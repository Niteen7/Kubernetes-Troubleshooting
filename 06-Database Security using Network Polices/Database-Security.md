# KUBERNETES NETWORKPOLICY - DATABASE SECURITY NOTES

## Purpose:

Secure a database running in Kubernetes by allowing network access only
from authorized application Pods and denying unnecessary traffic.

Main principle:

Only authorized application Pods should be able to reach the database,
only on the required database port.

1. RECOMMENDED SECURITY MODEL

---

Use the following approach:

```
Default Deny
     |
     v
Allow required application
     |
     v
Allow required database port
     |
     v
Allow required DNS
     |
     v
Test allowed and denied traffic
```

Example:

```
Database
   |
   +--> Default Deny
   |
   +--> Allow Backend
   |
   +--> Allow PostgreSQL TCP/5432
   |
   +--> Allow DNS if required
   |
   +--> Deny everything else
```

## 2. USE A DEDICATED DATABASE NAMESPACE

Recommended production structure:

```
production
   |
   +--> frontend
   +--> backend

database
   |
   +--> postgres
   +--> mysql
```

Benefits:

```
- Network isolation
- Better RBAC separation
- Easier security management
- Easier monitoring and auditing
- Better production organization
```

## 3. DEFAULT DENY

Start with a default-deny NetworkPolicy for the database Namespace.

Example:

```
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

Important:

```
podSelector: {}
```

means:

```
Select all Pods in this Namespace.
```

This provides a deny-by-default security baseline for the selected
Ingress and Egress traffic.

4. ALLOW ONLY THE REQUIRED APPLICATION

---

Example architecture:

```
Application Namespace = production
Application           = backend

Database Namespace    = database
Database              = postgres
```

Backend Pod labels:

```
app: backend
```

PostgreSQL Pod labels:

```
app: postgres
```

Allow only the backend application to access PostgreSQL:

```
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

## 5. NAMESPACESELECTOR + PODSELECTOR

This is an important concept.

Example:

```
namespaceSelector:
  matchLabels:
    kubernetes.io/metadata.name: production

podSelector:
  matchLabels:
    app: backend
```

This means:

```
Namespace must be production
              AND
Pod must have app=backend
```

Expected result:

```
production/backend       ---> ALLOW
production/frontend      ---> DENY
staging/backend          ---> DENY
development/backend     ---> DENY
other Pods               ---> DENY
```

This is more secure than allowing an entire Namespace.

6. RESTRICT THE DATABASE PORT

---

Only allow the port that the application actually needs.

PostgreSQL:

```
TCP 5432
```

MySQL:

```
TCP 3306
```

MongoDB:

```
TCP 27017
```

Example:

```
ports:
  - protocol: TCP
    port: 5432
```

Avoid unnecessarily allowing all ports:

```
ports:
  - {}
```

Security model:

```
Backend
   |
   | TCP 5432
   v
PostgreSQL
```

NOT:

```
Backend
   |
   | ALL PORTS
   v
PostgreSQL
```

## 7. UNDERSTAND INGRESS AND EGRESS

INGRESS:

Controls traffic coming INTO the database Pod.

```
Backend
   |
   | Ingress
   v
Database
```

EGRESS:

Controls traffic LEAVING a Pod.

```
Backend
   |
   | Egress
   v
Database
```

When troubleshooting database connectivity, check both sides:

```
Backend
   |
   | Egress
   v
Network
   |
   | Ingress
   v
Database
```

If the backend has an Egress policy, it must allow the database
connection.

If the database has an Ingress policy, it must allow the backend.

8. DO NOT FORGET DNS

---

One of the most common problems with Egress NetworkPolicies is
blocking DNS.

Application:

```
Backend
   |
   | DNS lookup
   v
CoreDNS
```

If DNS is blocked:

```
Backend
   |
   X
CoreDNS
```

The application may fail to resolve:

```
postgres.database.svc.cluster.local
```

When Egress policies are enabled, allow the required DNS traffic.

Normally:

```
UDP 53
TCP 53
```

The exact DNS Pod labels can vary by Kubernetes distribution, so
verify the labels in the cluster before creating the policy.

9. DO NOT GIVE THE DATABASE UNRESTRICTED INTERNET ACCESS

---

Avoid unrestricted Egress such as:

```
egress:
  - {}
```

unless there is a specific and documented requirement.

Preferred model:

```
Database
   |
   +--> DNS              ALLOW
   |
   +--> Backup service   ALLOW if required
   |
   +--> Monitoring       ALLOW if required
   |
   +--> Replication      ALLOW if required
   |
   +--> Internet         DENY
```

## 10. NETWORKPOLICY IS NOT COMPLETE DATABASE SECURITY

NetworkPolicy controls network connectivity.

It does NOT replace:

```
- Database authentication
- Database authorization
- TLS
- Secrets management
- Encryption
- Backups
- Monitoring
- Auditing
```

