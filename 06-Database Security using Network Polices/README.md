# Kubernetes Production Security Runbook: Secure Database Using NetworkPolicy

This guide explains how to secure a database running inside Kubernetes using `NetworkPolicy`.

The objective is to ensure that:

* Only authorized application Pods can connect to the database.
* Unrelated Pods cannot access the database.
* Database access is restricted to the required port.
* Traffic is restricted by Namespace and Pod labels.
* DNS continues to work when Egress policies are enabled.
* Database access follows a least-privilege model.
* NetworkPolicy changes can be tested and validated safely.

> **Important:** NetworkPolicy provides network-level access control. It does not replace database authentication, TLS, encryption, secrets management, backups, or database-level authorization.

---

# 1. Security Architecture

A recommended production architecture is:

```text
                         Kubernetes Cluster
                                |
                +---------------+---------------+
                |                               |
         Application Namespace            Database Namespace
                |                               |
         +------+-------+                       |
         |              |                       |
      Frontend        Backend                   |
                        |                       |
                        | TCP 5432              |
                        |                       |
                        v                       v
                 +-------------------------------+
                 |       Database Pod             |
                 |       PostgreSQL :5432         |
                 +-------------------------------+
                              ^
                              |
                       NetworkPolicy
                              |
                    Allow only Backend
```

Expected traffic:

```text
Frontend
   |
   | HTTPS
   v
Backend
   |
   | TCP 5432
   v
PostgreSQL
```

Unwanted traffic:

```text
Frontend --------------X--------------> PostgreSQL
Other Pod -------------X--------------> PostgreSQL
Unknown Namespace -----X--------------> PostgreSQL
Internet --------------X--------------> PostgreSQL
```

---

# 2. Recommended Security Model

Use multiple layers of protection:

```text
                Database Security
                      |
        +-------------+-------------+
        |             |             |
        v             v             v
 NetworkPolicy   DB Authentication   TLS
        |
        v
 Pod/Namespace
   Isolation
        |
        v
 Secrets Management
        |
        v
 Encryption / Backups
```

NetworkPolicy should be considered one layer of the overall security architecture.

---

# 3. Example Architecture

Assume:

```text
Application Namespace:
production

Database Namespace:
database

Application:
backend

Database:
postgres

Database Port:
5432
```

Backend Pod labels:

```yaml
labels:
  app: backend
```

PostgreSQL Pod labels:

```yaml
labels:
  app: postgres
```

---

# 4. Verify Labels Before Creating Policies

First verify the backend Pod:

```bash
kubectl get pods \
  -n production \
  --show-labels
```

Example:

```text
NAME                       LABELS
backend-7d8f8c7d9b-x2abc   app=backend
```

Verify the database Pod:

```bash
kubectl get pods \
  -n database \
  --show-labels
```

Example:

```text
NAME                       LABELS
postgres-0                 app=postgres
```

These labels are critical because NetworkPolicy uses selectors to determine which Pods can communicate.

---

# 5. Create a Default-Deny Policy for the Database Namespace

Start with a default-deny policy.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: database

spec:
  podSelector: {}

  policyTypes:
    - Ingress
    - Egress
```

Apply it:

```bash
kubectl apply -f database-default-deny.yaml
```

Verify:

```bash
kubectl get networkpolicy \
  -n database
```

This creates an isolated baseline for Pods in the `database` Namespace.

---

# 6. Allow Backend to Access PostgreSQL

Now explicitly allow only the backend application to connect to PostgreSQL.

Because the backend is in a different Namespace, use both:

* `namespaceSelector`
* `podSelector`

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-postgres
  namespace: database

spec:
  podSelector:
    matchLabels:
      app: postgres

  policyTypes:
    - Ingress

  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: production

          podSelector:
            matchLabels:
              app: backend

      ports:
        - protocol: TCP
          port: 5432
```

Apply:

```bash
kubectl apply -f allow-backend-to-postgres.yaml
```

---

# 7. Why Use Both Selectors?

This is important.

Consider:

