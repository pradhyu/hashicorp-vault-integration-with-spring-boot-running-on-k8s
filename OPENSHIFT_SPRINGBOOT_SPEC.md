# 🚀 HashiCorp Vault & Spring Boot Integration on Red Hat OpenShift

Comprehensive architecture, lifecycle mechanics, deployment manifests, application configurations, hot-reloading strategies, and TLS integration patterns between **HashiCorp Vault** and **Spring Boot** on **Red Hat OpenShift**.

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
| **Code Changes** | **None** (Optional Actuator for hot-reload) | **None** | **Requires Dependencies & Config** |
| **OpenShift Prerequisite** | Vault Agent Injector Webhook | Vault Secrets Operator (VSO) | Direct network route to Vault |
| **Secret Storage Location** | In-memory `emptyDir` in Pod | OpenShift `etcd` (v1/Secret) | JVM Heap Memory only (Zero-Disk) |
| **Dynamic Secrets / DB Leases** | Supported (Agent renews leases) | Limited / Best for Static KV | **Native & Full Dynamic Lifecycle** |
| **Properties Hot-Reload** | Via Sidecar hook + `@RefreshScope` | Pod restart or Config watcher | `@RefreshScope` / Actuator / Event |
| **TLS Hot-Reload (SSL)** | Native via Spring 3.2+ `SslBundle` | Secret volume reload | Programmatic KeyStore reload |
| **Local Dev Parity** | Requires Mock/Sidecar | Standard K8s/Env profile | Easy local test via profile/token |
| **Security Context (SCC)** | Ephemeral volume mount | Standard OpenShift Secret mount | Standard Pod execution |

---

## 2. Option 1: Vault Agent Sidecar Injection

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

### 2.2 Application Properties Hot-Reloading Mechanics

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
| **TLS / HTTPS Certificates** |  **YES** | Spring Boot 3.2+ `SslBundle` auto-reloads SSL Context natively. |
| **Database Credentials (`HikariCP`)** | ⚠️ **Complex** | Hikari maintains open connection pools. Requires DataSource reset or Rolling Pod Restart. |
| **JPA / Hibernate Metadata** | ❌ **NO** | Rebuilding SessionFactory causes memory leaks; requires Pod Restart. |
| **Server Port (`server.port`)** | ❌ **NO** | Cannot re-bind active socket; requires Pod Restart. |

> **Best Practice for Non-Reloadable Properties**: If rotating database credentials or infrastructure properties, configure Vault Agent to trigger a graceful pod shutdown (`vault.hashicorp.com/agent-inject-command: "pkill -SIGTERM java"`). OpenShift will perform a zero-downtime rolling update.

---

### 2.3 TLS / Certificate Integration (Spring Boot 3.2+ SSL Bundles)

Vault's **PKI Secrets Engine** generates short-lived X.509 certificates. Vault Agent renders them as PEM files (`tls.crt`, `tls.key`, `ca.crt`) onto `/vault/secrets/`.

Spring Boot 3.2+ natively supports **hot-reloading PEM certificates without restarting the JVM or dropping TCP connections**:

```text
+------------------------------------------------------------------------------------+
| OpenShift Pod                                                                      |
|                                                                                    |
|  +------------------------+                     +-------------------------------+  |
|  |  vault-agent (Sidecar) |                     |  Spring Boot 3.2+ (JVM)       |  |
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

### 2.4 Complete OpenShift Deployment Manifest (Sidecar + Secrets + TLS + Hot-Reload)

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

### 2.5 Spring Boot 4 Application Setup (Java Code & `application.yml`)

Spring Boot 4 baselines on **Java 21+**, **Jakarta EE 11+**, and **Virtual Threads**.

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
        <!-- Web & Actuator -->
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

#### `src/main/resources/application.yml` (Spring Boot 4):
```yaml
server:
  port: 8443
  ssl:
    bundle: "vault-ssl-bundle"

spring:
  threads:
    virtual:
      enabled: true               # Spring Boot 4 Virtual Threads (Project Loom)
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

### 2.6 Consul Template Patterns: Dynamic Key-Value Loops, JSON & YAML

Instead of hardcoding each secret key in the OpenShift manifest, Consul Template allows dynamic iteration and multi-format serialization:

#### 1. Dynamic Key-Value Iteration (`.properties`)
Automatically extracts all keys and values from Vault without listing individual key names:
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
Uses Consul Template's built-in `toJSONPretty` filter:
```yaml
vault.hashicorp.com/agent-inject-secret-application-vault.json: "secret/data/my-app/config"
vault.hashicorp.com/agent-inject-template-application-vault.json: |
  {{- with secret "secret/data/my-app/config" -}}
  {{ .Data.data | toJSONPretty }}
  {{- end -}}
```
*Spring Boot Location:* `SPRING_CONFIG_ADDITIONAL_LOCATION=file:/vault/secrets/application-vault.json`

#### 3. Native YAML Format (`.yml`)
Uses Consul Template's built-in `toYAML` filter:
```yaml
vault.hashicorp.com/agent-inject-secret-application-vault.yml: "secret/data/my-app/config"
vault.hashicorp.com/agent-inject-template-application-vault.yml: |
  {{- with secret "secret/data/my-app/config" -}}
  {{ .Data.data | toYAML }}
  {{- end -}}
```
*Spring Boot Location:* `SPRING_CONFIG_ADDITIONAL_LOCATION=file:/vault/secrets/application-vault.yml`

---

### 2.7 Where and How to Create the Secret in Vault for Spring Boot Properties

To ensure the rendered properties work seamlessly with Spring Boot 4's property binding (`@Value`, `@ConfigurationProperties`, and auto-configured `DataSource`), **name the keys in Vault using standard Spring Boot dot-notation**.

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

#### 1. Vault Secret Key-Value Schema

| Key in Vault (Dot Notation) | Example Value | Target Spring Boot Usage |
| :--- | :--- | :--- |
| `spring.datasource.url` | `jdbc:postgresql://postgres.my-app-namespace.svc:5432/appdb` | Auto-configured `HikariDataSource` URL |
| `spring.datasource.username` | `app_user` | Auto-configured DB Username |
| `spring.datasource.password` | `P@ssw0rd123!` | Auto-configured DB Password |
| `app.payment.api-key` | `sk_live_9988776655` | Injected into `@Value("${app.payment.api-key}")` |
| `app.features.enable-discounts` | `true` | Injected into `@Value("${app.features.enable-discounts}")` |
| `app.security.jwt-secret` | `4a8f9c1b7e2d0...` | Injected into JWT Token Validator Bean |

---

#### 2. Creating via Vault CLI

Run the following command in your terminal (or OpenShift Vault CLI pod):

```bash
# Set Vault Address and Token
export VAULT_ADDR="https://vault.vault-system.svc.cluster.local:8200"
export VAULT_TOKEN="<your-vault-token>"

# Create or Update the KV v2 secret with standard Spring Boot property keys
vault kv put secret/my-app/config \
  spring.datasource.url="jdbc:postgresql://postgres.my-app-namespace.svc:5432/appdb" \
  spring.datasource.username="app_user" \
  spring.datasource.password="P@ssw0rd123!" \
  app.payment.api-key="sk_live_9988776655" \
  app.features.enable-discounts="true" \
  app.security.jwt-secret="4a8f9c1b7e2d0a3f5e8b9c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1"
```

To verify the secret was created properly:
```bash
vault kv get secret/my-app/config
```

---

#### 3. Creating via Vault Web UI

1. Open your Vault Web UI (e.g. `https://vault-ui.apps.my-openshift-cluster.com`).
2. Navigate to **Secrets Engines** $\rightarrow$ click on `secret/` (KV v2).
3. Click **Create secret**.
4. Set **Path for this secret**: `my-app/config`.
5. Enter the Key-Value pairs:
   * **Key**: `spring.datasource.username` &nbsp;|&nbsp; **Value**: `app_user`
   * **Key**: `spring.datasource.password` &nbsp;|&nbsp; **Value**: `P@ssw0rd123!`
   * **Key**: `app.payment.api-key` &nbsp;|&nbsp; **Value**: `sk_live_9988776655`
   * **Key**: `app.features.enable-discounts` &nbsp;|&nbsp; **Value**: `true`
6. Click **Save**.

---

#### 4. Creating via Vault HTTP REST API (`curl`)

If managing secrets programmatically via CI/CD pipelines or scripts without installing the Vault CLI:

