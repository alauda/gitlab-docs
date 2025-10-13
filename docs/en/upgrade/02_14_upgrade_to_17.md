---
title: "Data migration from GitLab 14.0.12 to 17.8.z"
---

:::tip Problem to Solve
Versions 3.16 and 3.18 support GitLab versions that lag behind the official versions. Upgrading GitLab to the latest official version requires more than 10 upgrades to complete. After upgrading to 4.0, the product does not provide automatic upgrades to 17.8.z. This solution addresses how to upgrade from 14.0.12 to 17.8.z.

**Considering the large version gap, we adopt a data migration approach for the upgrade.**
:::

## Process Overview

:::warning
The time required for the upgrade varies significantly depending on the size of the GitLab data. It may take several days to complete the upgrade, **so it's necessary to evaluate the maintenance window in advance**.

Test data: Includes 3 projects (one large project of 666MB, 2 empty projects), backup file size of 668MB, upgrade time of 8 hours.
:::

Data migration Path
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

## Data migration Process

Backup the `platform-deployed` GitLab and restore it to an `all-in-one` image deployed GitLab, then upgrade to 17.8.1 using the `all-in-one` image, and finally restore the data backup to the `platform-deployed` 17.8.z instance.

1. Backup the `platform-deployed` GitLab 14.0.12
2. Restore the backup data to the `all-in-one` deployed 14.0.12
3. Upgrade the `all-in-one` deployed GitLab to 17.8.1
4. Backup the `all-in-one` deployed 17.8.1
5. Restore the `all-in-one` backup data to the `platform-deployed` 17.8.z

:::tip
For the latest GitLab instance and operator versions, please refer to the [Release Note](../overview/release_notes.mdx).
:::

## Operation Steps

### Prerequisites

