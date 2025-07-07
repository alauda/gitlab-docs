---
title: "GitLab Instance Deployment"
description: "GitLab is an open-source code hosting platform that provides code management, code collaboration, continuous deployment, and many other features."
weight: 30
---

This document introduces GitLab Operator subscription and GitLab instance deployment functionality.

## Prerequisites

- This document applies to GitLab 17 and above versions provided by the platform. It is decoupled from the platform based on technologies such as Operator.

- Please ensure that GitLab Operator has been deployed (subscribed) in the target cluster, i.e., the GitLab instance can be created from GitLab Operator.

## Deployment Planning

GitLab supports various resource configurations to accommodate different customer scenarios. In different scenarios, the required resources and configurations may vary significantly. Therefore, this section introduces the aspects to consider when planning a GitLab instance deployment, as well as the impact of decision points, to help users make informed decisions for subsequent instance deployments.

### Basic Information

1. The GitLab Operator provided by the platform is based on the official community GitLab Operator, with enterprise-level capability enhancements such as IPv6 support, ARM support, and security vulnerability fixes. It is fully compatible with the community version in terms of functionality, and enhances the convenience of GitLab deployment through optional and customizable templates.

2. A GitLab instance includes multiple components, such as the `Gitaly component` responsible for Git repository access, the `PostgreSQL` component providing storage for application metadata and user information, and the `Redis` component used for caching, etc. The platform provides professional PostgreSQL Operator and Redis Operator, so when deploying a GitLab instance, Redis and PostgreSQL resources are no longer deployed directly, but accessed through configuring specific access credentials to existing instances.

### Pre-deployment Resource Planning

Pre-deployment resource planning refers to the resource planning that needs to be decided before deployment and takes effect during deployment. The main content includes:

**High Availability**

- GitLab supports high availability deployment. The main impacts and limitations of this mode are:

   - Each component will be deployed with multiple replicas

   - Network access no longer supports `NodePort`, but requires access through domain names configured via `Ingress`

   - Storage method no longer supports `node storage`, but requires access through `StorageClass` or `PVC`

**Resources**

According to community recommendations and practices, a non-high-availability GitLab instance can run with a minimum of 3 cores and 6Gi of resources, while in high availability mode, a minimum of 18 cores and 22Gi of resources is required for stable operation.

**Storage**

- Common storage methods provided by the platform can be used for GitLab, such as storage classes, persistent volume claims, node storage, etc.

   - TopoLVM storage class is recommended for Gitaly storage

   - Node storage is not suitable for `high availability` mode, as it stores files in a specified path on the host node

- In addition, GitLab also supports object storage. In this mode, separate storage access method adaptation is required. Please consult the vendor for details.

**Network**

- The platform provides two mainstream network access methods: `NodePort` and `Ingress`

   - `NodePort` requires specifying HTTP and SSH ports, and ensuring that the ports are available. `NodePort` is not suitable for `high availability` mode

   - `Ingress` requires specifying a domain name and ensuring that the domain name resolution is working properly