```yaml
namespaceSelector:
  matchLabels:
    kubernetes.io/metadata.name: production

podSelector:
  matchLabels:
    app: backend
```

Together they mean:

```text
ONLY

Namespace:
production

AND

Pod:
app=backend
```

So the policy allows:

```text
production/backend
        |
        | TCP 5432
        v
database/postgres
```

It does NOT broadly allow:

```text
production/frontend
production/testing
staging/backend
development/backend
other Pods
```

This follows the principle of least privilege.

---

# 8. Important YAML Detail

Be careful with the indentation and structure.

This:

```yaml
from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: production
    podSelector:
      matchLabels:
        app: backend
```

means:

```text
Namespace = production
AND
Pod = backend
```

Whereas separate list entries can represent different allowed sources.

For example:

```yaml
from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: production

  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: monitoring
```

means traffic can be allowed from either Namespace, subject to the rest of the policy.

---

# 9. Allow Database DNS Egress

If the database Namespace has an Egress default-deny policy, the database may need access to cluster DNS.

A database may need DNS for:

* Replication endpoints
* External monitoring
* Backup services
* Other database dependencies
* Service discovery

Do not automatically allow unrestricted Egress.

Instead, explicitly allow only the required DNS traffic.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: database

spec:
  podSelector: {}

  policyTypes:
    - Egress

  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system

          podSelector:
            matchLabels:
              k8s-app: kube-dns

      ports:
        - protocol: UDP
          port: 53

        - protocol: TCP
          port: 53
```

Verify the actual DNS Pod labels in your cluster:

```bash
kubectl get pods \
  -n kube-system \
  --show-labels
```

The DNS labels can vary depending on the Kubernetes distribution.

---

# 10. Allow Database Egress Only When Required

A database should not normally have unrestricted Internet access.

Avoid:

```yaml
egress:
  - {}
```

unless there is a documented reason.

Instead, define the exact destinations required by the database.

For example:

```text
Database
   |
   +--> DNS
   |
   +--> Backup service
   |
   +--> Monitoring
   |
   +--> Replication
```

Every additional destination should have a documented business or operational requirement.

---

# 11. PostgreSQL Example

For PostgreSQL:

```text
TCP 5432
```

Database NetworkPolicy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-ingress
  namespace: database

spec:
  podSelector:
    matchLabels:
      app: postgres

  policyTypes:
    - Ingress

  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: production

          podSelector:
            matchLabels:
              app: backend

      ports:
        - protocol: TCP
          port: 5432
```

---

# 12. MySQL Example

For MySQL:

```text
TCP 3306
```

Policy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: mysql-ingress
  namespace: database

spec:
  podSelector:
    matchLabels:
      app: mysql

  policyTypes:
    - Ingress

  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: production

          podSelector:
            matchLabels:
              app: backend

      ports:
        - protocol: TCP
          port: 3306
```

---

# 13. MongoDB Example

For MongoDB:

```text
TCP 27017
```

Policy:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: mongodb-ingress
  namespace: database

spec:
  podSelector:
    matchLabels:
      app: mongodb

  policyTypes:
    - Ingress

  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: production

          podSelector:
            matchLabels:
              app: backend

      ports:
        - protocol: TCP
          port: 27017
```

---

# 14. Test Allowed Connectivity

From the backend Pod:

```bash
kubectl exec -it <backend-pod> \
  -n production \
  -- nc -vz postgres.database.svc.cluster.local 5432
```

Expected:

```text
Connection succeeded
```

For PostgreSQL, you can perform an application-level test where the PostgreSQL client is available:

```bash
kubectl exec -it <backend-pod> \
  -n production \
  -- psql \
  -h postgres.database.svc.cluster.local \
  -U <user> \
  -d <database>
```

---

# 15. Test Denied Connectivity

Testing only the allowed path is not enough.

Create or use an unrelated test Pod.

For example:

```bash
kubectl run network-test \
  -n production \
  --image=busybox:1.36 \
  --restart=Never \
  -- sleep 3600
```

Then test:

