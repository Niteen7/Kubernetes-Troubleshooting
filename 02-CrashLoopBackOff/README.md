# CrashLoopBackOff

When you see "CrashLoopBackOff," it means that kubelet is trying to run the container, but it keeps failing and crashing. After crashing, Kubernetes tries to restart the container automatically, but if the container keeps failing repeatedly, you end up in a loop of crashes and restarts, thus the term "CrashLoopBackOff." 

This situation indicates that something is wrong with the application or the configuration that needs to be fixed.

## Common Situations of CrashLoopBackOff

The CrashLoopBackOff error in Kubernetes indicates that a container is repeatedly crashing and restarting. Here are explanations of how the CrashLoopBackOff error can occur due to the specific reasons you listed:

### Misconfigurations

Misconfigurations can encompass a wide range of issues, from incorrect environment variables to improper setup of service ports or volumes. These misconfigurations can prevent the application from starting correctly, leading to crashes. For example, if an application expects a certain environment variable to connect to a database and that variable is not set or is incorrect, the application might crash as it cannot establish a database connection.

### Errors in the Liveness Probes

Liveness probes in Kubernetes are used to check the health of a container. If a liveness probe is incorrectly configured, it might falsely report that the container is unhealthy, causing Kubernetes to kill and restart the container repeatedly. For example, if the liveness probe checks a URL or port that the application does not expose or checks too soon before the application is ready, the container will be repeatedly terminated and restarted.

### The Memory Limits Are Too Low

If the memory limits set for a container are too low, the application might exceed this limit, especially under load, leading to the container being killed by Kubernetes. This can happen repeatedly if the workload does not decrease, causing a cycle of crashing and restarting. Kubernetes uses these limits to ensure that containers do not consume all available resources on a node, which can affect other containers.

### Wrong Command Line Arguments

Containers might be configured to start with specific command-line arguments. If these arguments are wrong or lead to the application exiting (for example, passing an invalid option to a command), the container will exit immediately. Kubernetes will then attempt to restart it, leading to the CrashLoopBackOff status. An example would be passing a configuration file path that does not exist or is inaccessible.

### Bugs & Exceptions

Bugs in the application code, such as unhandled exceptions or segmentation faults, can cause the application to crash. For instance, if the application tries to access a null pointer or fails to catch and handle an exception correctly, it might terminate unexpectedly. Kubernetes, detecting the crash, will restart the container, but if the bug is triggered each time the application runs, this leads to a repetitive crash loop.

# Kubernetes Production Troubleshooting Runbook: CrashLoopBackOff

A practical production troubleshooting runbook for diagnosing and resolving Kubernetes Pods stuck in `CrashLoopBackOff`.

---

# Table of Contents