- The platform supports the HTTPS protocol, which needs to be configured after instance deployment. See [Configure HTTPS](#gitlab_https) for details.

**Redis**

- The Redis component version currently required by GitLab is v6. It is recommended to deploy Redis instances using the Redis Operator provided by the platform, and then complete Redis integration by configuring access credentials.

   - Redis access is achieved by configuring a `secret` resource with specific format content. See [Configure Redis, PostgreSQL, and Account Credentials](./02_gitlab_credential.mdx) for details.

**PostgreSQL**

- The PostgreSQL component version currently required by GitLab is v14. It is recommended to deploy PostgreSQL instances using the PostgreSQL Operator provided by the platform, and then complete PostgreSQL integration by configuring access credentials.

   - PostgreSQL access is achieved by configuring a `secret` resource with specific format content. See [Configure Redis, PostgreSQL, and Account Credentials](./02_gitlab_credential.mdx) for details.

- Note: In high availability mode, the Gitaly component is in Cluster mode and requires an additional PostgreSQL credential configured in the same way as above to access a separate database. It can be two different databases under the same PostgreSQL instance.

**Account Credentials**

When creating a GitLab instance, you need to configure the root account and its password. This is done by configuring a `secret` resource. See [Configure Redis, PostgreSQL, and Account Credentials](./02_gitlab_credential.mdx) for details.


### Post-deployment Configuration Planning

Post-deployment configuration planning refers to configurations that do not need to be decided before deployment but can be completed as needed through standardized operations after deployment. These mainly include Single Sign-On (SSO), HTTPS configuration, external load balancer configuration, etc. For details, refer to [Next Steps](#gitlab_day2).


## Instance Deployment

The GitLab Operator provided by the platform offers two deployment methods: deployment from templates and deployment from YAML.

The platform provides two built-in templates: the `GitLab Quick Start` template and the `GitLab High Availability` template, while also supporting customer-defined templates to meet their specific scenarios.

Information about the built-in templates and YAML deployment is as follows:

### Deploying from the `GitLab Quick Start` Template

This template is used to quickly create a lightweight GitLab instance suitable for development and testing scenarios, not recommended for production environments.

- Computing resources: 3 CPU cores, 6Gi memory
- Storage method: Uses local node storage, requires configuring storage node IP and path
- Network access: Uses NodePort method, shares the node IP with storage, requires port specification
- Dependent services: Requires configuration of existing Redis and PostgreSQL access credentials
- Other settings: Requires account credentials configuration, SSO functionality is disabled by default

Complete the deployment by filling in the relevant information according to the template prompts.

### Deploying from the `GitLab High Availability` Template

Deploying a high-availability GitLab instance requires higher resource configuration and provides a higher availability standard.

- Computing resources: 18 CPU cores, 22Gi memory
- Storage method: Uses storage class resources to store Gitaly, the code repository component
- Network access: Uses Ingress method, requires domain name specification
- Dependent services: Requires configuration of existing Redis and PostgreSQL access credentials, and additionally requires separate access credentials for another independent database in the PG instance for the Gitaly component
- Other settings: Requires account credentials configuration, SSO functionality is disabled by default

To achieve GitLab high availability, external dependencies must meet the following conditions:

1. The `attachment storage class` must support multi-node read and write (ReadWriteMany)
2. The `Redis` and `PostgreSQL` instances must be highly available
3. The network load balancer must be highly available; when using ALB, a VIP must be configured
4. The cluster must have more than 2 nodes

Complete the deployment by filling in the relevant information according to the template prompts.

### Deploying from YAML

YAML deployment is the most basic and powerful deployment capability. Here we provide corresponding YAML snippets for each dimension from the `Deployment Planning` section, and then provide two complete YAML examples for different scenarios to help users understand the YAML configuration method and make changes as needed.

#### YAML Snippets Based on Deployment Planning

##### High Availability

In high availability mode, Gitaly needs to use cluster mode, and other components need at least 2 replicas. The YAML configuration snippet for high availability deployment is as follows:

```yaml
spec:
  helmValues:
    gitlab:
      gitlab-shell:
        maxReplicas: 2
        minReplicas: 2
      sidekiq:
        maxReplicas: 2
        minReplicas: 2
      webservice:
        maxReplicas: 2
        minReplicas: 2
    global:
      praefect:
        psql:
          host: host.example # PostgreSQL instance address
          port: 5432 # PostgreSQL instance port
          user: username # PostgreSQL instance access user
          dbName: praefect-db # PostgreSQL instance access database
          sslMode: disable # PostgreSQL instance SSL mode
        dbSecret:
          key: secret # Key in the PostgreSQL instance secret that stores the password
          secret: praefect-db # Secret for the PostgreSQL database used by Gitaly cluster
        enabled: true
        virtualStorages:
        - gitalyReplicas: 3
          maxUnavailable: 1
          name: default
```

##### Storage

GitLab data storage mainly includes two parts:

1. Gitaly storage: Stores code repository data
2. Attachment storage: Stores user avatars, user-uploaded images, etc.

Currently, three storage configuration methods are supported: storage class, PVC, and local node storage.

Storage class configuration snippet:

```yaml
spec:
  helmValues:
    gitlab:
      gitaly:
        persistence:
          enabled: true
          size: 20Gi # Gitaly storage capacity
          storageClass: topolvm # Storage class used for Gitaly
    global:
      uploads:
        persistence:
          enabled: true
          storageClass: ceph # Storage class used for attachments, needs to support multi-node read-write (ReadWriteMany)
          size: 10Gi # Attachment storage capacity
```

PVC configuration snippet:

```yaml
spec:
  helmValues:
    gitlab:
      gitaly:
        persistence:
          enabled: true
          existingClaim: pvc-gitaly # PVC name for Gitaly storage, needs to be created in advance
    global:
      uploads:
        persistence:
          enabled: true
          existingClaim: pvc-uploads # PVC name for attachment storage, needs to be created in advance
```

Local node storage configuration snippet:

```yaml
spec:
  helmValues:
    global:
      nodeSelector:
        kubernetes.io/hostname: node-1 # Node name, not the node IP address
      hostpath:
        enabled: true
        nodeName: node-1 # Node name
        path: /mnt/data/gitlab-hostpath # Node storage path, needs to be created in advance
```

##### Network Access

Network access mainly includes two methods: domain name access and NodePort access.

Domain name access configuration snippet:

```yaml
spec:
  helmValues:
    gitlab:
      webservice:
        ingress:
          enabled: true # Enable domain name access
    global:
      hosts:
        domain: gitlab.example.com # Domain name
        gitlab:
          hostnameOverride: gitlab.example.com # Domain name
          https: false # Use HTTP access instead of HTTPS
          name: gitlab.example.com # Domain name
        ssh: gitlab.example.com # Domain name
      ingress:
        class: "" # Leave empty to use the cluster default
        enabled: true # Enable domain name access
        tls:
          enabled: false # Use HTTP access instead of HTTPS
```

NodePort method requires configuring two ports: HTTP port and SSH port.

NodePort access configuration snippet:

```yaml
spec:
  helmValues:
    gitlab:
      gitlab-shell:
        service:
          nodePort: 30000 # SSH port number
          type: NodePort
      webservice:
        ingress:
          enabled: false # Disable ingress access
    global:
      appConfig:
        gitlabPort: 30010 # HTTP port number
      gitlabNodePort:
        http: 30010 # HTTP port number
        ssh: 30000 # SSH port number
      hosts:
        domain: 192.168.100.100 # Cluster Node IP
        gitlab:
          https: false
          name: 192.168.100.100 # Cluster Node IP
        ssh: 192.168.100.100 # Cluster Node IP
      ingress:
        enabled: false # Disable ingress access
      shell:
        port: 30000 # SSH port number
```

##### Redis Access Credential Configuration \{#configure-redis-credentials}

This refers to the configuration snippet in the GitLab instance for the Redis credential after configuring the `secret` resource with Redis credentials:

Standalone example:

```yaml
spec:
  helmValues:
    global:
      redis:
        auth:
          enabled: true
          key: password # Key in the Redis connection secret that stores the password
          secret: gitlab-redis-password # Secret containing Redis connection key
        host: 192.168.100.200 # Redis instance address
        port: 6379 # Redis instance port
```

Sentinel example:

```yaml
spec:
  helmValues:
    global:
      redis:
        auth:
          enabled: true
          key: password # Key in the Redis connection secret that stores the password
          secret: gitlab-redis-password # Secret containing Redis connection key
        host: mymaster # Sentinel monitor instances group name
        sentinels:
          - host: 192.168.100.200 # Sentinel node address
            port: 16379 # Sentinel node port
          - host: 192.168.100.201 # Sentinel node address
            port: 16379 # Sentinel node port
          - host: 192.168.100.202 # Sentinel node address
            port: 16379 # Sentinel node port
```

##### PostgreSQL Access Credential Configuration \{#configure-postgresql-credentials}

This refers to the configuration snippet in the GitLab instance for the PostgreSQL credential after configuring the `secret` resource with PostgreSQL credentials, which also includes the case of configuring two PostgreSQL databases in high availability mode:

```yaml
spec:
  helmValues:
    global:
      psql:
        database: gitlab-db # PostgreSQL instance access database
        host: 192.168.100.300 # PostgreSQL instance address
        password:
          key: password # Key in the PostgreSQL instance secret that stores the password
          secret: gitlab-pg-password # PostgreSQL instance secret
        port: 5432 # PostgreSQL instance port
        username: gitlab-user # PostgreSQL instance access user
```

##### Root Account Configuration

This refers to the configuration snippet in the GitLab instance for the account credential after configuring the `secret` resource with account credentials:

```yaml
spec:
  helmValues:
    global:
      initialRootPassword:
        key: password # Key in the GitLab root account secret that stores the password
        secret: gitlab-root-password # GitLab root account secret
```

#### Complete YAML Example: Single Instance, Node Storage, NodePort Network Access

```yaml
spec:
  helmValues:
    gitlab:
      gitaly:
        persistence:
          enabled: true
        resources:
          limits:
            cpu: 1
            memory: 1Gi
          requests:
            cpu: 500m
            memory: 512Mi
        volumePermissions:
          enabled: true
      gitlab-shell:
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 300m
            memory: 300Mi
      migrations:
        resources:
          limits:
            cpu: 2
            memory: 2Gi
          requests:
            cpu: 500m
            memory: 512Mi
      sidekiq:
        resources:
          limits:
            cpu: 500m
            memory: 1Gi
          requests:
            cpu: 200m
            memory: 200Mi
      webservice:
        ingress:
          enabled: false
        resources:
          limits:
            cpu: 1
            memory: 2.5Gi
          requests:
            cpu: 500m
            memory: 512Mi
        workhorse:
          resources:
            limits:
              cpu: 300m
              memory: 300Mi
            requests:
              cpu: 200m
              memory: 200Mi
    global:
      nodeSelector:
        kubernetes.io/hostname: node-1 # Node name
      appConfig:
        gitlabPort: 30010 # HTTP port number
        object_store:
          enabled: true
        omniauth:
          blockAutoCreatedUsers: false
          dex:
            enabled: false
          enabled: false
      gitlabNodePort:
        http: 30010 # HTTP port number
        ssh: 30000 # SSH port number
      hostpath:
        enabled: true
        nodeName: node-1 # Node name
        path: /mnt/data/gitlab-hostpath # Node storage path, needs to be created in advance
      hosts:
        domain: 192.168.100.100 # Cluster Node IP
        gitlab:
          https: false
          name: 192.168.100.100 # Cluster Node IP
        ssh: 192.168.100.100 # Cluster Node IP
      ingress:
        enabled: false
        tls:
          enabled: false
      initialRootPassword:
        key: password
        secret: gitlab-root-password # Root account secret
      psql:
        database: gitlab-db # PostgreSQL instance access database
        host: 192.168.100.100 # PostgreSQL instance address
        password:
          key: password # Key in the PostgreSQL instance secret that stores the password
          secret: gitlab-pg-password # PostgreSQL instance secret
        port: 5432 # PostgreSQL instance port
        username: gitlab-user # PostgreSQL instance access user
      redis:
        auth:
          enabled: true
          key: password # Key in the Redis instance secret that stores the password
          secret: gitlab-redis-password # Redis instance secret
        host: 192.168.100.200 # Redis instance address
        port: 6379 # Redis instance port
      uploads:
        persistence:
          enabled: true
```

#### Complete YAML Example: High Availability, Storage Class, Ingress Network Access

```yaml
spec:
  helmValues:
    gitlab:
      gitaly:
        persistence:
          enabled: true
          size: 20Gi # Gitaly storage capacity
          storageClass: topolvm # Storage class used for Gitaly
        resources:
          limits:
            cpu: 4
            memory: 4Gi
          requests:
            cpu: 1
            memory: 1Gi
        volumePermissions:
          enabled: true
      gitlab-shell:
        maxReplicas: 2
        minReplicas: 2
        resources:
          limits:
            cpu: 1
            memory: 1Gi
          requests:
            cpu: 250m
            memory: 600Mi
      migrations:
        resources:
          limits:
            cpu: "1"
            memory: 1Gi
      sidekiq:
        maxReplicas: 2
        minReplicas: 2
        resources:
          limits:
            cpu: 2
            memory: 2Gi
          requests:
            cpu: 1
            memory: 1Gi
      webservice:
        ingress:
          enabled: true # Enable Ingress access
        maxReplicas: 2
        minReplicas: 2
        resources:
          limits:
            cpu: 2
            memory: 4Gi
          requests:
            cpu: 1
            memory: 2Gi
        workhorse:
          resources:
            limits:
              cpu: 1
              memory: 1Gi
            requests:
              cpu: 200m
              memory: 200Mi
    global:
      appConfig:
        object_store:
          enabled: true
        omniauth:
          blockAutoCreatedUsers: false
          dex:
            enabled: false
          enabled: false
      hosts:
        domain: gitlab.example.com # Domain name
        gitlab:
          hostnameOverride: gitlab.example.com # Domain name
          https: false
          name: gitlab.example.com # Domain name
        ssh: gitlab.example.com # Domain name
      ingress:
        class: ""
        configureCertmanager: false
        enabled: true
        tls:
          enabled: false
      initialRootPassword:
        key: password
        secret: gitlab-root-password # Root account secret
      praefect:
        psql:
          host: 192.168.100.300 # PostgreSQL instance address
          port: 5432 # PostgreSQL instance port
          user: gitlab-user # PostgreSQL instance access user
          dbName: gitlab-praefect # PostgreSQL instance access database
          sslMode: disable
        dbSecret:
          key: password # Key in the PostgreSQL instance secret that stores the password
          secret: praefect-db # Secret for the PostgreSQL database used by Gitaly cluster
        enabled: true
        virtualStorages:
          - gitalyReplicas: 3
            maxUnavailable: 1
            name: default
      psql:
        database: gitlab-db # PostgreSQL instance access database
        host: 192.168.100.300 # PostgreSQL instance address
        password:
          key: password # Key in the PostgreSQL instance secret that stores the password
          secret: gitlab-pg # PostgreSQL instance secret
        port: 5432 # PostgreSQL instance port
        username: gitlab-user # PostgreSQL instance access user
      redis:
        auth:
          enabled: true
          key: password # Key in the Redis instance secret that stores the password
          secret: gitlab-redis-password # Redis instance secret
        host: 192.168.100.200 # Redis instance address
        port: 6379 # Redis instance port
      uploads:
        persistence:
          enabled: true
          size: 10Gi # Attachment storage capacity
          storageClass: ceph # Storage class used for attachments, needs to support multi-node read-write (ReadWriteMany)
```

## <span id ="gitlab_day2">Next Steps</span>

### <span id ="gitlab_sso">Configure Single Sign-On (SSO)</span>

Configuring SSO involves the following steps:

1. Register an SSO authentication client in the global cluster
2. Prepare the SSO authentication configuration
3. Configure the GitLab instance to use the SSO authentication configuration

Create the following Oauth2Client resource in the global cluster to register the SSO authentication client.

```yaml
apiVersion: dex.coreos.com/v1
kind: OAuth2Client
name: OIDC
metadata:
  name: m5uxi3dbmiwwizlyzpzjzzeeeirsk # This value is calculated based on the hash of the id field, online calculator: https://go.dev/play/p/QsoqUohsKok
  namespace: cpaas-system
alignPasswordDB: true
id: gitlab-dex # Client ID
public: false
redirectURIs:
  - <gitlab-host>/users/auth/dex/callback # GitLab authentication callback address, where <gitlab-host> is replaced with the GitLab instance access address
secret: Z2l0bGFiLW9mZmljaWFsLTAK # Client secret
spec: {}
```

Prepare the configuration content according to the comments in the JSON below.

```yaml
{
 "name": "openid_connect",
 "label": "OIDC",
 "args": {
   "name": "dex",
   "scope": ["openid", "profile", "email"],
   "response_type": "code",
   "issuer": "<platform-url>/dex", # Platform authentication address, where <platform-url> is replaced with the platform access address
   "discovery": true,
   "client_auth_method": "query",
   "client_options": {
     "identifier": "test-dex", # Client ID
     "secret": "Z2l0bGFiLW9mZmljaWFsLTAK", # Client secret
     "redirect_uri": "<gitlab-host>/users/auth/dex/callback" # GitLab authentication callback address, where <gitlab-host> is replaced with the GitLab instance access address
   }
 }
}
```

Save the configuration content to a secret and create it in the namespace where the GitLab instance is located.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-sso-config
  namespace: gitlab
type: Opaque
data:
  provider: <base64 encoded config> # Save the above configuration content base64 encoded to the provider field
```

Edit the GitLab instance to add the following configuration:

```yaml
spec:
  helmValues:
    global:
      appConfig:
        omniauth:
          enabled: true
          allowSingleSignOn:
            - dex
          syncProfileAttributes:
            - email
          syncProfileFromProvider:
            - dex
          blockAutoCreatedUsers: false
          providers:
            - key: provider
              secret: dex-provider # Secret containing SSO authentication configuration
```

**Enable SSO for Platforms Using Self-Signed Certificates**

If the platform is accessed via HTTPS and uses a self-signed certificate, you need to mount the self-signed certificate's CA to the GitLab instance as follows:

In the `cpaas-system` namespace of the global cluster, find the secret named `dex.tls`, get the `ca.crt` content from the secret, save it as a new secret, and create it in the namespace of the GitLab instance.

```yaml
apiVersion: v1
data:
  ca.crt: <base64 encode data>
kind: Secret
metadata:
  name: dex-tls
  namespace: cpaas-system
type: kubernetes.io/tls
```

Edit the GitLab instance to use this CA:

```yaml
spec:
  helmValues:
    global:
      certificates:
        customCAs:
          - secret: dex-tls
```

### <span id ="gitlab_https">Configure HTTPS</span>

After deploying the GitLab instance, you can configure HTTPS as needed.

First, create a TLS certificate secret in the namespace where the instance is located.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-tls-cert
  namespace: gitlab
type: kubernetes.io/tls
data:
  tls.crt: <base64 encoded cert>
  tls.key: <base64 encoded key>
```

Then edit the YAML configuration of the GitLab instance to enable HTTPS access:

```yaml
spec:
  helmValues:
    gitlab:
      webservice:
        ingress:
          enabled: true
    global:
      hosts:
        domain: gitlab.example.com # Domain name
        gitlab:
          hostnameOverride: gitlab.example.com # Domain name
          https: true # Enable HTTPS
          name: gitlab.example.com # Domain name
        ssh: gitlab.example.com # Domain name
      ingress:
        class: ""
        enabled: true
        tls:
          enabled: true # Enable HTTPS
          secretName: gitlab-tls-cert # Certificate secret name
```

### Configure Monitoring Dashboard

GitLab Operator provides default support for the platform operations center monitoring dashboard. When GitLab Operator and GitLab instance are deployed, GitLab Metrics will be automatically configured and can be viewed in the operations center monitoring dashboard.

### Configure External Load Balancer (LB)

Configure External Load Balancer (LB) for GitLab

## Additional Information

### Pod Security Policy

When deploying GitLab in a namespace with Pod Security Policy (PSP) configured, ensure that the policy's enforce level is set to `privileged`.

**Tip**: Setting the Pod Security Policy enforce level to `restricted` or `baseline` in the namespace may cause GitLab deployment to fail.
