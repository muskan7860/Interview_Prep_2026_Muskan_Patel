# Kubernetes Storage — Deep Dive Concept + Hands-On Lab
## PV · PVC · StorageClass · Dynamic Provisioning · emptyDir · hostPath · CSI Drivers
> Written for: Someone with 4 years of DevOps experience preparing for senior-level interviews
> Style: First-standard student explanation → deep technical truth → hands-on lab with line-by-line explanation

---

## 🧠 SECTION 1 — WHY DOES KUBERNETES NEED STORAGE? (Story First)

### Forget Kubernetes for 3 minutes — start with a story

Imagine you are working in a big office building.

You have a **desk** (your pod/container). Every day you come in, sit at your desk, and work. Your desk has a small tray where you keep today's papers (container's local filesystem).

Now three problems:

**Problem 1 — The fire drill problem:**
The fire alarm goes off. Everyone evacuates (pod restarts/crashes). When you come back, your desk is completely empty. All your papers are gone. Whatever you were working on — lost forever.

**This is what happens to container storage by default.** When a container restarts, its filesystem is wiped clean. A database pod that restarts loses ALL its data. This is unacceptable.

**Problem 2 — The office move problem:**
Your company moves you to a different floor (pod rescheduled to different node). You need your files to come WITH you. But your files were on the old floor's shelf — they don't move automatically.

**Problem 3 — The team sharing problem:**
Three people (three pods) need to read and write the SAME set of files at the same time. They can't all have their own separate copies — they need to share one storage location.

**Kubernetes Storage solves all three problems:**

```
Problem 1 → PersistentVolume (data survives pod restarts)
Problem 2 → PVC + StorageClass (storage follows pods, works on any node)
Problem 3 → ReadWriteMany access mode (multiple pods share one volume)
```

---

## 🏗️ SECTION 2 — THE BIG PICTURE: HOW KUBERNETES STORAGE WORKS

### Three Layers — Admin, Developer, Kubernetes

Kubernetes storage has a clean separation of concerns — three different people/roles are involved:

```
┌─────────────────────────────────────────────────────────────┐
│  CLUSTER ADMIN (Infrastructure team)                        │
│                                                             │
│  Creates PersistentVolumes (PV)                             │
│  "Here is 100GB of NFS storage available for the cluster"   │
│                                                             │
│  OR sets up StorageClass                                     │
│  "Use AWS EBS — Kubernetes can create disks automatically"  │
└─────────────────────────────────────────────────────────────┘
                         │
                         │ (matches)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  DEVELOPER (Application team)                               │
│                                                             │
│  Creates PersistentVolumeClaim (PVC)                        │
│  "I need 10GB of storage, ReadWriteOnce access"             │
│                                                             │
│  Does NOT care where the storage comes from                 │
│  Does NOT care about the underlying disk type               │
└─────────────────────────────────────────────────────────────┘
                         │
                         │ (binds)
                         ▼
┌─────────────────────────────────────────────────────────────┐
│  KUBERNETES (Control plane)                                 │
│                                                             │
│  Matches PVC with a suitable PV                             │
│  OR creates a new PV automatically (dynamic provisioning)   │
│  Mounts the storage into the pod                            │
└─────────────────────────────────────────────────────────────┘
```

This separation means developers can ask for storage without knowing anything about the infrastructure. The admin manages what's available. Kubernetes connects them.

---

## 📦 SECTION 3 — VOLUMES (The Foundation)

### What is a Volume?

Before PV and PVC, understand the basic concept: a **Volume** in Kubernetes is a **directory that is accessible to containers in a pod**.

```
Pod
├── Container 1 ─── can read/write /data
├── Container 2 ─── can read/write /data    ← SAME directory!
└── Volume: /data ─── the actual storage location
```

A volume's lifetime is tied to the POD — not the container. So if a container inside the pod crashes and restarts, the volume data is still there. But if the POD itself is deleted, the volume data depends on the volume TYPE.

### Volume Types Summary

```
EPHEMERAL (data lost when pod is deleted):
  emptyDir  → temporary directory, exists only while pod runs

NODE-LOCAL (data tied to a specific node):
  hostPath  → mount a directory from the HOST NODE's filesystem

NETWORK (data persists independently):
  PersistentVolume → backed by NFS, iSCSI, or cloud disk (EBS, GCS, Azure Disk)

CLOUD-SPECIFIC:
  awsElasticBlockStore → AWS EBS (deprecated — use CSI instead)
  gcePersistentDisk    → GCP Persistent Disk (deprecated — use CSI)

MODERN (CSI-based):
  Any storage system with a CSI driver
```

---

## 🟡 SECTION 4 — emptyDir (Simplest Volume)

### What is emptyDir?

**emptyDir** creates an **empty directory** when the pod starts. All containers in the same pod can access it. When the pod is DELETED — the directory and all its contents are permanently gone.

### Simple Story

emptyDir is like a **whiteboard in a meeting room**. When you walk in (pod starts), the whiteboard is blank and ready. Everyone in the room (all containers in the pod) can write on it and read from it. When the meeting ends and everyone leaves (pod deleted), someone erases the whiteboard. Fresh for the next meeting.

