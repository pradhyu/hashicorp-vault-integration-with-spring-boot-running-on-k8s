# 🚀 HashiCorp Vault & Spring Boot Integration on Red Hat OpenShift

Comprehensive architecture, deployment manifests, application configurations, and operational trade-offs for three integration patterns between **HashiCorp Vault** and **Spring Boot** on **Red Hat OpenShift**.

---

## 1. Executive Summary & Architecture Matrix

```
+-----------------------------------------------------------------------------------------------------+
|                                INTEGRATION PATTERNS OVERVIEW                                        |
+-----------------------------------+--------------------------------+--------------------------------+
|  Pattern 1: Vault Agent Sidecar   |  Pattern 2: Static Secret Sync | Pattern 3: Spring Cloud Vault  |
|  (Agent Injection Webhook)        |  (Vault Secrets Operator/ESO)  | (Direct JVM REST Integration)  |
+-----------------------------------+--------------------------------+--------------------------------+
| - Pod Mutation via Annotations    | - Kubernetes Operator Sync     | - In-memory secrets            |
| - Secret written to emptyDir      | - Stored in OpenShift Secret   | - Dynamic Leases / Token Mgmt  |
| - Zero Spring Boot code changes   | - Zero Spring Boot code changes| - Native Spring Cloud Config   |
+-----------------------------------+--------------------------------+--------------------------------+
```

### Feature Comparison Matrix

| Feature | Option 1: Vault Agent Sidecar | Option 2: Static Secret Sync (VSO/ESO) | Option 3: Direct Spring Cloud Vault |
| :--- | :--- | :--- | :--- |
| **Code Changes** | **None** | **None** | **Requires Dependencies & Config** |
| **OpenShift Prerequisite** | Vault Agent Injector Webhook | Vault Secrets Operator (VSO) | Direct network route to Vault |
| **Secret Storage Location** | In-memory `emptyDir` in Pod | OpenShift `etcd` (v1/Secret) | JVM Heap Memory only (Zero-Disk) |
| **Dynamic Secrets / DB Leases** | Supported (Agent renews leases) | Limited / Best for Static KV | **Native & Full Dynamic Lifecycle** |
| **Secret Rotation Handling** | File change / Pod restart | Pod restart or Config watcher | `@RefreshScope` / Actuator / Event |
| **Local Dev Parity** | Requires Mock/Sidecar | Standard K8s/Env profile | Easy local test via profile/token |
| **Security Context (SCC)** | Ephemeral volume mount | Standard OpenShift Secret mount | Standard Pod execution |

---

## 2. Option 1: Vault Agent Sidecar Injection

### 2.1 Architectural Flow

The Vault Agent Sidecar uses an OpenShift Mutating Admission Controller (Vault Agent Injector). When a Pod with specific annotations is scheduled, the webhook injects:
1. An **`initContainer`** (`vault-agent-init`) that blocks app startup until secrets are initially fetched and rendered.
2. A **`vault-agent`** sidecar container that manages ongoing token lifecycle, secret renewal, and templating.

```text
+------------------------------------------------------------------------------------+
| OpenShift Worker Node - Pod (`spring-boot-vault-sidecar`)                          |
|                                                                                    |
|  +------------------------------------------------------------------------------+  |
|  | Init Container: `vault-agent-init` (Fetches initial secrets before app boot) |  |
|  +--------------------------------------+---------------------------------------+  |
|                                         | Writes                                   |
|                                         v                                          |
|  +------------------------------------------------------------------------------+  |
|  | Shared Ephemeral Volume: `emptyDir` (Mounted at `/vault/secrets`)             |  |
|  | - File: `application-vault.properties`                                       |  |
|  +-------------------+----------------------------------+-----------------------+  |
|                      ^                                  |                          |
|      Maintains &     |                                  | Reads on boot            |
|      Renews Leases   |                                  v                          |
|  +-------------------+--------------+      +------------+-----------------------+  |
|  | Sidecar: `vault-agent`           |      | Container: `spring-boot-app`       |  |
|  | - Auth: ServiceAccount Token     |      | - JVM executes                     |  |
|  | - Consul Template Engine         |      | - `SPRING_CONFIG_ADDITIONAL_`      |  |
|  | - Token Lifecycle Renewal        |      |   `LOCATION=file:/vault/secrets/..`|  |
|  +-------------------+--------------+      +------------------------------------+  |
+----------------------|-------------------------------------------------------------+
                       |
                       | 1. POST /v1/auth/kubernetes/login (JWT SA Token)
                       | 2. GET  /v1/secret/data/my-app/config
                       v
        +------------------------------+
        |    HashiCorp Vault Server    |
        +------------------------------+
```

### 2.2 OpenShift RBAC Setup

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: spring-boot-sa
  namespace: my-app-namespace
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: spring-boot-vault-auth-binding
  namespace: my-app-namespace
subjects:
  - kind: ServiceAccount
    name: spring-boot-sa
    namespace: my-app-namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
