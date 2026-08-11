# Kubernetes Production Troubleshooting Runbook: NetworkPolicy

A production troubleshooting runbook for diagnosing and resolving Kubernetes network connectivity issues caused by `NetworkPolicy`.

This runbook covers:

* Pod-to-Pod communication
* Pod-to-Service communication
* Pod-to-external communication
* Ingress traffic
* Egress traffic
* Namespace-based policies
* Label selectors
* `podSelector`
* `namespaceSelector`
* `ipBlock`
* DNS connectivity
* CNI implementation
* NetworkPolicy enforcement
* Ingress and Egress rules
* Policy conflicts
* Cloud/network dependencies
* Production troubleshooting and validation

---

# Table of Contents

* [1. Purpose](#1-purpose)
* [2. What is a NetworkPolicy](#2-what-is-a-networkpolicy)
* [3. NetworkPolicy Traffic Flow](#3-networkpolicy-traffic-flow)
* [4. Important NetworkPolicy Concepts](#4-important-networkpolicy-concepts)
* [5. Initial Investigation](#5-initial-investigation)
* [6. Check Pod Connectivity](#6-check-pod-connectivity)
* [7. Check Existing NetworkPolicies](#7-check-existing-networkpolicies)
* [8. Describe the NetworkPolicy](#8-describe-the-networkpolicy)
* [9. Check Pod Labels](#9-check-pod-labels)
* [10. Check Namespace Labels](#10-check-namespace-labels)
* [11. Check Ingress Rules](#11-check-ingress-rules)
* [12. Check Egress Rules](#12-check-egress-rules)
* [13. Check DNS Connectivity](#13-check-dns-connectivity)
* [14. Check Service Connectivity](#14-check-service-connectivity)
* [15. Check Endpoint and EndpointSlice](#15-check-endpoint-and-endpointslice)
* [16. Check CNI Plugin](#16-check-cni-plugin)
* [17. Check NetworkPolicy Support](#17-check-networkpolicy-support)
* [18. Check Namespace Isolation](#18-check-namespace-isolation)
* [19. Check ipBlock Rules](#19-check-ipblock-rules)
* [20. Check Ports and Protocols](#20-check-ports-and-protocols)
* [21. Check Ingress and Egress Policy Interaction](#21-check-ingress-and-egress-policy-interaction)
* [22. Check External Connectivity](#22-check-external-connectivity)
* [23. Check NetworkPolicy with LoadBalancers and Ingress](#23-check-networkpolicy-with-loadbalancers-and-ingress)
* [24. Check NetworkPolicy with Service Mesh](#24-check-networkpolicy-with-service-mesh)
* [25. Check CNI Logs](#25-check-cni-logs)
* [26. Common NetworkPolicy Errors](#26-common-networkpolicy-errors)
* [27. Production Troubleshooting Flow](#27-production-troubleshooting-flow)
* [28. Production Checklist](#28-production-checklist)
* [29. Fast Incident Investigation](#29-fast-incident-investigation)
* [30. Root Cause Classification](#30-root-cause-classification)
* [31. Best Practices](#31-best-practices)
* [32. Interview Answer](#32-interview-answer)
* [33. Quick Reference](#33-quick-reference)
* [34. Summary](#34-summary)

---

# 1. Purpose

This runbook provides a structured approach for troubleshooting Kubernetes network connectivity problems related to `NetworkPolicy`.

Typical symptoms include:

```text
Connection timeout
Connection refused
DNS resolution failure
HTTP 502
HTTP 503
HTTP 504
Cannot connect to another Pod
Cannot connect to a Service
Cannot connect to database
Cannot access external API
Cannot access DNS
```

The objective is to determine whether the problem is caused by:

```text
NetworkPolicy
   |
   +--> Pod selector
   +--> Namespace selector
   +--> IP block
   +--> Port
   +--> Protocol
   +--> Ingress rule
   +--> Egress rule
   +--> CNI enforcement
   +--> DNS policy
   +--> Service/Endpoint issue
   +--> External network issue
```

---

# 2. What is a NetworkPolicy?

A Kubernetes `NetworkPolicy` controls network traffic to and from Pods.

It can control:

* Ingress traffic
* Egress traffic

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
  namespace: production

spec:
  podSelector:
    matchLabels:
      app: backend

  policyTypes:
    - Ingress
    - Egress

  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080

  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
```

This example allows:

```text
frontend Pod
     |
     | TCP 8080
     v
backend Pod
```

and:

```text
backend Pod
     |
     | TCP 5432
     v
database Pod
```

---

# 3. NetworkPolicy Traffic Flow

A simplified traffic flow is:

```text
                  Client Pod
                      |
                      |
                 Network Request
                      |
                      v
              +---------------+
              | NetworkPolicy |
              +---------------+
                      |
             +--------+--------+
             |                 |
             v                 v
          Allowed            Denied
             |                 |
             v                 v
        Destination          Timeout /
           Pod              Connection
```

For a connection to succeed, you need to consider both sides.

For example:

```text
Frontend
   |
   | Egress
   v
Backend
   |
   | Ingress
   v
```

The destination's ingress policy and source's egress policy can both affect connectivity.

---

# 4. Important NetworkPolicy Concepts

## 4.1 `podSelector`

Selects Pods in the same Namespace as the NetworkPolicy.

Example:

```yaml
podSelector:
  matchLabels:
    app: backend
```

This selects Pods with:

```text
app=backend
```

---

## 4.2 `namespaceSelector`

Selects Namespaces based on Namespace labels.

Example:

```yaml
namespaceSelector:
  matchLabels:
    environment: production
```

The Namespace must actually have:

```text
environment=production
```

Check:

```bash
kubectl get namespaces --show-labels
```

---

## 4.3 `ipBlock`

Allows traffic from or to specific IP ranges.

Example:

```yaml
ipBlock:
  cidr: 10.10.0.0/16
```

You can exclude a smaller range:

```yaml
ipBlock:
  cidr: 10.10.0.0/16
  except:
    - 10.10.10.0/24
```

Be careful when using IP ranges because Pod IPs can change.

---

# 5. Initial Investigation

When an application cannot communicate with another service, first identify:

```text
Source Pod
Destination Pod/Service
Destination Port
Protocol
Namespace
Expected traffic direction
```

Example:

```text
Source:
frontend-7d8f8c

Destination:
backend.production.svc.cluster.local

Port:
8080

Protocol:
TCP
```

Then test connectivity from the source Pod.

---

# 6. Check Pod Connectivity

Get the source Pod:

```bash
kubectl get pod <source-pod> \
  -n <source-namespace> \
  -o wide
```

Test DNS:

```bash
kubectl exec -it <source-pod> \
  -n <source-namespace> \
  -- nslookup <service-name>
```

Test TCP connectivity:

```bash
kubectl exec -it <source-pod> \
  -n <source-namespace> \
  -- nc -vz <service-name> <port>
```

Example:

```bash
kubectl exec -it frontend-0 \
  -n production \
  -- nc -vz backend 8080
```

If `nc` is not available, use another appropriate diagnostic utility such as `curl` where applicable.

Example:

```bash
kubectl exec -it frontend-0 \
  -n production \
  -- curl -v http://backend:8080/health
```

---

# 7. Check Existing NetworkPolicies

List NetworkPolicies:

```bash
kubectl get networkpolicy -A
```

For a specific Namespace:

```bash
kubectl get networkpolicy \
  -n production
```

Example:

```text
NAME              POD-SELECTOR   AGE
backend-policy    app=backend    30d
default-deny      <none>         30d
```

This is one of the first things to check when connectivity suddenly stops after a policy deployment.

---

# 8. Describe the NetworkPolicy

Run:

```bash
kubectl describe networkpolicy <policy-name> \
  -n <namespace>
```

Example:

```bash
kubectl describe networkpolicy backend-policy \
  -n production
```

Check:

```text
PodSelector
Policy Types
Ingress
Egress
Ports
From
To
```

---

# 9. Check Pod Labels

NetworkPolicies depend heavily on labels.

Check source Pod:

```bash
kubectl get pod <source-pod> \
  -n <namespace> \
  --show-labels
```

Check destination Pod:

```bash
kubectl get pod <destination-pod> \
  -n <namespace> \
  --show-labels
```

Example:

```text
app=frontend
```

and:

```text
app=backend
```

If the policy expects:

```yaml
matchLabels:
  app: backend
```

but the destination Pod actually has:

```text
app=backend-v2
```

the policy will not select it.

---

# 10. Check Namespace Labels

If the policy uses `namespaceSelector`, check Namespace labels.

Run:

```bash
kubectl get namespace \
  --show-labels
```

Example:

```text
NAME         LABELS
production   environment=production
staging      environment=staging
```

If the policy contains:

```yaml
namespaceSelector:
  matchLabels:
    environment: production
```

the destination/source Namespace must have the expected label.

---

# 11. Check Ingress Rules

Ingress controls traffic **coming into the selected Pods**.

Example:

```yaml
ingress:
  - from:
      - podSelector:
          matchLabels:
            app: frontend

    ports:
      - protocol: TCP
        port: 8080
```

This means:

```text
frontend
   |
   | TCP 8080
   v
backend
```

is allowed, assuming the relevant egress side also permits the traffic where applicable.

If traffic comes from:

```text
app=payments
```

instead of:

```text
app=frontend
```

the traffic may be denied.

---

# 12. Check Egress Rules

Egress controls traffic **leaving the selected Pods**.

Example:

```yaml
egress:
  - to:
      - podSelector:
          matchLabels:
            app: database

    ports:
      - protocol: TCP
        port: 5432
```

This allows:

```text
backend
   |
   | TCP 5432
   v
database
```

If Egress policies isolate the source Pod and do not allow the destination, the connection can fail.

---

# 13. Check DNS Connectivity

DNS is one of the most commonly overlooked requirements when using Egress NetworkPolicies.

A Pod may need to reach the Kubernetes DNS service, usually CoreDNS, over:

```text
UDP 53
TCP 53
```

If Egress is restricted, DNS traffic must be explicitly allowed.

Example:

```yaml
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

The exact CoreDNS labels can vary by Kubernetes distribution. Verify them in your cluster:

```bash
kubectl get pods \
  -n kube-system \
  --show-labels
```

Test DNS:

```bash
kubectl exec -it <pod> \
  -n <namespace> \
  -- nslookup kubernetes.default.svc
```

Also test:

```bash
kubectl exec -it <pod> \
  -n <namespace> \
  -- nslookup <service>.<namespace>.svc.cluster.local
```

---

# 14. Check Service Connectivity

A NetworkPolicy applies to Pods, not directly to a Kubernetes Service.

If an application connects to:

```text
backend.production.svc.cluster.local
```

the request ultimately reaches backend Pods selected by the Service.

Check the Service:

```bash
kubectl get svc backend \
  -n production
```

Describe it:

```bash
kubectl describe svc backend \
  -n production
```

Check:

```text
Selector
Port
TargetPort
Endpoints
```

---

# 15. Check Endpoint and EndpointSlice

If the Service has no endpoints, NetworkPolicy may not be the root cause.

Check:

```bash
kubectl get endpoints backend \
  -n production
```

Also check EndpointSlices:

```bash
kubectl get endpointslice \
  -n production
```

Example:

```text
NAME             ENDPOINTS
backend          10.10.2.15:8080
                 10.10.3.20:8080
```

If there are no endpoints:

```text
Service
   |
   X
   |
No matching Pods
```

Investigate:

* Service selector
* Pod labels
* Pod readiness
* EndpointSlice configuration

before changing NetworkPolicy.

---

# 16. Check CNI Plugin

NetworkPolicy enforcement is implemented by the cluster networking layer/CNI.

Common CNI implementations include:

```text
Calico
Cilium
Antrea
Azure CNI
AWS VPC CNI
Other vendor-specific implementations
```

Check the cluster's networking components:

```bash
kubectl get pods -n kube-system
```

Look for CNI-related Pods.

Examples:

```text
calico-node
cilium
antrea-agent
aws-node
```

The exact components depend on the Kubernetes distribution.

---

# 17. Check NetworkPolicy Support

Not every networking implementation historically supported NetworkPolicy in the same way.

Verify that your CNI supports and enforces Kubernetes NetworkPolicy.

Check:

```bash
kubectl get pods -n kube-system
```

Then consult your CNI's documentation and logs.

Important:

> Creating a NetworkPolicy object does not guarantee that traffic is being enforced unless the cluster's networking implementation supports NetworkPolicy.

---

# 18. Check Namespace Isolation

A common production configuration is:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

This selects all Pods in the Namespace.

It effectively establishes a default isolation posture for those Pods.

Once such a policy exists, additional policies must explicitly allow the required traffic.

Example:

```text
Default Deny
     |
     +--> Frontend -> Backend
     |       Allowed?
     |
     +--> Backend -> Database
     |       Allowed?
     |
     +--> Pod -> DNS
     |       Allowed?
     |
     +--> Pod -> External API
             Allowed?
```

---

# 19. Check ipBlock Rules

Example:

```yaml
egress:
  - to:
      - ipBlock:
          cidr: 10.20.0.0/16
```

Make sure the destination IP actually belongs to that CIDR.

Test:

```bash
kubectl exec -it <pod> \
  -n <namespace> \
  -- getent hosts <hostname>
```

Then determine the resolved IP.

Be careful when using:

```yaml
ipBlock:
```

with:

* Pod CIDRs
* Service CIDRs
* Node IPs
* NAT addresses
* Cloud load balancer addresses

The actual packet source/destination as seen by the network enforcement layer can differ depending on the cluster networking implementation and traffic path.

---

# 20. Check Ports and Protocols

A policy can allow the correct destination but the wrong port.

Example:

```yaml
ports:
  - protocol: TCP
    port: 8080
```

But the application is actually listening on:

```text
9090
```

The connection will fail.

Check the application:

```bash
kubectl get svc <service-name> \
  -n <namespace>
```

Check Pod ports:

```bash
kubectl get pod <pod-name> \
  -n <namespace> \
  -o yaml
```

Test the actual port:

```bash
kubectl exec -it <source-pod> \
  -n <namespace> \
  -- nc -vz <destination> 8080
```

Remember that:

```text
Service port
      |
      v
TargetPort
      |
      v
Container listening port
```

must be configured correctly.

---

# 21. Check Ingress and Egress Policy Interaction

Consider:

```text
Frontend Pod
     |
     | Egress
     v
Backend Pod
     |
     | Ingress
     v
```

If the frontend has an Egress policy that blocks the backend, connectivity fails.

If the backend has an Ingress policy that blocks the frontend, connectivity also fails.

A useful troubleshooting model is:

```text
Source Pod
    |
    | Egress
    v
Network
    |
    | Ingress
    v
Destination Pod
```

Check both sides when applicable.

---

# 22. Check External Connectivity

If a Pod cannot reach an external API:

```text
Pod
 |
 +--> NetworkPolicy
 |
 +--> CNI
 |
 +--> Node networking
 |
 +--> NAT / Gateway
 |
 +--> Firewall
 |
 +--> Internet
 |
 +--> External API
```

First test DNS:

```bash
kubectl exec -it <pod> \
  -n <namespace> \
  -- nslookup api.example.com
```

Then test HTTPS:

```bash
kubectl exec -it <pod> \
  -n <namespace> \
  -- curl -v https://api.example.com
```

If DNS works but HTTPS does not, investigate:

* Egress NetworkPolicy
* CNI
* Node routing
* NAT Gateway
* Firewall
* Proxy
* Cloud security controls
* External endpoint availability

---

# 23. Check NetworkPolicy with LoadBalancers and Ingress

When traffic originates outside the cluster:

```text
Internet
   |
   v
LoadBalancer
   |
   v
Ingress / Service
   |
   v
Pod
```

The source IP visible to NetworkPolicy enforcement may not always be the original client IP.

Depending on the networking architecture, traffic can pass through:

* Load balancer
* Ingress controller
* Node
* kube-proxy
* CNI
* NAT

Therefore, do not blindly add the external client's IP to an `ipBlock`.

First determine the actual source address seen by the relevant enforcement layer.

---

# 24. Check NetworkPolicy with Service Mesh

If using a service mesh such as:

```text
Istio
Linkerd
Other service-mesh implementations
```

traffic may pass through sidecar or node-level proxies.

The actual traffic path can become:

```text
Application
    |
    v
Sidecar Proxy
    |
    v
Network
    |
    v
Sidecar Proxy
    |
    v
Destination Application
```

When troubleshooting, determine:

* Which port the application uses
* Which port the sidecar uses
* Whether traffic is redirected
* Whether mTLS is enabled
* Whether NetworkPolicy allows the required traffic

Do not modify NetworkPolicy until you understand the actual traffic path.

---

# 25. Check CNI Logs

If the NetworkPolicy configuration looks correct but traffic is still blocked, investigate the CNI.

Find CNI Pods:

```bash
kubectl get pods -n kube-system
```

For example, if using Calico:

```bash
kubectl get pods -n kube-system | grep calico
```

If using Cilium:

```bash
kubectl get pods -n kube-system | grep cilium
```

Then inspect the relevant logs according to the CNI implementation.

Example:

```bash
kubectl logs <cni-pod> \
  -n kube-system
```

Check for:

```text
Policy errors
Datapath errors
Endpoint errors
iptables/eBPF errors
Connectivity errors
Configuration errors
```

---

# 26. Common NetworkPolicy Errors

| Symptom / Error                               | Likely Cause                                                |
| --------------------------------------------- | ----------------------------------------------------------- |
| Pod cannot connect to another Pod             | Ingress/Egress policy                                       |
| Pod cannot connect to Service                 | Policy or Service/Endpoint issue                            |
| DNS fails                                     | Egress policy blocks DNS                                    |
| `Connection timed out`                        | Traffic may be dropped by policy/firewall                   |
| `Connection refused`                          | Application may not be listening or Service target is wrong |
| HTTP `503`                                    | Backend unreachable/unhealthy                               |
| HTTP `504`                                    | Upstream connection timeout                                 |
| External API unavailable                      | Egress policy/NAT/firewall                                  |
| Policy has no effect                          | CNI may not enforce NetworkPolicy                           |
| Policy blocks unexpected Pods                 | Incorrect selector                                          |
| Namespace traffic blocked                     | Incorrect namespace selector                                |
| Correct port but connection fails             | Protocol/selector/policy issue                              |
| Service has no endpoints                      | Service selector/readiness issue                            |
| Policy works in one Namespace but not another | Namespace labels/policy differences                         |

---

# 27. Production Troubleshooting Flow

```text
                 Application Connectivity Issue
                              |
                              v
                    Identify Source Pod
                              |
                              v
                    Identify Destination
                              |
                              v
                  Check Service / Endpoint
                              |
                    +---------+---------+
                    |                   |
                  Valid               Invalid
                    |                   |
                    v                   v
              Check NetworkPolicy   Fix Service /
                    |                Endpoint issue
                    v
            Check Source Pod Egress
                    |
                    v
          Check Destination Ingress
                    |
                    v
             Check Pod Labels
                    |
                    v
          Check Namespace Labels
                    |
                    v
             Check Ports / Protocol
                    |
                    v
                 Check DNS
                    |
                    v
              Check CNI Plugin
                    |
                    v
              Check CNI Logs
                    |
                    v
          Check External Networking
                    |
                    v
             Validate Connectivity
```

---

# 28. Production Checklist

## Source Pod

* [ ] Source Pod exists
* [ ] Source Pod is Running
* [ ] Source Pod labels verified
* [ ] Source Namespace verified
* [ ] Egress policy checked
* [ ] DNS connectivity tested
* [ ] TCP/HTTP connectivity tested

## Destination Pod

* [ ] Destination Pod exists
* [ ] Destination Pod is Running
* [ ] Destination Pod labels verified
* [ ] Ingress policy checked
* [ ] Application port verified

## Service

* [ ] Service exists
* [ ] Service selector verified
* [ ] Service port verified
* [ ] TargetPort verified
* [ ] Endpoints verified
* [ ] EndpointSlices verified

## NetworkPolicy

* [ ] All NetworkPolicies listed
* [ ] Policy selectors verified
* [ ] Ingress rules verified
* [ ] Egress rules verified
* [ ] Namespace selectors verified
* [ ] `ipBlock` rules verified
* [ ] Ports verified
* [ ] Protocols verified
* [ ] `policyTypes` verified

## DNS

* [ ] DNS resolution works
* [ ] CoreDNS Pods healthy
* [ ] UDP 53 allowed
* [ ] TCP 53 allowed where required
* [ ] DNS service address reachable

## CNI

* [ ] CNI Pods healthy
* [ ] CNI configuration verified
* [ ] NetworkPolicy enforcement supported
* [ ] CNI logs checked
* [ ] No datapath errors

## External Connectivity

* [ ] External DNS works
* [ ] Egress NetworkPolicy allows destination
* [ ] NAT/Gateway verified
* [ ] Firewall rules verified
* [ ] Proxy configuration verified
* [ ] External endpoint reachable

---

# 29. Fast Incident Investigation

Use this sequence during a production incident.

### Step 1 — List NetworkPolicies

```bash
kubectl get networkpolicy -A
```

### Step 2 — Describe the Policy

```bash
kubectl describe networkpolicy \
  <policy-name> \
  -n <namespace>
```

### Step 3 — Check Source Pod

```bash
kubectl get pod <source-pod> \
  -n <namespace> \
  --show-labels
```

### Step 4 — Check Destination Pod

```bash
kubectl get pod <destination-pod> \
  -n <namespace> \
  --show-labels
```

### Step 5 — Check Namespace Labels

```bash
kubectl get namespaces --show-labels
```

### Step 6 — Check Service

```bash
kubectl get svc <service> \
  -n <namespace>
```

### Step 7 — Check Endpoints

```bash
kubectl get endpoints <service> \
  -n <namespace>
```

### Step 8 — Test DNS

```bash
kubectl exec -it <source-pod> \
  -n <namespace> \
  -- nslookup <service>
```

### Step 9 — Test Port

```bash
kubectl exec -it <source-pod> \
  -n <namespace> \
  -- nc -vz <service> <port>
```

### Step 10 — Test Application

```bash
kubectl exec -it <source-pod> \
  -n <namespace> \
  -- curl -v http://<service>:<port>
```

### Step 11 — Check CNI

```bash
kubectl get pods -n kube-system
```

### Step 12 — Check Events

```bash
kubectl get events -A \
  --sort-by='.lastTimestamp'
```

---

# 30. Root Cause Classification

## Category 1 — Policy Selector

```text
Incorrect podSelector
Incorrect namespaceSelector
Incorrect labels
Missing labels
```

## Category 2 — Ingress

```text
Ingress traffic denied
Incorrect source selector
Incorrect port
Incorrect protocol
```

## Category 3 — Egress

```text
Egress traffic denied
External destination not allowed
DNS traffic blocked
Database traffic blocked
```

## Category 4 — DNS

```text
DNS traffic blocked
CoreDNS unavailable
Incorrect DNS configuration
```

## Category 5 — Service

```text
Incorrect Service selector
No endpoints
Incorrect targetPort
Application not listening
```

## Category 6 — CNI

```text
CNI failure
NetworkPolicy enforcement problem
Datapath issue
CNI configuration problem
```

## Category 7 — External Network

```text
NAT failure
Firewall
Cloud security controls
Proxy
Routing
External service failure
```

---

# 31. Best Practices

## 31.1 Start with Default Deny

For production environments, consider a deliberate default-deny strategy where appropriate.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: production

spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

Then explicitly allow required traffic.

---

## 31.2 Always Allow Required DNS

When using Egress restrictions, ensure application Pods can reach the cluster DNS service.

Otherwise:

```text
Application
    |
    v
DNS lookup
    |
    X
NetworkPolicy
    |
    v
Application fails
```

---

## 31.3 Use Meaningful Labels

NetworkPolicy depends heavily on labels.

Use consistent labels such as:

```text
app
component
environment
team
tier
```

Example:

```yaml
labels:
  app: payment
  component: backend
  environment: production
```

---

## 31.4 Avoid Overly Broad `ipBlock`

Avoid unnecessarily broad rules such as:

```yaml
cidr: 0.0.0.0/0
```

unless there is a documented business requirement.

Broad rules can defeat the purpose of network segmentation.

---

## 31.5 Review Policy Changes Carefully

Treat NetworkPolicy changes as production network changes.

Before deployment, verify:

```text
Source
Destination
Port
Protocol
Namespace
Expected traffic
```

A small selector change can affect many applications.

---

## 31.6 Test Before Production

Test NetworkPolicies in a lower environment.

Validate:

```text
Allowed traffic
Denied traffic
DNS
Service communication
Database connectivity
External API connectivity
Monitoring
Logging
```

---

## 31.7 Monitor NetworkPolicy Changes

Track:

```text
Policy creation
Policy modification
Policy deletion
Application connectivity failures
CNI errors
```

NetworkPolicy changes should be included in deployment and audit processes.

---

# 32. Interview Answer

### Question

**How would you troubleshoot a Kubernetes application that cannot communicate with another Pod because of NetworkPolicy?**

### Answer

> First, I would identify the source Pod, destination Pod or Service, destination port, protocol, and Namespaces.
>
> Then I would check the Service and EndpointSlices to make sure the traffic is actually reaching the expected destination Pods.
>
> Next, I would list and describe the NetworkPolicies in both relevant Namespaces. I would verify the `podSelector`, `namespaceSelector`, `ipBlock`, ports, protocols, and `policyTypes`.
>
> I would check the source Pod's Egress policy and the destination Pod's Ingress policy. I would also verify that the Pod and Namespace labels match the selectors defined in the policies.
>
> If DNS is failing, I would verify that Egress traffic to the cluster DNS service is allowed, normally on UDP/TCP port 53 as required.
>
> If the NetworkPolicy configuration looks correct, I would investigate the CNI implementation and its logs to confirm that NetworkPolicy is actually being enforced correctly.
>
> For external connectivity, I would additionally check NAT, routing, firewall rules, proxies, and cloud networking.
>
> Finally, I would test the connection from the source Pod using DNS lookup, TCP connectivity, and an application-level request such as `curl`, and then identify the exact policy or networking layer responsible for the failure.

---

# 33. Quick Reference

```bash
# List NetworkPolicies
kubectl get networkpolicy -A

# List policies in a Namespace
kubectl get networkpolicy -n <namespace>

# Describe NetworkPolicy
kubectl describe networkpolicy <policy> -n <namespace>

# Get NetworkPolicy YAML
kubectl get networkpolicy <policy> \
  -n <namespace> \
  -o yaml

# Check Pod labels
kubectl get pod <pod> \
  -n <namespace> \
  --show-labels

# Check Namespace labels
kubectl get namespaces --show-labels

# Check Services
kubectl get svc -n <namespace>

# Describe Service
kubectl describe svc <service> -n <namespace>

# Check Endpoints
kubectl get endpoints <service> -n <namespace>

# Check EndpointSlices
kubectl get endpointslice -n <namespace>

# Check CNI / networking Pods
kubectl get pods -n kube-system

# Check DNS Pods
kubectl get pods -n kube-system | grep -i dns

# Test DNS
kubectl exec -it <pod> \
  -n <namespace> \
  -- nslookup <service>

# Test DNS using FQDN
kubectl exec -it <pod> \
  -n <namespace> \
  -- nslookup <service>.<namespace>.svc.cluster.local

# Test TCP connectivity
kubectl exec -it <pod> \
  -n <namespace> \
  -- nc -vz <destination> <port>

# Test HTTP
kubectl exec -it <pod> \
  -n <namespace> \
  -- curl -v http://<destination>:<port>

# Check Events
kubectl get events -A \
  --sort-by='.lastTimestamp'
```

---

# 34. Summary

For Kubernetes NetworkPolicy troubleshooting, follow the traffic path:

```text
Source Pod
    |
    v
Source Egress Policy
    |
    v
CNI / Network
    |
    v
Destination Ingress Policy
    |
    v
Destination Pod
```

For Service-based traffic:

```text
Source Pod
    |
    v
DNS
    |
    v
Service
    |
    v
EndpointSlice
    |
    v
Destination Pod
```

For external traffic:

```text
Source Pod
    |
    v
Egress NetworkPolicy
    |
    v
CNI
    |
    v
Node / NAT / Gateway
    |
    v
Firewall / Internet
    |
    v
External Service
```

The key questions are:

```text
1. Who is sending the traffic?
2. Who is receiving the traffic?
3. Which Namespace are they in?
4. What are the Pod labels?
5. Is there an Egress policy on the source?
6. Is there an Ingress policy on the destination?
7. Do the selectors match?
8. Is the correct port allowed?
9. Is the correct protocol allowed?
10. Is DNS allowed?
11. Does the Service have endpoints?
12. Is the CNI enforcing NetworkPolicy?
13. Are CNI components healthy?
14. Is external traffic allowed?
15. Are cloud/network firewalls involved?
```

## Golden Rule

> **When troubleshooting Kubernetes NetworkPolicy, never look at only one policy. Trace the complete traffic path: Source Pod → Egress → CNI/Network → Destination Ingress → Destination Pod. Also verify DNS, Services, EndpointSlices, ports, protocols, Pod labels, Namespace labels, and the CNI implementation before changing the policy.**