### When to Use emptyDir

```
✅ Temporary workspace (processing files, then discarding)
✅ Sharing files between containers in the SAME pod
✅ Cache that doesn't need to survive restarts
✅ Scratch space for sorting large datasets

❌ NOT for data that must survive pod deletion
❌ NOT for database storage
❌ NOT for sharing between DIFFERENT pods
```

### emptyDir YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "echo 'hello from writer' > /shared/data.txt && sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  - name: reader
    image: busybox
    command: ["sh", "-c", "sleep 5 && cat /shared/data.txt && sleep 3600"]
    volumeMounts:
    - name: shared-data
      mountPath: /shared
  volumes:
  - name: shared-data
    emptyDir: {}
```

**Every line explained:**

```yaml
  containers:
  - name: writer
```
→ First container named `writer`. This pod has TWO containers — writer and reader.

```yaml
    command: ["sh", "-c", "echo 'hello from writer' > /shared/data.txt && sleep 3600"]
```
→ `command` overrides the container's default entrypoint.
→ `echo 'hello from writer' > /shared/data.txt` → write text to a file in the shared directory.
→ `&& sleep 3600` → then sleep 1 hour (keeps container running so we can inspect).

```yaml
    volumeMounts:
    - name: shared-data
      mountPath: /shared
```
→ `volumeMounts` → mount a volume INTO this container.
→ `name: shared-data` → which volume to mount (matches the name in `volumes:` section below).
→ `mountPath: /shared` → WHERE inside the container to mount it. The container sees this as a regular directory.

```yaml
  - name: reader
    ...
    volumeMounts:
    - name: shared-data
      mountPath: /shared
```
→ Second container ALSO mounts the SAME volume (`shared-data`) at `/shared`.
→ Both containers share the same directory. Writer writes, reader reads the SAME file.

```yaml
  volumes:
  - name: shared-data
    emptyDir: {}
```
→ `volumes:` → define the volumes available to this pod.
→ `name: shared-data` → the name containers reference in their `volumeMounts`.
→ `emptyDir: {}` → the `{}` means empty config — use all defaults. The directory starts empty and lives in memory (by default on node's disk, but can be `medium: Memory` for RAM-backed tmpfs).

### emptyDir with Memory Backing

```yaml
  volumes:
  - name: cache
    emptyDir:
      medium: Memory      # stored in RAM, not disk (faster, but uses node memory)
      sizeLimit: 256Mi    # maximum size — prevents runaway memory use
```
→ `medium: Memory` → backed by RAM (tmpfs). Faster than disk. Lost when node reboots.
→ `sizeLimit: 256Mi` → pod is evicted if this volume exceeds 256Mi.

---

## 🔴 SECTION 5 — hostPath (Node's Filesystem)

### What is hostPath?

**hostPath** mounts a **directory from the HOST NODE's filesystem** directly into the container. The container can read and write files that exist on the actual machine running the pod.

### Simple Story

hostPath is like giving a hotel guest the KEY TO THE BUILDING'S STORAGE ROOM. The storage room is part of the building (the node). Different guests (pods) on that floor (node) can access it. But if you move to a different hotel (different node) — that storage room doesn't come with you.

### When to Use hostPath

```
✅ Reading node-level files (system logs, device files)
✅ Running node monitoring agents that need to read /proc, /sys
✅ DaemonSets that need access to node resources
✅ Development environments (simple local storage)

❌ NEVER in production for application data
❌ Data is NODE-SPECIFIC — if pod moves to different node, data is gone
❌ Security risk — pod can read/write host filesystem (container escape risk)
```

### hostPath YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: host-logs
      mountPath: /var/log/nginx-app    # inside container
  volumes:
  - name: host-logs
    hostPath:
      path: /var/log/nginx-app         # on the HOST NODE
      type: DirectoryOrCreate          # create if doesn't exist
```

**Explaining hostPath type values:**

```yaml
type: DirectoryOrCreate
```
→ If the directory doesn't exist on the host → create it. If it exists → use it.

Other type values:
```
Directory          → must already exist on host. Fails if doesn't.
File               → mount a specific FILE (not directory) from host
FileOrCreate       → mount a file, create if doesn't exist
Socket             → mount a Unix socket from the host
CharDevice         → mount a character device
BlockDevice        → mount a block device
```

### Real DaemonSet Use Case

```yaml
# Fluentd log collector DaemonSet — reads logs from every node
volumes:
- name: varlog
  hostPath:
    path: /var/log          # all container logs are here on the host
- name: varlibdockercontainers
  hostPath:
    path: /var/lib/docker/containers  # docker container logs
```

Fluentd mounts these host directories to read log files that containers write there. This is the legitimate production use of hostPath.

---

## 🟢 SECTION 6 — PersistentVolume (PV)

### What is a PersistentVolume?

A **PersistentVolume (PV)** is a piece of storage in the cluster that has been **provisioned by an administrator** (or automatically by a StorageClass). It is a **cluster-level resource** — it doesn't belong to any namespace.

