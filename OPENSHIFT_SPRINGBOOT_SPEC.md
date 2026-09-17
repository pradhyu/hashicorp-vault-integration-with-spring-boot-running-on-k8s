# 🚀 HashiCorp Vault & Spring Boot 4 Integration on Red Hat OpenShift

A complete architectural specification, deployment manifest reference, and implementation guide for three distinct HashiCorp Vault integration patterns on **Red Hat OpenShift** with **Spring Boot 4 & Java 21+**.

---

## 1. Executive Summary & Architecture Matrix

```
+-----------------------------------------------------------------------------------------------------+
|                                THREE INTEGRATION PATTERNS OVERVIEW                                  |
+-----------------------------------+--------------------------------+--------------------------------+
|  SECTION 2: VAULT AGENT SIDECAR   | SECTION 3: STATIC SECRET SYNC  | SECTION 4: SPRING CLOUD VAULT  |
|  (Mutating Admission Webhook)     | (Vault Secrets Operator / ESO) | (Direct In-JVM REST API SDK)   |
+-----------------------------------+--------------------------------+--------------------------------+
| - Pod Mutation via Annotations    | - Kubernetes Operator Sync     | - Zero disk / In-memory only   |
| - Secret rendered to emptyDir     | - Stored in OpenShift Secret   | - Native Spring ConfigData API |
| - Zero Spring Boot code changes   | - Zero Spring Boot code changes| - Dynamic DB user leasing      |
| - Sidecar handles token lifecycle | - Operator handles lifecycle   | - JVM handles token lifecycle  |
+-----------------------------------+--------------------------------+--------------------------------+
```

### Feature & Capability Comparison Matrix

| Feature | Pattern 1: Vault Agent Sidecar (Sec. 2) | Pattern 2: Static Secret Sync (Sec. 3) | Pattern 3: Spring Cloud Vault (Sec. 4) |
| :--- | :--- | :--- | :--- |
| **Code Changes** | **None** (Optional Actuator for hot-reload) | **None** | **Requires Dependencies & Config** |
| **OpenShift Prerequisite** | Vault Agent Injector Webhook | Vault Secrets Operator (VSO) | Direct network route to Vault Server |
| **Secret Storage Location** | In-memory `emptyDir` in Pod | OpenShift `etcd` (`v1/Secret`) | JVM Heap Memory only (Zero-Disk) |
| **Dynamic Secrets / DB Leases** | Supported (Agent renews leases) | Limited / Best for Static KV | **Native & Full Dynamic Lifecycle** |
| **Properties Hot-Reload** | Via Sidecar hook + `@RefreshScope` | Pod restart or Config watcher | `@RefreshScope` / Actuator / Event |
| **TLS Hot-Reload (SSL)** | Native via Spring 4 `SslBundle` | Secret volume reload | Programmatic KeyStore reload |
| **Local Dev Parity** | Requires Mock/Sidecar | Standard K8s/Env profile | Easy local test via profile/token |
| **Security Context (SCC)** | Ephemeral volume mount | Standard OpenShift Secret mount | Standard Pod execution |

---

## 2. Pattern 1: Vault Agent Sidecar (Mutating Webhook)

### 2.1 Pod Lifecycle: Init Container vs. Sidecar Container

When Vault Agent injection is enabled via OpenShift annotations, the **Vault Mutating Webhook** injects two distinct containers into your Pod:

```text
Pod Lifecycle Timeline
------------------------------------------------------------------------------------>
[ 1. Pod Scheduled on OpenShift Node ] 
        |
        v
[ 2. Init Container: vault-agent-init ]  --> Authenticates with Vault (ServiceAccount JWT)
        |                                --> Fetches initial secrets & TLS certificates
        |                                --> Renders files to shared emptyDir (/vault/secrets)
        |                                --> Exits with code 0 (Terminates successfully)
        v
[ 3. Main Containers Start Simultaneously ]
        |
        +---> [ App Container: spring-boot-app ]  --> Reads /vault/secrets on JVM boot
        |                                         --> Starts Tomcat/Netty & binds ports
        |
        +---> [ Sidecar: vault-agent ]            --> Runs continuously in background
                                                  --> Renews Vault tokens before TTL expires
                                                  --> Renews dynamic secret leases
                                                  --> Re-renders files when secrets rotate
                                                  --> Executes reload hooks (/actuator/refresh)
```