```bash
kubectl exec -it network-test \
  -n production \
  -- nc -vz postgres.database.svc.cluster.local 5432
```

Expected:

```text
Connection should fail
```

This validates that the policy is actually restricting access.

Remove the test Pod afterward:

```bash
kubectl delete pod network-test \
  -n production
```

---

# 16. Test DNS Separately

From the backend:

```bash
kubectl exec -it <backend-pod> \
  -n production \
  -- nslookup postgres.database.svc.cluster.local
```

If DNS fails:

```text
Application
    |
    v
DNS lookup
    |
    X
Egress NetworkPolicy
```

Check:

* DNS Service
* CoreDNS Pods
* DNS labels
* UDP 53
* TCP 53
* Egress policies

---

# 17. Check the Database Service

Verify:

```bash
kubectl get svc \
  -n database
```

Example:

```text
NAME       TYPE        CLUSTER-IP      PORT(S)
postgres   ClusterIP   10.100.20.10    5432/TCP
```

Describe:

```bash
kubectl describe svc postgres \
  -n database
```

Check:

```text
Selector
Port
TargetPort
Endpoints
```

---

# 18. Check EndpointSlices

Verify that the Service points to the expected database Pods:

```bash
kubectl get endpointslice \
  -n database
```

Also:

```bash
kubectl get endpoints postgres \
  -n database
```

If there are no endpoints, NetworkPolicy may not be the actual problem.

Investigate:

```text
Service selector
Pod labels
Pod readiness
Database Pod state
EndpointSlice
```

---

# 19. Secure Database from Internet Access

A database should generally not be directly exposed to the Internet.

Avoid:

```text
Internet
   |
   v
LoadBalancer
   |
   v
Database
```

Prefer:

```text
Internet
   |
   v
Application/API
   |
   v
Backend
   |
   | TCP 5432
   v
Database
```

The database should normally be reachable only from the workloads that require it.

---

# 20. Separate Database Namespace

For production, consider placing databases in a dedicated Namespace:

```text
production
   |
   +--> frontend
   +--> backend

database
   |
   +--> postgres
   +--> mysql
```

This makes it easier to apply:

* Network isolation
* RBAC controls
* Resource policies
* Monitoring
* Security policies
* Auditing

---

# 21. Database Security Should Not Depend Only on NetworkPolicy

NetworkPolicy controls network reachability.

It does not provide:

```text
Database authentication
Database authorization
Encryption at rest
Encryption in transit
Password management
Backup
Recovery
Audit logging
SQL-level access control
```

Use multiple layers:

```text
                 Production Database
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
 NetworkPolicy      DB Authentication     TLS
        |                |                |
        v                v                v
 Pod Isolation       DB Roles          Encryption
        |
        v
 Kubernetes RBAC
        |
        v
 Secrets Management
        |
        v
 Backups / Recovery
```

---

# 22. Use Kubernetes Secrets Carefully

Do not hard-code database credentials inside:

```yaml
Deployment
ConfigMap
NetworkPolicy
Git repository
Dockerfile
```

Use a proper secrets-management solution appropriate for your environment.

Example Kubernetes Secret reference:

```yaml
env:
  - name: DB_USERNAME
    valueFrom:
      secretKeyRef:
        name: database-credentials
        key: username

  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: database-credentials
        key: password
```

NetworkPolicy controls:

```text
Who can connect?
```

Database authentication controls:

```text
Who can authenticate?
```

These are different security layers.

---

# 23. Use TLS for Database Connections

NetworkPolicy does not encrypt database traffic.

If the database supports TLS, configure the application to use it.

For example:

```text
Backend
   |
   | TLS
   | TCP 5432
   v
PostgreSQL
```

NetworkPolicy:

```text
Controls connectivity
```

TLS:

```text
Protects data in transit
```

Database authentication:

```text
Verifies identity
```

All three provide different protections.

---

# 24. Production Policy Model

A strong production model is:

```text
                         Database
                            |
                +-----------+-----------+
                |                       |
          Default Deny              Authentication
                |                       |
                v                       v
         NetworkPolicy             DB Credentials
                |
                v
       Explicit Application
              Access
                |
                v
          TLS Encryption
                |
                v
          Audit / Monitoring
                |
                v
        Backup / Recovery
```

---

# 25. Common Mistakes

## Mistake 1 — Allowing the Entire Namespace

Avoid unnecessarily broad policies such as:

```yaml
from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: production
```

This allows all matching Pods in the Namespace.

Prefer:

```yaml
from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: production

    podSelector:
      matchLabels:
        app: backend
```

when only the backend needs access.

---

## Mistake 2 — Allowing All Ports

Avoid:

```yaml
ports:
  - {}
```

when only one database port is required.

Prefer:

```yaml
ports:
  - protocol: TCP
    port: 5432
```

---

## Mistake 3 — Allowing All Egress

Avoid:

```yaml
egress:
  - {}
```

for a sensitive database unless unrestricted egress is explicitly required and justified.

---

## Mistake 4 — Forgetting DNS

A default-deny Egress policy can break:

```text
DNS
```

which can make applications appear to have database connectivity problems.

Always test:

```bash
nslookup postgres.database.svc.cluster.local
```

---

## Mistake 5 — Relying Only on IP Addresses

Avoid using Pod IPs as the primary authorization mechanism.

Pod IPs can change.

Prefer:

```text
Namespace labels
+
Pod labels
```

for Kubernetes-native identity.

---

## Mistake 6 — Assuming NetworkPolicy Is a Firewall

NetworkPolicy is not a complete replacement for:

```text
Cloud firewall
Database firewall
Service mesh authorization
Database authentication
TLS
Secrets management
```

Use the appropriate security layer for each requirement.

---

# 26. Production Troubleshooting

If the backend cannot connect to PostgreSQL:

### Step 1 — Check Backend

```bash
kubectl get pod \
  -n production \
  --show-labels
```

### Step 2 — Check Database

```bash
kubectl get pod \
  -n database \
  --show-labels
```

### Step 3 — Check Policies

```bash
kubectl get networkpolicy \
  -A
```

### Step 4 — Describe Database Policy

```bash
kubectl describe networkpolicy \
  allow-backend-to-postgres \
  -n database
```

### Step 5 — Check Namespace Labels

```bash
kubectl get namespace \
  production \
  --show-labels
```

### Step 6 — Check Service

```bash
kubectl get svc postgres \
  -n database
```

### Step 7 — Check Endpoints

```bash
kubectl get endpoints postgres \
  -n database
```

### Step 8 — Test DNS

```bash
kubectl exec -it <backend-pod> \
  -n production \
  -- nslookup postgres.database.svc.cluster.local
```

### Step 9 — Test TCP

```bash
kubectl exec -it <backend-pod> \
  -n production \
  -- nc -vz postgres.database.svc.cluster.local 5432
```

### Step 10 — Check CNI

```bash
kubectl get pods \
  -n kube-system
```

Then inspect the networking implementation's logs and policy enforcement behavior.

---

# 27. Recommended Production Policy Set

A typical database Namespace might have:

```text
database Namespace
       |
       +--> Default Deny
       |
       +--> Allow Backend -> PostgreSQL :5432
       |
       +--> Allow DNS
       |
       +--> Allow Monitoring (if required)
       |
       +--> Allow Backup (if required)
       |
       +--> Allow Replication (if required)
```

Everything else should be denied unless explicitly required.

---

# 28. Production Validation Checklist

## Database

* [ ] Database runs in a dedicated Namespace
* [ ] Database has stable labels
* [ ] Database Service is correct
* [ ] Database EndpointSlices are healthy
* [ ] Database port is verified
* [ ] Database authentication is enabled
* [ ] TLS is enabled where required
* [ ] Credentials are stored securely

## NetworkPolicy

* [ ] Default-deny policy exists where appropriate
* [ ] Database Ingress is explicitly restricted
* [ ] Only required application Pods are allowed
* [ ] Namespace selector is correct
* [ ] Pod selector is correct
* [ ] Database port is restricted
* [ ] Protocol is restricted
* [ ] Unnecessary Egress is denied
* [ ] DNS is explicitly allowed if needed