Think of it as a **storage unit in a warehouse**. The warehouse manager (admin) goes and rents a storage unit (creates an EBS volume, NFS share, etc.) and then registers it with the cluster: "Here is a 50GB storage unit, available for anyone who needs it."

```
CLUSTER ADMIN does:
  → Creates an AWS EBS volume: vol-0abc123def (50GB)
  → Creates a PV YAML pointing to that EBS volume
  → Applies it: kubectl apply -f pv.yaml
  → PV now shows as "Available" in the cluster

KUBERNETES knows:
  → There is 50GB of EBS storage available
  → It has certain access modes and reclaim policies
  → Someone can claim it with a PVC
```

### PV YAML (Static Provisioning)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: postgres-pv
spec:
  capacity:
    storage: 50Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  awsElasticBlockStore:
    volumeID: vol-0abc123def456
    fsType: ext4
```

**Every line explained:**

```yaml
  capacity:
    storage: 50Gi
```
→ `capacity.storage` → how much storage this PV offers. 50Gi = 50 Gibibytes.
→ A PVC requesting 30Gi can bind to this 50Gi PV — it can request UP TO the PV's size but not more.

```yaml
  volumeMode: Filesystem
```
→ `volumeMode` → how the volume is presented to the pod.
→ `Filesystem` → the pod sees it as a mounted directory with files. This is the default.
→ `Block` → the pod gets raw block device access (used for databases that manage their own filesystem).

```yaml
  accessModes:
  - ReadWriteOnce
```
→ `accessModes` → how many nodes can mount this volume and in what way.

The three access modes — understand them deeply:
```
ReadWriteOnce (RWO):
  → Only ONE node can mount this volume at a time
  → That node can have MULTIPLE pods reading/writing to it
  → Like a single USB drive — one laptop at a time
  → Most cloud disks (EBS, Azure Disk) support only RWO
  → ⚠️ COMMON MISTAKE: people think RWO = one pod.
    It's one NODE. Multiple pods on same node can all use it.

ReadOnlyMany (ROX):
  → MULTIPLE nodes can mount this volume SIMULTANEOUSLY
  → But ALL mounts are READ-ONLY — no writing
  → Like a CD-ROM shared across many computers

ReadWriteMany (RWX):
  → MULTIPLE nodes can mount this volume SIMULTANEOUSLY
  → AND all can read AND write
  → Like a shared NFS network drive
  → EBS does NOT support this. NFS, CephFS, Azure Files support it.

ReadWriteOncePod (RWOP) [Kubernetes 1.22+]:
  → Only ONE POD in the entire cluster can mount this volume
  → Stronger than RWO (which allows multiple pods on same node)
```

```yaml
  persistentVolumeReclaimPolicy: Retain
```
→ What happens to the PV (and the actual data) when the PVC is DELETED.

Three reclaim policies:
```
Retain:
  → PV is NOT deleted when PVC is deleted
  → PV goes to "Released" state (not available for new PVCs)
  → Data is PRESERVED on the underlying storage
  → Admin must manually clean up and make it available again
  → USE FOR: production databases — you NEVER want accidental data deletion

Delete:
  → PV AND the underlying storage are DELETED when PVC is deleted
  → Data is PERMANENTLY GONE
  → USE FOR: temporary/dev storage where data doesn't matter

Recycle (deprecated):
  → Wipes the data (rm -rf) and makes PV available again
  → Deprecated — use dynamic provisioning instead
```

```yaml
  storageClassName: manual
```
→ The storage class name this PV belongs to.
→ A PVC with `storageClassName: manual` will look for PVs with this same class.
→ PVs with class `manual` and PVCs with class `manual` can bind to each other.
→ If PVC has no storageClassName and PV has no storageClassName → they can bind (empty class matches empty class).

```yaml
  awsElasticBlockStore:
    volumeID: vol-0abc123def456
    fsType: ext4
```
→ The actual underlying storage — in this case an AWS EBS volume.
→ `volumeID` → the EBS volume ID from AWS console.
→ `fsType: ext4` → what filesystem to format it with (ext4 is standard Linux filesystem).
→ This is STATIC provisioning — admin manually created the EBS volume first, then referenced it here.

---

## 🔵 SECTION 7 — PersistentVolumeClaim (PVC)

### What is a PVC?

A **PersistentVolumeClaim (PVC)** is a **request for storage by a user or application**. It is like a **voucher** — you say "I need 10GB of storage with ReadWriteOnce access" and Kubernetes finds a matching PV and gives it to you.

### Simple Story

PV is a **parking space** in a parking lot. PVC is your **parking ticket**. You hand in your ticket (PVC) saying "I need one parking space for a large car" (size + access mode). The parking attendant (Kubernetes) finds a space that fits your car (matching PV) and assigns it to you. Your car (pod) then parks in that specific space.

### PVC YAML

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: banking
spec:
  accessModes:
  - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 20Gi
  storageClassName: manual
```

**Every line explained:**