#### Detailed Container Responsibilities:

| Container | Type | Lifecycle | Key Responsibilities |
| :--- | :--- | :--- | :--- |
| **`vault-agent-init`** | `initContainer` | Runs once before app startup, then terminates (`exit 0`). | 1. Reads OpenShift SA token at `/var/run/secrets/kubernetes.io/serviceaccount/token`.<br>2. Authenticates to Vault (`POST /v1/auth/kubernetes/login`).<br>3. Fetches initial properties & TLS certificates.<br>4. Renders files into shared `emptyDir`.<br>5. **Fails the Pod fast** if Vault is unreachable or permissions are invalid. |
| **`vault-agent`** | `sidecar` | Runs continuously in parallel with Spring Boot container. | 1. **Token Lifecycle**: Automatically calls `/v1/auth/token/renew-self` before token expiry.<br>2. **Lease Management**: Keeps dynamic database/secret leases alive via `/v1/sys/leases/renew`.<br>3. **Secret Watching**: Polls Vault for KV changes or PKI renewals.<br>4. **Atomic Re-rendering**: Overwrites `/vault/secrets/*` files.<br>5. **Trigger Hooks**: Runs commands (e.g., `curl -X POST http://localhost:8080/actuator/refresh` or `pkill -SIGTERM java`). |

---

### 2.2 Authentication Mechanics: The 3-Way Kubernetes Auth Handshake

Neither the application nor the sidecar uses static passwords or pre-shared tokens to authenticate with Vault. Instead, authentication uses **OpenShift ServiceAccount Identity (`auth/kubernetes`)** via a 3-way cryptographic validation handshake:

```text
+-----------------------------------------------------------------------------------------------------+
|                                 3-WAY KUBERNETES AUTHENTICATION HANDSHAKE                            |
+-----------------------------------------------------------------------------------------------------+

  [ OpenShift Worker Node ]                  [ Vault Server ]               [ OpenShift Control Plane ]
     Pod: spring-boot-app                      (HashiCorp)                       (Kubernetes API)
  (with ServiceAccount Token)
              |                                     |                                     |
              | 1. Read projected JWT from:         |                                     |
              |    /var/run/secrets/.../token       |                                     |
              |                                     |                                     |
              | 2. POST /v1/auth/kubernetes/login   |                                     |
              |    { "role": "spring-boot-role",    |                                     |
              |      "jwt":  "<sa-jwt-token>" }     |                                     |
              |------------------------------------>|                                     |
              |                                     | 3. POST /apis/authentication.k8s.io/|
              |                                     |    v1/tokenreviews                  |
              |                                     |    (Is this JWT valid for           |
              |                                     |     sa:spring-boot-sa in my-app-ns?)|
              |                                     |------------------------------------>|
              |                                     |                                     |
              |                                     | 4. TokenReview Response:            |
              |                                     |    { "authenticated": true,         |
              |                                     |      "user": { "username": ... } }  |
              |                                     |<------------------------------------|
              |                                     |                                     |
              |                                     | 5. Verify against Role:             |
              |                                     |    - SA matches "spring-boot-sa"?   |
              |                                     |    - NS matches "my-app-namespace"? |
              |                                     |    - Generate Vault Client Token    |
              |                                     |                                     |
              | 6. HTTP 200 OK                      |                                     |
              |    { "client_token": "hvs.CAE...",  |                                     |
              |      "lease_duration": 3600 }       |                                     |
              |<------------------------------------|                                     |
              |                                     |                                     |
              | 7. GET /v1/secret/data/my-app/config|                                     |
              |    Header: X-Vault-Token: hvs.CAE...|                                     |
              |------------------------------------>|                                     |
              |                                     |                                     |
              | 8. Returns Secret Payload           |                                     |
              |<------------------------------------|                                     |
```

---

### 2.3 ServiceAccount Security Gates & OpenShift RBAC Requirements

#### 1. Can ANY ServiceAccount in OpenShift Connect to Vault?
**NO.** Vault enforces a strict **4-gate security model**:

| Security Gate | What Vault Checks | Failure Scenario |
| :--- | :--- | :--- |
| **Gate 1: Bound Namespaces** | `bound_service_account_namespaces=["my-app-namespace"]` | A ServiceAccount created in `hacker-namespace` is rejected. |
| **Gate 2: Bound SA Names** | `bound_service_account_names=["spring-boot-sa"]` | A different SA (e.g. `default` or `builder`) in `my-app-namespace` is rejected. |
| **Gate 3: OpenShift Signature** | Cryptographic verification via OpenShift `TokenReview` API | Forged, expired, or tampered JWT tokens fail immediately. |
| **Gate 4: ACL Policy Boundary** | Attached policy (`path "secret/data/my-app/*" { capabilities = ["read"] }`) | Even after login, the token cannot access secrets belonging to other apps. |

#### 2. OpenShift RBAC Roles Required
- **Application SA (`spring-boot-sa`)**: **Zero special RBAC / ClusterRoles needed.** Standard unprivileged pod execution rights.
- **Vault Reviewer SA (`vault-auth-sa`)**: Bound to ClusterRole **`system:auth-delegator`** so Vault can query OpenShift's `TokenReview` API.

```yaml
# In Vault Namespace:
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: vault-token-reviewer-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: vault-auth-sa
    namespace: vault-system
```

---

### 2.4 Application Properties Hot-Reloading Mechanics

#### The Problem with Default Spring Boot
Standard Spring Boot loads properties from `SPRING_CONFIG_ADDITIONAL_LOCATION` **only once during startup**. When the sidecar overwrites `/vault/secrets/application-vault.properties`, the disk changes but the JVM memory remains unchanged.

#### The Solution: Sidecar Triggered `/actuator/refresh` + `@RefreshScope`

```text
+-----------------------------------------------------------------------------------------+
| OpenShift Pod                                                                           |
|                                                                                         |
| 1. Vault Agent detects secret change in Vault                                           |
| 2. Vault Agent overwrites /vault/secrets/application-vault.properties                   |
| 3. Vault Agent executes `command`: curl -X POST http://localhost:8080/actuator/refresh   |
|                                        |                                                |
|                                        v                                                |
| 4. Spring Boot Actuator receives `/actuator/refresh`                                    |
| 5. Spring re-reads `/vault/secrets/application-vault.properties` into `Environment`     |
| 6. Spring re-instantiates all beans annotated with `@RefreshScope`                      |
| 7. Next request receives NEW property values with ZERO downtime!                        |
+-----------------------------------------------------------------------------------------+
```

#### Reloadability Matrix: What Can vs Cannot be Hot-Reloaded

| Property Category | Hot-Reloadable via `@RefreshScope`? | Behavior & Best Practice |
| :--- | :---: | :--- |
| **API Keys, Secrets, Feature Flags** |  **YES** | Instantly re-injected into `@RefreshScope` controllers/services. |
| **Business Logic Configs & URLs** |  **YES** | Reloaded in-memory without dropping requests. |
| **TLS / HTTPS Certificates** |  **YES** | Spring Boot 4 `SslBundle` auto-reloads SSL Context natively. |
| **Database Credentials (`HikariCP`)** | ⚠️ **Complex** | Hikari maintains open connection pools. Requires DataSource reset or Rolling Pod Restart. |
| **JPA / Hibernate Metadata** | ❌ **NO** | Rebuilding SessionFactory causes memory leaks; requires Pod Restart. |
| **Server Port (`server.port`)** | ❌ **NO** | Cannot re-bind active socket; requires Pod Restart. |

---

### 2.5 TLS / Certificate Integration (Spring Boot 4 SSL Bundles)

Vault's **PKI Secrets Engine** generates short-lived X.509 certificates. Vault Agent renders them as PEM files (`tls.crt`, `tls.key`, `ca.crt`) onto `/vault/secrets/`.

Spring Boot 4 natively supports **hot-reloading PEM certificates without restarting the JVM or dropping TCP connections**:

```text
+------------------------------------------------------------------------------------+
| OpenShift Pod                                                                      |
|                                                                                    |
|  +------------------------+                     +-------------------------------+  |
|  |  vault-agent (Sidecar) |                     |  Spring Boot 4 (JVM)          |  |
|  +-----------+------------+                     +---------------+---------------+  |
|              |                                                  ^                  |
|              | Renders PEM certificates                         | SslBundle        |
|              v                                                  | File Watcher     |
|  +--------------------------------------------------------------+---------------+  |
|  | Shared Volume (/vault/secrets/)                                              |  |
|  | ├── tls.crt          (Server X.509 Certificate)                              |  |
|  | ├── tls.key          (RSA / ECDSA Private Key)                               |  |
|  | ├── ca.crt           (Issuing CA Trust Chain)                                |  |
|  | └── application-vault.properties                                             |  |
|  +------------------------------------------------------------------------------+  |
+------------------------------------------------------------------------------------+
```

---

### 2.6 Complete OpenShift Deployment Manifest (Sidecar Pattern)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-boot-vault-sidecar
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
        # 1. Enable Vault Agent Injection
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "spring-boot-role"
        vault.hashicorp.com/agent-inject-status: "update"
        vault.hashicorp.com/auth-path: "auth/kubernetes"
        
        # 2. Render Application Properties (Dynamic Key-Value Loop) & Trigger Actuator Refresh
        vault.hashicorp.com/agent-inject-secret-application-vault.properties: "secret/data/my-app/config"
        vault.hashicorp.com/agent-inject-template-application-vault.properties: |
          {{- with secret "secret/data/my-app/config" -}}
          {{- range $key, $value := .Data.data }}
          {{ $key }}={{ $value }}
          {{- end }}
          {{- end -}}
        vault.hashicorp.com/agent-inject-command-application-vault.properties: |
          curl -s -X POST http://localhost:8080/actuator/refresh || true

        # 3. Render TLS Server Certificate (PKI Engine)
        vault.hashicorp.com/agent-inject-secret-tls.crt: "pki/issue/spring-boot-role"
        vault.hashicorp.com/agent-inject-template-tls.crt: |
          {{- with secret "pki/issue/spring-boot-role" "common_name=my-app.my-app-namespace.svc" "ttl=24h" -}}
          {{ .Data.certificate }}
          {{- end -}}

        # 4. Render TLS Private Key
        vault.hashicorp.com/agent-inject-secret-tls.key: "pki/issue/spring-boot-role"
        vault.hashicorp.com/agent-inject-template-tls.key: |
          {{- with secret "pki/issue/spring-boot-role" "common_name=my-app.my-app-namespace.svc" "ttl=24h" -}}
          {{ .Data.private_key }}
          {{- end -}}

        # 5. Render CA Certificate Chain
        vault.hashicorp.com/agent-inject-secret-ca.crt: "pki/issue/spring-boot-role"
        vault.hashicorp.com/agent-inject-template-ca.crt: |
          {{- with secret "pki/issue/spring-boot-role" "common_name=my-app.my-app-namespace.svc" "ttl=24h" -}}
          {{ .Data.issuing_ca }}
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
            - containerPort: 8443
              name: https
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
```

---

### 2.7 Spring Boot 4 Application Code & Configuration

#### `pom.xml` (Spring Boot 4 & Java 21+):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>4.0.0</version>
        <relativePath/>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>spring-boot-vault-app</artifactId>
    <version>1.0.0</version>

    <properties>
        <java.version>21</java.version>
        <spring-cloud.version>2025.0.0</spring-cloud.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
        <!-- Spring Cloud Context for @RefreshScope Hot-Reloading -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-bootstrap</artifactId>
            <version>4.2.0</version>
        </dependency>
    </dependencies>
</project>
```

#### `src/main/resources/application.yml`:
```yaml
server:
  port: 8443
  ssl:
    bundle: "vault-ssl-bundle"

spring:
  threads:
    virtual:
      enabled: true               # Spring Boot 4 Virtual Threads
  ssl:
    bundle:
      pem:
        vault-ssl-bundle:
          reload-on-update: true   # Dynamic SSL Context hot-reload when sidecar rotates certs!
          keystore:
            certificate: "file:/vault/secrets/tls.crt"
            private-key: "file:/vault/secrets/tls.key"
          truststore:
            certificate: "file:/vault/secrets/ca.crt"

management:
  endpoints:
    web:
      exposure:
        include: "health,info,refresh"
```

