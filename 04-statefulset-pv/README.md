# Kubernetes Production Troubleshooting Runbook: StatefulSet Pod & PersistentVolume (PV/PVC)

A production troubleshooting runbook for diagnosing Kubernetes StatefulSet Pods that are stuck in `Pending`, fail to start, or cannot mount their PersistentVolumeClaim (PVC).

This runbook focuses on storage-related issues involving:

* StatefulSets
* Pods
* PersistentVolumeClaims (PVCs)
* PersistentVolumes (PVs)
* StorageClasses
* Dynamic volume provisioning
* Volume attachment
* Volume mounting
* Availability Zone / topology constraints
* Access modes
* Volume permissions
* CSI drivers
* Cloud storage backends

---

# Table of Contents

* [1. Purpose](#1-purpose)
* [2. StatefulSet Storage Architecture](#2-statefulset-storage-architecture)
* [3. Understand the Problem](#3-understand-the-problem)
* [4. Initial Investigation](#4-initial-investigation)
* [5. Check StatefulSet](#5-check-statefulset)
* [6. Check StatefulSet Pod](#6-check-statefulset-pod)
* [7. Check Pod Events](#7-check-pod-events)
* [8. Check PVC](#8-check-pvc)
* [9. Check PVC Events](#9-check-pvc-events)
* [10. Check PV](#10-check-pv)
* [11. Check StorageClass](#11-check-storageclass)
* [12. Check CSI Driver](#12-check-csi-driver)
* [13. Check Volume Binding Mode](#13-check-volume-binding-mode)
* [14. Check Access Modes](#14-check-access-modes)
* [15. Check Availability Zone and Topology](#15-check-availability-zone-and-topology)
* [16. Check Volume Attachment](#16-check-volume-attachment)
* [17. Check Volume Mount](#17-check-volume-mount)
* [18. Check Filesystem and Device](#18-check-filesystem-and-device)
* [19. Check Volume Permissions](#19-check-volume-permissions)
* [20. Check Security Context](#20-check-security-context)
* [21. Check Storage Capacity](#21-check-storage-capacity)
* [22. Check Cloud Provider Permissions](#22-check-cloud-provider-permissions)
* [23. Check StatefulSet VolumeClaimTemplates](#23-check-statefulset-volumeclaimtemplates)
* [24. Check PVC/PV Lifecycle](#24-check-pv-pvclifecycle)
* [25. Common Error Messages](#25-common-error-messages)
* [26. Production Troubleshooting Flow](#26-production-troubleshooting-flow)
* [27. Production Checklist](#27-production-checklist)
* [28. Fast Incident Investigation](#28-fast-incident-investigation)
* [29. Root Cause Classification](#29-root-cause-classification)
* [30. Best Practices](#30-best-practices)
* [31. Interview Answer](#31-interview-answer)
* [32. Quick Reference](#32-quick-reference)
* [33. Summary](#33-summary)

---

# 1. Purpose

This runbook provides a systematic approach to troubleshoot StatefulSet workloads where Pods cannot start because of PersistentVolume or PersistentVolumeClaim problems.

Typical symptoms include:

```text
Pod: Pending
Pod: ContainerCreating
Pod: Init:0/1
Pod: CrashLoopBackOff
PVC: Pending
PVC: Lost
PV: Released
PV: Failed
```

Storage-related failures can occur at multiple stages:

```text
StatefulSet
    |
    v
Pod
    |
    v
PVC
    |
    v
PV
    |
    v
StorageClass
    |
    v
CSI Driver
    |
    v
Cloud Storage / Storage Backend
```

The first objective is to identify **which layer is failing**.

---

# 2. StatefulSet Storage Architecture

A StatefulSet commonly uses `volumeClaimTemplates` to create a separate PVC for each Pod.

Example:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: database
spec:
  serviceName: database
  replicas: 3

  selector:
    matchLabels:
      app: database

  template:
    metadata:
      labels:
        app: database

    spec:
      containers:
        - name: database
          image: postgres:16
          volumeMounts:
            - name: database-storage
              mountPath: /var/lib/postgresql/data

  volumeClaimTemplates:
    - metadata:
        name: database-storage
      spec:
        accessModes:
          - ReadWriteOnce

        storageClassName: gp3

        resources:
          requests:
            storage: 20Gi
```

For three replicas, Kubernetes creates separate PVCs:

```text
database-0
   |
   +--> database-storage-database-0
          |
          +--> PV
                 |
                 +--> EBS Volume


database-1
   |
   +--> database-storage-database-1
          |
          +--> PV
                 |
                 +--> EBS Volume


database-2
   |
   +--> database-storage-database-2
          |
          +--> PV
                 |
                 +--> EBS Volume
```

Each StatefulSet Pod normally has its own persistent storage.

---

# 3. Understand the Problem

Before making changes, determine the exact state.

Run:

```bash
kubectl get statefulset -n <namespace>
```

Then:

```bash
kubectl get pods -n <namespace>
```

Then:

```bash
kubectl get pvc -n <namespace>
```

And:

```bash
kubectl get pv
```

A typical failure may look like:

```text
NAME          READY   STATUS    RESTARTS   AGE
database-0    0/1     Pending   0          10m
database-1    1/1     Running   0          30m
database-2    1/1     Running   0          30m
```

PVC:

```text
NAME                         STATUS    VOLUME
database-storage-database-0 Pending
database-storage-database-1 Bound
database-storage-database-2 Bound
```

This immediately indicates that the problem may be specific to the storage associated with `database-0`.

---

# 4. Initial Investigation

Start with:

```bash
kubectl get statefulset -n <namespace>
kubectl get pods -n <namespace> -o wide
kubectl get pvc -n <namespace>
kubectl get pv
kubectl get storageclass
```

Then investigate the affected Pod:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Always check the Events section.

---

# 5. Check StatefulSet

Check the StatefulSet:

```bash
kubectl get statefulset <statefulset-name> \
  -n <namespace>
```

Example:

```text
NAME       READY   AGE
database   2/3     30d
```

This means:

```text
Desired replicas: 3
Ready replicas:   2
```

Get the StatefulSet YAML:

```bash
kubectl get statefulset database \
  -n production \
  -o yaml
```

Check:

```yaml
volumeClaimTemplates:
```

Also verify:

```yaml
storageClassName:
accessModes:
resources:
  requests:
    storage:
```

---

# 6. Check StatefulSet Pod

List Pods:

```bash
kubectl get pods -n <namespace> -o wide
```

For StatefulSets, Pods normally have predictable names:

```text
database-0
database-1
database-2
```

Check the affected Pod:

```bash
kubectl describe pod database-0 -n production
```

Check:

```text
Node
Status
Volumes
Volume Mounts
Events
```

---

# 7. Check Pod Events

This is one of the most important steps.

Run:

```bash
kubectl describe pod database-0 -n production
```

Look at:

```text
Events:
```

Examples:

```text
FailedScheduling
FailedAttachVolume
FailedMount
MountVolume.SetUp failed
```

Typical messages:

```text
pod has unbound immediate PersistentVolumeClaims
```

or:

```text
FailedAttachVolume
```

or:

```text
FailedMount
```

The error message usually identifies the failing layer.

---

# 8. Check PVC

List PVCs:

```bash
kubectl get pvc -n production
```

Example:

```text
NAME                         STATUS   VOLUME                                     CAPACITY
database-storage-database-0 Pending
database-storage-database-1 Bound    pvc-12345678                              20Gi
database-storage-database-2 Bound    pvc-87654321                              20Gi
```

The important field is:

```text
STATUS
```

Common states:

```text
Pending
Bound
Lost
```

A healthy StatefulSet PVC should normally be:

```text
Bound
```

---

# 9. Check PVC Events

Describe the PVC:

```bash
kubectl describe pvc <pvc-name> -n <namespace>
```

Example:

```bash
kubectl describe pvc database-storage-database-0 \
  -n production
```

Check:

```text
StorageClass
Status
Volume
Capacity
Access Modes
Events
```

Example problem:

```text
Events:
  Warning  ProvisioningFailed
  failed to provision volume
```

This means the problem may be with dynamic provisioning, the StorageClass, CSI driver, or cloud storage backend.

---

# 10. Check PV

List PVs:

```bash
kubectl get pv
```

A healthy PV should normally be:

```text
Bound
```

Example:

```text
NAME                                       STATUS   CLAIM
pvc-12345678                               Bound    production/database-storage-database-0
```

Describe it:

```bash
kubectl describe pv <pv-name>
```

Check:

```text
Status
Claim
StorageClass
Capacity
Access Modes
Node Affinity
Reclaim Policy
CSI
```

---

# 11. Check StorageClass

List StorageClasses:

```bash
kubectl get storageclass
```

Example:

```text
NAME            PROVISIONER
gp3 (default)   ebs.csi.aws.com
```

Describe:

```bash
kubectl describe storageclass gp3
```

Check:

```text
Provisioner
Parameters
VolumeBindingMode
AllowVolumeExpansion
ReclaimPolicy
```

For AWS EBS, a typical provisioner is:

```text
ebs.csi.aws.com
```

For other environments, the CSI provisioner will be different.

---

# 12. Check CSI Driver

Storage provisioning and mounting are usually handled by a CSI driver.

Check CSI drivers:

```bash
kubectl get csidrivers
```

Example:

```text
NAME
ebs.csi.aws.com
```

Check CSI Pods:

```bash
kubectl get pods -n kube-system | grep -i csi
```

For example:

```text
ebs-csi-controller
ebs-csi-node
```

Check CSI controller logs when provisioning or attachment fails:

```bash
kubectl logs <csi-controller-pod> \
  -n kube-system \
  -c csi-provisioner
```

The exact container names depend on the CSI implementation.

---

# 13. Check Volume Binding Mode

StorageClasses commonly use one of two binding modes.

Check:

```bash
kubectl get storageclass <storage-class> \
  -o yaml
```

Look for:

```yaml
volumeBindingMode:
```

Common values:

```text
Immediate
WaitForFirstConsumer
```

## Immediate

The volume may be provisioned before the Pod is scheduled.

## WaitForFirstConsumer

Volume provisioning waits until Kubernetes has enough information about the Pod's placement.

This is particularly useful for topology-aware storage.

Example:

```text
Pod
 |
 v
Scheduler determines Node / Zone
 |
 v
CSI provisions volume
 |
 v
Volume attached
 |
 v
Pod starts
```

---

# 14. Check Access Modes

PVCs define how storage can be accessed.

Common access modes include:

```text
ReadWriteOnce (RWO)
ReadOnlyMany (ROX)
ReadWriteMany (RWX)
ReadWriteOncePod (RWOP)
```

For example:

```yaml
accessModes:
  - ReadWriteOnce
```

`ReadWriteOnce` generally means the volume can be mounted read-write by Pods on a single Node at a time.

This is important for StatefulSet workloads.

Do not assume that one storage volume can safely be mounted read-write from multiple Nodes.

---

# 15. Check Availability Zone and Topology

This is a very common issue with cloud block storage.

Example:

```text
PV / Volume
     |
     v
Availability Zone: us-east-1a

Node
     |
     v
Availability Zone: us-east-1b
```

The volume may not be attachable to the Node because the storage is zone-specific.

Check Node topology:

```bash
kubectl get nodes \
  -L topology.kubernetes.io/zone
```

Example:

```text
NAME       STATUS   ZONE
worker-01  Ready    us-east-1a
worker-02  Ready    us-east-1b
worker-03  Ready    us-east-1c
```

Check PV topology:

```bash
kubectl describe pv <pv-name>
```

Look for:

```text
Node Affinity
```

---

# 16. Check Volume Attachment

If the PVC is `Bound` and the Pod is scheduled but the Pod remains in:

```text
ContainerCreating
```

investigate volume attachment.

Check:

```bash
kubectl get volumeattachments
```

Find the relevant attachment:

```bash
kubectl get volumeattachments \
  -o wide
```

Describe it:

```bash
kubectl describe volumeattachment <name>
```

Look for:

```text
Attacher
Node
Source
Status
```

A volume may fail to attach because:

* The cloud volume does not exist
* Cloud API call failed
* IAM permissions are insufficient
* Volume is already attached elsewhere
* Availability Zone is incompatible
* CSI driver has an issue
* Node is unhealthy

---

# 17. Check Volume Mount

If the volume is attached but the Pod cannot mount it:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Look for:

```text
FailedMount
MountVolume.SetUp failed
```

Example:

```text
MountVolume.SetUp failed for volume
```

Possible causes include:

* Filesystem problem
* Incorrect mount configuration
* Device unavailable
* Permission issue
* CSI driver problem
* Node-level problem

---

# 18. Check Filesystem and Device

For node-level mount problems, investigate the affected Node.

Check:

```bash
kubectl describe node <node-name>
```

If you have appropriate access to the worker Node, inspect:

```bash
lsblk
```

and:

```bash
df -h
```

Check mounted filesystems:

```bash
mount
```

Also inspect kernel messages where appropriate:

```bash
dmesg | tail -100
```

Use these commands carefully in production and follow your organization's host-access procedures.

---

# 19. Check Volume Permissions

A volume can be successfully mounted but the application may not be able to write to it.

Example error:

```text
Permission denied
```

Check the Pod:

```bash
kubectl logs <pod-name> -n <namespace>
```

Also inspect:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Check the Pod's:

```yaml
securityContext:
```

For example:

```yaml
securityContext:
  fsGroup: 1000
```

The correct UID/GID depends on the application image and storage requirements.

Do not change ownership or permissions blindly on production data volumes.

---

# 20. Check Security Context

Inspect:

```bash
kubectl get pod <pod-name> \
  -n <namespace> \
  -o yaml
```

Check:

```yaml
securityContext:
```

and container-level:

```yaml
securityContext:
```

Potentially relevant settings include:

```text
runAsUser
runAsGroup
fsGroup
readOnlyRootFilesystem
privileged
allowPrivilegeEscalation
```

Be particularly careful when changing security settings in production.

---

# 21. Check Storage Capacity

Check the PVC:

```bash
kubectl get pvc -n <namespace>
```

Example:

```text
CAPACITY
20Gi
```

If the application reports:

```text
No space left on device
```

check the filesystem from inside the container:

```bash
kubectl exec -it <pod-name> -n <namespace> -- df -h
```

For example:

```bash
kubectl exec -it database-0 \
  -n production \
  -- df -h /var/lib/postgresql/data
```

Also check inode usage:

```bash
kubectl exec -it <pod-name> \
  -n <namespace> \
  -- df -i
```

A filesystem can run out of inodes even when free disk space remains.

---

# 22. Check Cloud Provider Permissions

For cloud-managed storage, the CSI driver needs appropriate permissions to:

* Create volumes
* Delete volumes
* Attach volumes
* Detach volumes
* Describe volumes
* Mount volumes

For example, in AWS EKS, verify that the EBS CSI driver's IAM configuration has the required permissions.

Possible causes:

```text
AccessDenied
UnauthorizedOperation
Failed to create volume
Failed to attach volume
```

Check the CSI controller logs and the cloud provider's audit/event logs.

Do not grant broad administrator permissions as a troubleshooting shortcut. Identify the exact missing permission and apply least privilege.

---

# 23. Check StatefulSet VolumeClaimTemplates

This is one of the most important StatefulSet-specific checks.

Run:

```bash
kubectl get statefulset <statefulset-name> \
  -n <namespace> \
  -o yaml
```

Find:

```yaml
volumeClaimTemplates:
```

Example:

```yaml
volumeClaimTemplates:
  - metadata:
      name: database-storage
    spec:
      accessModes:
        - ReadWriteOnce

      storageClassName: gp3

      resources:
        requests:
          storage: 20Gi
```

Verify:

* PVC name
* StorageClass
* Access mode
* Requested capacity
* Volume mount name
* Mount path

The PVC name generated by a StatefulSet follows a predictable pattern:

```text
<volumeClaimTemplate-name>-<statefulset-name>-<ordinal>
```

Example:

```text
database-storage-database-0
database-storage-database-1
database-storage-database-2
```

---

# 24. Check PVC/PV Lifecycle

A typical lifecycle is:

```text
PVC Created
    |
    v
Provisioning
    |
    v
PV Created
    |
    v
PVC Bound
    |
    v
Pod Scheduled
    |
    v
Volume Attached
    |
    v
Volume Mounted
    |
    v
Container Starts
```

If the process stops at a particular stage, investigate that layer.

### PVC Pending

```text
PVC
 |
 X
 |
PV
```

Investigate:

* StorageClass
* CSI provisioner
* Cloud provider
* Capacity
* Topology
* Provisioner events

### PVC Bound but Pod Pending

```text
PVC
 |
 v
PV
 |
 v
Pod Pending
```

Investigate:

* Scheduler
* Node resources
* Taints
* Affinity
* Volume topology

### Pod Scheduled but ContainerCreating

```text
Pod
 |
 v
Node
 |
 X
 |
Volume Attach/Mount
```

Investigate:

* VolumeAttachment
* CSI node plugin
* Node health
* Cloud provider
* Mount errors

---

# 25. Common Error Messages

| Error                                              | Likely Cause                             |
| -------------------------------------------------- | ---------------------------------------- |
| `pod has unbound immediate PersistentVolumeClaims` | PVC is not bound                         |
| `no persistent volumes available`                  | No suitable PV                           |
| `failed to provision volume`                       | CSI/provisioner/storage backend issue    |
| `ProvisioningFailed`                               | Dynamic provisioning failed              |
| `FailedAttachVolume`                               | Volume could not be attached             |
| `FailedMount`                                      | Volume could not be mounted              |
| `MountVolume.SetUp failed`                         | Mount/setup failure                      |
| `volume node affinity conflict`                    | PV/volume topology does not match Node   |
| `Multi-Attach error`                               | RWO volume is already attached elsewhere |
| `AccessDenied`                                     | Cloud/CSI permissions issue              |
| `No space left on device`                          | Filesystem capacity exhausted            |
| `Permission denied`                                | Filesystem/application permissions       |
| `Volume is already exclusively attached`           | Existing attachment conflict             |
| `timed out waiting for volume`                     | Storage backend/CSI issue                |
| `node is not ready`                                | Worker Node problem                      |
| `PVC is Pending`                                   | Provisioning or binding issue            |

---

# 26. Production Troubleshooting Flow

```text
              StatefulSet Pod Problem
                       |
                       v
                Check Pod Status
                       |
                       v
              kubectl describe pod
                       |
             +---------+---------+
             |                   |
             v                   v
       PVC Pending?         PVC Bound?
             |                   |
            Yes                  Yes
             |                   |
             v                   v
       Check PVC Events     Is Pod Scheduled?
             |                   |
             v              +----+----+
       Check StorageClass   |         |
             |             No        Yes
             v              |         |
       Check CSI Driver     v         v
             |          Scheduler   Volume
             v          problem     Attach/
       Check Cloud                    Mount
       Provider                         |
             |                     +----+----+
             v                     |         |
       PVC Bound?                Attach    Mount
             |                   Failed    Failed
             v                     |         |
           Yes                     v         v
             |                  Check     Check
             v                  CSI       Node
       Check Pod                 Driver    / CSI
             |                            |
             v                            v
       Check Volume                    Check
       Topology                       Permissions
             |
             v
       Check Node
       Availability Zone
             |
             v
       Pod Scheduled
             |
             v
       Volume Attached
             |
             v
       Volume Mounted
             |
             v
       Container Starts
```

---

# 27. Production Checklist

## StatefulSet

* [ ] StatefulSet exists
* [ ] Desired replicas verified
* [ ] Ready replicas verified
* [ ] `volumeClaimTemplates` checked
* [ ] StorageClass verified
* [ ] Access mode verified
* [ ] Requested storage verified
* [ ] `volumeMounts` verified

## Pod

* [ ] Pod status checked
* [ ] Pod Events checked
* [ ] `FailedScheduling` checked
* [ ] `FailedAttachVolume` checked
* [ ] `FailedMount` checked
* [ ] Node assignment checked
* [ ] Volume configuration checked

## PVC

* [ ] PVC exists
* [ ] PVC is `Bound`
* [ ] PVC capacity verified
* [ ] Access mode verified
* [ ] StorageClass verified
* [ ] PVC Events checked

## PV

* [ ] PV exists
* [ ] PV is `Bound`
* [ ] PV is bound to the expected PVC
* [ ] Capacity verified
* [ ] Access mode verified
* [ ] Reclaim policy checked
* [ ] Node affinity checked
* [ ] CSI configuration checked

## StorageClass

* [ ] StorageClass exists
* [ ] Correct provisioner
* [ ] Correct parameters
* [ ] Binding mode verified
* [ ] Reclaim policy verified
* [ ] Volume expansion configuration verified

## CSI

* [ ] CSI driver installed
* [ ] CSI controller healthy
* [ ] CSI node plugin healthy
* [ ] CSI events/logs checked
* [ ] Required cloud permissions verified

## Node

* [ ] Node is `Ready`
* [ ] Node is schedulable
* [ ] Node has sufficient CPU
* [ ] Node has sufficient memory
* [ ] Node has sufficient disk
* [ ] Node is in the correct Availability Zone
* [ ] Node has required labels

## Cloud Storage

* [ ] Volume exists
* [ ] Volume is healthy
* [ ] Volume is in the correct Zone
* [ ] Volume can be attached
* [ ] IAM permissions verified
* [ ] Cloud provider events checked

---

# 28. Fast Incident Investigation

During a production incident, use this sequence.

### Step 1 — StatefulSet

```bash
kubectl get sts -n <namespace>
```

### Step 2 — Pods

```bash
kubectl get pods -n <namespace> -o wide
```

### Step 3 — PVC

```bash
kubectl get pvc -n <namespace>
```

### Step 4 — PV

```bash
kubectl get pv
```

### Step 5 — Pod Events

```bash
kubectl describe pod <pod-name> -n <namespace>
```

### Step 6 — PVC Events

```bash
kubectl describe pvc <pvc-name> -n <namespace>
```

### Step 7 — StorageClass

```bash
kubectl get storageclass
kubectl describe storageclass <storage-class>
```

### Step 8 — CSI Drivers

```bash
kubectl get csidrivers
```

### Step 9 — CSI Pods

```bash
kubectl get pods -n kube-system | grep -i csi
```

### Step 10 — Volume Attachments

```bash
kubectl get volumeattachments
```

### Step 11 — Nodes and Zones

```bash
kubectl get nodes \
  -L topology.kubernetes.io/zone
```

### Step 12 — Check Pod Logs

If the container has started:

```bash
kubectl logs <pod-name> -n <namespace>
```

---

# 29. Root Cause Classification

## Category 1 — PVC

```text
PVC Pending
PVC Lost
PVC cannot bind
No matching PV
```

## Category 2 — StorageClass

```text
Incorrect StorageClass
Incorrect provisioner
Incorrect parameters
Incorrect binding mode
```

## Category 3 — CSI

```text
CSI controller failure
CSI node plugin failure
Provisioning failure
Attach failure
Mount failure
```

## Category 4 — PV

```text
PV unavailable
PV Released
PV Failed
PV node affinity mismatch
PV bound to unexpected claim
```

## Category 5 — Node

```text
Node NotReady
Node cordoned
Insufficient resources
Incorrect Availability Zone
Disk problem
```

## Category 6 — Cloud Storage

```text
Volume unavailable
Volume attachment failure
IAM permission failure
Availability Zone mismatch
Cloud provider API failure
```

## Category 7 — Application Permissions

```text
Permission denied
Incorrect UID/GID
Incorrect fsGroup
Filesystem ownership issue
Read-only filesystem
```

---

# 30. Best Practices

## 30.1 Use `volumeClaimTemplates` for StatefulSet Storage

For StatefulSet workloads, use `volumeClaimTemplates` when each replica requires its own persistent storage.

Example:

```yaml
volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes:
        - ReadWriteOnce
      storageClassName: gp3
      resources:
        requests:
          storage: 20Gi
```

---

## 30.2 Do Not Delete PVCs During Troubleshooting

Be extremely careful with:

```bash
kubectl delete pvc
```

For StatefulSet workloads, PVCs often contain application data.

Deleting a PVC can result in:

* Data loss
* Loss of application state
* PV lifecycle changes
* Irreversible production impact

Always determine the application's backup and recovery strategy before deleting storage resources.

---

## 30.3 Verify Backups Before Storage Changes

For databases and other stateful applications, verify:

```text
Backup exists
Backup is recent
Restore procedure is known
Data loss impact is understood
```

before performing destructive storage operations.

---

## 30.4 Use Topology-Aware Storage

For zone-aware workloads, configure storage so that Kubernetes can make correct scheduling and provisioning decisions.

`WaitForFirstConsumer` can be useful for topology-constrained storage because provisioning is coordinated with Pod scheduling.

---

## 30.5 Monitor PVC Capacity

Monitor:

```text
Filesystem usage
Filesystem inode usage
PVC capacity
Application storage growth
Volume expansion
```

A full filesystem can cause application failures even when Kubernetes reports the PVC as healthy.

---

## 30.6 Monitor CSI Components

Monitor:

```text
CSI controller
CSI node plugin
Volume provisioning
Volume attachment
Volume mounting
Storage API errors
```

Storage infrastructure is a critical dependency for StatefulSet workloads.

---

# 31. Interview Answer

### Question

**How would you troubleshoot a StatefulSet Pod that is stuck in `Pending` because of a PV/PVC issue?**

### Answer

> First, I would check the StatefulSet, Pod, PVC, PV, and StorageClass to understand where the failure is occurring.
>
> I would run `kubectl describe pod` and check the Events section for messages such as `FailedScheduling`, `FailedAttachVolume`, or `FailedMount`.
>
> Next, I would check the PVC status. If the PVC is `Pending`, I would investigate the StorageClass, CSI provisioner, PV availability, capacity, access modes, and CSI controller events.
>
> If the PVC is `Bound` but the Pod is still Pending, I would check Node resources, taints, tolerations, affinity, and volume topology.
>
> If the Pod is scheduled but remains in `ContainerCreating`, I would investigate volume attachment and mounting, including `VolumeAttachment`, CSI node plugins, Node health, and cloud provider storage events.
>
> I would also verify Availability Zone compatibility, especially for cloud block storage such as AWS EBS, because the volume and Node may need to be in compatible topology domains.
>
> Finally, I would identify the exact failing layer, make the smallest safe change, and verify that the volume is attached, mounted, and the StatefulSet Pod becomes Ready.
>
> I would never delete a production PVC or PV as a first troubleshooting step because it may result in permanent data loss.

---

# 32. Quick Reference

```bash
# StatefulSets
kubectl get sts -n <namespace>

# StatefulSet YAML
kubectl get sts <sts> -n <namespace> -o yaml

# Pods
kubectl get pods -n <namespace> -o wide

# Describe Pod
kubectl describe pod <pod> -n <namespace>

# Pod YAML
kubectl get pod <pod> -n <namespace> -o yaml

# PVCs
kubectl get pvc -n <namespace>

# Describe PVC
kubectl describe pvc <pvc> -n <namespace>

# PVs
kubectl get pv

# Describe PV
kubectl describe pv <pv>

# StorageClasses
kubectl get storageclass

# Describe StorageClass
kubectl describe storageclass <storageclass>

# CSI Drivers
kubectl get csidrivers

# CSI Pods
kubectl get pods -n kube-system | grep -i csi

# VolumeAttachments
kubectl get volumeattachments

# Describe VolumeAttachment
kubectl describe volumeattachment <volumeattachment>

# Nodes
kubectl get nodes -o wide

# Nodes with Zones
kubectl get nodes \
  -L topology.kubernetes.io/zone

# Node labels
kubectl get nodes --show-labels

# Node details
kubectl describe node <node>

# Events
kubectl get events -n <namespace> \
  --sort-by='.lastTimestamp'

# Pod logs
kubectl logs <pod> -n <namespace>

# Execute df inside Pod
kubectl exec -it <pod> -n <namespace> -- df -h

# Check inode usage
kubectl exec -it <pod> -n <namespace> -- df -i
```

---

# 33. Summary

For a StatefulSet storage incident, troubleshoot from the top down:

```text
StatefulSet
     |
     v
Pod
     |
     v
PVC
     |
     v
PV
     |
     v
StorageClass
     |
     v
CSI Driver
     |
     v
Volume Attachment
     |
     v
Volume Mount
     |
     v
Cloud Storage
```

The key diagnostic questions are:

```text
1. Does the StatefulSet exist?
2. Does the affected Pod exist?
3. Is the Pod Pending or ContainerCreating?
4. Is the PVC Pending or Bound?
5. Does the corresponding PV exist?
6. Is the PV Bound to the correct PVC?
7. Is the correct StorageClass being used?
8. Is the CSI driver healthy?
9. Can the volume be provisioned?
10. Can the volume be attached?
11. Can the volume be mounted?
12. Is the volume in a compatible Availability Zone?
13. Does the Node have sufficient resources?
14. Does the application have permission to access the mounted filesystem?
15. Is the underlying cloud storage healthy?
```

## Golden Rule

> **For StatefulSet storage issues, do not troubleshoot only the Pod. Trace the complete storage chain: StatefulSet → Pod → PVC → PV → StorageClass → CSI Driver → Volume Attachment → Node → Storage Backend. Identify the exact layer that is failing before making any production change.**