```yaml
  accessModes:
  - ReadWriteOnce
```
→ The PVC requests this access mode. Kubernetes only binds this PVC to PVs that SUPPORT this mode.

```yaml
  resources:
    requests:
      storage: 20Gi
```
→ `resources.requests.storage` → how much storage the application needs. 20Gi.
→ Kubernetes finds a PV with AT LEAST 20Gi capacity.
→ If only a 50Gi PV is available → PVC binds to it and gets all 50Gi (PVC gets the full PV).

```yaml
  storageClassName: manual
```
→ Only look at PVs with `storageClassName: manual`.
→ If no matching PV exists → PVC stays in `Pending` state until one becomes available.

### How PVC Binding Works

```
PVC created with:
  storage: 20Gi
  accessModes: ReadWriteOnce
  storageClassName: manual

Kubernetes searches available PVs:
  PV1: 50Gi, ReadWriteOnce, storageClassName: manual → MATCH ✅
  PV2: 10Gi, ReadWriteOnce, storageClassName: manual → TOO SMALL ❌
  PV3: 50Gi, ReadWriteMany, storageClassName: manual → WRONG MODE ❌
  PV4: 50Gi, ReadWriteOnce, storageClassName: fast  → WRONG CLASS ❌

Kubernetes binds PVC to PV1.
PVC status: Bound
PV1 status: Bound (no longer available for other PVCs)
```

### Using PVC in a Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: postgres-pod
  namespace: banking
spec:
  containers:
  - name: postgres
    image: postgres:15
    env:
    - name: POSTGRES_PASSWORD
      value: "mysecretpassword"
    volumeMounts:
    - name: postgres-storage
      mountPath: /var/lib/postgresql/data   # where postgres stores data
  volumes:
  - name: postgres-storage
    persistentVolumeClaim:
      claimName: postgres-pvc               # reference to our PVC
```

**Key lines:**

```yaml
  volumes:
  - name: postgres-storage
    persistentVolumeClaim:
      claimName: postgres-pvc
```
→ `persistentVolumeClaim` → use a PVC as the volume source.
→ `claimName: postgres-pvc` → which PVC to use (must be in same namespace as the pod).

```yaml
    volumeMounts:
    - name: postgres-storage
      mountPath: /var/lib/postgresql/data
```
→ Mount the volume at this path inside the container.
→ `/var/lib/postgresql/data` is where PostgreSQL stores all its databases and data files.
→ Even if the pod is deleted and recreated → the data at this path persists on the PV.

---

## ⚡ SECTION 8 — StorageClass (Dynamic Provisioning)

### The Problem with Static Provisioning

With static provisioning (manual PV creation):
```
Admin needs to:
  1. Go to AWS console → create EBS volume
  2. Note the volume ID
  3. Write PV YAML with that ID
  4. Apply PV YAML
  5. Wait for developer to submit PVC
  6. Repeat for EVERY storage request

In a company with 50 teams → 50 separate manual processes
When a PVC is deleted → admin must manually clean up the EBS volume
This does NOT scale.
```

### StorageClass Solves This — Dynamic Provisioning

A **StorageClass** defines a **recipe** for automatically creating PVs on demand.

When a developer creates a PVC that references a StorageClass:
1. Kubernetes calls the StorageClass's provisioner
2. The provisioner (e.g., AWS EBS CSI driver) creates the actual disk
3. A PV is automatically created pointing to that disk
4. The PVC binds to the new PV

**No admin manual work required.** Storage is created on-demand, automatically.

### StorageClass YAML

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

**Every line explained:**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
```
→ StorageClass belongs to `storage.k8s.io` API group.

```yaml
  name: fast-ssd
```
→ Name of this StorageClass. PVCs reference this name.

```yaml
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
```
→ Mark this as the DEFAULT StorageClass.
→ When a PVC doesn't specify a `storageClassName` → it uses the default StorageClass automatically.
→ Only ONE StorageClass should be marked as default.

```yaml
provisioner: ebs.csi.aws.com
```
→ Which plugin creates the actual storage.
→ `ebs.csi.aws.com` → the AWS EBS CSI driver. It knows how to create EBS volumes via AWS API.
→ Other provisioners: `pd.csi.storage.gke.io` (GCP), `disk.csi.azure.com` (Azure), `nfs.csi.k8s.io` (NFS).

```yaml
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
```
→ `parameters` → provisioner-specific settings. These are AWS EBS-specific.
→ `type: gp3` → EBS volume type. gp3 is the modern general-purpose SSD (better than gp2).
→ `iops: "3000"` → I/O operations per second. gp3 allows up to 16,000 IOPS.
→ `throughput: "125"` → MB/s throughput.
→ `encrypted: "true"` → encrypt the EBS volume at rest (mandatory in banking/compliance environments).

```yaml
reclaimPolicy: Delete
```
→ What happens to the dynamically created EBS volume when the PVC is deleted.
→ `Delete` → EBS volume is automatically deleted. Data is gone. Good for dev/test.
→ `Retain` → EBS volume is kept. Admin must clean up. Good for production databases.

