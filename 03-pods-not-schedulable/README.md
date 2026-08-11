In Kubernetes, the scheduler is responsible for assigning pods to nodes in the cluster based on various criteria. Sometimes, you might encounter situations where pods are not being scheduled as expected. This can happen due to factors such as node constraints, pod requirements, or cluster configurations.

1. Node Selector

Node Selector is a simple way to constrain pods to nodes with specific labels. It allows you to specify a set of key-value pairs that must match the node's labels for a pod to be scheduled on that node.
Usage: Include a nodeSelector field in the pod's YAML definition to specify the required labels.

```
spec:
    containers:
    - name: my-app
    image: my-image
    nodeSelector:
    disktype: ssd
```

2. Node Affinity

Node Affinity is a more expressive way to specify rules about the placement of pods relative to nodes' labels. It allows you to specify rules that apply only if certain conditions are met.
Usage: Define nodeAffinity rules in the pod's YAML definition, specifying required and preferred node selectors.

```
spec:
    containers:
    - name: my-app
    image: my-image
    affinity:
    nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
            - key: disktype
            operator: In
            values:
            - ssd
```

3. Taints

Taints are applied to nodes to repel certain pods. They allow nodes to refuse pods unless the pods have a matching toleration.
Usage: Use kubectl taint command to apply taints to nodes. Include tolerations field in the pod's YAML definition to tolerate specific taints.

```
kubectl taint nodes node1 disktype=ssd:NoSchedule
```

```
spec:
    containers:
    - name: my-app
    image: my-image
    tolerations:
    - key: disktype
      operator: Equal
      value: ssd
      effect: NoSchedule
```

4. Tolerations

Tolerations are applied to pods and allow them to schedule onto nodes with matching taints. They override the effect of taints.

Usage: Include tolerations field in the pod's YAML definition to specify which taints the pod tolerates.

```
spec:
  containers:
  - name: my-app
    image: my-image
  tolerations:
  - key: disktype
    operator: Equal
    value: ssd
    effect: NoSchedule
```

# Kubernetes Production Troubleshooting Runbook: Pod Not Scheduled

A practical production troubleshooting runbook for diagnosing and resolving Kubernetes Pods that remain in `Pending` because the Kubernetes Scheduler cannot assign them to a suitable Node.

---

# Table of Contents