1. Install [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-on-linux), [yq](https://mikefarah.gitbook.io/yq#install) on the execution host.
2. Configure environment variables (Note: at this point, the gitlab-operator version has not been upgraded and corresponds to the operator version for GitLab 14.0.12)

    ```bash
    export GITLAB_NAME=<old gitlab name>
    export GITLAB_NAMESPACE=<gitlab deploy namespace>
    ```

3. Prepare PVCs for the upgrade, which need to be created in the **same namespace** of the old GitLab instance

    ```yaml
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: backup-pvc
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 20Gi # Reassess the size based on actual usage
      storageClassName: nfs # Replace with the storage class name in your cluster
      volumeMode: Filesystem
    ---
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: upgrade-pvc
    spec:
      accessModes:
        - ReadWriteMany
      resources:
        requests:
          storage: 20Gi # Reassess the size based on actual usage
      storageClassName: nfs # Replace with the storage class name in your cluster
      volumeMode: Filesystem
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

### Data migration Steps

#### Backup the `platform-deployed` GitLab 14.0.12

:::warning
During the backup process, the task-runner container may be killed due to insufficient resources. To avoid this, configure resource requests and limits for the task-runner container (at least 2 CPU and 4 GiB memory).
:::

1. Enable the task-runner on GitLab 14 to execute backup commands

    ```shell
    $ kubectl patch gitlabofficials.operator.devops.alauda.io ${GITLAB_NAME} -n $GITLAB_NAMESPACE --type='merge' -p='{"spec":{"helmValues":{"gitlab":{"task-runner":{"enabled":"true","resources":{"limits":{"cpu":"2","memory":"4G"}}}}}}}'
    # Execute the command and confirm that the ${GITLAB_NAME}-task-runner deployment exists
    $ kubectl get deploy -n $GITLAB_NAMESPACE ${GITLAB_NAME}-task-runner
    NAME                                    READY   UP-TO-DATE   AVAILABLE   AGE
    high-sc-sso-ingress-https-task-runner   1/1     1            1           178m
    ```

2. Set the gitlab 14 to read-only mode

    ```bash
    $ kubectl patch deploy ${GITLAB_NAME}-webservice-default -n $GITLAB_NAMESPACE --type='merge' -p='{"metadata":{"annotations":{"skip-sync":"true"}}}'
    $ kubectl patch deploy ${GITLAB_NAME}-sidekiq-all-in-1-v1 -n $GITLAB_NAMESPACE --type='merge' -p='{"metadata":{"annotations":{"skip-sync":"true"}}}'
    $ kubectl scale deploy ${GITLAB_NAME}-webservice-default -n $GITLAB_NAMESPACE --replicas 0
    $ kubectl scale deploy ${GITLAB_NAME}-sidekiq-all-in-1-v1 -n $GITLAB_NAMESPACE --replicas 0
    ```

3. Set the backup PVC for gitlab 14 task-runner, the PVC name is backup-pvc (the PVC needs to be created in advance, the size needs to be evaluated based on the gitlab data volume)

    ```bash
    $ kubectl patch deploy ${GITLAB_NAME}-task-runner -n $GITLAB_NAMESPACE -p='{"metadata":{"annotations":{"skip-sync":"true"}}}'
    $ kubectl patch deploy ${GITLAB_NAME}-task-runner -n $GITLAB_NAMESPACE --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/volumes/1", "value": {"name": "task-runner-tmp", "persistentVolumeClaim": {"claimName": "backup-pvc"}}}]'
    ```

4. Backup gitlab 14 data

    ```bash
    # Get the task-runner name, ensure the pod status is running
    $ kubectl get po -n $GITLAB_NAMESPACE -l app=task-runner,release=${GITLAB_NAME}
    gitlab-task-runner-b4459444b-bklh4   1/1     Running   0          84m

    $ kubectl exec -ti -n $GITLAB_NAMESPACE gitlab-task-runner-b4459444b-bklh4 -- bash

    $ gitlab-rake gitlab:backup:create
    2024-11-01 06:16:06 +0000 -- Dumping database ...
    Dumping PostgreSQL database gitlabhq_production ... [DONE]
    2024-11-01 06:16:18 +0000 -- done
    2024-11-01 06:16:18 +0000 -- Dumping repositories ...
    ....
    done
    Deleting old backups ... skipping
    Warning: Your gitlab.rb and gitlab-secrets.json files contain sensitive data
    and are not included in this backup. You will need these files to restore a backup.
    Please back them up manually.
    Backup task is done.

    # Add permissions to the backup file, use the user of the backup pod, which may be different from the backup pod user.
    $ ls /srv/gitlab/tmp/backups
    17302758xx_xxxx_xx_xx_14.0.12_gitlab_backup.tar

    $ chmod 777 /srv/gitlab/tmp/backups/17302758xx_xxxx_xx_xx_14.0.12_gitlab_backup.tar
    $ exit

    # Backup rails-secret
    kubectl get secrets -n ${GITLAB_NAMESPACE} ${GITLAB_NAME}-rails-secret -o jsonpath="{.data['secrets\.yml']}" | base64 --decode | yq -o json > gitlab14-rails-secret.yaml
    ```

#### Restore backup data to `all-in-one` deployment of 14.0.12

1.  Use the `all-in-one` image to deploy 14.0.12 gitlab.

    Apply the following yaml to the gitlab deployment namespace, the yaml has already mounted the backup pvc (backup-pvc) to the backup directory (if you need to access the gitlab instance during the upgrade process, you need to replace the IP and nodeport port in the yaml).

    ```yaml
    ---
    apiVersion: v1
    kind: Service
    metadata:
      name: gitlab-http-nodeport
    spec:
      externalTrafficPolicy: Cluster
      internalTrafficPolicy: Cluster
      ports:
        - appProtocol: tcp
          name: web
          port: 30855
          protocol: TCP
          targetPort: 30855
          nodePort: 30855
      selector:
        deploy: gitlab
      sessionAffinity: None
      type: NodePort
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: gitlab
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
                - name: GITLAB_OMNIBUS_CONFIG
                  value: external_url 'http://192.168.132.128:30855' # External access address of the `all-in-one gitlab`, need to replace with the IP and nodeport port of a cluster node
              image: gitlab/gitlab-ce:14.0.12-ce.0 # Can be replaced with an image that can be pulled
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
                  cpu: "4"
                  memory: 8Gi
                requests:
                  cpu: 2
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
    ```


2. Restore the backup data to the 14.0.12 gitlab deployed with `all-in-one` image.

    :::tip
    During the recovery process, database-related errors may occur, for example:
    - `ERROR:  role "xxx" does not exist`
    - `ERROR:  function xxx does not exist`

    These errors can be ignored, details can be found at (https://gitlab.com/gitlab-org/gitlab/-/issues/266988).
    :::

    ```bash
    # Use deployment to start 14.0.12 gitlab
    # Enter the deployment deployed gitlab
    # Get the gitlab 14's gitlab-rails file.
    $ kubectl get secrets -n ${GITLAB_NAMESPACE} ${GITLAB_NAME}-rails-secret -o jsonpath="{.data['secrets\.yml']}" | base64 --decode | yq -o json
    {
      "production": {
        "secret_key_base": "abfb978b1a25ff3887458849fea585f8d14ec7295720c98d4bd4xxxxxxfbd83268730fe71841af490607ccc7deb5974acdcde9238c441b19b4aa6c568",
        "otp_key_base": "8d1dac40d1c80c8c3fd9507417c6c3ef8e7d2078722de844d78b0xxxxxx3256cde5ec2f6ed954ae131a3604905d5470fce7cfcf887c3b8f55f6f8b0aac",
        "db_key_base": "2491be299da2a44310bdd3a9999c27c685a98xxxxx9932be0e040211344aed132544efa5eb6d46670e365b24451a7d7ce61a721792c0",
        "openid_connect_signing_key": "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAxxxxxFtr64yx4zyrDMOV5QpY03jxr8Wqm/LkjJh3Y7DJHBSQIDAQxeem9Tlx8L\nS5OBfHS8QoH1NFGxM30o6mlUmiSGWhnQoRIUi3+9vR0WmvA1Ab904NL7HmWh4p5WpC7\n3lMXsAyOaRBmLcmXQuTzsB5OD6RuTqYMCqaBaWxewMqP/TrluaO4dObeSzRRf7PV\nElvQsrqTsqWV+a38bTlG4Z0GJLT76Eq4KGn0mDnnEGx6uezvJf4MUiECgYEA0BNa\nz6xQddjDC2rm9YNkQZQPBBdn0nqq0h+Qd6Ymap16VFVSSebjI4lshqiUOpDPbw2o\nD3eJMeVbONguYFHqD+RqzgmFMTyvXAOEY1lvVMfiPaTXRzSuYz31WGr11tmeG7OF\nHdGUQ6+Zc4h3YLDO8y4hME7o/FquMxElvbKZWikCgYAKuHI8MJOy1i1/u1i6Svy3\nzJfbzLzDsPBS2LOI7bkHdBOpf+DsnGX9yHnDFemupC81Vhyvq0MYtVsJyFOkUhU4\n6i3EpiBIJRbFKi0hQhAz3mBhvrrK8t8j3K15Bj6tlh7JPltNlTKLcsrwFJTriUoj\nSvCRMgfyai29sqiOa1+ywQKBgQCZll/GwSOXCUxXRi569ORw/4/h7kDlfURP25qw\npsTel6UvUNdv02y/03V3JEJdxHxJNeRinlJ3sRuXpwL8eBp0Zp9rvF1DTc8G9VWo\nW+CwzOYzqFR7q+g5Owe5nyId1/475lQRAZ0WJSz4ubeceIYZvGglF2oks+63pSWd\nk5JcmQKBgQDuE/LftG8cMAet/bg6LkXeK2DVultT7UXD7bUNcm7hlliZL8piVNrW\nMBq4VRfuO/2kcYA4NyMUDFuRQCxQ9lsOkRKXBUv/PutyRo/jFe/d/jfvEbM7MSph\n/Puu6GQPYjW3MY9CV59uRAMKWE1vK2zjPume143wyILu6SVDQaKNTA==\n-----END RSA PRIVATE KEY-----\n",
        "ci_jwt_signing_key": "-----BEGIN RSA PRIVATE KEY-----\nMIIEpAIBAAKCAQEAw+emHJQxCFmM/9oaUqjRfEC4oha0qL/ahIsf11oFayGTlR0K\nYr+xxx+k19pvD2LbnEsDOv/a\nFp0MBLkdL87UB5u3c/v67IuVOZhObOP/iIboWmINHMxc2J73QnfzSYK0sUuSqc74AgEJyUsVzIO62dtNBMUqXbM5nMaUlZfYP\nUnUotz1xi6atvrFtoju0/TExMLV3hoChNv0ByO5t4TdGemBa7RQ0W1cMz2CY3wJz\nuvlzlz0kIHqlcn2ZkzLQ7OvYIsqkl4lUKhThE6ko9xqUrtmmHjrmELmoDqHu0D4j\nKbTDvJ3x5T5cVvlnqDBisO2zKGRTDuQgGhyX6eyBFFtmvFw9hujMGHivku951DkA\nOm7nVz8ZhkKu8FkxIUAEaXqMCmirjeoWzdRmbEAeBrL01FTOfUQY3BcsB3qahND/\nDIBCPcECgYEA+L/IhmM6+6cDWiIXzXR28ZQ5uqRPF25sqN//+cNLOCnGx/QAxXag\n1TzVxNiBhZeS+BuwpYMXLaFekzIHFV3URpqjsHjmB2399zSPwFdX5Q5DRr8cgvIY\nLbTIA5FCEsyPRgLL4fPT1ZEbAkA5quMi8UzqzyJvRlE8quuwHob1qI0CgYEAyZ2H\n5npGb66/U1NjG06PiWUdgMD9fOGSOAaeNoRmNyo2t6zYtPQ3qtGIKPUtcTUASY4P\nHtkt3w2EB0z7XIe8eRn6aCc0vJ8zEBAz/6IQ9bCHSC/te+Jj5lXG7OVO8TxhmgtY\n7UmW8JzHMYaeXlGBIxoQ973Gnyb01hOJe7lit4kCgYEAjcljl5aATGlKc9nzD11P\nXyxKK6T0oDqFHU1xLwCuo3jMobTnq6aOzn06rFVsnqVjVKET84PhdlUA/44Ik5lE\nImqK21BObfW4SWxgdBZVN28F0hGlQs6UEZl2WPI3Y1fOYu29ITJGkPmBF6tcM5f8\nluZtAVxzaPVtS0/Et+HdrRECgYEAocq49EvLmnQxNT0FmzRAG5H5SwmUYlLic/Nb\no4Q8QqitoFgkz5Hr2jire7LE9MQDpwNJPwgpt4WxHeq5DFgg903RlSNhPrzCzXEz\nSUFVOtSeu186xN+4K29KY3DhGNXLvUK96i3T4uLtNuFA1Y+ygei5FRZF/hHVCLZE\n7fSnM4ECgYAdVC85PiYuwmU1FxHmlgXC97Wv4/idBBqRT/rDfc/tdB2XcAJz9Jzw\nk5QiFgGbfFHWqlMH1XZ5MSVJqnl7PYtOHKBi5xV2VIbbbC4givWzAU3YJcmO7YuA\ndCa5xXeck9DptcpNMSXL81DaeSjk3+5027IaK9F9pcSYKP9dBCzfeQ==\n-----END RSA PRIVATE KEY-----"
      }
    }

    $ cat << 'EOF' > check_gitlab.sh
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

    # Enter the pod to execute the recovery
    $ kubectl get po -l deploy=gitlab -n $GITLAB_NAMESPACE
    gitlab-98c4b9f4-lcjwn   1/1     Running   0          9s

    $ bash check_gitlab.sh gitlab-98c4b9f4-lcjwn
    2024-11-01 16:02:47 - Starting monitoring script for GitLab pod: gitlab-7ff8d674bd-m65nz on port: 30855
    command terminated with exit code 7
    2024-11-01 16:12:59 - HTTP not ready. Retrying in 10 seconds...
    command terminated with exit code 7
    2024-11-01 16:13:09 - HTTP not ready. Retrying in 10 seconds...
    2024-11-01 16:13:19 - HTTP not ready. Retrying in 10 seconds...
    2024-11-01 16:13:29 - HTTP not ready. Retrying in 10 seconds...
    2024-11-01 16:13:40 - HTTP not ready. Retrying in 10 seconds...
    2024-11-01 16:13:50 - GitLab is ready

    #############
    # Replace the corresponding fields in /etc/gitlab/gitlab-secrets.json with the data from the gitlab 14 rails-secret.
    # Note: Do not overwrite the YAML, only replace the following fields:
    # - secret_key_base, otp_key_base
    # - db_key_base
    # - openid_connect_signing_key
    # - ci_jwt_signing_key
    # Replace with the corresponding field data from the backup.
    ##############
    $ kubectl exec -ti gitlab-98c4b9f4-lcjwn -n $GITLAB_NAMESPACE -- bash
    vi /etc/gitlab/gitlab-secrets.json
    exit

    # Restart the instance
    $ kubectl delete po -n $GITLAB_NAMESPACE gitlab-98c4b9f4-lcjwn
    # Re-enter the pod, confirm that gitlab is started successfully
    $ kubectl get po -l deploy=gitlab -n $GITLAB_NAMESPACE
    NAME                    READY   STATUS    RESTARTS   AGE
    gitlab-98c4b9f4-lcjwn   1/1     Running   0          9s

    $ bash check_gitlab.sh gitlab-98c4b9f4-lcjwn
    2024-11-01 16:02:47 - Starting monitoring script for GitLab pod: gitlab-7ff8d674bd-m65nz on port: 30855
    command terminated with exit code 7
    2024-11-01 16:12:59 - HTTP not ready. Retrying in 10 seconds...
    command terminated with exit code 7
    2024-11-01 16:13:09 - HTTP not ready. Retrying in 10 seconds...
    2024-11-01 16:13:19 - HTTP not ready. Retrying in 10 seconds...
    2024-11-01 16:13:29 - HTTP not ready. Retrying in 10 seconds...
    2024-11-01 16:13:40 - HTTP not ready. Retrying in 10 seconds...
    2024-11-01 16:13:50 - GitLab is ready

    $ kubectl -n $GITLAB_NAMESPACE exec -ti gitlab-98c4b9f4-lcjwn -- bash
    # Set gitlab to read-only
    $ gitlab-ctl stop puma
    $ gitlab-ctl stop sidekiq
    # Confirm that puma and sidekiq have been stopped
    $ gitlab-ctl status

    # Execute backup
    $ ls /var/opt/gitlab/backups/
    17302758xx_xxxx_xx_xx_14.0.12_gitlab_backup.tar

    # This step will prompt you multiple times to confirm whether to continue.
    # Please select "yes" each time to proceed.
    $ gitlab-backup restore BACKUP=17302758xx_xxxx_xx_xx_14.0.12
    $ exit

    # Restart gitlab
    $ kubectl -n $GITLAB_NAMESPACE delete po gitlab-98c4b9f4-lcjwn

    # Check the recovery status
    $ kubectl get po -l deploy=gitlab -n $GITLAB_NAMESPACE
    NAME                    READY   STATUS    RESTARTS   AGE
    gitlab-98c4b9f4-lcjwn   1/1     Running   0          9s

    # Wait for GitLab background data migration tasks to complete
    # This process may take anywhere from a few minutes to up to 20 minutes, please be patient
    $ bash check_gitlab.sh gitlab-98c4b9f4-lcjwn
    $ kubectl -n $GITLAB_NAMESPACE exec -ti gitlab-98c4b9f4-lcjwn -- gitlab-rake gitlab:check SANITIZE=true
    ```

#### Upgrade `all-in-one` deployed gitlab to 17.8.1

Use the `all-in-one` mode to upgrade gitlab to 17.8.1. You need to replace the gitlab image in the upgrade path one by one until you upgrade to 17.8.1.

Upgrade path: 14.0.12 → 14.3.6 → 14.9.5 → 14.10.5 → 15.0.5 → 15.4.6 → 15.11.13 → 16.3.8 → 16.7.9 → 16.11.10 → 17.3.6 → 17.5.5 → 17.8.1

1. Upgrade 14.0.12 to 14.3.6

    ```bash
    # Modify the gitlab image in the deployment, the image repository address should be modified according to the actual address, here 127.0.0.1 is used as an example
    $ kubectl -n $GITLAB_NAMESPACE set image deployment/gitlab gitlab=127.0.0.1/gitlab/gitlab-ce:14.3.6-ce.0
    $ cat << 'EOF' > monitor_gitlab.sh
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

    $ kubectl -n $GITLAB_NAMESPACE get po -l deploy=gitlab
    NAME                    READY   STATUS    RESTARTS   AGE
    gitlab-98c4b9f4-lcjwn   1/1     Running   0          9s
    # Wait for the pod to start, then check if the application has started successfully
    $ bash monitor_gitlab.sh gitlab-98c4b9f4-lcjwn
    2024-11-01 16:02:47 - Starting monitoring script for GitLab pod: gitlab-7ff8d674bd-m65nz on port: 30855
    command terminated with exit code 7
    2024-11-01 16:12:59 - HTTP not ready. Retrying in 10 seconds...
    command terminated with exit code 7
    2024-11-01 16:13:09 - HTTP not ready. Retrying in 10 seconds...
    2024-11-01 16:13:19 - HTTP not ready. Retrying in 10 seconds...
    2024-11-01 16:13:29 - HTTP not ready. Retrying in 10 seconds...
    2024-11-01 16:13:40 - HTTP not ready. Retrying in 10 seconds...
    2024-11-01 16:13:50 - GitLab is ready, all migrations are done
    ```

2. Upgrade 14.3.6 to 14.9.5: Replace the image tag with 14.9.5-ce.0, other operations are the same as "14.0.12 to 14.3.6".
3. Upgrade 14.9.5 to 14.10.5: Replace the image tag with 14.10.5-ce.0, other operations are the same as "14.0.12 to 14.3.6".
4. Upgrade 14.10.5 to 15.0.5: Replace the image tag with 15.0.5-ce.0, other operations are the same as "14.0.12 to 14.3.6".
5. Upgrade 15.0.5 to 15.4.6: Replace the image tag with 15.4.6-ce.0, other operations are the same as "14.0.12 to 14.3.6".
6. Upgrade 15.4.6 to 15.11.13: Replace the image tag with 15.11.13-ce.0, other operations are the same as "14.0.12 to 14.3.6".
7. Upgrade 15.11.13 to 16.3.8: Replace the image tag with 16.3.8-ce.0, other operations are the same as "14.0.12 to 14.3.6".
8. Upgrade 16.3.8 to 16.7.9: Replace the image tag with 16.7.9-ce.0, other operations are the same as "14.0.12 to 14.3.6".
9. Upgrade 16.7.9 to 16.11.10: Replace the image tag with 16.11.10-ce.0, other operations are the same as "14.0.12 to 14.3.6".
10. Upgrade 16.11.10 to 17.3.6: Replace the image tag with 17.3.6-ce.0, other operations are the same as "14.0.12 to 14.3.6".
11. Upgrade 17.3.6 to 17.5.5: Replace the image tag with 17.5.5-ce.0, other operations are the same as "14.0.12 to 14.3.6".
12. Upgrade 17.5.5 to 17.8.1: Replace the image tag with 17.8.1-ce.0, other operations are the same as "14.0.12 to 14.3.6".

### Backup the data from 17.8.1 `all-in-one` instance

In the 17.8.1 deployment pod, execute the backup.

```yaml
kubectl -n $GITLAB_NAMESPACE get po -l deploy=gitlab
NAME                    READY   STATUS    RESTARTS   AGE
gitlab-98c4b9f4-lcjwn   1/1     Running   0          9s
kubectl -n $GITLAB_NAMESPACE exec -ti gitlab-98c4b9f4-lcjwn -- bash
gitlab-ctl stop puma
gitlab-ctl stop sidekiq
gitlab-ctl status
gitlab-rake gitlab:backup:create
ls /var/opt/gitlab/backups/
17302758xx_xxxx_xx_xx_14.0.12_gitlab_backup.tar
chmod 777 /var/opt/gitlab/backups/17302758xx_xxxx_xx_xx_14.0.12_gitlab_backup.tar
exit

# Backup gitlab 17.8.1 deployment's gitlab-secrets
kubectl -n $GITLAB_NAMESPACE cp gitlab-98c4b9f4-lcjwn:/etc/gitlab/gitlab-secrets.json ./gitlab-secrets.json
yq -P '{"production": .gitlab_rails}' gitlab-secrets.json -o yaml > gitlab-secrets.yaml
```

### Restore `all-in-one` backup data to `platform-deployed` 17.8.z

:::warning
During the restore process, the toolbox container may be killed due to insufficient resources. To avoid this, configure resource requests and limits for the toolbox container (at least 2 CPU and 4 GiB memory).
:::

1. Deploy gitlab 17.8.z on the platform using the operator and enable the toolbox (note that it needs to be in the same namespace as the old instance), then restore the data to the gitlab 17.8.z deployed by the operator (set the name of the gitlab deployed on the platform to the environment variable NEW_GITLAB_NAME).

    In the gitlab 17.8.z, add the following values to enable the gitlab 17.8.z toolbox:

    ```yaml
    spec:
      helmValues:
        gitlab:
          toolbox:
            enabled: true
            resources:
              limits:
                cpu: 2
                memory: 4G
    ```

    :::tip
    During the recovery process, database-related errors may occur, for example:
    - `ERROR:  role "xxx" does not exist`
    - `ERROR:  function xxx does not exist`

    These errors can be ignored, details can be found at (https://gitlab.com/gitlab-org/gitlab/-/issues/266988).
    :::

    ```bash
    export NEW_GITLAB_NAME=<platform gitlab instance name>

    # Replace the rails-secret of the deployment deployed by the operator with the rails-secret of the platform deployed gitlab
    kubectl -n $GITLAB_NAMESPACE get secrets -l release=${NEW_GITLAB_NAME}| grep rails-secret
    gitlab-rails-secret                                             Opaque               1      2d2h

    kubectl -n $GITLAB_NAMESPACE delete secret ${NEW_GITLAB_NAME}-rails-secret
    kubectl -n $GITLAB_NAMESPACE create secret generic ${NEW_GITLAB_NAME}-rails-secret --from-file=secrets.yml=gitlab-secrets.yaml

    # Restart the pod to make the changes take effect
    kubectl delete pod -n $GITLAB_NAMESPACE -l release=$NEW_GITLAB_NAME,app=webservice
    kubectl delete pod -n $GITLAB_NAMESPACE -l release=$NEW_GITLAB_NAME,app=sidekiq
    kubectl delete pod -n $GITLAB_NAMESPACE -l release=$NEW_GITLAB_NAME,app=toolbox
    # Wait for the pod to be ready before proceeding with the next steps
    kubectl get po -n $GITLAB_NAMESPACE -l release=$NEW_GITLAB_NAME

    # Close the external access of the 17.8.z gitlab
    kubectl patch deploy ${NEW_GITLAB_NAME}-webservice-default -n $GITLAB_NAMESPACE --type='merge' -p='{"metadata":{"annotations":{"skip-sync":"true"}}}'
    kubectl patch deploy ${NEW_GITLAB_NAME}-sidekiq-all-in-1-v2 -n $GITLAB_NAMESPACE --type='merge' -p='{"metadata":{"annotations":{"skip-sync":"true"}}}'
    kubectl scale deploy ${NEW_GITLAB_NAME}-webservice-default -n $GITLAB_NAMESPACE --replicas 0
    kubectl scale deploy ${NEW_GITLAB_NAME}-sidekiq-all-in-1-v2 -n $GITLAB_NAMESPACE --replicas 0

    # Modify the toolbox, mount the backup pvc to the toolbox
    kubectl patch deploy ${NEW_GITLAB_NAME}-toolbox -n $GITLAB_NAMESPACE -p='{"metadata":{"annotations":{"skip-sync":"true"}}}'
    kubectl patch deploy ${NEW_GITLAB_NAME}-toolbox -n $GITLAB_NAMESPACE --type='json' -p='[{"op": "replace", "path": "/spec/template/spec/volumes/1", "value": {"name": "toolbox-tmp", "persistentVolumeClaim": {"claimName": "backup-pvc"}}}]'

    # Enter the pod to restore the data
    kubectl get po -n $GITLAB_NAMESPACE -l release=$NEW_GITLAB_NAME,app=toolbox
    NAME READY STATUS RESTARTS AGE
    gitlab-toolbox-58b7db895c-8k99q 1/1 Running 0 119m

    kubectl exec -ti -n $GITLAB_NAMESPACE gitlab-toolbox-58b7db895c-8k99q -- bash
    ls /srv/gitlab/tmp/backups/
    17302758xx_xxxx_xx_xx_17.8.1_gitlab_backup.tar

    # Note that this step will delete the backup file, you can back it up in advance: cp /srv/gitlab/tmp/backups/17302758xx_xxxx_xx_xx_17.8.1_gitlab_backup.tar /srv/gitlab/tmp/backups/17302758xx_xxxx_xx_xx_17.8.1_gitlab_backup2.tar

    backup-utility --restore -f file:///srv/gitlab/tmp/backups/17302758xx_xxxx_xx_xx_17.8.1_gitlab_backup.tar
    exit

    kubectl patch deploy ${NEW_GITLAB_NAME}-webservice-default -n $GITLAB_NAMESPACE --type='json' -p='[{"op":"remove","path":"/metadata/annotations/skip-sync"}]'
    kubectl patch deploy ${NEW_GITLAB_NAME}-sidekiq-all-in-1-v2 -n $GITLAB_NAMESPACE --type='json' -p='[{"op":"remove","path":"/metadata/annotations/skip-sync"}]'
    kubectl patch deploy ${NEW_GITLAB_NAME}-toolbox -n $GITLAB_NAMESPACE --type='json' -p='[{"op":"remove","path":"/metadata/annotations/skip-sync"}]'

    kubectl scale deploy ${NEW_GITLAB_NAME}-webservice-default -n $GITLAB_NAMESPACE --replicas 1
    kubectl scale deploy ${NEW_GITLAB_NAME}-sidekiq-all-in-1-v2 -n $GITLAB_NAMESPACE --replicas 1
    ```

2. Verify that the data after the upgrade is different from the data before the upgrade.
   - Enter the GitLab UI in administrator view and check whether the repositories and user data are normal.
   - Select some repositories and verify whether the code, branches, and merge requests are functioning properly.
3. After the data verification is complete, manually clean up the old gitlab instance and the old operator if necessary.

### FAQ

1. The migrated data does not include avatars and attachments, so the new GitLab instance cannot display this content. To resolve this issue, you can replace the upload PVC of GitLab 17.8.z with the PVC used by 14.0.12 or manually copy these data from the old instance to the new instance.