```yaml
allowVolumeExpansion: true
```
→ Can PVCs using this StorageClass be resized (expanded) after creation?
→ `true` → yes. Developer can edit PVC and request more storage. The underlying EBS volume grows.
→ `false` → storage size is fixed at creation time.
→ **Note:** You can EXPAND but NEVER SHRINK a PVC.

```yaml
volumeBindingMode: WaitForFirstConsumer
```
→ When should the actual storage be created and bound?

Two modes:
```
Immediate:
  → PV is created and PVC is bound as SOON as PVC is created
  → The disk is created even before any pod uses it
  → Problem: disk might be in wrong availability zone
    (pod lands on Node in us-east-1a but disk was created in us-east-1b)

WaitForFirstConsumer:
  → PV creation WAITS until a pod actually tries to use the PVC
  → Kubernetes knows which node the pod will run on first
  → Creates the disk in the SAME availability zone as the node
  → CORRECT approach for cloud environments with multiple AZs
```

### Dynamic Provisioning Flow

```
DEVELOPER creates PVC:
  storageClassName: fast-ssd
  storage: 50Gi
  accessModes: ReadWriteOnce
         │
         ▼ (WaitForFirstConsumer — waits)

DEVELOPER creates Pod that uses the PVC
         │
         ▼
SCHEDULER assigns pod to Node in us-east-1a
         │
         ▼
KUBERNETES: "PVC is needed by pod on us-east-1a node"
         │
         ▼
AWS EBS CSI DRIVER called with parameters from StorageClass
         │
         ▼
EBS VOLUME CREATED in us-east-1a (50GB, gp3, encrypted)
         │
         ▼
PV AUTOMATICALLY CREATED pointing to new EBS volume
         │
         ▼
PVC BOUND to new PV
         │
         ▼
POD STARTS with EBS volume mounted at the specified path
```

---

## 🔌 SECTION 9 — CSI DRIVERS (Container Storage Interface)

### What is CSI?

**CSI** stands for **Container Storage Interface**. Just like CNI is the standard for networking plugins, CSI is the standard for storage plugins.

Before CSI, storage drivers were built INTO the Kubernetes binary. To add support for a new storage system, you had to wait for a Kubernetes release. This was slow and messy.

After CSI, any storage vendor can write a CSI driver as a separate project. The driver is deployed in the cluster and Kubernetes calls it via the standard CSI interface. No changes to Kubernetes itself needed.

### Simple Story

CSI is like a **USB standard**. Kubernetes is your laptop — it has USB ports (CSI interface). Any storage vendor (AWS, Google, NetApp, Ceph) writes a USB device (CSI driver). Any USB device plugs into any USB port. No laptop modification needed.

### How a CSI Driver Works

```
CSI Driver components (deployed as pods in kube-system):

1. CSI Controller (Deployment — runs once):
   → Handles CreateVolume, DeleteVolume, AttachVolume, DetachVolume
   → Talks to cloud API (e.g., AWS EC2 API to create EBS volumes)
   → Runs on control plane nodes usually

2. CSI Node Plugin (DaemonSet — runs on every node):
   → Handles NodeStageVolume, NodePublishVolume (actually mount the disk)
   → Runs on every worker node because mounting happens on the node

3. External Provisioner (sidecar):
   → Watches for PVCs with matching StorageClass
   → Calls CSI controller to create volume
   → Creates PV object in Kubernetes

4. External Attacher (sidecar):
   → Watches for pods that need volumes attached
   → Calls CSI controller to attach volume to the right node
```

### Common CSI Drivers

```
AWS EBS CSI Driver:
  → GitHub: kubernetes-sigs/aws-ebs-csi-driver
  → Provisioner: ebs.csi.aws.com
  → Creates: EBS volumes (gp2, gp3, io1, io2, st1, sc1)
  → Access mode: ReadWriteOnce only (EBS can only attach to one EC2)

AWS EFS CSI Driver:
  → Provisioner: efs.csi.aws.com
  → Creates: EFS mount points
  → Access mode: ReadWriteMany (EFS supports multiple mounts)
  → Great for: shared config files, media files, log aggregation

GCP PD CSI Driver:
  → Provisioner: pd.csi.storage.gke.io
  → Creates: GCP Persistent Disks (pd-standard, pd-ssd, pd-balanced)

Azure Disk CSI Driver:
  → Provisioner: disk.csi.azure.com
  → Creates: Azure Managed Disks

Azure File CSI Driver:
  → Provisioner: file.csi.azure.com
  → Creates: Azure File Shares (ReadWriteMany capable)

NFS CSI Driver:
  → Provisioner: nfs.csi.k8s.io
  → Creates: NFS shares
  → ReadWriteMany capable — multiple pods across nodes

Rook-Ceph CSI Driver:
  → Provisioner: rook-ceph.rbd.csi.ceph.com
  → Creates: Ceph RBD (block) or CephFS (file) volumes
  → Self-hosted distributed storage — no cloud needed
  → ReadWriteMany with CephFS
```

### How to Install AWS EBS CSI Driver