Think of database security as multiple layers:

```
NetworkPolicy
      +
Database Authentication
      +
Database Authorization
      +
TLS
      +
Secrets Management
      +
Encryption
      +
Backups
      +
Monitoring
      +
Auditing
```

## 11. NETWORKPOLICY VS DATABASE AUTHENTICATION

NetworkPolicy answers:

```
"Can this Pod establish a network connection to the database?"
```

Database authentication answers:

```
"Does this user/application have valid database credentials?"
```

Database authorization answers:

```
"What is this authenticated user allowed to do?"
```

Example:

```
Backend
   |
   | Network connection
   v
Database
   |
   X
Invalid credentials
```

Therefore, both NetworkPolicy and database authentication are required.

12. USE TLS

---

NetworkPolicy does not encrypt database traffic.

Without TLS:

```
Backend
   |
   | Plain TCP
   v
Database
```

With TLS:

```
Backend
   |
   | Encrypted TLS
   v
Database
```

For production databases, use TLS where supported and required.

13. DO NOT HARD-CODE DATABASE CREDENTIALS

---

Do not store database credentials in:

```
- Dockerfile
- Git repository
- NetworkPolicy
- ConfigMap
- Application source code
```

Use a proper secrets-management solution.

Example Kubernetes Secret reference:

```
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: database-credentials
        key: password
```

Remember:

```
NetworkPolicy = Network connectivity

Secret = Credentials
```

They solve different security problems.

14. VERIFY THE DATABASE SERVICE

---

Before blaming NetworkPolicy, verify the Kubernetes Service.

```
kubectl get svc postgres -n database
```

Then:

```
kubectl describe svc postgres -n database
```

Check:

```
- Service selector
- Service port
- TargetPort
- Endpoints
```

A NetworkPolicy cannot fix an incorrectly configured Service.

15. CHECK ENDPOINTS AND ENDPOINTSLICES

---

Check:

```
kubectl get endpoints postgres -n database
```

Also:

```
kubectl get endpointslice -n database
```

If there are no endpoints:

```
Service
   |
   X
No matching database Pods
```

Investigate:

```
- Service selector
- Pod labels
- Pod readiness
- Database Pod status
- EndpointSlices
```

The issue may not be NetworkPolicy.

16. VERIFY POD LABELS

---

NetworkPolicy relies heavily on labels.

Check database Pods:

```
kubectl get pods -n database --show-labels
```

Check application Pods:

```
kubectl get pods -n production --show-labels
```

Example:

Policy expects:

```
app=backend
```

Actual Pod:

```
app=backend-v2
```

Result:

```
Selector does not match.
```

Always verify the actual labels before troubleshooting the policy.

17. VERIFY NAMESPACE LABELS

---

If using namespaceSelector, verify Namespace labels.

```
kubectl get namespace production --show-labels
```

For example:

```
kubernetes.io/metadata.name=production
```

The label used by the NetworkPolicy must match the actual Namespace
label.

18. TEST BOTH ALLOWED AND DENIED TRAFFIC

---

Do not only test:

```
Backend -> Database
```

Also test:

```
Unauthorized Pod -> Database
```

Expected result:

```
Backend ----------------> Database
          ALLOWED

Frontend ---------------> Database
          DENIED

Unknown Pod ------------> Database
          DENIED
```

Testing both positive and negative cases confirms that the
NetworkPolicy is enforcing the intended security boundary.

19. BASIC TROUBLESHOOTING COMMANDS

---

List all NetworkPolicies:

```
kubectl get networkpolicy -A
```

List policies in database Namespace:

```
kubectl get networkpolicy -n database
```

Describe a policy:

```
kubectl describe networkpolicy <policy> -n database
```

Get policy YAML:

```
kubectl get networkpolicy <policy> \
  -n database \
  -o yaml
```

Check database Pods and labels:

```
kubectl get pods -n database --show-labels
```

Check application Pods and labels:

```
kubectl get pods -n production --show-labels
```

Check Namespace labels:

```
kubectl get namespace production --show-labels
```

Check database Service:

```
kubectl get svc postgres -n database
```

Describe database Service:

```
kubectl describe svc postgres -n database
```

Check endpoints:

```
kubectl get endpoints postgres -n database
```

Check EndpointSlices:

```
kubectl get endpointslice -n database
```

Test DNS:

```
kubectl exec -it <backend-pod> \
  -n production \
  -- nslookup postgres.database.svc.cluster.local
```

Test TCP connectivity:

```
kubectl exec -it <backend-pod> \
  -n production \
  -- nc -vz postgres.database.svc.cluster.local 5432
```

Check CNI components:

```
kubectl get pods -n kube-system
```

## 20. PRODUCTION TROUBLESHOOTING FLOW

When the application cannot connect to the database:

```
Application cannot connect to DB
              |
              v
       Check DB Pod
              |
              v
      Check DB Service
              |
              v
      Check Endpoints
              |
              v
         Check DNS
              |
              v
   List NetworkPolicies
              |
              v
    Check Backend Egress
              |
              v
   Check Database Ingress
              |
              v
    Check Pod selectors
              |
              v
  Check Namespace selectors
              |
              v
      Check DB port
              |
              v
        Check CNI
              |
              v
    Test connectivity
```

## 21. COMMON MISTAKES

Mistake 1:
Allowing the entire Namespace when only one application needs
database access.

Mistake 2:
Allowing all ports instead of only the required database port.

Mistake 3:
Forgetting DNS when Egress policies are enabled.

Mistake 4:
Using incorrect Pod labels.

Mistake 5:
Using incorrect Namespace labels.

Mistake 6:
Assuming NetworkPolicy provides database authentication.

Mistake 7:
Assuming NetworkPolicy encrypts database traffic.

Mistake 8:
Exposing the database directly through a public LoadBalancer
without a strong business/security requirement.

Mistake 9:
Allowing unrestricted Egress from the database.

22. PRODUCTION SECURITY CHECKLIST

---

Database:

```
[ ] Dedicated database Namespace
[ ] Correct Pod labels
[ ] Correct Service configuration
[ ] Correct EndpointSlices
[ ] Database authentication enabled
[ ] TLS enabled where required
[ ] Credentials stored securely
[ ] Backups configured
```

NetworkPolicy:

```
[ ] Default Deny configured
[ ] Only authorized applications allowed
[ ] Namespace selector verified
[ ] Pod selector verified
[ ] Database port restricted
[ ] Protocol restricted
[ ] DNS allowed where required
[ ] Unnecessary Egress denied
[ ] Internet access denied unless required
[ ] CNI supports and enforces NetworkPolicy
```

Testing:

```
[ ] Authorized application can connect
[ ] Unauthorized Pod cannot connect
[ ] DNS resolution works
[ ] Service resolution works
[ ] Database authentication works
[ ] TLS works
[ ] CNI enforcement verified
```

## 23. GOLDEN ARCHITECTURE

```
                     INTERNET
                        |
                        X
                        |
                +---------------+
                |   Database    |
                |   Namespace   |
                +---------------+
                        |
                 Default Deny
                        |
                        v
               Explicit Allow
                        |
                        v
               +----------------+
               | Backend Pods   |
               | app=backend    |
               +----------------+
                        |
                   TCP 5432
                        |
                        v
               +----------------+
               | PostgreSQL     |
               | app=postgres   |
               +----------------+
```

Expected traffic:

```
Backend  -------- ALLOW --------> PostgreSQL :5432

Frontend -------- DENY ---------> PostgreSQL

Unknown Pod ----- DENY ---------> PostgreSQL

Internet -------- DENY ---------> PostgreSQL
```

## 24. INTERVIEW ANSWER

Question:

How do you secure a database using Kubernetes NetworkPolicy?

Answer:

I would first isolate the database in a dedicated Namespace and
apply a default-deny NetworkPolicy.

Then I would create an explicit Ingress policy on the database that
allows only the required application Pods. I would preferably use
both Namespace and Pod selectors so that only the required
application from the required Namespace can access the database.

I would restrict access to the required database port, such as
TCP 5432 for PostgreSQL.

If Egress policies are enabled, I would also make sure that the
application can reach the database and that required DNS traffic
is allowed.

I would verify the database Service, EndpointSlices, Pod labels,
Namespace labels, selectors, ports, and CNI implementation.

Finally, I would test both allowed and denied connectivity.

NetworkPolicy is only one layer of database security. I would also
use database authentication and authorization, TLS, secure secret
management, encryption, monitoring, auditing, and backups.

25. FINAL TAKEAWAYS

---

1. Use default deny for database network access where appropriate.

2. Allow only authorized application Pods.

3. Use Namespace + Pod selectors for precise access control.

4. Restrict the database to the required port.

5. PostgreSQL = TCP 5432.

6. MySQL = TCP 3306.

7. MongoDB = TCP 27017.

8. Do not expose the database directly to the Internet.

9. Allow DNS explicitly when Egress restrictions are enabled.

10. Avoid unrestricted database Egress.

11. Verify Service and EndpointSlices before assuming NetworkPolicy
    is the problem.

12. Verify Pod and Namespace labels.

13. Test both allowed and denied traffic.

14. Verify that the CNI supports and enforces NetworkPolicy.

15. NetworkPolicy does not replace database authentication.

16. NetworkPolicy does not encrypt database traffic.

17. Use NetworkPolicy together with authentication, authorization,
    TLS, secrets management, encryption, monitoring, auditing,
    backups, and other required security controls.

## ONE-LINE PRODUCTION RULE

Only the required application Pods should be able to reach the
database, only on the required port, from the required Namespace,
with all other unnecessary network traffic denied.