* [1. Purpose](#1-purpose)
* [2. What is CrashLoopBackOff](#2-what-is-crashloopbackoff)
* [3. CrashLoopBackOff Troubleshooting Flow](#3-crashloopbackoff-troubleshooting-flow)
* [4. Initial Pod Investigation](#4-initial-pod-investigation)
* [5. Check Container Logs](#5-check-container-logs)
* [6. Check Previous Container Logs](#6-check-previous-container-logs)
* [7. Check Pod Events](#7-check-pod-events)
* [8. Check Container Exit Code](#8-check-container-exit-code)
* [9. Check Application Configuration](#9-check-application-configuration)
* [10. Check ConfigMaps](#10-check-configmaps)
* [11. Check Secrets](#11-check-secrets)
* [12. Check Environment Variables](#12-check-environment-variables)
* [13. Check Application Startup Command](#13-check-application-startup-command)
* [14. Check Dockerfile and Container Image](#14-check-dockerfile-and-container-image)
* [15. Check Resource Limits](#15-check-resource-limits)
* [16. OOMKilled](#16-oomkilled)
* [17. Check CPU and Memory Usage](#17-check-cpu-and-memory-usage)
* [18. Check Liveness and Readiness Probes](#18-check-liveness-and-readiness-probes)
* [19. Check Dependencies](#19-check-dependencies)
* [20. Check Database Connectivity](#20-check-database-connectivity)
* [21. Check External Services](#21-check-external-services)
* [22. Check Permissions](#22-check-permissions)
* [23. Check SecurityContext](#23-check-securitycontext)
* [24. Check Volume Mounts](#24-check-volume-mounts)
* [25. Check Application Port](#25-check-application-port)
* [26. Jenkins CI/CD Troubleshooting](#26-jenkins-cicd-troubleshooting)
* [27. Common Exit Codes](#27-common-exit-codes)
* [28. Common Errors](#28-common-errors)
* [29. Production Troubleshooting Decision Tree](#29-production-troubleshooting-decision-tree)
* [30. Production Checklist](#30-production-checklist)
* [31. Fast Incident Investigation](#31-fast-incident-investigation)
* [32. Root Cause Classification](#32-root-cause-classification)
* [33. Best Practices](#33-best-practices)
* [34. Interview Answer](#34-interview-answer)
* [35. Quick Reference](#35-quick-reference)
* [36. Summary](#36-summary)

---

# 1. Purpose

This runbook provides a structured approach for troubleshooting Kubernetes Pods that are repeatedly starting and crashing.

It focuses on:

* `CrashLoopBackOff`
* Application startup failures
* Container exit codes
* Application logs
* Previous container logs
* Configuration problems
* ConfigMaps
* Secrets
* Environment variables
* Resource limits
* `OOMKilled`
* Liveness probes
* Readiness probes
* Database connectivity
* External dependencies
* File permissions
* SecurityContext
* Volume mounts
* Container commands
* Dockerfile issues
* Jenkins CI/CD issues

The objective is to identify the root cause systematically and safely resolve the production issue.

---

# 2. What is CrashLoopBackOff?

`CrashLoopBackOff` means that a container inside a Pod has repeatedly started and then terminated.

Kubernetes restarts the container, but because it keeps failing, Kubernetes progressively increases the delay before restarting it.

The lifecycle generally looks like:

```text
Container starts
      |
      v
Application starts
      |
      v
Application crashes
      |
      v
Container exits
      |
      v
Kubernetes restarts container
      |
      v
Application crashes again
      |
      v
Back-off
      |
      v
CrashLoopBackOff
```

### Important distinction

`CrashLoopBackOff` does **not** necessarily mean Kubernetes itself is broken.

It usually means:

> **The container starts but the process inside the container repeatedly terminates or is repeatedly killed/restarted.**

Common causes include:

* Application error
* Incorrect configuration
* Missing environment variables
* Missing Secret
* Incorrect ConfigMap
* Database connection failure
* External dependency failure
* Incorrect startup command
* Application port/configuration mismatch
* File permission problems
* Missing files
* Insufficient memory
* `OOMKilled`
* Incorrect liveness probe
* Application startup failure
* Dependency/version incompatibility

---

# 3. CrashLoopBackOff Troubleshooting Flow

Use the following high-level process:

```text
                    CrashLoopBackOff
                           |
                           v
                 kubectl describe pod
                           |
                           v
                    Check Events
                           |
                           v
                  Check Application Logs
                           |
                           v
              Check Previous Container Logs
                           |
                           v
                    Check Exit Code
                           |
          +----------------+----------------+
          |                                 |
          v                                 v
    Application Error                  Container Killed
          |                                 |
          v                                 v
 Config / Dependency /              OOM / Probe /
 Code / Permissions                 Resource Issue
          |                                 |
          +----------------+----------------+
                           |
                           v
                    Verify Configuration
                           |
                           v
                  Verify Dependencies
                           |
                           v
                    Apply Correct Fix
                           |
                           v
                     Validate Pod
```

---

# 4. Initial Pod Investigation

## 4.1 List Pods

```bash
kubectl get pods -n <namespace>
```

Example:

```bash
kubectl get pods -n production
```

Example output:

```text
NAME                            READY   STATUS             RESTARTS   AGE
payment-api-7d8f9c8b7f          0/1     CrashLoopBackOff   8          10m
```

The `RESTARTS` column is important.

For example:

```text
RESTARTS
   0
```

means the container has not restarted.

But:

```text
RESTARTS
   25
```

indicates repeated container failures.

---

# 5. Check Container Logs

The first troubleshooting step after identifying `CrashLoopBackOff` should usually be checking the application logs.

```bash
kubectl logs <pod-name> -n <namespace>
```

Example:

```bash
kubectl logs payment-api-7d8f9c8b7f -n production
```

Look for:

* Exception
* Stack trace
* Configuration error
* Database connection error
* Authentication failure
* File-not-found error
* Permission denied
* Port binding failure
* Out-of-memory error
* Application startup failure

Example:

```text
ERROR: Unable to connect to database
Connection refused: db.production.svc.cluster.local:5432

Application startup failed.
```

This immediately points toward a database connectivity problem.

---

# 6. Check Previous Container Logs

This is one of the most important commands for `CrashLoopBackOff`.

When the container has already crashed and restarted, the current logs may not contain the information you need.

Use:

```bash
kubectl logs <pod-name> -n <namespace> --previous
```

Example:

```bash
kubectl logs payment-api-7d8f9c8b7f \
  -n production \
  --previous
```

For multi-container Pods:

```bash
kubectl logs <pod-name> \
  -n <namespace> \
  -c <container-name> \
  --previous
```

### Why `--previous` is important

Consider:

```text
Container starts
     |
     v
Application prints error
     |
     v
Container crashes
     |
     v
Container restarts
```

The logs from the previous container instance may contain the actual failure.

Therefore:

```bash
kubectl logs <pod> --previous
```

should be one of your first commands during a `CrashLoopBackOff` investigation.

---

# 7. Check Pod Events

Run:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Example:

```bash
kubectl describe pod payment-api-7d8f9c8b7f -n production
```

Look at:

```text
Events:
```

Possible events include:

```text
Back-off restarting failed container
```

or:

```text
Liveness probe failed
```

or:

```text
Killing container
```

or:

```text
OOMKilled
```

The Events section helps determine whether Kubernetes itself is killing the container or whether the application is terminating by itself.

---

# 8. Check Container Exit Code

Inspect the container's termination state:

```bash
kubectl get pod <pod-name> -n <namespace> -o yaml
```

Look for:

```yaml
containerStatuses:
  - name: payment-api
    lastState:
      terminated:
        exitCode: 1
        reason: Error
```

You can also use:

```bash
kubectl get pod <pod-name> -n <namespace> \
  -o jsonpath='{.status.containerStatuses[*].lastState.terminated.exitCode}'
```

Example:

```text
1
```

Exit codes provide useful information about how the process terminated.

---

# 9. Common Exit Codes

| Exit Code | Common Meaning                                              |
| --------- | ----------------------------------------------------------- |
| `0`       | Process exited successfully                                 |
| `1`       | Generic application error                                   |
| `2`       | Incorrect usage/syntax or application-specific error        |
| `126`     | Command found but cannot be executed                        |
| `127`     | Command not found                                           |
| `128`     | Invalid exit argument                                       |
| `137`     | Process killed with `SIGKILL`; commonly associated with OOM |
| `143`     | Process terminated with `SIGTERM`                           |
| `139`     | Segmentation fault (`SIGSEGV`)                              |

### Important

Exit codes are clues, not absolute diagnoses.

For example:

```text
137
```

often indicates an OOM-related termination, but you should verify the Pod status and events rather than assuming it.

---

# 10. Check Application Configuration

Incorrect application configuration is one of the most common causes of `CrashLoopBackOff`.

Check:

* Environment variables
* ConfigMaps
* Secrets
* Application configuration files
* Database configuration
* API endpoints
* Credentials
* Ports
* Feature flags
* Runtime configuration

Inspect the Deployment:

```bash
kubectl get deployment <deployment-name> \
  -n <namespace> \
  -o yaml
```

---

# 11. Check ConfigMaps

List ConfigMaps:

```bash
kubectl get configmaps -n <namespace>
```

Inspect a ConfigMap:

```bash
kubectl describe configmap <configmap-name> -n <namespace>
```

Or:

```bash
kubectl get configmap <configmap-name> \
  -n <namespace> \
  -o yaml
```

Check whether the application expects configuration that is missing.

Example:

```yaml
envFrom:
  - configMapRef:
      name: payment-api-config
```

Verify:

```bash
kubectl get configmap payment-api-config -n production
```

If the ConfigMap does not exist, the application may fail during startup depending on how it is referenced and configured.

---

# 12. Check Secrets

List Secrets:

```bash
kubectl get secrets -n <namespace>
```

Check a Secret:

```bash
kubectl describe secret <secret-name> -n <namespace>
```

Do **not** expose decoded Secret values in logs, tickets, chat messages, or Git repositories.

Verify:

* Secret exists
* Correct Secret name
* Correct namespace
* Required keys exist
* Application expects the same key names

Example:

```yaml
env:
  - name: DB_USERNAME
    valueFrom:
      secretKeyRef:
        name: database-secret
        key: username

  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: database-secret
        key: password
```

Verify the Secret:

```bash
kubectl get secret database-secret -n production
```

---

# 13. Check Environment Variables

Inspect the Deployment:

```bash
kubectl get deployment <deployment-name> \
  -n <namespace> \
  -o yaml
```

Look for:

```yaml
env:
  - name: DB_HOST
    value: "postgres"
  - name: DB_PORT
    value: "5432"
```

or:

```yaml
envFrom:
  - configMapRef:
      name: payment-api-config
  - secretRef:
      name: payment-api-secret
```

Common problems:

```text
Missing environment variable
Wrong environment variable name
Wrong value
Wrong database hostname
Wrong port
Wrong credentials
Wrong API endpoint
```

---

# 14. Check Application Startup Command

A container may immediately crash because the configured command or arguments are incorrect.

Check the Deployment:

```bash
kubectl get deployment <deployment-name> \
  -n <namespace> \
  -o yaml
```

Look for:

```yaml
command:
  - "/app/start.sh"

args:
  - "--environment=production"
```

Verify that:

* The command exists.
* The script exists.
* The script has execute permission.
* Arguments are valid.
* Required files exist.
* The working directory is correct.

---

# 15. Check Dockerfile and Container Image

Review the Dockerfile.

Example:

```dockerfile
FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci --omit=dev

COPY . .

CMD ["node", "server.js"]
```

Verify:

* Correct base image
* Correct runtime version
* Application files are copied
* Dependencies are installed
* Startup command is correct
* Working directory is correct
* Required files exist
* Required permissions are available

Test the image locally:

```bash
docker run --rm <image>:<tag>
```

If the container crashes locally with the same configuration, the problem is likely inside the image/application rather than Kubernetes.

---

# 16. Check Resource Limits

Inspect resource configuration:

```bash
kubectl get deployment <deployment-name> \
  -n <namespace> \
  -o yaml
```

Look for:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"

  limits:
    cpu: "500m"
    memory: "512Mi"
```

Problems can occur when the application requires more memory than its configured limit.

---

# 17. OOMKilled

`OOMKilled` means the container was terminated because it exceeded its memory limit or the node experienced memory pressure.

Check:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Look for:

```text
Reason: OOMKilled
```

You can also check:

```bash
kubectl get pod <pod-name> -n <namespace> \
  -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
```

Expected:

```text
OOMKilled
```

Check the configured limit:

```bash
kubectl get deployment <deployment-name> \
  -n <namespace> \
  -o jsonpath='{.spec.template.spec.containers[*].resources.limits.memory}'
```

---

# 18. Check CPU and Memory Usage

If Metrics Server is available:

```bash
kubectl top pod <pod-name> -n <namespace>
```

Example:

```text
NAME             CPU(cores)   MEMORY(bytes)
payment-api      450m         490Mi
```

Compare this with the configured limit.

Example:

```yaml
resources:
  limits:
    memory: "512Mi"
```

If the application consistently approaches the memory limit, investigate:

* Memory leak
* Large requests
* Large data processing
* Incorrect JVM heap settings
* Incorrect application configuration
* Insufficient container memory limit

Do not blindly increase memory limits without understanding why memory usage is high.

---

# 19. Check Liveness and Readiness Probes

Incorrect probes are another common cause of container restarts.

Example:

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
```

If the application takes 60 seconds to start but the liveness probe begins checking after 10 seconds, Kubernetes may repeatedly kill the container before it finishes starting.

Check:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Look for:

```text
Liveness probe failed
```

---

# 20. Liveness vs Readiness

Understanding the difference is important.

### Liveness Probe

Answers:

> Is the application still alive?

If the liveness probe repeatedly fails, Kubernetes may restart the container.

### Readiness Probe

Answers:

> Is the application ready to receive traffic?

If readiness fails, Kubernetes normally removes the Pod from Service endpoints but does not restart the container.

### Startup Probe

Useful for applications that take a long time to start.

Example:

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

This gives the application more time to start before liveness checking becomes active.

---

# 21. Check Dependencies

Applications often depend on other services:

```text
Application
    |
    +--> Database
    |
    +--> Redis
    |
    +--> Kafka
    |
    +--> External API
    |
    +--> Authentication Service
```

If a dependency is unavailable and the application exits instead of retrying, the Pod may enter `CrashLoopBackOff`.

Check the application logs:

```bash
kubectl logs <pod-name> -n <namespace> --previous
```

Look for:

```text
Connection refused
Connection timeout
DNS resolution failed
Authentication failed
TLS handshake failed
Connection reset
```

---

# 22. Check Database Connectivity

Verify the database Service:

```bash
kubectl get svc -n <namespace>
```

Example:

```text
NAME       TYPE        CLUSTER-IP      PORT(S)
postgres   ClusterIP   10.96.100.20    5432/TCP
```

Check endpoints:

```bash
kubectl get endpoints postgres -n <namespace>
```

Or, depending on the cluster version/configuration:

```bash
kubectl get endpointslice -n <namespace>
```

Verify that the database has healthy backend Pods.

```bash
kubectl get pods -n <namespace> -l app=postgres
```

---

# 23. Check DNS

If the application cannot resolve a dependency:

```text
postgres.production.svc.cluster.local
```

investigate Kubernetes DNS.

Check CoreDNS:

```bash
kubectl get pods -n kube-system \
  -l k8s-app=kube-dns
```

Check CoreDNS logs:

```bash
kubectl logs -n kube-system \
  -l k8s-app=kube-dns
```

You can also use a temporary troubleshooting Pod:

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --rm -it \
  --restart=Never \
  -- sh
```

Inside the Pod:

```sh
nslookup kubernetes.default
```

---

# 24. Check External Services

If the application depends on an external API:

```text
Payment API
     |
     v
External Payment Provider
```

verify:

* DNS
* Network connectivity
* TLS certificates
* Proxy configuration
* API credentials
* Endpoint URL
* Firewall rules
* Network policies
* Service availability

Check the application logs for the exact failure.

---

# 25. Check Permissions

The application may crash because the container user does not have permission to access a file or directory.

Example:

```text
Permission denied
```

Check the Pod's `securityContext`:

```bash
kubectl get pod <pod-name> \
  -n <namespace> \
  -o yaml
```

Example:

```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
```

Verify that the application has permission to:

* Read configuration files
* Write log files
* Write temporary files
* Access mounted volumes
* Execute startup scripts

---

# 26. Check SecurityContext

Common security settings include:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  allowPrivilegeEscalation: false
```

These settings improve security but can expose applications that incorrectly assume they can run as root.

For example, an application may attempt to write to:

```text
/root
```

while running as:

```text
UID 1000
```

This can result in:

```text
Permission denied
```

Do not simply remove security restrictions in production. Fix the application's file ownership and permissions instead.

---

# 27. Check Volume Mounts

Check the Pod:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Look for:

```text
Mounts:
Volumes:
```

Common problems:

* Volume does not exist
* Incorrect mount path
* Read-only filesystem
* Incorrect file permissions
* Missing configuration file
* PVC not mounted correctly

Check PVCs:

```bash
kubectl get pvc -n <namespace>
```

Check:

```bash
kubectl describe pvc <pvc-name> -n <namespace>
```

---

# 28. Check Application Port

Verify the application listens on the expected port.

Example:

```yaml
containers:
  - name: payment-api
    ports:
      - containerPort: 8080
```

Check the application configuration.

For example:

```text
Application listens on:
8080

Container expects:
8080

Service targets:
8080
```

A port mismatch may cause connectivity or probe failures.

However, remember:

> A port mismatch by itself does not necessarily cause `CrashLoopBackOff`.

It may instead result in failed readiness/liveness probes or Service connectivity issues.

---

# 29. Jenkins CI/CD Troubleshooting

Jenkins can introduce application-level failures if the wrong image is built or deployed.

Typical flow:

```text
Git
 |
 v
Jenkins
 |
 +--> Build
 |
 +--> Test
 |
 +--> Docker Build
 |
 +--> Push Image
 |
 +--> Update Kubernetes Manifest
 |
 v
Kubernetes
 |
 v
Pod
 |
 v
Application
```

Verify:

* Correct Git branch
* Correct commit
* Correct Dockerfile
* Correct application version
* Correct dependencies
* Correct environment
* Correct image tag
* Correct image pushed
* Correct Kubernetes manifest
* Correct ConfigMaps
* Correct Secrets

---

# 30. Common CrashLoopBackOff Scenarios

## Scenario 1: Application Configuration Error

Logs:

```text
ERROR: DB_HOST environment variable is missing
Application startup failed
```

Root cause:

```text
Required environment variable is missing.
```

Resolution:

Check:

```yaml
env:
envFrom:
configMapRef:
secretRef:
```

---

## Scenario 2: Database Connection Failure

Logs:

```text
Connection refused to postgres:5432
```

Check:

```bash
kubectl get svc postgres -n production
```

Then:

```bash
kubectl get endpoints postgres -n production
```

Verify database health and connectivity.

---

## Scenario 3: OOMKilled

Pod status:

```text
Reason: OOMKilled
```

Check:

```bash
kubectl top pod <pod-name> -n production
```

Review memory usage and application behavior before changing limits.

---

## Scenario 4: Incorrect Startup Command

Logs:

```text
/bin/start.sh: not found
```

Check:

```yaml
command:
args:
```

Then inspect the Docker image and Dockerfile.

---

## Scenario 5: Permission Denied

Logs:

```text
Permission denied: /app/logs/application.log
```

Check:

```yaml
securityContext:
volumeMounts:
volumes:
```

Correct file ownership/permissions rather than unnecessarily running the container as root.

---

## Scenario 6: Liveness Probe Failure

Events:

```text
Liveness probe failed
Killing container
```

Check:

```yaml
livenessProbe:
startupProbe:
initialDelaySeconds:
periodSeconds:
timeoutSeconds:
failureThreshold:
```

If the application needs significant startup time, consider a properly configured `startupProbe`.

---

# 31. Common Error Messages

| Error                       | Likely Cause                             |
| --------------------------- | ---------------------------------------- |
| `CrashLoopBackOff`          | Container repeatedly exits               |
| `OOMKilled`                 | Container exceeded memory constraints    |
| `Connection refused`        | Target service unavailable/not listening |
| `Connection timed out`      | Network/connectivity problem             |
| `Permission denied`         | File/security permission problem         |
| `command not found`         | Incorrect command or missing executable  |
| `No such file or directory` | Missing file/path                        |
| `Liveness probe failed`     | Kubernetes health check failed           |
| `Readiness probe failed`    | Application is not ready                 |
| `Address already in use`    | Port already occupied                    |
| `Authentication failed`     | Invalid credentials                      |
| `ConfigMap not found`       | Missing/incorrect ConfigMap              |
| `Secret not found`          | Missing/incorrect Secret                 |
| `Invalid configuration`     | Application configuration problem        |

---

# 32. Production Troubleshooting Decision Tree

```text
                    CrashLoopBackOff
                           |
                           v
                kubectl describe pod
                           |
                           v
                    Check Events
                           |
                           v
                 kubectl logs --previous
                           |
                           v
                  Identify root cause
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
     App Error          OOMKilled       Probe Failure
          |                |                |
          v                v                v
    Check Config       Check Memory     Check Probe
    Check Secrets      Check Limits     Check Startup
    Check DB           Check Usage      Check Timing
    Check Code
          |
          v
     Dependency Issue?
          |
     +----+----+
     |         |
    Yes        No
     |         |
     v         v
 Check DB/    Check Command
 Redis/API    Check Files
 Kafka/etc.   Check Permissions
     |         |
     +----+----+
          |
          v
       Fix Issue
          |
          v
      Validate Pod
```

---

# 33. Production Checklist

## Pod

* [ ] Pod is in the expected namespace
* [ ] Pod status confirmed
* [ ] Restart count checked
* [ ] Pod Events checked
* [ ] Current logs checked
* [ ] Previous container logs checked
* [ ] Exit code checked
* [ ] Termination reason checked

## Application

* [ ] Application startup logs reviewed
* [ ] Configuration verified
* [ ] Environment variables verified
* [ ] ConfigMaps verified
* [ ] Secrets verified
* [ ] Database connectivity verified
* [ ] External dependencies verified
* [ ] Startup command verified
* [ ] Application port verified

## Resources

* [ ] CPU requests verified
* [ ] CPU limits verified
* [ ] Memory requests verified
* [ ] Memory limits verified
* [ ] `OOMKilled` checked
* [ ] Actual resource usage checked

## Probes

* [ ] Liveness probe verified
* [ ] Readiness probe verified
* [ ] Startup probe verified
* [ ] Probe endpoint verified
* [ ] Probe port verified
* [ ] Probe timing verified

## Security

* [ ] SecurityContext verified
* [ ] Container user verified
* [ ] File permissions verified
* [ ] Volume permissions verified
* [ ] Required permissions verified

## Storage

* [ ] PVC status verified
* [ ] Volume mount verified
* [ ] Mount path verified
* [ ] Read/write permissions verified

## CI/CD

* [ ] Correct Git commit deployed
* [ ] Correct application version
* [ ] Correct Dockerfile
* [ ] Correct image
* [ ] Correct image tag
* [ ] Correct ConfigMap/Secret version
* [ ] Jenkins build successful
* [ ] Deployment updated successfully

---

# 34. Fast Incident Investigation

During a production incident, run these commands first.

### 1. Check Pod

```bash
kubectl get pods -n <namespace>
```

### 2. Check Pod Details and Events

```bash
kubectl describe pod <pod-name> -n <namespace>
```

### 3. Check Current Logs

```bash
kubectl logs <pod-name> -n <namespace>
```

### 4. Check Previous Logs

```bash
kubectl logs <pod-name> \
  -n <namespace> \
  --previous
```

### 5. Check Container Status

```bash
kubectl get pod <pod-name> \
  -n <namespace> \
  -o jsonpath='{.status.containerStatuses[*]}'
```

### 6. Check Exit Code

```bash
kubectl get pod <pod-name> \
  -n <namespace> \
  -o jsonpath='{.status.containerStatuses[*].lastState.terminated.exitCode}'
```

### 7. Check Termination Reason

```bash
kubectl get pod <pod-name> \
  -n <namespace> \
  -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'
```

### 8. Check Resource Usage

```bash
kubectl top pod <pod-name> -n <namespace>
```

### 9. Check Deployment

```bash
kubectl get deployment <deployment-name> \
  -n <namespace> \
  -o yaml
```

---

# 35. Root Cause Classification

After investigation, classify the issue.

## Category 1 — Application

```text
Application exception
Startup failure
Code bug
Dependency failure
Incorrect runtime version
```

## Category 2 — Configuration

```text
Missing environment variable
Incorrect ConfigMap
Incorrect Secret
Incorrect configuration file
Incorrect endpoint
```

## Category 3 — Resource

```text
OOMKilled
Insufficient memory
CPU throttling
Incorrect resource limits
```

## Category 4 — Probe

```text
Incorrect liveness probe
Incorrect readiness probe
Incorrect startup probe
Incorrect port
Incorrect health endpoint
Incorrect timing
```

## Category 5 — Dependency

```text
Database unavailable
Redis unavailable
Kafka unavailable
External API unavailable
DNS failure
```

## Category 6 — Security

```text
Permission denied
Incorrect user
Incorrect SecurityContext
File ownership problem
Volume permissions
```

## Category 7 — CI/CD

```text
Wrong image
Wrong image tag
Wrong application version
Incorrect Dockerfile
Incorrect configuration deployed
Bad build artifact
```

---

# 36. Best Practices

## 36.1 Use Startup Probes for Slow-Starting Applications

For applications that require significant startup time:

```yaml
startupProbe:
  httpGet:
    path: /health
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

This prevents the liveness probe from killing the application while it is still starting.

---

## 36.2 Configure Resource Requests and Limits

Example:

```yaml
resources:
  requests:
    cpu: "250m"
    memory: "256Mi"

  limits:
    cpu: "1"
    memory: "512Mi"
```

Use observed workload behavior to determine appropriate values.

---

## 36.3 Implement Graceful Shutdown

Applications should handle `SIGTERM` correctly.

Typical flow:

```text
Kubernetes
    |
    | SIGTERM
    v
Application
    |
    +--> Stop accepting new requests
    |
    +--> Finish active requests
    |
    +--> Close connections
    |
    +--> Flush data
    |
    v
Exit
```

This is especially important during:

* Rolling deployments
* Scaling
* Node maintenance
* Pod eviction

---

## 36.4 Make Applications Resilient to Dependencies

Avoid immediately terminating the application when a temporary dependency is unavailable.

Prefer:

```text
Retry
Backoff
Timeout
Circuit breaker
Graceful degradation
```

where appropriate.

---

## 36.5 Use Structured Logging

Prefer logs such as:

```text
2026-08-11T12:30:45Z ERROR database connection failed
host=postgres
port=5432
retry=3
```

instead of unstructured messages that make production troubleshooting difficult.

Never log:

* Passwords
* API keys
* Tokens
* Private keys
* Sensitive credentials

---

# 37. Interview Answer

### Question

**How would you troubleshoot `CrashLoopBackOff` in Kubernetes?**

### Answer

> I would first check the Pod status and restart count using `kubectl get pods`. Then I would run `kubectl describe pod` and check the Events section. After that, I would check the current container logs and, more importantly, the previous container logs using `kubectl logs --previous`, because the previous container instance may contain the actual application failure.
>
> I would then check the container's exit code and termination reason. If it is `OOMKilled`, I would investigate memory usage and resource limits. If there is a liveness probe failure, I would verify the probe configuration, endpoint, port, and startup timing.
>
> Next, I would verify ConfigMaps, Secrets, environment variables, application startup commands, volume mounts, file permissions, and SecurityContext settings. I would also check dependencies such as databases, Redis, Kafka, external APIs, and DNS.
>
> Finally, I would verify the Docker image and CI/CD pipeline to make sure the correct application version and configuration were built and deployed. Once I identify the root cause, I would apply the smallest safe fix, monitor the Pod, and verify that the application remains healthy.

---

# 38. Quick Reference

```bash
# List Pods
kubectl get pods -n <namespace>

# Describe Pod
kubectl describe pod <pod> -n <namespace>

# Current logs
kubectl logs <pod> -n <namespace>

# Previous container logs
kubectl logs <pod> -n <namespace> --previous

# Specific container logs
kubectl logs <pod> \
  -c <container> \
  -n <namespace>

# Previous logs for specific container
kubectl logs <pod> \
  -c <container> \
  -n <namespace> \
  --previous

# Pod YAML
kubectl get pod <pod> -n <namespace> -o yaml

# Deployment YAML
kubectl get deployment <deployment> \
  -n <namespace> \
  -o yaml

# Container exit code
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.status.containerStatuses[*].lastState.terminated.exitCode}'

# Termination reason
kubectl get pod <pod> -n <namespace> \
  -o jsonpath='{.status.containerStatuses[*].lastState.terminated.reason}'

# Resource usage
kubectl top pod <pod> -n <namespace>

# ConfigMaps
kubectl get configmaps -n <namespace>

# Secrets
kubectl get secrets -n <namespace>

# Services
kubectl get svc -n <namespace>

# Endpoints
kubectl get endpoints -n <namespace>

# PVCs
kubectl get pvc -n <namespace>

# Nodes
kubectl get nodes -o wide
```

---

# 39. Summary

When troubleshooting `CrashLoopBackOff`, remember:

```text
1. Check Pod status
        ↓
2. Check restart count
        ↓
3. Check Pod Events
        ↓
4. Check current logs
        ↓
5. Check previous logs
        ↓
6. Check exit code
        ↓
7. Check termination reason
        ↓
8. Check configuration
        ↓
9. Check resources / OOMKilled
        ↓
10. Check probes
        ↓
11. Check dependencies
        ↓
12. Check permissions / volumes
        ↓
13. Check Docker image / CI-CD
        ↓
14. Apply fix
        ↓
15. Validate and monitor
```

## Golden Rule

> **For `CrashLoopBackOff`, don't immediately restart or delete the Pod. First capture the evidence: Pod Events, current logs, previous logs, exit code, and termination reason. Then determine whether the failure is caused by the application, configuration, resources, probes, dependencies, permissions, storage, or CI/CD.**