* [1. Purpose](#1-purpose)
* [2. What Does Pod Not Scheduled Mean](#2-what-does-pod-not-scheduled-mean)
* [3. Pod Scheduling Flow](#3-pod-scheduling-flow)
* [4. Initial Pod Investigation](#4-initial-pod-investigation)
* [5. Check Pod Events](#5-check-pod-events)
* [6. Check Scheduler Status](#6-check-scheduler-status)
* [7. Check Node Availability](#7-check-node-availability)
* [8. Check Node Conditions](#8-check-node-conditions)
* [9. Check CPU and Memory Resources](#9-check-cpu-and-memory-resources)
* [10. Check Resource Requests](#10-check-resource-requests)
* [11. Check Taints and Tolerations](#11-check-taints-and-tolerations)
* [12. Check Node Labels](#12-check-node-labels)
* [13. Check NodeSelector](#13-check-nodeselector)
* [14. Check Node Affinity](#14-check-node-affinity)
* [15. Check Pod Affinity and Anti-Affinity](#15-check-pod-affinity-and-anti-affinity)
* [16. Check Topology Spread Constraints](#16-check-topology-spread-constraints)
* [17. Check Pod Capacity and Limits](#17-check-pod-capacity-and-limits)
* [18. Check Namespace ResourceQuota](#18-check-namespace-resourcequota)
* [19. Check LimitRange](#19-check-limitrange)
* [20. Check PersistentVolume and PVC](#20-check-persistentvolume-and-pvc)
* [21. Check StorageClass](#21-check-storageclass)
* [22. Check Volume Topology](#22-check-volume-topology)
* [23. Check Pod Security and Admission Policies](#23-check-pod-security-and-admission-policies)
* [24. Check DaemonSet and System Resource Consumption](#24-check-daemonset-and-system-resource-consumption)
* [25. Check Cluster Autoscaler](#25-check-cluster-autoscaler)
* [26. Check Node Groups](#26-check-node-groups)
* [27. Check Scheduler Logs](#27-check-scheduler-logs)
* [28. Common Scheduling Errors](#28-common-scheduling-errors)
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

This runbook provides a structured approach for troubleshooting Kubernetes Pods that remain in:

```text
Pending
```

because the Kubernetes Scheduler cannot find a suitable Node.

It covers common scheduling problems involving:

* CPU and memory
* Node availability
* Node conditions
* Taints and tolerations
* Node selectors
* Node affinity
* Pod affinity
* Pod anti-affinity
* Topology spread constraints
* ResourceQuota
* LimitRange
* PersistentVolumeClaims
* StorageClasses
* Volume topology
* Cluster Autoscaler
* Node groups
* Scheduler configuration
* Admission policies

---

# 2. What Does Pod Not Scheduled Mean?

A Pod is **not scheduled** when Kubernetes has created the Pod object but the Scheduler has not assigned the Pod to a Node.

The Pod commonly remains in:

```text
Pending
```

Example:

```text
NAME                    READY   STATUS    RESTARTS   AGE
payment-api-7d8f9c8b7f  0/1     Pending   0          5m
```

This is different from:

```text
ImagePullBackOff
```

and:

```text
CrashLoopBackOff
```

### Important distinction

```text
Pending
   |
   +--> Pod has not been successfully scheduled
         |
         +--> Scheduler / resource / placement problem


ImagePullBackOff
   |
   +--> Pod is scheduled
         |
         +--> Node cannot pull the image


CrashLoopBackOff
   |
   +--> Pod is scheduled and container started
         |
         +--> Container repeatedly crashes
```

---

# 3. Pod Scheduling Flow

A simplified scheduling flow looks like this:

```text
                 Pod Created
                      |
                      v
              Kubernetes API
                      |
                      v
                  Scheduler
                      |
          +-----------+-----------+
          |                       |
          v                       v
    Find Suitable Nodes      No Suitable Node
          |                       |
          v                       v
      Select Node              Pod Pending
          |
          v
      Bind Pod
          |
          v
     Kubelet Starts Pod
```

The Scheduler evaluates factors such as:

* CPU availability
* Memory availability
* Taints
* Tolerations
* Node labels
* Node selectors
* Node affinity
* Pod affinity
* Pod anti-affinity
* Topology constraints
* Storage constraints
* Resource quotas
* Other scheduling policies

---

# 4. Initial Pod Investigation

## 4.1 Check Pod Status

```bash
kubectl get pods -n <namespace>
```

Example:

```bash
kubectl get pods -n production
```

Example:

```text
NAME                    READY   STATUS    RESTARTS   AGE
payment-api-7d8f9c8b7f  0/1     Pending   0          10m
```

---

# 5. Check Pod Events

The most important first troubleshooting command is:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Example:

```bash
kubectl describe pod payment-api-7d8f9c8b7f -n production
```

Look at the bottom:

```text
Events:
```

You may see:

```text
Warning  FailedScheduling
0/5 nodes are available:
3 Insufficient cpu,
2 Insufficient memory.
```

This message is extremely useful because it tells you why the Scheduler rejected the available Nodes.

---

# 6. Check Scheduler Status

Get the Pod's scheduling information:

```bash
kubectl get pod <pod-name> \
  -n <namespace> \
  -o wide
```

For a Pod that has not been scheduled, the `NODE` field may be empty.

Example:

```text
NAME           READY   STATUS    IP     NODE
payment-api    0/1     Pending          <none>
```

Check the scheduler-related events:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Look for:

```text
FailedScheduling
```

---

# 7. Check Node Availability

List Nodes:

```bash
kubectl get nodes
```

Example:

```text
NAME              STATUS   ROLES    AGE
worker-node-01    Ready    <none>   20d
worker-node-02    Ready    <none>   20d
worker-node-03    Ready    <none>   20d
```

A healthy cluster should normally have available worker Nodes.

Check more details:

```bash
kubectl get nodes -o wide
```

---

# 8. Check Node Conditions

Inspect a Node:

```bash
kubectl describe node <node-name>
```

Check:

```text
Conditions:
```

Important conditions include:

```text
Ready
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
```

A healthy Node generally has:

```text
Ready              True
MemoryPressure     False
DiskPressure       False
PIDPressure        False
```

If a Node is:

```text
Ready: False
```

or:

```text
Ready: Unknown
```

the Scheduler may not consider it suitable.

---

# 9. Check CPU and Memory Resources

One of the most common scheduling problems is insufficient CPU or memory.

Run:

```bash
kubectl describe node <node-name>
```

Look at:

```text
Allocatable:
```

and:

```text
Allocated resources:
```

Example:

```text
Allocatable:
  cpu:     4
  memory:  16Gi

Allocated resources:
  cpu:     3900m
  memory:  15Gi
```

If the Pod requests:

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "2Gi"
```

the Node may not have enough allocatable resources to schedule it.

---

# 10. Check Resource Requests

Inspect the Deployment:

```bash
kubectl get deployment <deployment-name> \
  -n <namespace> \
  -o yaml
```

Look for:

```yaml
resources:
  requests:
    cpu: "1"
    memory: "2Gi"

  limits:
    cpu: "2"
    memory: "4Gi"
```

### Important concept

The Scheduler primarily uses **resource requests**, not actual current CPU/memory usage, when determining whether a Pod can fit on a Node.

For example:

```text
Node allocatable CPU:
4 CPU

Existing Pod requests:
3.5 CPU

New Pod request:
1 CPU
```

Even if the existing Pods are currently using only 1 CPU, the Scheduler may still reject the new Pod because:

```text
3.5 + 1 = 4.5 CPU
```

which exceeds the Node's allocatable CPU.

---

# 11. Check Taints and Tolerations

A Node can have a taint that prevents Pods from being scheduled there unless they have a matching toleration.

Check Node taints:

```bash
kubectl describe node <node-name>
```

Look for:

```text
Taints:
```

Example:

```text
Taints: dedicated=database:NoSchedule
```

This means normal Pods cannot be scheduled onto that Node.

A Pod would need a matching toleration:

```yaml
tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"
```

---

# 12. Taint Effects

There are three common taint effects:

### `NoSchedule`

New Pods without a matching toleration will not be scheduled.

```text
Pod
 |
 X
 |
Node
```

### `PreferNoSchedule`

The Scheduler tries to avoid placing the Pod on the Node but does not strictly prohibit it.

### `NoExecute`

Pods that do not tolerate the taint can be evicted from the Node, and new Pods without tolerations cannot be scheduled there.

---

# 13. Check Node Labels

List Node labels:

```bash
kubectl get nodes --show-labels
```

Example:

```text
worker-node-01
  environment=production
  workload=application
```

You can inspect a specific Node:

```bash
kubectl get node <node-name> --show-labels
```

---

# 14. Check NodeSelector

A Pod may require a specific Node label.

Example:

```yaml
nodeSelector:
  workload: application
```

This means the Pod can only be scheduled on Nodes with:

```text
workload=application
```

Check Nodes:

```bash
kubectl get nodes \
  -l workload=application
```

If no Nodes are returned, the Pod cannot be scheduled.

---

# 15. Check Node Affinity

Node affinity provides more advanced placement rules.

Example:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: workload
              operator: In
              values:
                - application
```

The Scheduler must find a Node satisfying the required affinity.

Check:

```bash
kubectl get nodes --show-labels
```

Verify that the required labels actually exist.

---

# 16. Required vs Preferred Affinity

### Required

```text
requiredDuringSchedulingIgnoredDuringExecution
```

This is a hard requirement.

If no Node satisfies the rule:

```text
Pod remains Pending
```

### Preferred

```text
preferredDuringSchedulingIgnoredDuringExecution
```

This is a preference.

The Scheduler attempts to satisfy it but can schedule the Pod elsewhere if necessary.

---

# 17. Check Pod Affinity and Anti-Affinity

Pod affinity can require a Pod to run close to other Pods.

Example:

```yaml
affinity:
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: redis
        topologyKey: kubernetes.io/hostname
```

If there is no Node satisfying the affinity rule, the Pod may remain Pending.

---

# 18. Check Pod Anti-Affinity

Pod anti-affinity can prevent multiple replicas from running on the same Node.

Example:

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: payment-api
        topologyKey: kubernetes.io/hostname
```

If there are more replicas than available suitable Nodes, additional Pods may remain Pending.

Example:

```text
3 replicas
+
3 available Nodes
=
Can schedule

5 replicas
+
3 available Nodes
=
2 Pods may remain Pending
```

---

# 19. Check Topology Spread Constraints

Topology spread constraints help distribute Pods across:

* Nodes
* Availability Zones
* Regions
* Other topology domains

Example:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: payment-api
```

If the topology requirements cannot be satisfied, the Pod may remain Pending.

Check the Pod:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Look for:

```text
FailedScheduling
```

and topology-related scheduling messages.

---

# 20. Check Pod Capacity and Limits

A Node has finite capacity.

Check:

```bash
kubectl describe node <node-name>
```

Important fields:

```text
Capacity
Allocatable
Allocated resources
```

Example:

```text
Capacity:
  cpu:     8
  memory:  32Gi

Allocatable:
  cpu:     7.5
  memory:  30Gi
```

Remember that Kubernetes system components also consume resources.

Therefore:

```text
Node Capacity
      |
      +--> Kubernetes system components
      |
      +--> DaemonSets
      |
      +--> Existing workloads
      |
      +--> Available capacity for new Pods
```

---

# 21. Check Namespace ResourceQuota

A Namespace may have a ResourceQuota.

List quotas:

```bash
kubectl get resourcequota -n <namespace>
```

Example:

```text
NAME              AGE
production-quota  20d
```

Describe:

```bash
kubectl describe resourcequota production-quota \
  -n production
```

Example:

```text
Resource       Used     Hard
--------       ----     ----
requests.cpu   7        8
requests.memory 14Gi    16Gi
pods           48       50
```

If the Namespace has reached its quota, new workloads may not be admitted or may fail to obtain the requested resources.

---

# 22. Check LimitRange

Check:

```bash
kubectl get limitrange -n <namespace>
```

Describe:

```bash
kubectl describe limitrange <limitrange-name> \
  -n <namespace>
```

LimitRange can define default or maximum/minimum resource requirements.

For example:

```text
Default CPU request
Default memory request
Maximum CPU
Maximum memory
Minimum CPU
Minimum memory
```

This can affect the resources assigned to Pods.

---

# 23. Check PersistentVolume and PVC

Storage can prevent scheduling.

Check PVC:

```bash
kubectl get pvc -n <namespace>
```

Example:

```text
NAME             STATUS    VOLUME   CAPACITY
payment-data     Pending
```

If the PVC is Pending, investigate it:

```bash
kubectl describe pvc payment-data -n production
```

Check PersistentVolumes:

```bash
kubectl get pv
```

---

# 24. Check StorageClass

List StorageClasses:

```bash
kubectl get storageclass
```

Check the StorageClass:

```bash
kubectl describe storageclass <storage-class>
```

Verify:

* Correct provisioner
* Correct parameters
* Correct availability zone configuration
* Correct cloud permissions
* Correct volume type

---

# 25. Check Volume Topology

Some storage volumes are tied to a particular Availability Zone.

Example:

```text
Volume:
Zone A

Node:
Zone B
```

The Pod may not be able to run on a Node in Zone B.

For AWS EBS-backed workloads, for example, verify:

```bash
kubectl get nodes \
  --show-labels
```

Look for:

```text
topology.kubernetes.io/zone
```

Example:

```text
worker-01   zone-a
worker-02   zone-b
worker-03   zone-c
```

The storage volume and Pod scheduling constraints must be compatible.

---

# 26. Check Pod Security and Admission Policies

Admission controls and security policies can affect whether a Pod can be created or scheduled.

Potential mechanisms include:

* Pod Security Admission
* Validating admission policies
* Mutating admission webhooks
* OPA/Gatekeeper
* Kyverno
* Custom admission controllers

First determine whether the Pod was actually created.

If the Pod exists but remains Pending, focus primarily on Scheduler-related events.

If the Pod cannot be created at all, investigate admission failures instead.

---

# 27. Check DaemonSet and System Resource Consumption

DaemonSets typically run a Pod on every Node.

Examples:

```text
CNI
kube-proxy
Monitoring agents
Logging agents
Security agents
Node exporters
```

These workloads consume CPU and memory.

Check DaemonSets:

```bash
kubectl get daemonsets -A
```

Check Node resource allocation:

```bash
kubectl describe node <node-name>
```

Look at:

```text
Allocated resources
```

A Node may appear healthy but have little allocatable capacity because of DaemonSets and other workloads.

---

# 28. Check Cluster Autoscaler

If the cluster does not have enough capacity, Cluster Autoscaler may add Nodes.

Check whether Autoscaler is deployed:

```bash
kubectl get pods -n kube-system | grep -i autoscaler
```

If the Pod remains Pending, determine whether Autoscaler is expected to scale the cluster.

Typical scenario:

```text
Pod requires:
4 CPU

Available Nodes:
2 CPU free

        |
        v

Cluster Autoscaler
        |
        v

Adds larger Node
        |
        v

Pod gets scheduled
```

If Autoscaler cannot scale, investigate:

* Maximum Node group size
* Minimum Node group size
* Cloud provider permissions
* Instance availability
* Node group configuration
* Pod scheduling constraints
* Availability Zone capacity

---

# 29. Check Node Groups

For managed Kubernetes platforms, verify the underlying Node groups.

Examples:

* EKS Node Groups
* GKE Node Pools
* AKS Node Pools

Check:

```bash
kubectl get nodes
```

If Nodes are missing, investigate the cloud provider.

For example, in AWS:

```text
EKS
 |
 +--> Managed Node Group
 |
 +--> Auto Scaling Group
 |
 +--> EC2 Instances
```

A Pod cannot be scheduled if the required Node capacity does not exist.

---

# 30. Check Scheduler Logs

In clusters where the control plane is accessible, inspect the Kubernetes Scheduler.

For self-managed clusters:

```bash
kubectl get pods -n kube-system
```

Find the Scheduler Pod:

```bash
kubectl get pods -n kube-system | grep scheduler
```

Then:

```bash
kubectl logs <scheduler-pod> -n kube-system
```

For managed Kubernetes services, control-plane Scheduler logs may not be directly accessible in the same way.

Use the Pod's `FailedScheduling` events as the primary diagnostic source.

---

# 31. Common Scheduling Errors

| Error                                              | Likely Cause                                                  |
| -------------------------------------------------- | ------------------------------------------------------------- |
| `Insufficient cpu`                                 | Not enough CPU requested capacity                             |
| `Insufficient memory`                              | Not enough memory requested capacity                          |
| `node(s) had untolerated taint`                    | Pod lacks required toleration                                 |
| `node(s) didn't match Pod's node affinity`         | Node affinity mismatch                                        |
| `node(s) didn't match Pod's node selector`         | NodeSelector mismatch                                         |
| `Too many pods`                                    | Node reached Pod capacity                                     |
| `pod has unbound immediate PersistentVolumeClaims` | PVC is not bound                                              |
| `volume node affinity conflict`                    | Volume and Node topology mismatch                             |
| `didn't match pod anti-affinity rules`             | Anti-affinity prevents placement                              |
| `topology spread constraints`                      | Pod cannot satisfy topology rules                             |
| `node(s) are unschedulable`                        | Node is cordoned/unschedulable                                |
| `node(s) not ready`                                | Node is unhealthy                                             |
| `Insufficient ephemeral-storage`                   | Not enough ephemeral storage                                  |
| `No preemption victims found`                      | Scheduler cannot free sufficient resources through preemption |

---

# 32. Production Troubleshooting Decision Tree

```text
                     Pod Pending
                         |
                         v
                kubectl describe pod
                         |
                         v
                  FailedScheduling
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
     Insufficient      Taint /        Affinity /
     CPU/Memory       Toleration      Selector
          |              |              |
          v              v              v
     Check Node       Check Pod      Check Node
     Resources        Tolerations    Labels
          |              |              |
          +--------------+--------------+
                         |
                         v
                    Storage Issue?
                         |
                    +----+----+
                    |         |
                   Yes        No
                    |         |
                    v         v
                  PVC       Check
                  PV        Topology
                  Zone      Constraints
                    |         |
                    +----+----+
                         |
                         v
                   Node Available?
                         |
                    +----+----+
                    |         |
                   No        Yes
                    |         |
                    v         v
             Check Node     Check
             Group /        Scheduler
             Autoscaler     Constraints
                    |
                    v
                 Scale Up
                    |
                    v
              Pod Scheduled
```

---

# 33. Production Checklist

## Pod

* [ ] Pod status checked
* [ ] Pod Events checked
* [ ] `FailedScheduling` reason identified
* [ ] Pod resource requests checked
* [ ] Node selector checked
* [ ] Node affinity checked
* [ ] Pod affinity checked
* [ ] Pod anti-affinity checked
* [ ] Topology spread constraints checked
* [ ] Tolerations checked

## Nodes

* [ ] Nodes are `Ready`
* [ ] Nodes are not cordoned
* [ ] CPU capacity checked
* [ ] Memory capacity checked
* [ ] Ephemeral storage checked
* [ ] Pod capacity checked
* [ ] Node taints checked
* [ ] Node labels checked
* [ ] Node conditions checked

## Resources

* [ ] CPU requests checked
* [ ] Memory requests checked
* [ ] ResourceQuota checked
* [ ] LimitRange checked
* [ ] Existing workload resource usage checked
* [ ] DaemonSet resource usage checked

## Storage

* [ ] PVC status checked
* [ ] PV status checked
* [ ] StorageClass checked
* [ ] Volume binding checked
* [ ] Volume topology checked
* [ ] Availability Zone compatibility checked

## Autoscaling

* [ ] Cluster Autoscaler checked
* [ ] Node group capacity checked
* [ ] Maximum Node group size checked
* [ ] Cloud provider permissions checked
* [ ] Instance availability checked

---

# 34. Fast Incident Investigation

During a production incident, start with these commands.

### Step 1 — Check Pod

```bash
kubectl get pods -n <namespace>
```

### Step 2 — Describe Pod

```bash
kubectl describe pod <pod-name> -n <namespace>
```

### Step 3 — Check Node Status

```bash
kubectl get nodes
```

### Step 4 — Check Node Resources

```bash
kubectl describe node <node-name>
```

### Step 5 — Check Pod Resource Requests

```bash
kubectl get pod <pod-name> \
  -n <namespace> \
  -o jsonpath='{.spec.containers[*].resources}'
```

### Step 6 — Check Node Selector

```bash
kubectl get pod <pod-name> \
  -n <namespace> \
  -o jsonpath='{.spec.nodeSelector}'
```

### Step 7 — Check Tolerations

```bash
kubectl get pod <pod-name> \
  -n <namespace> \
  -o jsonpath='{.spec.tolerations}'
```

### Step 8 — Check Node Labels

```bash
kubectl get nodes --show-labels
```

### Step 9 — Check PVC

```bash
kubectl get pvc -n <namespace>
```

### Step 10 — Check ResourceQuota

```bash
kubectl get resourcequota -n <namespace>
```

### Step 11 — Check LimitRange

```bash
kubectl get limitrange -n <namespace>
```

### Step 12 — Check DaemonSets

```bash
kubectl get daemonsets -A
```

---

# 35. Root Cause Classification

After investigation, classify the issue.

## Category 1 — Resource

```text
Insufficient CPU
Insufficient memory
Insufficient ephemeral storage
Too many Pods
```

## Category 2 — Taints

```text
Node has NoSchedule taint
Pod lacks toleration
Incorrect toleration
```

## Category 3 — Node Placement

```text
NodeSelector mismatch
Node affinity mismatch
Pod affinity mismatch
Pod anti-affinity conflict
Topology constraint conflict
```

## Category 4 — Storage

```text
PVC Pending
PV unavailable
StorageClass problem
Volume topology conflict
Availability Zone mismatch
```

## Category 5 — Node Health

```text
Node NotReady
MemoryPressure
DiskPressure
PIDPressure
Node cordoned
```

## Category 6 — Namespace

```text
ResourceQuota reached
LimitRange configuration
Namespace-specific constraints
```

## Category 7 — Cluster Capacity

```text
Node group too small
Autoscaler disabled
Autoscaler at maximum size
Cloud provider capacity issue
Instance type unavailable
```

---

# 36. Best Practices

## 36.1 Define Appropriate Resource Requests

Do not use unnecessarily large requests.

For example:

```yaml
resources:
  requests:
    cpu: "4"
    memory: "8Gi"
```

If the application only requires:

```text
CPU: 500m
Memory: 512Mi
```

the unnecessarily large request can make scheduling difficult.

At the same time, do not underestimate resource requirements.

Use actual workload metrics to determine appropriate requests.

---

## 36.2 Avoid Overly Restrictive Node Affinity

Avoid unnecessary hard constraints such as:

```yaml
requiredDuringSchedulingIgnoredDuringExecution:
```

unless the workload genuinely requires them.

Where appropriate, consider:

```yaml
preferredDuringSchedulingIgnoredDuringExecution:
```

to express a preference rather than an absolute requirement.

---

## 36.3 Use Taints Carefully

Taints are useful for dedicated workloads.

Example:

```text
dedicated=database:NoSchedule
```

But every workload that needs that Node must have the appropriate toleration.

---

## 36.4 Plan Cluster Capacity

Monitor:

* CPU utilization
* Memory utilization
* Pod count
* Node capacity
* Node group capacity
* Autoscaler activity

Do not wait until the cluster reaches 100% capacity before planning expansion.

---

## 36.5 Use Cluster Autoscaler Carefully

Configure:

```text
Minimum Nodes
Maximum Nodes
Instance types
Availability Zones
Scaling policies
```

Make sure the autoscaler can actually provision a Node capable of satisfying your workloads.

---

## 36.6 Avoid Excessive Pod Anti-Affinity

Hard anti-affinity rules can make workloads unschedulable when the cluster does not have enough Nodes.

For example:

```text
10 replicas
+
5 Nodes
+
required anti-affinity
=
Potential scheduling problem
```

Use preferred anti-affinity where strict separation is not mandatory.

---

# 37. Interview Answer

### Question

**How would you troubleshoot a Pod that is stuck in `Pending` and not scheduled?**

### Answer

> I would first run `kubectl describe pod` and check the Events section, especially the `FailedScheduling` message. This usually tells me why the Scheduler cannot find a suitable Node.
>
> If the error is `Insufficient cpu` or `Insufficient memory`, I would compare the Pod's resource requests with the Nodes' allocatable resources. I would also check whether DaemonSets and other workloads are consuming the available capacity.
>
> Next, I would check Node taints and the Pod's tolerations, followed by node selectors, node affinity, pod affinity, anti-affinity, and topology spread constraints.
>
> I would then investigate storage if the Pod uses a PVC, including the PVC/PV status, StorageClass, volume binding, and Availability Zone or topology constraints.
>
> I would also check whether Nodes are healthy and schedulable, whether the cluster has enough capacity, and whether Cluster Autoscaler can add Nodes if required.
>
> Finally, I would identify the exact scheduling constraint causing the problem, make the smallest safe configuration or capacity change, and verify that the Pod gets scheduled successfully.

---

# 38. Quick Reference

```bash
# List Pods
kubectl get pods -n <namespace>

# Describe Pod
kubectl describe pod <pod> -n <namespace>

# Get Pod details
kubectl get pod <pod> -n <namespace> -o wide

# Get Pod YAML
kubectl get pod <pod> -n <namespace> -o yaml

# List Nodes
kubectl get nodes

# List Nodes with labels
kubectl get nodes --show-labels

# Describe Node
kubectl describe node <node>

# Check Node labels
kubectl get node <node> --show-labels

# Check Pods matching a Node selector
kubectl get nodes -l <key>=<value>

# Check PVCs
kubectl get pvc -n <namespace>

# Describe PVC
kubectl describe pvc <pvc> -n <namespace>

# Check PVs
kubectl get pv

# Check StorageClasses
kubectl get storageclass

# Check ResourceQuota
kubectl get resourcequota -n <namespace>

# Check LimitRange
kubectl get limitrange -n <namespace>

# Check DaemonSets
kubectl get daemonsets -A

# Check resource usage
kubectl top nodes
kubectl top pods -n <namespace>

# Check scheduler-related events
kubectl get events -n <namespace> \
  --sort-by='.lastTimestamp'
```

---

# 39. Summary

When troubleshooting a Pod that is not scheduled:

```text
1. Check Pod status
        ↓
2. Run kubectl describe pod
        ↓
3. Check FailedScheduling events
        ↓
4. Check Node availability
        ↓
5. Check CPU / Memory requests
        ↓
6. Check Node taints / Pod tolerations
        ↓
7. Check NodeSelector
        ↓
8. Check Node affinity
        ↓
9. Check Pod affinity / anti-affinity
        ↓
10. Check topology constraints
        ↓
11. Check PVC / PV / StorageClass
        ↓
12. Check Node health
        ↓
13. Check ResourceQuota / LimitRange
        ↓
14. Check Cluster Autoscaler
        ↓
15. Check Node group capacity
        ↓
16. Apply the smallest safe fix
        ↓
17. Validate Pod scheduling
```

## Golden Rule

> **For a Pod stuck in `Pending`, always start with `kubectl describe pod` and the `FailedScheduling` event. The Scheduler normally tells you which constraints prevented the Pod from being placed on a Node. Work backward from that specific reason instead of changing resources, taints, affinity, or Node configuration blindly.**