```bash
# Install using kubectl (EKS cluster)
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.25"

# OR using Helm
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system \
  --set controller.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=arn:aws:iam::ACCOUNT_ID:role/AmazonEKS_EBS_CSI_DriverRole
```

---

## 📊 SECTION 10 — STATEFULSET WITH VOLUMECLAIMTEMPLATES

### Why StatefulSet Needs Different Storage

With a regular Deployment and a PVC — all 3 replicas share the SAME PVC (and the same PV). For a web server — fine. For a database — DISASTER (three database processes writing to the same files simultaneously → corruption).

StatefulSet solves this with **volumeClaimTemplates** — a TEMPLATE for creating one DEDICATED PVC per pod.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: banking
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        env:
        - name: POSTGRES_PASSWORD
          value: "mysecretpassword"
        volumeMounts:
        - name: postgres-data
          mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 10Gi
```

**The magic:**
```yaml
  volumeClaimTemplates:
  - metadata:
      name: postgres-data
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: fast-ssd
      resources:
        requests:
          storage: 10Gi
```
→ This is NOT a regular PVC. It is a TEMPLATE.
→ Kubernetes creates one PVC FROM this template for EACH replica:
  - `postgres-data-postgres-0` → 10Gi EBS volume → pod postgres-0
  - `postgres-data-postgres-1` → 10Gi EBS volume → pod postgres-1
  - `postgres-data-postgres-2` → 10Gi EBS volume → pod postgres-2
→ Each pod has its OWN dedicated, isolated storage. No sharing. No corruption.
→ If postgres-1 dies and restarts → it comes back as postgres-1 and reconnects to SAME `postgres-data-postgres-1` PVC.
→ **PVCs from volumeClaimTemplates are NOT deleted when the StatefulSet is deleted.** They must be manually cleaned up. This is the safety default.

---

## 🔄 SECTION 11 — PV LIFECYCLE STATES

```
AVAILABLE → PV exists, no PVC has claimed it yet. Ready to be claimed.
BOUND     → PV is claimed by a PVC. In use.
RELEASED  → PVC that was using this PV has been deleted.
            But reclaim policy = Retain, so PV is not deleted.
            PV has the old data still. NOT available for new PVCs until admin clears it.
FAILED    → PV failed its reclaim (automatic reclaim went wrong)
```

### PVC Lifecycle States

```
PENDING  → Waiting for a matching PV to become available
           OR waiting for first pod to be scheduled (WaitForFirstConsumer)
BOUND    → Successfully bound to a PV. Ready for pods to use.
LOST     → The PV that this PVC was bound to no longer exists.
           Data may be lost. Very bad situation.
```

---

## 💻 SECTION 12 — HANDS-ON LAB

> Every command explained word by word. Every flag explained. Nothing skipped.

---

### LAB 1 — emptyDir: Two Containers Sharing Data

```bash
# Create the pod with two containers sharing emptyDir
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-lab
spec:
  containers:
  - name: writer
    image: busybox
    command: ["sh", "-c", "while true; do echo $(date) >> /shared/log.txt; sleep 5; done"]
    volumeMounts:
    - name: shared
      mountPath: /shared
  - name: reader
    image: busybox
    command: ["sh", "-c", "tail -f /shared/log.txt"]
    volumeMounts:
    - name: shared
      mountPath: /shared
  volumes:
  - name: shared
    emptyDir: {}
EOF
```

**Explaining the command in the writer container:**
- `while true; do ... done` → infinite loop (keeps running)
- `echo $(date)` → print current date/time. `$(date)` runs the `date` command and inserts its output.
- `>> /shared/log.txt` → APPEND to the file (not overwrite)
- `sleep 5` → wait 5 seconds between each write

```bash
# Watch what the reader container sees
kubectl logs emptydir-lab -c reader -f
```
- `logs` → show container logs
- `emptydir-lab` → pod name
- `-c reader` → `-c` = container. Show logs from the `reader` container specifically.
- `-f` → follow (stream logs in real time)

You should see dates being printed every 5 seconds — written by the writer, read by the reader from the SHARED emptyDir volume.

```bash
# Verify the volume is there
kubectl exec emptydir-lab -c writer -- ls -la /shared
kubectl exec emptydir-lab -c reader -- cat /shared/log.txt
```

```bash
# Delete the pod and see the data is GONE
kubectl delete pod emptydir-lab
kubectl get pod emptydir-lab   # doesn't exist anymore
# The /shared/log.txt file is gone — emptyDir is ephemeral
```

---

### LAB 2 — hostPath: Mount Node Directory

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-lab
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo 'pod was here' > /host-data/marker.txt && sleep 3600"]
    volumeMounts:
    - name: host-volume
      mountPath: /host-data
  volumes:
  - name: host-volume
    hostPath:
      path: /tmp/k8s-hostpath-test
      type: DirectoryOrCreate
EOF
```

```bash
# Verify the file was written
kubectl exec hostpath-lab -- cat /host-data/marker.txt
```

```bash
# Find which node this pod is on
kubectl get pod hostpath-lab -o wide
```
- `-o wide` → shows NODE column