```bash
# Write / Create Secret via HTTP API (KV v2)
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

To read and verify the secret via REST API:
```bash
curl --silent --header "X-Vault-Token: ${VAULT_TOKEN}" \
  "https://vault.vault-system.svc.cluster.local:8200/v1/secret/data/my-app/config" | jq .data.data
```

---

#### 5. Patching / Updating an Existing Key (CLI & REST API)

When you only need to rotate a single password or API key without overwriting all other properties:

**Via Vault CLI:**
```bash
# Update just the database password and API key
vault kv patch secret/my-app/config \
  spring.datasource.password="NewSuperSecretPass456!" \
  app.payment.api-key="sk_live_new_key_112233"
```

**Via HTTP REST API:**
```bash
# Patch specific keys via HTTP PATCH
curl --silent --location --request PATCH "https://vault.vault-system.svc.cluster.local:8200/v1/secret/data/my-app/config" \
  --header "X-Vault-Token: ${VAULT_TOKEN}" \
  --header "Content-Type: application/merge-patch+json" \
  --data '{
    "data": {
      "spring.datasource.password": "NewSuperSecretPass456!",
      "app.payment.api-key": "sk_live_new_key_112233"
    }
  }'
```

---

#### 5. How Spring Boot Automatically Consumes the Rendered Properties

When Vault Agent executes the dynamic template:
```yaml
{{- with secret "secret/data/my-app/config" -}}
{{- range $key, $value := .Data.data }}
{{ $key }}={{ $value }}
{{- end }}
{{- end -}}
```

It renders `/vault/secrets/application-vault.properties`:
```properties
app.features.enable-discounts=true
app.payment.api-key=sk_live_9988776655
app.security.jwt-secret=4a8f9c1b7e2d0a3f5e8b9c1d2e3f4a5b
spring.datasource.password=P@ssw0rd123!
spring.datasource.url=jdbc:postgresql://postgres.my-app-namespace.svc:5432/appdb
spring.datasource.username=app_user
```

Because `SPRING_CONFIG_ADDITIONAL_LOCATION=file:/vault/secrets/application-vault.properties` is set on the container:
1. Spring Boot's auto-configuration binds `spring.datasource.*` directly into HikariCP to establish database connections.
2. `@Value("${app.payment.api-key}")` and `@RefreshScope` controllers immediately receive their injected strings.

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

---

## 4. Option 3: Direct Spring Boot Integration (Spring Cloud Vault)

### 4.1 Architectural Flow

The Spring Boot application communicates directly with the Vault REST API using `spring-cloud-starter-vault-config`. It mounts the OpenShift ServiceAccount JWT token, exchanges it for a Vault client token, and loads configuration into the Spring `Environment` in-memory.

```text
+---------------------------------------------------------------------------------------+
| OpenShift Pod                                                                         |
|                                                                                       |
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

### 4.2 Spring Boot 4 Maven Dependencies (`pom.xml`)

```xml
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
    <!-- Spring Cloud Starter Vault Config -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-vault-config</artifactId>
    </dependency>

    <!-- Spring Boot 4 Actuator for Refresh Endpoints -->
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
      # Dynamic Database Secrets Configuration
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
        |                   |                                 Option 3: Spring Cloud
   [Ephemeral]        [GitOps Native]                                 Vault
        |                   |                             (Dynamic DB, in-memory only)
        v                   v
 Option 1: Sidecar    Option 2: Static Secret
 (Agent Injector)     (VSO / Operator)
```

1. **Choose Option 1 (Vault Agent Sidecar)** when:
   - Migrating existing applications with zero code refactoring.
   - Applications require short-lived TLS X.509 certificates rendered to disk.
   - You want hot-reloading of properties via Actuator and hot-reloading of TLS via Spring 3.2+ `SslBundle`.
   - Credentials must never be stored in OpenShift `etcd`.

2. **Choose Option 2 (Static Secret Sync - VSO/ESO)** when:
   - Operating in standard GitOps environments (e.g., ArgoCD) where platform engineers manage Secrets as native OpenShift objects.
   - Running lightweight microservices where extra sidecar memory overhead is prohibited.

3. **Choose Option 3 (Direct Spring Cloud Vault)** when:
   - Leveraging Vault's dynamic database secret engines (auto-generating short-lived PostgreSQL/Oracle users).
   - Dynamic secret renewal and runtime `@RefreshScope` reloads are needed without pod restarts.
   - Running in strict compliance environments requiring zero secrets written to disk or etcd.