## Testing

* [ ] Authorized application can connect
* [ ] Unauthorized Pod cannot connect
* [ ] DNS works
* [ ] Service resolution works
* [ ] Database authentication works
* [ ] TLS validation works
* [ ] CNI enforcement is verified

## Operations

* [ ] Policy changes are reviewed
* [ ] Policy changes are audited
* [ ] Monitoring is configured
* [ ] Database backups are configured
* [ ] Recovery procedures are tested

---

# 29. Reference Architecture

```text
                         Kubernetes Cluster
                                |
          +---------------------+---------------------+
          |                                           |
          |                                           |
   production Namespace                       database Namespace
          |                                           |
   +------+-------+                            +------+------+
   |              |                            |             |
Frontend        Backend                     PostgreSQL     Monitoring
   |              |                            |             |
   |              |                            |             |
   |              +---------- TCP 5432 --------+             |
   |                           ALLOWED                        |
   |                                                          |
   +--------------------X------------------------------------+
                     DENIED
```

Traffic policy:

```text
frontend  -> backend       ALLOW
backend   -> postgres      ALLOW
frontend  -> postgres      DENY
unknown   -> postgres      DENY
internet  -> postgres      DENY
```

---

# 30. Golden Rules

> **1. Default deny first, then explicitly allow required database traffic.**

> **2. Restrict database access using both Namespace and Pod selectors whenever possible.**

> **3. Allow only the required database port and protocol.**

> **4. Do not expose the database directly to the Internet unless there is a specific, reviewed requirement.**

> **5. Remember that NetworkPolicy controls network connectivity; it does not replace database authentication or TLS.**

> **6. Always test both positive and negative connectivity.**

> **7. Verify DNS when Egress policies are enabled.**

> **8. Use stable Pod and Namespace labels rather than relying on dynamic Pod IP addresses.**

> **9. Verify the CNI implementation actually enforces NetworkPolicy.**

> **10. Treat NetworkPolicy changes as production security changes and review them accordingly.**

---

# 31. Quick Reference

```bash
# Check database Pods
kubectl get pods -n database --show-labels

# Check backend Pods
kubectl get pods -n production --show-labels

# Check Namespace labels
kubectl get namespace production --show-labels

# List all policies
kubectl get networkpolicy -A

# Get database policies
kubectl get networkpolicy -n database

# Describe database policy
kubectl describe networkpolicy <policy> -n database

# Check database Service
kubectl get svc postgres -n database

# Check database endpoints
kubectl get endpoints postgres -n database

# Check EndpointSlices
kubectl get endpointslice -n database

# Test DNS
kubectl exec -it <backend-pod> \
  -n production \
  -- nslookup postgres.database.svc.cluster.local

# Test PostgreSQL port
kubectl exec -it <backend-pod> \
  -n production \
  -- nc -vz postgres.database.svc.cluster.local 5432

# Check CNI components
kubectl get pods -n kube-system

# Check cluster events
kubectl get events -A --sort-by='.lastTimestamp'
```

---

# 32. Final Security Model

For a production Kubernetes database, aim for:

```text
                    INTERNET
                       |
                       X
                       |
                  DATABASE
                       ^
                       |
                 NetworkPolicy
                       ^
                       |
                Backend Pods
                       ^
                       |
                 Application
```

More specifically:

```text
Database Namespace
       |
       +------------------------------+
       |                              |
       v                              v
  Default Deny                  DB Authentication
       |                              |
       v                              v
Explicit Backend Access             TLS
       |
       v
TCP 5432 only
       |
       v
PostgreSQL
```

The core principle is:

> **A database should be reachable only by the workloads that require access, on the specific port they require, from the specific Namespace and Pod identities that are authorized. NetworkPolicy should be combined with database authentication, TLS, secrets management, monitoring, backups, and other appropriate security controls for defense in depth.**
