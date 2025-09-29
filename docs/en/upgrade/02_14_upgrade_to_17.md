---
title: "Data migration from GitLab 14.0.12 to 17.8.z"
---

:::tip Problem to Solve
Versions 3.16 and 3.18 support GitLab versions that lag behind the official versions. Upgrading GitLab to the latest official version requires more than 10 upgrades to complete. After upgrading to 4.0, the product does not provide automatic upgrades to 17.8.z. This solution addresses how to migrate data from 14.0.12 to 17.8.z.

**Considering the large version gap, we adopt a data migration approach for the upgrade.**
:::

## Terminology

| Term                        | Explanation                                                                                                              |
|-----------------------------|--------------------------------------------------------------------------------------------------------------------------|
| `all-in-one` GitLab         | All GitLab components are packaged and deployed together in a single container or Pod. Used for rapid version upgrades. |
| `platform-deployed` GitLab  | GitLab is deployed and managed by the platform using the Operator, with components separated.                           |



## Process Overview

### Data Migration Path

According to the [official upgrade path](https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/?current=14.0.12&target=17.8.1&distro=docker&edition=ce) documentation, the data migration path is as follows:

- 14.0.12 -> 14.3.6
- 14.3.6 -> 14.9.5
- 14.9.5 -> 14.10.5
- 14.10.5 -> 15.0.5
- 15.0.5 -> 15.4.6
- 15.4.6 -> 15.11.13
- 15.11.13 -> 16.3.8
- 16.3.8 -> 16.7.9
- 16.7.9 -> 16.11.10
- 16.11.10 -> 17.3.6
- 17.3.6 → 17.5.5
- 17.5.5 → 17.8.1

Finally, back up the data from 17.8.1 and import it into the `platform-deployed` 17.8.z instance to complete the data migration.

### Data Migration Process

Back up the `platform-deployed` GitLab and restore it to an `all-in-one` image deployed GitLab, then upgrade to 17.8.1 using the `all-in-one` image, and finally restore the data backup to the `platform-deployed` 17.8.z instance.

1. Back up the `platform-deployed` GitLab 14.0.12 instance.
2. Restore the backup to an `all-in-one` deployment of GitLab 14.0.12.
3. Perform a rolling upgrade of the `all-in-one` deployment to GitLab 17.8.1.
4. Back up the upgraded `all-in-one` GitLab 17.8.1 instance.
5. Restore the backup from the `all-in-one` 17.8.1 to a `platform-deployed` GitLab 17.8.z instance.

:::tip
For the latest GitLab instance and operator versions, please refer to the [Release Note](../overview/release_notes.mdx).
:::

:::tip Migration Duration
The migration process involves database and repository backup/restore operations:

- Larger databases increase migration time.
- A higher number of repositories or larger single repositories significantly extends the duration.
- Storage performance also impacts efficiency — using topolvm is recommended for better performance.

Test setup:

- One large repository (~600 MB), others are small initial repositories
- GitLab instance: 6,700 issues, 3,600 merge requests

|Migration Step	| Key Factor	| 1,000 Repos	 |10,000 Repos|
|----|---|---|---|
|Back up platform-deployed GitLab 14.0.12	| Repo size & count	| 10 min	| 30 min|
|Restore to all-in-one GitLab 14.0.12	|Repo size & count	| 20 min	| 1 hr|
|Rolling upgrade to GitLab 17.8.1	| Database size	| 1 hr 30 min	| 1 hr 40 min|
|Back up all-in-one GitLab 17.8.1	| Repo size & count	| 3 min	| 10 min|
|Restore to platform-deployed GitLab | Repo size & count	| 30 min	| 1 hr|

:::


## Prerequisites

1. Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-on-linux) (>=1.30.0) and [yq](https://mikefarah.gitbook.io/yq#install) (>=4.45.0) on the execution host.
    ```bash
    kubectl version --client
    yq --version

    # Output:
    # Client Version: v1.30.0
    # Kustomize Version: v5.4.2
    # yq (https://github.com/mikefarah/yq/) version v4.45.4
    ```

2. Configure environment variables:

    ```bash
    # Replace with the name of the old GitLab CR
    export GITLAB_NAME=<old gitlab name>
    export GITLAB_NAMESPACE=<gitlab deploy namespace>
    ```

3. Prepare the PVCs required for the upgrade. These must be created in the **same namespace** as the old GitLab instance:

    :::tip
    A simple way to estimate the required storage size for the original instance is to sum the PVC sizes used by the database and the Gitaly components.
    :::

  - `backup-pvc`: Used to store backup tarballs. The recommended capacity is twice the size of the original instance.
  - `upgrade-pvc`: Used to store GitLab data during the rolling upgrade process. The recommended capacity is 1.2 times the size of the original instance.

    ```bash
    kubectl apply -f - <<EOF
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: backup-pvc
      namespace: ${GITLAB_NAMESPACE}
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 40Gi # Reassess the size based on actual usage
      storageClassName: ceph # Replace with the storage class name in your cluster
      volumeMode: Filesystem
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: upgrade-pvc
      namespace: ${GITLAB_NAMESPACE}
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 20Gi # Reassess the size based on actual usage
      storageClassName: ceph # Replace with the storage class name in your cluster
      volumeMode: Filesystem
    EOF

    # Output:
    # persistentvolumeclaim/backup-pvc created
    # persistentvolumeclaim/upgrade-pvc created
    ```

4. Prepare the images needed for the upgrade. Download each version of the image from the attachments and push them to an image repository that the GitLab deployment cluster can pull from. Image download links:

    ```shell
    # amd images
    https://cloud.alauda.cn/attachments/knowledge/242090509/gitlab-ce-14-0-12-amd.tgz
    https://cloud.alauda.cn/attachments/knowledge/242090509/gitlab-ce-14-3-6-amd.tgz
    https://cloud.alauda.cn/attachments/knowledge/242090509/gitlab-ce-14-9-5-amd.tgz
    https://cloud.alauda.cn/attachments/knowledge/242090509/gitlab-ce-14-10-5-amd.tgz
    https://cloud.alauda.cn/attachments/knowledge/242090509/gitlab-ce-15-0-5-amd.tgz
    https://cloud.alauda.cn/attachments/knowledge/242090509/gitlab-ce-15-4-6-amd.tgz
    https://cloud.alauda.cn/attachments/knowledge/242090509/gitlab-ce-15-11-13-amd.tgz
    https://cloud.alauda.cn/attachments/knowledge/242090509/gitlab-ce-16-3-8-amd.tgz
    https://cloud.alauda.cn/attachments/knowledge/242090509/gitlab-ce-16-7-9-amd.tgz
    https://cloud.alauda.cn/attachments/knowledge/242090509/gitlab-ce-16-11-10-amd.tgz
    https://cloud.alauda.cn/attachments/knowledge/242090509/gitlab-ce-17-3-6-amd.tgz
    https://cloud.alauda.cn/attachments/knowledge/242090509/gitlab-ce-17-5-5-amd.tgz
    https://cloud.alauda.cn/attachments/knowledge/242090509/gitlab-ce-17-8-1-amd.tgz
    ```

5. Prepare the scripts needed for the upgrade.

   Execute the following command to generate four scripts:

  - `check_gitlab.sh`: Check if the GitLab pod is ready.
  - `finalize_migrations.sql`: Finalize the migrations.
  - `monitor_gitlab.sh`: Monitor the GitLab pod to check if data migration is complete.
  - `rolling_upgrade.sh`: Rolling upgrade the GitLab instances one by one.

    ```bash
    cat << 'EOF' > check_gitlab.sh
    #!/bin/bash

    gitlab_pod=$1
    port=${2:-30855}

    log() {
        echo "$(date '+%Y-%m-%d %H:%M:%S') - $1"
    }

    log "Starting monitoring script for GitLab pod: ${gitlab_pod} on port: ${port}"

    while true; do
      curl_result=$(kubectl -n $GITLAB_NAMESPACE exec ${gitlab_pod} -- curl -s localhost:${port} | grep "You are being")
      if [[ -z "${curl_result}" ]]; then
          log "HTTP not ready. Retrying in 10 seconds..."
          sleep 10
          continue
      fi

      log "GitLab is ready"
      break
    done
    EOF

    cat << 'EOF' > finalize_migrations.sql
    select concat(
      'gitlab-rake gitlab:background_migrations:finalize[',
      job_class_name, ',',
      table_name, ',',
      column_name, $$,'$$,
      REPLACE(cast(job_arguments as text), ',', $$\,$$),
      $$']$$
    )
    from batched_background_migrations WHERE status NOT IN(3, 6);
    EOF

    cat << 'EOF' > monitor_gitlab.sh
    #!/bin/bash

    gitlab_pod=$1
    port=${2:-30855}

    log() {
        echo "$(date '+%Y-%m-%d %H:%M:%S') - $1"
    }

    log "Starting monitoring script for GitLab pod: ${gitlab_pod} on port: ${port}"

    while true; do
      curl_result=$(kubectl -n $GITLAB_NAMESPACE exec ${gitlab_pod} -- curl -s localhost:${port} | grep "You are being")
      if [[ -z "${curl_result}" ]]; then
          log "HTTP check not ready. Retrying in 10 seconds..."
          sleep 10
          continue
      fi

      migration_result=$(kubectl -n $GITLAB_NAMESPACE exec ${gitlab_pod} -- gitlab-psql -c "SELECT job_class_name, table_name, column_name, job_arguments FROM batched_background_migrations WHERE status NOT IN(3, 6);" | grep "(0 rows)")

      if [[ -z "${migration_result}" ]]; then
          log "Database migration is running. Retrying in 10 seconds..."
          sleep 10
          continue
      fi

      log "GitLab is ready, all migrations are done"
      break
    done
    EOF

    cat << 'EOF' > rolling_upgrade.sh
    #!/bin/bash

    image=$(kubectl -n $GITLAB_NAMESPACE get deployment gitlab -o jsonpath='{.spec.template.spec.containers[?(@.name=="gitlab")].image}')

    versions=(
      14.0.12-ce.0
      14.3.6-ce.0
      14.9.5-ce.0
      14.10.5-ce.0
      15.0.5-ce.0
      15.4.6-ce.0
      15.11.13-ce.0
      16.3.8-ce.0
      16.7.9-ce.0
      16.11.10-ce.0
      17.3.6-ce.0
      17.5.5-ce.0
      17.8.1-ce.0
    )

    for version in "${versions[@]}"; do
      echo "Upgrading to ${version}..."
      new_image="${image%:*}:${version}"
      kubectl -n $GITLAB_NAMESPACE set image deployment/gitlab gitlab=$new_image

      echo "Waiting for the GitLab pod to start..."
      sleep 10
      kubectl -n $GITLAB_NAMESPACE wait deploy gitlab --for condition=available --timeout=3000s
      sleep 10
      kubectl -n $GITLAB_NAMESPACE wait pod -l deploy=gitlab --for=condition=Ready --timeout=3000s

      new_pod_name=$(kubectl -n $GITLAB_NAMESPACE get po -l deploy=gitlab --sort-by=.metadata.creationTimestamp --field-selector=status.phase=Running -o jsonpath='{.items[-1].metadata.name}')
      echo "Waiting for GitLab to be ready..."
      bash check_gitlab.sh $new_pod_name

      echo "Waiting for migrations to finish..."
      kubectl -n $GITLAB_NAMESPACE cp finalize_migrations.sql $new_pod_name:/tmp/finalize_migrations.sql
      for i in {1..3}; do
        echo "Running migration tasks (attempt $i/3)..."
        if kubectl -n $GITLAB_NAMESPACE exec -ti $new_pod_name -- bash -c "gitlab-psql -t -A -f /tmp/finalize_migrations.sql > /tmp/run_migration_tasks.sh && xargs -d \"\n\" -P 3 -I {} bash -c \"{}\" < /tmp/run_migration_tasks.sh"; then
          echo "Migration tasks completed successfully"
          break
        fi
        sleep 20
      done
      bash monitor_gitlab.sh $new_pod_name
      echo "Upgraded to ${version} successfully"
    done
    EOF
    ```

## Backup the `platform-deployed` GitLab 14.0.12

1. Enable the task-runner component on GitLab 14 to execute backup commands

    ```bash
    kubectl patch gitlabofficials.operator.devops.alauda.io ${GITLAB_NAME} -n $GITLAB_NAMESPACE --type='merge' -p='
    {
      "spec": {
        "helmValues": {
          "gitlab": {
            "task-runner": {
              "enabled": "true",
              "resources": { "limits": { "cpu": "2", "memory": "4G" } }
            }
          }
        }
      }
    }'
    # Output:
    # gitlabofficial.operator.devops.alauda.io/xxx patched

    # Wait for the task-runner deployment to be available
    echo "Waiting for the task-runner deployment to be available..."
    sleep 60

    # Execute the command and confirm that the ${GITLAB_NAME}-task-runner deployment exists
    kubectl wait deploy ${GITLAB_NAME}-task-runner -n $GITLAB_NAMESPACE --for condition=available --timeout=3000s
    kubectl get deploy -n $GITLAB_NAMESPACE ${GITLAB_NAME}-task-runner

    # Output:
    # NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
    #high-sc-sso-ingress-https-task-runner   1/1     1            1           178m
    ```

2. Set GitLab 14 to read-only mode

    ```bash
    kubectl patch deploy ${GITLAB_NAME}-webservice-default -n $GITLAB_NAMESPACE --type='merge' -p='{"metadata":{"annotations":{"skip-sync":"true"}}}'
    kubectl patch deploy ${GITLAB_NAME}-sidekiq-all-in-1-v1 -n $GITLAB_NAMESPACE --type='merge' -p='{"metadata":{"annotations":{"skip-sync":"true"}}}'
    kubectl scale deploy ${GITLAB_NAME}-webservice-default -n $GITLAB_NAMESPACE --replicas 0
    kubectl scale deploy ${GITLAB_NAME}-sidekiq-all-in-1-v1 -n $GITLAB_NAMESPACE --replicas 0

    # Output:
    # deployment.apps/gitlab-old-webservice-default patched
    # deployment.apps/gitlab-old-sidekiq-all-in-1-v1 patched
    # deployment.apps/gitlab-old-webservice-default scaled
    # deployment.apps/gitlab-old-sidekiq-all-in-1-v1 scaled
    ```

3. Patch the backup PVC for the GitLab 14 task-runner. The PVC should be named `backup-pvc`.

    ```bash
    kubectl patch deploy ${GITLAB_NAME}-task-runner -n $GITLAB_NAMESPACE --type='json' -p='
    [
      {
        "op": "add",
        "path": "/metadata/annotations/skip-sync",
        "value": "true"
      },
      {
        "op": "replace",
        "path": "/spec/template/spec/volumes/1",
        "value": {"name": "task-runner-tmp", "persistentVolumeClaim": {"claimName": "backup-pvc"}}
      }
    ]'

    # Output:
    # deployment.apps/gitlab-old-task-runner patched

    echo "Waiting for the task-runner deployment to be available..."
    sleep 60

    # Execute the command and confirm that the ${GITLAB_NAME}-task-runner deployment exists
    kubectl wait deploy ${GITLAB_NAME}-task-runner -n $GITLAB_NAMESPACE --for condition=available --timeout=3000s
    kubectl get deploy -n $GITLAB_NAMESPACE ${GITLAB_NAME}-task-runner
    ```

4. Back up GitLab 14 data

    ```bash
    pod_name=$(kubectl get po -n $GITLAB_NAMESPACE -l app=task-runner,release=${GITLAB_NAME} -o jsonpath='{.items[0].metadata.name}')
    kubectl exec -ti -n "$GITLAB_NAMESPACE" $pod_name -- gitlab-rake gitlab:backup:create

    # Output:
    # 2024-11-01 06:16:06 +0000 -- Dumping database ...
    # Dumping PostgreSQL database gitlabhq_production ... [DONE]
    # 2024-11-01 06:16:18 +0000 -- done
    # 2024-11-01 06:16:18 +0000 -- Dumping repositories ...
    # ....
    # done
    # Deleting old backups ... skipping
    # Warning: Your gitlab.rb and gitlab-secrets.json files contain sensitive data
    # and are not included in this backup. You will need these files to restore a backup.
    # Please back them up manually.
    # Backup task is done.

    # Add permissions to the backup file.
    kubectl exec -ti -n "$GITLAB_NAMESPACE" $pod_name -- bash -c '
    chmod 777 /srv/gitlab/tmp/backups/*_gitlab_backup.tar
    ls -lh /srv/gitlab/tmp/backups
    '

    # Output:
    # total 300K
    # -rwxrwxrwx 1 git git 300K Sep 15 09:15 1757927758_2025_09_15_14.0.12_gitlab_backup.tar
    ```

    Back up the rails-secret by executing the following command, which will save the rails-secret to a file named `gitlab14-rails-secret.yaml` in the current directory.

    ```bash
    kubectl get secrets -n ${GITLAB_NAMESPACE} ${GITLAB_NAME}-rails-secret -o jsonpath="{.data['secrets\.yml']}" | base64 --decode | yq -o yaml > gitlab14-rails-secret.yaml
    ```

5. Remove the task-runner component after backup is complete.

    ```bash
    kubectl patch gitlabofficials.operator.devops.alauda.io ${GITLAB_NAME} -n $GITLAB_NAMESPACE --type='merge' -p='
    {
      "spec": {
        "helmValues": {
          "gitlab": {
            "task-runner": null
          }
        }
      }
    }'

    echo "Waiting for the task-runner deployment to be removed..."
    sleep 30

    kubectl get po -n "$GITLAB_NAMESPACE" -l app=task-runner,release=${GITLAB_NAME}

    # Output:
    # No resources found in xxxx namespace.
    ```

## Restore backup data to `all-in-one` deployment of 14.0.12

1. Use the `all-in-one` image to deploy GitLab 14.0.12.

    First, set the `NODE_IP` environment variable to access the `all-in-one` GitLab instance through this IP and NodePort port.

    ```bash
    export NODE_IP=<node_ip>
    ```

    Create the `all-in-one` GitLab instance.

    ```bash
    export GITLAB_IMAGE=<registry-host>/gitlab/gitlab-ce:14.0.12-ce.0

    kubectl apply -f - <<EOF
    apiVersion: v1
    kind: Service
    metadata:
      name: gitlab-http-nodeport
      namespace: ${GITLAB_NAMESPACE}
    spec:
      ports:
        - appProtocol: tcp
          name: web
          port: 30855
          protocol: TCP
          targetPort: 30855
          nodePort: 30855
      selector:
        deploy: gitlab
      type: NodePort
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: gitlab
      namespace: ${GITLAB_NAMESPACE}
    spec:
      progressDeadlineSeconds: 600
      replicas: 1
      revisionHistoryLimit: 10
      selector:
        matchLabels:
          deploy: gitlab
      strategy:
        rollingUpdate:
          maxSurge: 1
          maxUnavailable: 1
        type: RollingUpdate
      template:
        metadata:
          labels:
            deploy: gitlab
        spec:
          affinity: {}
          containers:
            - env:
                # External access address of the all-in-one gitlab, need to replace with the IP and nodeport port of a cluster node
                - name: GITLAB_OMNIBUS_CONFIG
                  value: external_url 'http://${NODE_IP}:30855'
              image: ${GITLAB_IMAGE} # Can be replaced with an image that can be pulled
              imagePullPolicy: IfNotPresent
              name: gitlab
              ports:
                - containerPort: 30855
                  name: http
                  protocol: TCP
                - containerPort: 2424
                  name: ssh
                  protocol: TCP
              resources:
                limits:
                  cpu: "8"
                  memory: 8Gi
                requests:
                  cpu: 4
                  memory: "4Gi"
              securityContext:
                privileged: true
              terminationMessagePath: /dev/termination-log
              terminationMessagePolicy: File
              volumeMounts:
                - mountPath: /var/opt/gitlab/backups
                  name: backup
                  subPath: backups
                - mountPath: /etc/gitlab
                  name: gitlab-upgrade-data
                  subPath: config
                - mountPath: /var/log/gitlab
                  name: gitlab-upgrade-data
                  subPath: logs
                - mountPath: /var/opt/gitlab
                  name: gitlab-upgrade-data
                  subPath: data
          dnsPolicy: ClusterFirst
          restartPolicy: Always
          schedulerName: default-scheduler
          securityContext: {}
          terminationGracePeriodSeconds: 30
          volumes:
            - name: backup
              persistentVolumeClaim:
                claimName: backup-pvc
            - name: gitlab-upgrade-data
              persistentVolumeClaim:
                claimName: upgrade-pvc
    EOF

    # Output:
    # service/gitlab-http-nodeport created
    # deployment.apps/gitlab created
    ```

    Check if the `all-in-one` GitLab instance started successfully.

    ```bash
    pod_name=$(kubectl get po -l deploy=gitlab -n $GITLAB_NAMESPACE -o jsonpath='{.items[0].metadata.name}')
    bash check_gitlab.sh $pod_name

    # Output:
    # 2024-11-01 16:02:47 - Starting monitoring script for GitLab pod: gitlab-7ff8d674bd-m65nz on port: 30855
    # 2024-11-01 16:12:59 - HTTP not ready. Retrying in 10 seconds...
    # 2024-11-01 16:13:09 - HTTP not ready. Retrying in 10 seconds...
    # 2024-11-01 16:13:19 - HTTP not ready. Retrying in 10 seconds...
    # 2024-11-01 16:13:29 - HTTP not ready. Retrying in 10 seconds...
    # 2024-11-01 16:13:40 - HTTP not ready. Retrying in 10 seconds...
    # 2024-11-01 16:13:50 - GitLab is ready
    ```

2. Modify the rails secret content of the `all-in-one` GitLab. Replace the `secret_key_base`, `otp_key_base`, `db_key_base`, `openid_connect_signing_key`, `ci_jwt_signing_key` fields with the corresponding fields from the backed up GitLab 14 rails secret.

    ```bash
    # Copy the gitlab-secrets.json to the local
    pod_name=$(kubectl get po -l deploy=gitlab -n $GITLAB_NAMESPACE -o jsonpath='{.items[0].metadata.name}')
    kubectl -n $GITLAB_NAMESPACE cp $pod_name:/etc/gitlab/gitlab-secrets.json all-in-one-gitlab-secrets.json

    # Replace with the data backup from the gitlab 14 rails-secret.
    KEYS=(secret_key_base otp_key_base db_key_base openid_connect_signing_key ci_jwt_signing_key)
    for key in "${KEYS[@]}"; do
      echo "Replace ${key} ..."
      export KEY_VALUE=$(yq eval -r --unwrapScalar=false ".production.${key}" gitlab14-rails-secret.yaml)
      yq eval ".gitlab_rails.${key} = env(KEY_VALUE)" all-in-one-gitlab-secrets.json -i
    done

    # Copy the gitlab-secrets.json to the pod
    kubectl -n $GITLAB_NAMESPACE cp all-in-one-gitlab-secrets.json $pod_name:/etc/gitlab/gitlab-secrets.json
    ```

    Disable unnecessary components.

    ```bash
    cat <<EOF >> gitlab.rb
    prometheus['enable'] = false
    gitlab_kas['enable'] = false
    redis_exporter['enable'] = false
    gitlab_exporter['enable'] = false
    postgres_exporter['enable'] = false
    sidekiq['enable'] = false
    EOF

    # Copy the gitlab.rb to the pod
    kubectl -n $GITLAB_NAMESPACE cp gitlab.rb $pod_name:/etc/gitlab/gitlab.rb
    ```

    Restart the pod and wait for it to start up.

    ```bash
    # Restart the pod
    kubectl delete po -l deploy=gitlab -n $GITLAB_NAMESPACE

    # Output:
    # pod "gitlab-xxxx" deleted

    echo "Waiting for the pod to start..."
    sleep 10

    pod_name=$(kubectl get po -l deploy=gitlab -n $GITLAB_NAMESPACE -o jsonpath='{.items[0].metadata.name}')
    bash check_gitlab.sh $pod_name

    # Output:
    # 2024-11-01 16:02:47 - Starting monitoring script for GitLab pod: gitlab-7ff8d674bd-m65nz on port: 30855
    # 2024-11-01 16:12:59 - HTTP not ready. Retrying in 10 seconds...
    # 2024-11-01 16:13:09 - HTTP not ready. Retrying in 10 seconds...
    # 2024-11-01 16:13:19 - HTTP not ready. Retrying in 10 seconds...
    # 2024-11-01 16:13:29 - HTTP not ready. Retrying in 10 seconds...
    # 2024-11-01 16:13:40 - HTTP not ready. Retrying in 10 seconds...
    # 2024-11-01 16:13:50 - GitLab is ready
    ```

3. Restore the backup data.

    :::tip
    During the recovery process, database-related errors may occur, for example:
    - `ERROR:  role "xxx" does not exist`
    - `ERROR:  function xxx does not exist`
    - `ERROR:  permission denied to create extension "pg_stat_statements"`
    - `ERROR:  could not open extension control file`

    These errors can be ignored, details can be found at (https://gitlab.com/gitlab-org/gitlab/-/issues/266988).
    :::

    ```bash
    # Set gitlab to read-only
    pod_name=$(kubectl get po -l deploy=gitlab -n $GITLAB_NAMESPACE -o jsonpath='{.items[0].metadata.name}')
    kubectl -n $GITLAB_NAMESPACE exec -ti $pod_name -- gitlab-ctl stop puma
    kubectl -n $GITLAB_NAMESPACE exec -ti $pod_name -- gitlab-ctl stop sidekiq

    # Output:
    # ok: down: puma: 666s, normally up
    # ok: down: sidekiq: 660s, normally up


    # Confirm that puma and sidekiq have been stopped
    kubectl -n $GITLAB_NAMESPACE exec -ti $pod_name -- gitlab-ctl status

    # Output:
    # ...
    # run: postgresql: (pid 436) 1137s; run: log: (pid 519) 1134s
    # run: prometheus: (pid 1142) 1043s; run: log: (pid 680) 1092s
    # down: puma: 695s, normally up; run: log: (pid 536) 1128s
    # down: sidekiq: 687s, normally up; run: log: (pid 551) 1124s
    # run: sshd: (pid 42) 1171s; run: log: (pid 41) 1171s
    # command terminated with exit code 6

    # Execute backup
    backup_id=$(kubectl -n $GITLAB_NAMESPACE exec -ti $pod_name -- bash -c 'ls /var/opt/gitlab/backups/*14.0.12_gitlab_backup.tar | head -n1 | sed "s|.*/\([0-9_]*_14\.0\.12\)_gitlab_backup\.tar|\1|"')
    echo "Backup ID: ${backup_id}"

    echo "This step will prompt you multiple times to confirm whether to continue. \nPlease select \"yes\" each time to proceed."
    kubectl -n $GITLAB_NAMESPACE exec -ti $pod_name -- bash -c 'gitlab-backup restore BACKUP=${backup_id}'

    # Output:
    # ...
    # This task will now rebuild the authorized_keys file.
    # You will lose any data stored in the authorized_keys file.
    # Do you want to continue (yes/no)? yes
    #
    # Warning: Your gitlab.rb and gitlab-secrets.json files contain sensitive data
    # and are not included in this backup. You will need to restore these files manually.
    # Restore task is done.
    ```

    Restart the pod and wait for it to start up, then execute `gitlab-rake gitlab:check SANITIZE=true` to check GitLab integrity.

    ```bash
    # Restart the pod
    kubectl delete po -l deploy=gitlab -n $GITLAB_NAMESPACE

    # Output:
    # pod "gitlab-xxxx" deleted

    # Waiting for the pod to start
    sleep 10
    pod_name=$(kubectl get po -l deploy=gitlab -n $GITLAB_NAMESPACE -o jsonpath='{.items[0].metadata.name}')
    bash check_gitlab.sh $pod_name

    # Output:
    # 2024-11-01 16:02:47 - Starting monitoring script for GitLab pod: gitlab-7ff8d674bd-m65nz on port: 30855
    # 2024-11-01 16:12:59 - HTTP not ready. Retrying in 10 seconds...
    # 2024-11-01 16:13:09 - HTTP not ready. Retrying in 10 seconds...
    # 2024-11-01 16:13:19 - HTTP not ready. Retrying in 10 seconds...
    # 2024-11-01 16:13:29 - HTTP not ready. Retrying in 10 seconds...
    # 2024-11-01 16:13:40 - HTTP not ready. Retrying in 10 seconds...
    # 2024-11-01 16:13:50 - GitLab is ready

    kubectl -n $GITLAB_NAMESPACE exec -ti $pod_name -- gitlab-rake gitlab:check SANITIZE=true

    # Output:
    # ...
    # GitLab configured to store new projects in hashed storage? ... yes
    # All projects are in hashed storage? ... yes
    # Checking GitLab App ... Finished
    # Checking GitLab subtasks ... Finished
    ```

    Log in to GitLab to check if the instance restoration was successful (root user password remains the same as the original instance), and execute the following command to get the access address of the `all-in-one` instance.

    ```bash
    kubectl -n $GITLAB_NAMESPACE get pod -l deploy=gitlab -o yaml | grep external_url

    # Output:
    #       value: external_url 'http://192.168.130.174:30855'
    ```

## Rolling upgrade `all-in-one` deployed gitlab to 17.8.1

Execute the `rolling_upgrade.sh` script, which will upgrade the all-in-one instance step by step to version 17.8.1 according to the upgrade path.

   ```bash
   bash -x rolling_upgrade.sh
   ```

After the script execution is complete, enter the GitLab Web UI to check if the instance version is 17.8.1, and verify that the data is complete, including code repositories, users, issues, merge requests, etc.

## Back up the data from 17.8.1 `all-in-one` instance

Stop the `all-in-one` instance service and back up the data. The backup files will be saved in the `/var/opt/gitlab/backups/` directory.

```bash
pod_name=$(kubectl get po -l deploy=gitlab -n $GITLAB_NAMESPACE -o jsonpath='{.items[0].metadata.name}')
kubectl -n $GITLAB_NAMESPACE exec -ti $pod_name -- bash -c '
gitlab-ctl stop puma
gitlab-ctl stop sidekiq
gitlab-ctl status
gitlab-rake gitlab:backup:create
'

# Output:
# ok: down: puma: 0s, normally up
# ok: down: sidekiq: 0s, normally up
# ...
# 2025-09-15 03:24:37 UTC -- Deleting backups/tmp ...
# 2025-09-15 03:24:37 UTC -- Deleting backups/tmp ... done
# 2025-09-15 03:24:37 UTC -- Warning: Your gitlab.rb and gitlab-secrets.json files contain sensitive data
# and are not included in this backup. You will need these files to restore a backup.
# Please back them up manually.
# 2025-09-15 03:24:37 UTC -- Backup 1757906670_2025_09_15_17.8.1 is done.
# 2025-09-15 03:24:37 UTC -- Deleting backup and restore PID file at [/opt/gitlab/embedded/service/gitlab-rails/tmp/backup_restore.pid] ... done
```

Stop the all-in-one instance service.

```bash
kubectl scale deploy gitlab -n $GITLAB_NAMESPACE --replicas 0

# Output:
# deployment.apps/gitlab scaled
```

## Restore `all-in-one` backup data to `platform-deployed` 17.8.z

:::warning Namespace of the new gitlab instance
The namespace of the new gitlab instance must be the same as the namespace of the old gitlab instance.
:::

1. Refer to the [GitLab Deployment Guide](../install/03_gitlab_deploy.mdx) to use the operator for deploying a new GitLab 17.8.z instance.

    ::: tip
     1. The newly deployed instance must be in the same namespace as the old instance
     2. The access method of the newly deployed instance must not conflict with the old instance, such as domain name, nodeport, etc.
    :::

    After the new instance is deployed, set the environment variable for the new instance name:
    ```bash
    export NEW_GITLAB_NAME=<name of the new GitLab instance>
    ```

2. Enable the toolbox component for the new GitLab instance.

    ```yaml
    kubectl patch gitlabofficial.operator.alaudadevops.io ${NEW_GITLAB_NAME} -n $GITLAB_NAMESPACE --type='merge' -p='
    {
      "spec": {
        "helmValues": {
          "gitlab": {
            "toolbox": {
              "enabled": true,
              "resources": {
                "limits": {
                  "cpu": 2,
                  "memory": "4G"
                }
              }
            }
          }
        }
      }
    }'
    ```

    Wait for the instance to complete redeployment.

    ```bash
    kubectl get po -l app=toolbox,release=${NEW_GITLAB_NAME} -n $GITLAB_NAMESPACE

    # Output:
    # NAME                                    READY   STATUS    RESTARTS   AGE
    # xx-gitlab-toolbox-xxxx-xxxx-xxxx-xxxx   1/1     Running   0          178m
    ```

3. Restore backup data to the GitLab 17.8.z deployed by the operator

    :::tip
    During the recovery process, database-related errors may occur, for example:
    - `ERROR:  role "xxx" does not exist`
    - `ERROR:  function xxx does not exist`
    - `ERROR:  must be owner of extension`

    These errors can be ignored, details can be found at (https://gitlab.com/gitlab-org/gitlab/-/issues/266988).
    :::

    Close the external access of the new 17.8.z gitlab instance, and patch the toolbox component to mount the backup pvc.

    ```bash
    kubectl patch deploy ${NEW_GITLAB_NAME}-webservice-default -n $GITLAB_NAMESPACE --type='merge' -p='{"metadata":{"annotations":{"skip-sync":"true"}}}'
    kubectl patch deploy ${NEW_GITLAB_NAME}-sidekiq-all-in-1-v2 -n $GITLAB_NAMESPACE --type='merge' -p='{"metadata":{"annotations":{"skip-sync":"true"}}}'
    kubectl scale deploy ${NEW_GITLAB_NAME}-webservice-default -n $GITLAB_NAMESPACE --replicas 0
    kubectl scale deploy ${NEW_GITLAB_NAME}-sidekiq-all-in-1-v2 -n $GITLAB_NAMESPACE --replicas 0

    # Modify the toolbox, mount the backup pvc to the toolbox
    kubectl patch deploy ${NEW_GITLAB_NAME}-toolbox -n $GITLAB_NAMESPACE -p='{"metadata":{"annotations":{"skip-sync":"true"}}}'
    kubectl patch deploy ${NEW_GITLAB_NAME}-toolbox -n $GITLAB_NAMESPACE --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/volumes/1", "value": {"name": "toolbox-tmp", "persistentVolumeClaim": {"claimName": "backup-pvc"}}}]'

    # Output:
    # deployment.apps/xxxx-xxxx-toolbox patched
    ```

    Wait for the toolbox pod to be ready, and then restore the backup data to the new GitLab instance.

    ```bash
    toolbox_pod_name=$(kubectl get po -n $GITLAB_NAMESPACE -l release=$NEW_GITLAB_NAME,app=toolbox -o jsonpath='{.items[0].metadata.name}')

    # backup the old backup file
    kubectl -n $GITLAB_NAMESPACE exec -ti $toolbox_pod_name -- bash -c '
    mkdir -p /srv/gitlab/tmp/backup_tarballs
    mv /srv/gitlab/tmp/backups/*gitlab_backup.tar /srv/gitlab/tmp/backup_tarballs || true
    '

    backup_file=$(kubectl -n $GITLAB_NAMESPACE exec -ti $toolbox_pod_name -- bash -c "ls /srv/gitlab/tmp/backup_tarballs/*17.8.1_gitlab_backup.tar | head -n1 | tr -d '\r\n'")
    echo "Backup file of 17.8.1: ${backup_file}"

    # Output:
    # Backup file of 17.8.1: 1757906670_2025_09_15_17.8.1_gitlab_backup.tar

    kubectl -n $GITLAB_NAMESPACE exec -ti $toolbox_pod_name -- backup-utility --restore --skip-restore-prompt -f "file://${backup_file}"

    # Output:
    # 2025-09-15 06:59:21 UTC -- Restoring repositories ... done
    # 2025-09-15 06:59:21 UTC -- Deleting backup and restore PID file at [/srv/gitlab/tmp/backup_restore.pid] ... done
    # 2025-09-15 06:59:41 UTC -- Restoring builds ...
    # 2025-09-15 06:59:41 UTC -- Restoring builds ... done
    # 2025-09-15 06:59:41 UTC -- Deleting backup and restore PID file at [/srv/gitlab/tmp/backup_restore.pid] ... done
    # Backup tarball not from a Helm chart based installation. Not processing files in object storage.
    ```

4. Replace the rails-secret for the new GitLab instance.

    ```bash
    kubectl get secrets -n ${GITLAB_NAMESPACE} ${NEW_GITLAB_NAME}-rails-secret -o jsonpath="{.data['secrets\.yml']}" | base64 --decode | yq -o yaml > gitlab17-rails-secret.yaml
    yq eval-all 'select(fileIndex == 0) * select(fileIndex == 1)' gitlab17-rails-secret.yaml gitlab14-rails-secret.yaml > final-rails-secret.yml
    kubectl create secret generic rails-secret -n ${GITLAB_NAMESPACE} --from-file=secrets.yml=final-rails-secret.yml

    # Output:
    # secret/rails-secret created

    kubectl patch gitlabofficials.operator.alaudadevops.io ${NEW_GITLAB_NAME} -n $GITLAB_NAMESPACE --type='merge' -p='
    {
      "spec": {
        "helmValues": {
          "global": {
            "railsSecrets": {
              "secret": "rails-secret"
            }
          }
        }
      }
    }'

    # Output:
    # gitlabofficial.operator.alaudadevops.io/xxxx-xxxx patched

    kubectl patch deploy ${NEW_GITLAB_NAME}-webservice-default -n $GITLAB_NAMESPACE --type='json' -p='[{"op":"remove","path":"/metadata/annotations/skip-sync"}]'
    kubectl patch deploy ${NEW_GITLAB_NAME}-sidekiq-all-in-1-v2 -n $GITLAB_NAMESPACE --type='json' -p='[{"op":"remove","path":"/metadata/annotations/skip-sync"}]'
    kubectl patch deploy ${NEW_GITLAB_NAME}-toolbox -n $GITLAB_NAMESPACE --type='json' -p='[{"op":"remove","path":"/metadata/annotations/skip-sync"}]'

    kubectl delete pod -lrelease=fm-test-gitlab -n $GITLAB_NAMESPACE
    kubectl scale deploy ${NEW_GITLAB_NAME}-webservice-default -n $GITLAB_NAMESPACE --replicas 1
    kubectl scale deploy ${NEW_GITLAB_NAME}-sidekiq-all-in-1-v2 -n $GITLAB_NAMESPACE --replicas 1
    ```

    The operator will automatically redeploy the new GitLab instance. Wait for the instance to complete redeployment.

5. Verify that the data after the upgrade is normal.

   - Enter the GitLab UI in administrator view and check whether the repositories and user data are normal.
   - Select some repositories and verify whether the code, branches, and merge requests are functioning properly.

6. Clean up unused resources:

   - Delete the old GitLab instance and the old operator if necessary.
   - Delete the `all-in-one` GitLab instance if necessary.

## FAQ

1. The migrated data does not include avatars and attachments, so the new GitLab instance cannot display this content. To resolve this issue, you can replace the upload PVC of GitLab 17.8.z with the PVC used by 14.0.12.

    ```bash
    export UPLOADS_PVC_NAME=<name of the uploads pvc>

    kubectl patch gitlabofficials.operator.alaudadevops.io ${NEW_GITLAB_NAME} -n $GITLAB_NAMESPACE --type='json' -p='
    {
      "spec": {
        "helmValues": {
          "global": {
            "uploads": {
              "persistence": {
                "enabled": true,
                "existingClaim": "${UPLOADS_PVC_NAME}"
              }
            }
          }
        }
      }
    }'
    ```

   If your old GitLab instance uses HostPath storage, you must manually copy the data from the old instance's uploads directory to the `/srv/gitlab/public/uploads` path of the webservice component in the new instance.