#### `@RefreshScope` Controller (`PaymentController.java`):
```java
package com.example.app;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RefreshScope  // <-- Reloads in-memory when sidecar calls /actuator/refresh
public class PaymentController {

    @Value("${app.payment.api-key:default-key}")
    private String apiKey;

    @Value("${app.features.enable-discounts:false}")
    private boolean discountsEnabled;

    @GetMapping("/api/v1/payment/config")
    public ResponseEntity<String> getConfig() {
        return ResponseEntity.ok("Active Key: " + apiKey + ", Discounts: " + discountsEnabled);
    }
}
```

---

### 2.8 Consul Template Patterns: Dynamic Key-Value Loops, JSON & YAML

#### 1. Dynamic Key-Value Iteration (`.properties`)
```yaml
vault.hashicorp.com/agent-inject-secret-application-vault.properties: "secret/data/my-app/config"
vault.hashicorp.com/agent-inject-template-application-vault.properties: |
  {{- with secret "secret/data/my-app/config" -}}
  {{- range $key, $value := .Data.data }}
  {{ $key }}={{ $value }}
  {{- end }}
  {{- end -}}
```

#### 2. Native JSON Format (`.json`)
```yaml
vault.hashicorp.com/agent-inject-secret-application-vault.json: "secret/data/my-app/config"
vault.hashicorp.com/agent-inject-template-application-vault.json: |
  {{- with secret "secret/data/my-app/config" -}}
  {{ .Data.data | toJSONPretty }}
  {{- end -}}
```

#### 3. Native YAML Format (`.yml`)
```yaml
vault.hashicorp.com/agent-inject-secret-application-vault.yml: "secret/data/my-app/config"
vault.hashicorp.com/agent-inject-template-application-vault.yml: |
  {{- with secret "secret/data/my-app/config" -}}
  {{ .Data.data | toYAML }}
  {{- end -}}
```

---

### 2.9 Where & How to Create the Secret in Vault (CLI & REST API)

```text
+-------------------------------------------------------------------------------------------------+
|                                END-TO-END SECRET TRANSFORMATION FLOW                            |
+-------------------------------------------------------------------------------------------------+
| 1. RAW SECRET IN VAULT (`secret/data/my-app/config`):                                           |
|    "spring.datasource.username" : "app_user"                                                    |
|    "spring.datasource.password" : "P@ssw0rd123!"                                                |
|    "app.payment.api-key"        : "sk_live_9988776655"                                          |
|    "app.features.enable-discounts" : "true"                                                     |
+-------------------------------------------------------------------------------------------------+
                                               |
                                               | Read by `vault-agent` Sidecar
                                               v
+-------------------------------------------------------------------------------------------------+
| 2. CONSUL TEMPLATE EVALUATION IN OPENSHIFT POD:                                                 |
|    {{- with secret "secret/data/my-app/config" -}}                                              |
|    {{- range $key, $value := .Data.data }}                                                      |
|    {{ $key }}={{ $value }}                                                                      |
|    {{- end }}                                                                                   |
|    {{- end -}}                                                                                  |
+-------------------------------------------------------------------------------------------------+
                                               |
                                               | Renders file on `emptyDir` mount
                                               v
+-------------------------------------------------------------------------------------------------+
| 3. RENDERED FILE (`/vault/secrets/application-vault.properties`):                               |
|    app.features.enable-discounts=true                                                           |
|    app.payment.api-key=sk_live_9988776655                                                      |
|    spring.datasource.password=P@ssw0rd123!                                                      |
|    spring.datasource.username=app_user                                                          |
+-------------------------------------------------------------------------------------------------+
                                               |
                                               | Consumed via SPRING_CONFIG_ADDITIONAL_LOCATION
                                               v
+-------------------------------------------------------------------------------------------------+
| 4. SPRING BOOT 4 APPLICATION CONTEXT (JVM MEMORY):                                              |
|    - HikariDataSource -> connects with "app_user" / "P@ssw0rd123!"                              |
|    - @Value("${app.payment.api-key}") -> receives "sk_live_9988776655"                          |
|    - @Value("${app.features.enable-discounts}") -> receives true                                |
+-------------------------------------------------------------------------------------------------+
```