```bash
# SSH to that node and check the host directory directly
# (If using MicroK8s on local machine, just check locally)
ls /tmp/k8s-hostpath-test/
cat /tmp/k8s-hostpath-test/marker.txt
# File is on the HOST MACHINE, not just inside the container
```

```bash
# Delete the pod — file STAYS on host
kubectl delete pod hostpath-lab
ls /tmp/k8s-hostpath-test/marker.txt   # still there!
# hostPath data survives pod deletion (it's on the host, not the pod)
```

---

### LAB 3 — Static PV and PVC (Manual Provisioning)

```bash
# Step 1: Create a PV (using hostPath for local lab — production would use EBS/NFS)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: lab-pv
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /tmp/k8s-pv-data
    type: DirectoryOrCreate
EOF
```

**Note:** Using hostPath for the PV backend in a local lab. In production this would be `awsElasticBlockStore`, `nfs`, or `csi`.

```bash
# Check PV status — should be Available
kubectl get pv lab-pv
```
Output:
```
NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      STORAGECLASS
lab-pv   1Gi        RWO            Retain            Available   manual
```
- `STATUS: Available` → PV exists, no PVC has claimed it yet

```bash
# Step 2: Create a PVC to claim this PV
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lab-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: manual
EOF
```

```bash
# Check PVC status — should be Bound
kubectl get pvc lab-pvc
```
Output:
```
NAME      STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
lab-pvc   Bound    lab-pv   1Gi        RWO            manual
```
- `STATUS: Bound` → PVC found a matching PV and is bound to it
- `VOLUME: lab-pv` → which PV it bound to
- `CAPACITY: 1Gi` → PVC gets the FULL PV capacity even though it requested only 500Mi

```bash
# Check PV status — now Bound
kubectl get pv lab-pv
```
Output:
```
NAME     CAPACITY   STATUS   CLAIM           STORAGECLASS
lab-pv   1Gi        Bound    default/lab-pvc  manual
```
- `STATUS: Bound`
- `CLAIM: default/lab-pvc` → which PVC claimed it (namespace/pvc-name)

```bash
# Step 3: Create a pod that uses the PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pvc-lab-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo 'persistent data written by pod' > /storage/data.txt && sleep 3600"]
    volumeMounts:
    - name: my-storage
      mountPath: /storage
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: lab-pvc
EOF
```

```bash
# Verify data was written
kubectl exec pvc-lab-pod -- cat /storage/data.txt
```

```bash
# Step 4: Test persistence — delete and recreate the pod
kubectl delete pod pvc-lab-pod

# Recreate the same pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pvc-lab-pod-2
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "cat /storage/data.txt && sleep 3600"]
    volumeMounts:
    - name: my-storage
      mountPath: /storage
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: lab-pvc
EOF
```

```bash
# Data still there even though first pod was deleted!
kubectl logs pvc-lab-pod-2
```
- Output: `persistent data written by pod`
- The data persisted across pod deletion and recreation — this is PersistentVolume working correctly

---

### LAB 4 — StorageClass and Dynamic Provisioning

```bash
# Check existing StorageClasses in your cluster
kubectl get storageclass
```

Output (in MicroK8s):
```
NAME                          PROVISIONER            RECLAIMPOLICY   VOLUMEBINDINGMODE
microk8s-hostpath (default)   microk8s.io/hostpath   Delete          WaitForFirstConsumer
```
- `(default)` → this is the default StorageClass
- PVCs without storageClassName use this automatically

```bash
# Create a PVC that uses the default StorageClass (no storageClassName needed)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

```bash
# Watch PVC status — starts Pending, becomes Bound after pod uses it
kubectl get pvc dynamic-pvc -w
```

```bash
# Create a pod to use it — this triggers volume creation (WaitForFirstConsumer)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-pvc-pod
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "echo dynamic! > /data/test.txt && sleep 3600"]
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: dynamic-pvc
EOF
```

```bash
# Check PVC — now Bound (triggered by pod creation)
kubectl get pvc dynamic-pvc
kubectl get pv   # a new PV was automatically created!
```

```bash
# Verify data
kubectl exec dynamic-pvc-pod -- cat /data/test.txt
```

---

### LAB 5 — Resize a PVC (Volume Expansion)

```bash
# Check if storageclass allows expansion
kubectl get storageclass -o yaml | grep allowVolumeExpansion
```

```bash
# Expand the PVC (if StorageClass supports it)
kubectl patch pvc dynamic-pvc -p '{"spec":{"resources":{"requests":{"storage":"2Gi"}}}}'
```
- `patch pvc dynamic-pvc` → patch the PVC object
- `-p '...'` → the JSON patch: change `spec.resources.requests.storage` to `2Gi`

```bash
# Check expansion status
kubectl get pvc dynamic-pvc
kubectl describe pvc dynamic-pvc | grep -A5 Conditions
```

---

### LAB 6 — Inspect Storage: Full Status Check

```bash
# See all PVs in cluster with details
kubectl get pv -o wide

# See all PVCs in all namespaces
kubectl get pvc -A

