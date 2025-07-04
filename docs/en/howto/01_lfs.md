---
title: "Managing Large Files with LFS"
---

This article will introduce how to use LFS to manage large files, including GitLab instance configuration and Git client configuration.

Applicable scenarios:

- Using LFS to manage executable files in code repositories, such as compiled JAR packages
- Using LFS to manage AI large model files in code repositories

## GitLab Instance Configuration

### Prerequisites

- A GitLab instance has been deployed according to the [GitLab Instance Deployment](../install/03_gitlab_deploy.md) documentation.

GitLab supports [two methods](https://docs.gitlab.com/ee/administration/lfs/?tab=Helm+chart+%28Kubernetes%29) for configuring LFS, with the main difference being the storage location of LFS files.

1. Using local storage to save LFS files
2. Using object storage to save LFS files

> Since LFS typically stores large files, storage capacity usage can be significant. Therefore, it's important to plan storage capacity according to requirements before deployment.

### Using Local Storage for LFS Files

By default, the deployed GitLab instance has already enabled the LFS feature and uses local storage to save LFS files.

Locally stored LFS files are saved in the `attachment storage`, with the path: `shareds/lfs-objects`.

The attachment storage varies depending on the deployment method:

- GitLab instances deployed using the HostPath method use node storage. Due to the difficulty of expanding node storage, this method is not recommended for production environments.
- GitLab instances deployed using storage classes or specified PVC methods both use PVC as the storage medium.

### Using Object Storage for LFS Files (Recommended)

GitLab officially recommends using object storage to save LFS files.

GitLab [supports multiple types of object storage](https://docs.gitlab.com/ee/administration/object_storage.html#supported-object-storage-providers). The following example uses MinIO to illustrate how to configure GitLab to use object storage.

Create the following buckets in MinIO:

1. `git-lfs`
2. `gitlab-uploads`

The method to create buckets using the [mc cli](https://min.io/docs/minio/linux/reference/minio-mc.html#id3) command is as follows:

```bash
# Set MinIO access address, access key, and secret key
export MINIO_HOST=minio.example.com
export MINIO_ACCESS_KEY=your-access-key
export MINIO_SECRET_KEY=your-secret-key
mc alias set minio http://${MINIO_HOST} ${MINIO_ACCESS_KEY} ${MINIO_SECRET_KEY}

# Create git-lfs and gitlab-uploads buckets
mc mb minio/git-lfs
mc mb minio/gitlab-uploads
```

Prepare a MinIO configuration file named `rails-storage.yaml` with the following content:

```yaml
provider: AWS
region: us-east-1
aws_access_key_id: example
aws_secret_access_key: example
endpoint: "http://minio.example.com"
path_style: true
```

Where:

- `provider` is the type of object storage, MinIO uses the **fixed value** `AWS`
- `region` is the region of the object storage, MinIO uses the **fixed value** `us-east-1`
- `aws_access_key_id` is the access key ID for the object storage
- `aws_secret_access_key` is the access key for the object storage
- `endpoint` is the access address for the object storage
- `path_style` is the access method for the object storage, using the **fixed value** `true` here

Save the configuration file as a secret in the cluster, noting that the namespace must match that of the GitLab instance:

```bash
export GITLAB_NS=<your-gitlab-namespace>
kubectl create secret generic gitlab-rails-storage \
    -n ${GITLAB_NS} \
    --from-file=connection=rails-storage.yaml
```

Modify the GitLab instance configuration by adding the following to enable object storage:

```yaml
spec:
  helmValues:
    global:
      appConfig:
        object_store:
          connection:
            key: connection # key in the secret
            secret: gitlab-rails-storage # name of the secret created in the previous step
          enabled: true
```

### Resource and Parameter Configuration

Unlike regular API requests, when uploading and downloading LFS files, the workhorse component consumes more CPU resources. The resources of the workhorse component directly affect push and pull performance.

| Workhorse Component Resource Configuration | Push Peak Bandwidth | CPU Usage | Memory Usage |
| ------------------------------------------ | ------------------ | --------- | ------------ |
| 1C 500Mi                                   | 70 MBps            | 1C(100%)  | 100Mi(20%)   |
| 2C 500Mi                                   | 140 MBps           | 2C(100%)  | 100Mi(20%)   |

Modify the GitLab instance configuration by adding the following to adjust the workhorse component resources:

```yaml
spec:
  helmValues:
    gitlab:
      webservice:
        workhorse:
          resources:
            limits:
              cpu: 2
              memory: 500Mi
            requests:
              cpu: 200m
              memory: 200Mi
```

In addition, you need to increase the timeout for the webservice component. The configuration method is as follows:

```yaml
spec:
  helmValues:
    gitlab:
      webservice:
        extraEnv:
          GITLAB_RAILS_RACK_TIMEOUT: "300"
    global:
      webservice:
        workerTimeout: 300
```

## Git Client Configuration

### Prerequisites

- Git client is installed, execute the `git version` command to check the Git version
- Git-lfs client is installed, execute the `git-lfs version` command to check the git-lfs version

### Configuring Git Client Parameters

Execute the following command to save clone credentials, avoiding password input for every pull and push:

```bash
git config --global credential.helper store
```

Execute the following command to set the concurrent transfer count for LFS files, which can effectively improve the stability of push and pull, especially when pushing a large number of files at once:

```bash
git config --global lfs.concurrenttransfers 2
```

Execute the following command to set the LFS activity timeout. This parameter must be added if GitLab uses object storage:

```bash
git config --global lfs.activitytimeout 36000
```

### Git Project Configuration

Execute the following commands to clone the Git repository to your local machine and run `git lfs install` to install the LFS git hook:

```bash
git clone http://example.com/root/local-lfs.git
cd local-lfs
git lfs install
```

Execute the following command to set the tracking pattern for LFS files, for example, to track all `.safetensors` files:

```bash
git lfs track "*.safetensors"
```

The above operation will generate a `.gitattributes` file. Execute the following commands to commit this file to the remote repository first:

```bash
git add .gitattributes
git commit -m "Add LFS tracking for .safetensors files"
git push
```

Afterwards, adding or updating `.safetensors` files to the repository will automatically save them using LFS.

To verify if a file is saved using LFS, you can view the file in the GitLab repository. If there is a small LFS icon next to the file name, it indicates that the file is saved using LFS.

## Common Issues

### Clone Failure for Very Large Repositories

When cloning very large repositories, if the client has insufficient resources, the client may be killed by the system after running for a while. The solution is:

1. Add the `GIT_LFS_SKIP_SMUDGE=1` parameter when cloning to skip LFS file dumping
2. Enter the local code directory and execute the `git lfs pull` command to pull LFS files to the local machine

```bash
GIT_LFS_SKIP_SMUDGE=1 git clone http://example.com/root/local-lfs.git
cd local-lfs
git lfs pull
```