```

### 2.3 OpenShift Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-sidecar-app
  namespace: my-app-namespace
  labels:
    app.kubernetes.io/name: spring-boot-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: spring-boot-app
  template:
    metadata:
      labels:
        app: spring-boot-app
      annotations:
        # 1. Enable Sidecar Injection
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "spring-boot-role"
        vault.hashicorp.com/agent-pre-populate-only: "false"
        
        # 2. Configure Vault Server Endpoint & Auth Path
        vault.hashicorp.com/agent-inject-status: "update"
        vault.hashicorp.com/auth-path: "auth/kubernetes"
        
        # 3. Define Secret Path and Target File
        vault.hashicorp.com/agent-inject-secret-application-vault.properties: "secret/data/my-app/config"
        
        # 4. Consul Template definition
        vault.hashicorp.com/agent-inject-template-application-vault.properties: |
          {{- with secret "secret/data/my-app/config" -}}
          spring.datasource.url=jdbc:postgresql://{{ .Data.data.db_host }}:5432/{{ .Data.data.db_name }}
          spring.datasource.username={{ .Data.data.db_username }}
          spring.datasource.password={{ .Data.data.db_password }}
          app.security.jwt-secret={{ .Data.data.jwt_secret }}
          app.payment.api-key={{ .Data.data.api_key }}
          {{- end -}}
    spec:
      serviceAccountName: spring-boot-sa
      containers:
        - name: spring-boot-app
          image: image-registry.openshift-image-stream.local/my-app-namespace/spring-boot-app:latest
          env:
            - name: SPRING_CONFIG_ADDITIONAL_LOCATION
              value: "file:/vault/secrets/application-vault.properties"
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
```

### 2.4 Spring Boot Application Configuration
- No dependencies are added to `pom.xml`.
- Standard Spring Boot property bindings (e.g. `@Value("${app.payment.api-key}")` or `@ConfigurationProperties`) consume the properties automatically.

---

## 3. Option 2: Static Secret Sync (Vault Secrets Operator / External Secrets)

### 3.1 Architectural Flow

The **Vault Secrets Operator (VSO)** or **External Secrets Operator (ESO)** runs as an OpenShift controller. It synchronizes Vault secrets into native OpenShift `Secret` objects.

```text
+-------------------------------------------------------------------------------+
| OpenShift Control Plane & Namespace                                           |
|                                                                               |
|  +------------------------------------+                                       |
|  | Vault Secrets Operator (Controller)|                                       |
|  +-----------------+------------------+                                       |
|                    | 1. Reconcile CRD & Pull Secrets                           |
|                    v                                                          |
|  +------------------------------------+                                       |
|  | OpenShift Native `v1/Secret`       |                                       |
|  | Name: `spring-boot-app-secret`     |                                       |
|  +-----------------+------------------+                                       |
|                    |                                                          |
|                    | 2. Injected via `envFrom` or `volumeMount`               |
|                    v                                                          |
|  +------------------------------------+                                       |
|  | Pod: `spring-boot-app`             |                                       |
|  | - Consumes standard OS environment |                                       |
|  | - No sidecars, no Vault calls     |                                       |
|  +------------------------------------+                                       |
+-------------------------------------------------------------------------------+
       ^
       | Auth & Periodic Sync (e.g. 60s polling)
+------+-----------------------+
|    HashiCorp Vault Server    |
+------------------------------+
```

### 3.2 Vault Connection CRDs (Vault Secrets Operator)

#### 1. `VaultConnection` & `VaultAuth`
```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  name: default-vault-conn
  namespace: my-app-namespace
spec:
  address: https://vault.vault-system.svc.cluster.local:8200
---
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: spring-boot-vault-auth
  namespace: my-app-namespace
spec:
  vaultConnectionRef: default-vault-conn
  method: kubernetes
  mount: kubernetes
  kubernetes:
    role: spring-boot-role
    serviceAccount: spring-boot-sa
```

#### 2. `VaultStaticSecret` CRD
```yaml
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: spring-boot-vault-sync
  namespace: my-app-namespace
spec:
  vaultAuthRef: spring-boot-vault-auth
  mount: secret
  type: kv-v2
  path: my-app/config
  destination:
    name: spring-boot-app-secret
    create: true
  refreshAfter: 60s
```

### 3.3 OpenShift Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-static-sync
  namespace: my-app-namespace
spec:
  replicas: 2
  selector:
    matchLabels:
      app: spring-boot-app
  template:
    metadata:
      labels:
        app: spring-boot-app
    spec:
      serviceAccountName: spring-boot-sa
      containers:
        - name: spring-boot-app
          image: image-registry.openshift-image-stream.local/my-app-namespace/spring-boot-app:latest
          envFrom:
            - secretRef:
                name: spring-boot-app-secret
          ports:
            - containerPort: 8080
```

### 3.4 Spring Boot `application.yml`
```yaml
spring:
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:5432/${DB_NAME:appdb}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