#### 1. Creating via Vault CLI
```bash
export VAULT_ADDR="https://vault.vault-system.svc.cluster.local:8200"
export VAULT_TOKEN="<your-vault-token>"

vault kv put secret/my-app/config \
  spring.datasource.url="jdbc:postgresql://postgres.my-app-namespace.svc:5432/appdb" \
  spring.datasource.username="app_user" \
  spring.datasource.password="P@ssw0rd123!" \
  app.payment.api-key="sk_live_9988776655" \
  app.features.enable-discounts="true" \
  app.security.jwt-secret="4a8f9c1b7e2d0a3f5e8b9c1d2e3f4a5b"
```

#### 2. Creating via Vault HTTP REST API (`curl`)
```bash
curl --silent --location --request POST "https://vault.vault-system.svc.cluster.local:8200/v1/secret/data/my-app/config" \
  --header "X-Vault-Token: ${VAULT_TOKEN}" \
  --header "Content-Type: application/json" \
  --data '{
    "data": {
      "spring.datasource.url": "jdbc:postgresql://postgres.my-app-namespace.svc:5432/appdb",
      "spring.datasource.username": "app_user",
      "spring.datasource.password": "P@ssw0rd123!",
      "app.payment.api-key": "sk_live_9988776655",
      "app.features.enable-discounts": "true",
      "app.security.jwt-secret": "4a8f9c1b7e2d0a3f5e8b9c1d2e3f4a5b"
    }
  }'
```

#### 3. Patching / Rotating a Single Key
```bash
# Via CLI
vault kv patch secret/my-app/config spring.datasource.password="NewSuperSecretPass456!"

# Via REST API
curl --silent --location --request PATCH "https://vault.vault-system.svc.cluster.local:8200/v1/secret/data/my-app/config" \
  --header "X-Vault-Token: ${VAULT_TOKEN}" \
  --header "Content-Type: application/merge-patch+json" \
  --data '{"data": {"spring.datasource.password": "NewSuperSecretPass456!"}}'
```

---

## 3. Pattern 2: Static Secret Sync (Vault Secrets Operator / ESO)

### 3.1 Architectural Flow

The **Vault Secrets Operator (VSO)** runs as an OpenShift controller. It synchronizes Vault secrets into native OpenShift `Secret` objects.

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
|  | - No sidecars, zero Vault calls    |                                       |
|  +------------------------------------+                                       |
+-------------------------------------------------------------------------------+
       ^
       | Auth & Periodic Sync (e.g. 60s polling)
+------+-----------------------+
|    HashiCorp Vault Server    |
+------------------------------+
```

---

### 3.2 Operator CRDs: `VaultConnection`, `VaultAuth`, `VaultStaticSecret`

```yaml
# 1. Vault Server Connection Endpoint
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultConnection
metadata:
  name: default-vault-conn
  namespace: my-app-namespace
spec:
  address: https://vault.vault-system.svc.cluster.local:8200
---
# 2. Kubernetes Authentication Binding
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
---
# 3. Secret Synchronization Definition
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

---