# Describe a PVC for full details
kubectl describe pvc lab-pvc

# See which PV a PVC is bound to
kubectl get pvc lab-pvc -o jsonpath='{.spec.volumeName}'

# See which node a PV's hostPath is on
kubectl get pv lab-pv -o jsonpath='{.spec.hostPath.path}'

# See StorageClass of a PVC
kubectl get pvc lab-pvc -o jsonpath='{.spec.storageClassName}'
```

---

### LAB 7 — Cleanup and Observe Reclaim Policy

```bash
# Delete the PVC
kubectl delete pvc lab-pvc

# Check PV status — should be Released (not deleted because Retain policy)
kubectl get pv lab-pv
```
Output:
```
NAME     CAPACITY   STATUS     CLAIM            STORAGECLASS
lab-pv   1Gi        Released   default/lab-pvc   manual
```
- `STATUS: Released` → PVC was deleted, PV still exists with Retain policy
- PV is NOT available for new PVCs in Released state — admin must manually clean it

```bash
# To make the PV available again, admin must remove the claimRef
kubectl patch pv lab-pv -p '{"spec":{"claimRef":null}}'
kubectl get pv lab-pv   # STATUS should be Available now
```

```bash
# Final cleanup
kubectl delete pv lab-pv
kubectl delete pod pvc-lab-pod-2 dynamic-pvc-pod
kubectl delete pvc dynamic-pvc
rm -rf /tmp/k8s-pv-data /tmp/k8s-hostpath-test
```

---

## 📊 SECTION 13 — SUMMARY TABLES

### Volume Types Comparison

| Volume Type | Data Survives Pod Restart? | Data Survives Pod Delete? | Multi-Pod Access? | Use Case |
|-------------|--------------------------|--------------------------|-------------------|----------|
| emptyDir | ✅ Yes | ❌ No | Same pod only | Temp scratch space, inter-container sharing |
| hostPath | ✅ Yes | ✅ Yes (on node) | Same node pods | Node agents, local dev |
| PV (Retain) | ✅ Yes | ✅ Yes | Depends on mode | Production databases |
| PV (Delete) | ✅ Yes | ❌ No | Depends on mode | Dev/test workloads |

### Access Modes Comparison

| Mode | Short | Multiple Nodes? | Write? | Supported By |
|------|-------|-----------------|--------|-------------|
| ReadWriteOnce | RWO | ❌ No (one node) | ✅ Yes | EBS, Azure Disk, GCP PD |
| ReadOnlyMany | ROX | ✅ Yes | ❌ No | NFS, CephFS |
| ReadWriteMany | RWX | ✅ Yes | ✅ Yes | NFS, EFS, CephFS, Azure Files |
| ReadWriteOncePod | RWOP | ❌ No (one pod) | ✅ Yes | EBS CSI (K8s 1.22+) |

### Reclaim Policy Comparison

| Policy | PV Deleted? | Data Deleted? | Use Case |
|--------|-------------|---------------|----------|
| Retain | ❌ No | ❌ No | Production — never lose data |
| Delete | ✅ Yes | ✅ Yes | Dev/test — auto cleanup |
| Recycle | ❌ No | ✅ Yes (rm -rf) | Deprecated |

---

## 🔑 SECTION 14 — KEY TERMS TO REMEMBER

| Term | Simple Meaning |
|------|---------------|
| **Volume** | A directory accessible to containers in a pod |
| **emptyDir** | Temporary directory — lives and dies with the pod |
| **hostPath** | Mount a directory from the host node's filesystem |
| **PersistentVolume (PV)** | Pre-provisioned storage — cluster level resource |
| **PersistentVolumeClaim (PVC)** | Request for storage — namespaced resource |
| **StorageClass** | Recipe for auto-creating PVs on demand |
| **Dynamic Provisioning** | Kubernetes auto-creates storage when PVC is made |
| **Static Provisioning** | Admin manually creates PV, then PVC claims it |
| **CSI** | Container Storage Interface — standard for storage plugins |
| **CSI Driver** | Plugin that connects Kubernetes to a specific storage system |
| **volumeClaimTemplates** | StatefulSet feature — creates one PVC per pod replica |
| **ReadWriteOnce (RWO)** | One node at a time can mount — most cloud disks |
| **ReadWriteMany (RWX)** | Multiple nodes can mount simultaneously — NFS, EFS |
| **Retain** | Keep PV and data after PVC is deleted |
| **Delete** | Delete PV and underlying storage when PVC is deleted |
| **WaitForFirstConsumer** | Create disk in same AZ as pod — prevents AZ mismatch |
| **Immediate** | Create disk as soon as PVC is created (AZ risk) |
| **allowVolumeExpansion** | StorageClass setting — allow PVC size to be increased |
| **volumeBindingMode** | When to create and bind the actual storage |
| **claimRef** | Field in PV pointing to the PVC bound to it |

---

*File: K8s_Storage_Concept_and_Lab.md*
*Repository: Interview_Preparation_2026 → Kubernetes/*
*Next: K8s_Storage_Interview_Questions.md*