app:
  payment:
    api-key: ${API_KEY}
```

---

## 4. Option 3: Direct Spring Boot Integration (Spring Cloud Vault)

### 4.1 Architectural Flow

The Spring Boot application communicates directly with the Vault REST API using `spring-cloud-starter-vault-config`. It mounts the OpenShift ServiceAccount JWT token, exchanges it for a Vault client token, and loads configuration into the Spring `Environment` in-memory.

```text
+---------------------------------------------------------------------------------------+
| OpenShift Pod                                                                         |
|                                                                               |
|  +---------------------------------------------------------------------------------+  |
|  | Spring Boot JVM Process                                                         |  |
|  |                                                                                 |  |
|  |  +----------------------------------+     +----------------------------------+  |  |
|  |  | Spring Cloud Vault Bootstrap     | --> | In-Memory `ConfigurableEnvironment`|  |
|  |  +-----------------+----------------+     +----------------+-----------------+  |  |
|  |                    |                                       |                    |  |
|  |                    | 1. Read JWT Token                     v                    |  |
|  |                    v                         [@Value / @ConfigurationProperties]|  |
|  |  [/var/run/secrets/kubernetes.io/serviceaccount/token]                          |  |
|  +--------------------+------------------------------------------------------------+  |
+-----------------------|---------------------------------------------------------------+
                        |
                        | 2. POST /v1/auth/kubernetes/login (JWT)
                        | 3. GET  /v1/secret/data/my-app
                        | 4. GET  /v1/database/creds/app-db-role (Dynamic Creds)
                        v
         +------------------------------+
         |    HashiCorp Vault Server    |
         +------------------------------+
```

### 4.2 Spring Boot Maven Dependencies (`pom.xml`)

```xml
<properties>
    <java.version>17</java.version>
    <spring-cloud.version>2023.0.3</spring-cloud.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>${spring-cloud.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <!-- Spring Cloud Starter Vault -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-vault-config</artifactId>
    </dependency>

    <!-- Spring Boot Actuator for Refresh Endpoints -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>
</dependencies>
```

### 4.3 Spring Boot Configuration (`src/main/resources/application.yml`)

```yaml
spring:
  application:
    name: my-app
  config:
    import: "vault://"
  cloud:
    vault:
      uri: https://vault.vault-system.svc.cluster.local:8200
      connection-timeout: 5000
      read-timeout: 15000
      kv:
        enabled: true
        backend: secret
        default-context: my-app/config
      authentication: KUBERNETES
      kubernetes:
        role: spring-boot-role
        kubernetes-path: kubernetes
        service-account-token-file: /var/run/secrets/kubernetes.io/serviceaccount/token
      # Dynamic Database Secrets Configuration (Optional)
      database:
        enabled: true
        role: app-db-role
        backend: database

management:
  endpoints:
    web:
      exposure:
        include: "health,info,refresh"
```

### 4.4 OpenShift Deployment Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-direct-vault
  namespace: my-app-namespace
spec:
  replicas: 2
  selector:
    matchLabels:
      app: spring-boot-app
  template:
    metadata:
      labels:
        app: spring-boot-app
    spec:
      serviceAccountName: spring-boot-sa
      containers:
        - name: spring-boot-app
          image: image-registry.openshift-image-stream.local/my-app-namespace/spring-boot-app:latest
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "openshift"
```

---

## 5. Security & Operational Decision Matrix

```
                             +-------------------------------+
                             | Which approach should I pick? |
                             +---------------+---------------+
                                             |
                  +--------------------------+--------------------------+
                  |                                                     |
         [Zero Code Changes]                                    [Deep Spring Hook]
                  |                                                     |
        +---------+---------+                                           v
        |                   |                                 Option 3: Spring Cloud
   [Ephemeral]        [GitOps Native]                                 Vault
        |                   |                             (Dynamic DB, in-memory only)
        v                   v
 Option 1: Sidecar    Option 2: Static Secret
 (Agent Injector)     (VSO / Operator)
```

1. **Choose Option 1 (Vault Agent Sidecar)** when:
   - Migrating existing applications with zero code refactoring.
   - Applications must never persist credentials to OpenShift `etcd`.
   - You need sidecar templating to produce custom configuration file formats (e.g., XML, YAML, JKS truststores).

2. **Choose Option 2 (Static Secret Sync - VSO/ESO)** when:
   - Operating in standard GitOps environments where infrastructure teams manage Secrets as native Kubernetes objects.
   - Running lightweight microservices where extra sidecar memory overhead is prohibited.

3. **Choose Option 3 (Direct Spring Cloud Vault)** when:
   - Leveraging Vault's dynamic database secret engines (auto-generating short-lived PostgreSQL/Oracle users).
   - Dynamic secret renewal and runtime `@RefreshScope` reloads are needed without pod restarts.
   - Running in strict compliance environments requiring zero secrets written to disk or etcd.