### 3.3 OpenShift Deployment Manifest (Consuming Native Secret)

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
      annotations:
        # Stakater Reloader triggers zero-downtime rolling restart when the Secret updates:
        secret.reloader.stakater.com/reload: "spring-boot-app-secret"
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
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
```

---

### 3.4 Spring Boot 4 Application Setup (`application.yml`)

Spring Boot's Relaxed Binding automatically converts environment variables like `SPRING_DATASOURCE_PASSWORD` or dot-notation properties:

```yaml
spring:
  threads:
    virtual:
      enabled: true
  datasource:
    url: ${SPRING_DATASOURCE_URL:jdbc:postgresql://localhost:5432/appdb}
    username: ${SPRING_DATASOURCE_USERNAME}
    password: ${SPRING_DATASOURCE_PASSWORD}

app:
  payment:
    api-key: ${APP_PAYMENT_API_KEY}
  features:
    enable-discounts: ${APP_FEATURES_ENABLE_DISCOUNTS:false}
```

---

### 3.5 Where & How to Create the Secret in Vault

```bash
# Using uppercase or standard keys for env injection
vault kv put secret/my-app/config \
  SPRING_DATASOURCE_URL="jdbc:postgresql://postgres.my-app-namespace.svc:5432/appdb" \
  SPRING_DATASOURCE_USERNAME="app_user" \
  SPRING_DATASOURCE_PASSWORD="P@ssw0rd123!" \
  APP_PAYMENT_API_KEY="sk_live_9988776655" \
  APP_FEATURES_ENABLE_DISCOUNTS="true"
```

---

## 4. Pattern 3: Direct In-App Integration (Spring Cloud Vault)

### 4.1 Architectural Flow

The Spring Boot application communicates directly with the Vault REST API using `spring-cloud-starter-vault-config`. It mounts the OpenShift ServiceAccount JWT token, exchanges it for a Vault client token, and loads configuration into the Spring `Environment` in-memory with **Zero-Disk footprint**.

```text
+---------------------------------------------------------------------------------------+
| OpenShift Pod                                                                         |
|                                                                                       |
|  +---------------------------------------------------------------------------------+  |
|  | Spring Boot 4 JVM Process (Java 21+)                                            |  |
|  |                                                                                 |  |
|  |  +----------------------------------+     +----------------------------------+  |  |
|  |  | Spring Cloud Vault ConfigData    | --> | In-Memory `ConfigurableEnvironment`|  |
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
                        | 4. GET  /v1/database/creds/app-db-role (Dynamic DB Users)
                        v
         +------------------------------+
         |    HashiCorp Vault Server    |
         +------------------------------+
```

---

### 4.2 Spring Boot 4 Maven Dependencies (`pom.xml`)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>4.0.0</version>
        <relativePath/>
    </parent>

    <groupId>com.example</groupId>
    <artifactId>spring-boot-direct-vault</artifactId>
    <version>1.0.0</version>

    <properties>
        <java.version>21</java.version>
        <spring-cloud.version>2025.0.0</spring-cloud.version>
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
        <!-- Spring Cloud Vault Starter -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-vault-config</artifactId>
        </dependency>

        <!-- Web & Actuator -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>
    </dependencies>
</project>
```

---

### 4.3 Spring Boot 4 Configuration (`src/main/resources/application.yml`)

```yaml
spring:
  application:
    name: my-app
  config:
    import: "vault://"
  threads:
    virtual:
      enabled: true
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
      # Dynamic Database Secret Engine (Ephemeral Short-Lived Database Users)
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

---

### 4.4 OpenShift Deployment Manifest (Zero Sidecars)

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
          resources:
            requests:
              cpu: "250m"
              memory: "512Mi"
            limits:
              cpu: "1000m"
              memory: "1Gi"
```

---

### 4.5 Where & How to Create the Secret in Vault

```bash
# Provision Static KV v2 properties
vault kv put secret/my-app/config \
  app.payment.api-key="sk_live_9988776655" \
  app.features.enable-discounts="true"

# Configure Dynamic Database Secret Engine Role
vault write database/roles/app-db-role \
    db_name=postgresql \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
        GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    default_ttl="1h" \
    max_ttl="24h"
```

---

## 5. Security & Operational Decision Guide

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
        |                   |                                 Pattern 3: Spring Cloud
   [Ephemeral]        [GitOps Native]                                 Vault
        |                   |                             (Dynamic DB, in-memory only)
        v                   v
 Pattern 1: Sidecar   Pattern 2: Static Secret
 (Agent Injector)     (VSO / Operator)
```

1. **Choose Pattern 1 (Vault Agent Sidecar - Section 2)** when:
   - Migrating existing applications with zero code refactoring.
   - Applications require short-lived TLS X.509 certificates rendered to disk.
   - You want hot-reloading of properties via Actuator and hot-reloading of TLS via Spring 4 `SslBundle`.
   - Credentials must never be stored in OpenShift `etcd`.

2. **Choose Pattern 2 (Static Secret Sync - Section 3)** when:
   - Operating in standard GitOps environments (e.g., ArgoCD) where platform engineers manage Secrets as native OpenShift objects.
   - Running lightweight microservices where extra sidecar memory overhead is prohibited.

3. **Choose Pattern 3 (Direct Spring Cloud Vault - Section 4)** when:
   - Leveraging Vault's dynamic database secret engines (auto-generating short-lived PostgreSQL/Oracle users).
   - Dynamic secret renewal and runtime `@RefreshScope` reloads are needed without pod restarts.
   - Running in strict compliance environments requiring zero secrets written to disk or etcd.
