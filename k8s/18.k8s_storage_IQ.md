# Kubernetes Storage — Interview Questions & Answers
## PV · PVC · StorageClass · Dynamic Provisioning · emptyDir · hostPath · CSI
> Target: 4 Years DevOps Experience | Senior-Level Interviews
> Style: Easy to memorize + Professional to say out loud
> Tricky questions marked with ⚠️

---

## 💡 HOW TO USE THIS FILE

- Read the question first
- Try answering in your own words
- Then read the answer — written to say out loud in an interview
- Real banking/production examples included

---

## SECTION A — BASICS

---

### Q1. Why does Kubernetes need persistent storage? What problem does it solve?

**Answer:**

> "Containers have an ephemeral filesystem by default. Every time a container restarts — whether from a crash, an OOMKill, or a rolling update — the filesystem is wiped clean. The container starts fresh as if it was brand new.
>
> For stateless applications like web servers and REST APIs, this is fine — they don't store anything locally. But for stateful applications — databases, message queues, file storage systems — losing all data on every restart is catastrophic.
>
> Kubernetes Persistent Volumes solve this by providing storage that exists OUTSIDE and INDEPENDENTLY of any individual pod. The storage is provisioned separately, and pods connect to it. If the pod dies and restarts, or even gets rescheduled to a different node — the same storage is reconnected to the new pod. The data survives.
>
> In our banking project, our PostgreSQL database used a PersistentVolume backed by AWS EBS. We deliberately tested this by force-deleting the database pod. The new pod came up, reconnected to the same EBS volume, and all transaction data was intact. That's the guarantee PV provides."

---

### Q2. What is the difference between emptyDir and hostPath?

**Answer:**

> "Both are simpler volume types that don't require a PV or PVC.
>
> emptyDir creates a fresh empty directory when the pod starts. All containers in that same pod can read and write to it — it's perfect for sharing data between containers in a pod, like a sidecar log-shipper reading logs written by the main container. The critical point: when the POD is deleted, the emptyDir and all its contents are permanently deleted. It's truly ephemeral — scoped to the pod's lifetime.
>
> hostPath mounts a directory from the HOST NODE's actual filesystem into the container. The data exists on the node and survives pod deletion. But it's tied to that specific node — if the pod moves to a different node, the data doesn't follow. This is the main danger. I only use hostPath for DaemonSets that need to read node-level files — like Fluentd reading container logs from /var/log, or monitoring agents reading /proc and /sys. Never for application data in production.
>
> Security note: hostPath is a security risk because a pod with hostPath can potentially read or write sensitive files on the host. Many security policies block it."

---

### Q3. What is a PersistentVolume (PV)? Who creates it?

**Answer:**

> "A PersistentVolume is a cluster-level storage resource that represents an actual piece of storage — an EBS volume, an NFS share, a Ceph block device. It's created and managed by cluster administrators, not by application developers.
>
> The PV describes: how much storage is available (capacity), what access modes are supported (RWO, RWX), what happens to the data when the PVC is deleted (reclaim policy), and a reference to the actual underlying storage (the EBS volumeID, the NFS server address, etc.).
>
> PVs are not namespaced — they're cluster-level resources. Any PVC in any namespace can claim a PV (as long as the specs match).
>
> In static provisioning, the admin creates PVs manually before developers request storage. In dynamic provisioning with a StorageClass, PVs are created automatically by the provisioner when a PVC is submitted — the admin doesn't need to pre-create anything."

---

### Q4. What is a PersistentVolumeClaim (PVC)? How does it relate to a PV?

**Answer:**

> "A PVC is a namespaced request for storage made by a developer or application. The developer says 'I need 20GB of storage with ReadWriteOnce access from the fast-ssd class' — they don't care where it comes from or what the underlying storage technology is.
>
> Kubernetes matches the PVC to a suitable PV by checking three things: the storage size (PV must have at least as much as PVC requests), the access modes (PV must support what PVC needs), and the storageClassName (both must match).
>
> When a match is found, Kubernetes binds the PVC to that PV — both objects change status to Bound. The PVC now has exclusive access to that PV — no other PVC can claim the same PV simultaneously.
>
> The pod then references the PVC name in its volume definition. The pod doesn't reference the PV directly — it goes PVC, which points to PV, which points to actual storage. This indirection is the power — developers work with PVCs and are completely isolated from infrastructure details."

---

### Q5. What are the three access modes? Which cloud storage supports which?

**Answer:**

> "There are three main access modes — and knowing which storage supports which is genuinely important in production.
>
> **ReadWriteOnce (RWO)** — only ONE NODE at a time can mount this volume, but multiple pods on that same node can all read and write to it. Most cloud block storage supports only RWO — AWS EBS, GCP Persistent Disk, Azure Disk. This is the most common mode for databases.
>
> **ReadOnlyMany (ROX)** — multiple nodes can mount simultaneously but only for reading. All mounts are read-only. Used for distributing shared config or reference data.
>
> **ReadWriteMany (RWX)** — multiple nodes can mount simultaneously and all can read AND write. This is what you need for shared storage. EBS does NOT support this. NFS supports it, AWS EFS supports it, CephFS supports it, Azure Files supports it.
>
> There's also a fourth mode in newer Kubernetes: **ReadWriteOncePod (RWOP)** — only a single pod in the entire cluster can mount it. Stricter than RWO which is per-node.
>
> Common interview trap: RWO means one NODE, not one pod. Multiple pods on the same node can all use an RWO volume."

---

### Q6. What are the three reclaim policies? When do you use each?

**Answer:**

> "Reclaim policy controls what happens to the PV and the underlying storage when the PVC that claimed it is deleted.
>
> **Retain** — the PV is NOT deleted. It goes to Released state with the data still intact. The underlying storage (EBS volume, NFS share) is preserved. An admin must manually clean up and decide whether to reuse it. I always use Retain for production databases. Accidentally deleting a PVC should never destroy your database data. In our banking project, all database PVs had Retain policy — even if someone ran `kubectl delete pvc postgres-pvc` by mistake, the EBS volume was still there.
>
> **Delete** — when the PVC is deleted, the PV is also deleted AND the underlying storage is destroyed. The AWS EBS volume is terminated. Data is permanently gone. I use Delete for dev/test environments or temporary workloads where the data has no long-term value — it also prevents cloud storage cost accumulation from forgotten volumes.
>
> **Recycle** — deprecated, don't use. It used to wipe the data with rm -rf and make the PV available again. Replaced by dynamic provisioning which handles cleanup more elegantly."

---

## SECTION B — StorageClass AND DYNAMIC PROVISIONING

---

### Q7. What is a StorageClass? Why was it introduced?

**Answer:**

> "A StorageClass is a template that tells Kubernetes HOW to automatically create PVs on demand. It defines the provisioner (which CSI driver to use), parameters (what type of disk, encryption, IOPS), and policies (reclaim, binding mode).
>
> It was introduced to solve the scalability problem of static provisioning. With static provisioning, an admin manually creates every PV before it can be claimed. In a large organization with 50 teams submitting storage requests, this becomes a full-time job. It's slow, error-prone, and doesn't scale.
>
> With dynamic provisioning via StorageClass, the admin sets up the StorageClass once and then steps back. When a developer submits a PVC with a storageClassName, Kubernetes automatically calls the provisioner, creates the underlying storage (EBS volume in AWS), creates a PV, and binds the PVC — all without admin involvement. The whole process takes seconds.
>
> I think of StorageClass as 'storage tier definitions.' In our banking cluster we had three: `standard` for development (gp2 EBS, no IOPS guarantee), `fast` for production databases (gp3 EBS with 3000 IOPS), and `archive` for log storage (st1 EBS, cheap, sequential access). Developers pick the tier they need; the infrastructure is invisible to them."

---

### Q8. ⚠️ What is `volumeBindingMode: WaitForFirstConsumer` and why is it critical in AWS?

**Answer:**

> "volumeBindingMode controls WHEN the PV is created and the PVC is bound.
>
> With `Immediate` — the PV is created as soon as the PVC is submitted. The problem: Kubernetes doesn't know yet which node will run the pod. The AWS EBS volume gets created in some availability zone — say us-east-1a. Later when the pod is scheduled, if it lands on a node in us-east-1b — the pod cannot start. EBS volumes can only be attached to EC2 instances in the SAME availability zone. You get a scheduling failure that's extremely confusing to debug.
>
> With `WaitForFirstConsumer` — PV creation is delayed until a pod is actually trying to use the PVC. By then, the Scheduler has already decided which node the pod will run on. Kubernetes knows 'this pod goes to a node in us-east-1a.' The EBS volume is created in us-east-1a. Node and volume are in the same zone. It works.
>
> This is not just best practice — it's essential correctness in multi-AZ cloud deployments. Always use WaitForFirstConsumer for cloud block storage StorageClasses. I learned this the hard way: an early cluster at Atos used Immediate, and about 30% of pod startups failed with cryptic volume attachment errors until we understood the AZ mismatch issue and switched to WaitForFirstConsumer."

---

### Q9. What is `allowVolumeExpansion` and how do you actually resize a PVC?

**Answer:**

> "allowVolumeExpansion is a boolean field in the StorageClass that controls whether PVCs using that class can be expanded after creation. When set to true, a developer can edit a PVC and request more storage — Kubernetes and the CSI driver handle resizing the underlying storage.
>
> To resize a PVC: `kubectl patch pvc my-pvc -p '{\"spec\":{\"resources\":{\"requests\":{\"storage\":\"50Gi\"}}}}'` — changing from the original size to the new larger size.
>
> Two things to know: first, you can ONLY expand, never shrink. Reducing a PVC's storage request is not supported and will be rejected. Second, after the PVC is updated, the filesystem inside the container may need to be expanded too. For EBS with ext4, this usually happens automatically when the pod restarts. For some storage types you may need to manually run resize2fs inside the container.
>
> In our banking environment, we had a reporting database that was filling up its 100GB volume. We patched the PVC to 200GB, the EBS volume expanded online (EBS supports live expansion), and the filesystem resized on the next pod restart — no downtime, no data migration needed."

---

### Q10. ⚠️ What is the default StorageClass and what happens if a PVC doesn't specify one?

**Answer:**

> "A default StorageClass is marked with the annotation `storageclass.kubernetes.io/is-default-class: 'true'`. When a PVC is submitted with no storageClassName field, Kubernetes automatically assigns it the default StorageClass and proceeds with dynamic provisioning.
>
> The tricky scenario: what if there's NO default StorageClass? A PVC with no storageClassName goes into Pending state and waits forever for a matching PV. No dynamic provisioning happens. This is a common source of confusion — developers create PVCs without specifying storageClassName assuming Kubernetes will 'just figure it out,' but if there's no default, it doesn't.
>
> Another tricky scenario: a PVC can explicitly opt OUT of dynamic provisioning by setting `storageClassName: ''` — an empty string. This tells Kubernetes 'do NOT use any StorageClass — only bind to a PV that also has no StorageClass.' This is used for manually pre-provisioned PVs in environments where you don't want dynamic provisioning.
>
> Check default: `kubectl get storageclass` — the default has `(default)` next to its name. Only one should be default. If multiple are marked default, behavior is undefined."

---

## SECTION C — CSI DRIVERS

---

### Q11. What is CSI and why was it introduced?

**Answer:**

> "CSI stands for Container Storage Interface — it's a standardized API specification that defines how storage systems integrate with container orchestrators like Kubernetes.
>
> Before CSI existed, storage drivers were built directly into the Kubernetes codebase — what people call in-tree plugins. To add support for a new storage vendor, someone had to submit code to the Kubernetes repository, wait for it to be reviewed and merged, and wait for a Kubernetes release. This meant storage innovation was bottlenecked by Kubernetes release cycles — typically 3-4 months per release.
>
> With CSI, storage vendors write their driver as a completely separate project. The driver is deployed in the cluster as pods — a controller deployment and a DaemonSet on every node. Kubernetes calls the driver through the standard CSI interface — gRPC calls for CreateVolume, DeleteVolume, AttachVolume, MountVolume. The vendor can release updates to their driver independently of Kubernetes.
>
> This is why all modern cloud storage uses CSI drivers — AWS EBS CSI driver, GCP PD CSI driver, Azure Disk CSI driver. The old in-tree plugins like `awsElasticBlockStore` are deprecated and will be removed. If you're creating new clusters today, always use CSI-based StorageClasses."

---

### Q12. What are the components of a CSI driver? What does each do?

**Answer:**

> "A CSI driver has two main components deployed in Kubernetes.
>
> The **CSI Controller** runs as a Deployment — typically one or a few replicas, usually on control plane nodes. It handles volume lifecycle operations that communicate with the storage backend: CreateVolume (make the EBS volume), DeleteVolume (terminate it), ControllerPublishVolume (attach the EBS volume to an EC2 instance), ControllerUnpublishVolume (detach it). These operations call cloud APIs.
>
> The **CSI Node Plugin** runs as a DaemonSet — one pod on every worker node. It handles the node-local operations: NodeStageVolume (format the disk and mount it to a staging path on the node), NodePublishVolume (bind mount from staging to the pod's directory). These operations actually put the volume into the pod's filesystem.
>
> There are also sidecar containers that bridge Kubernetes and the CSI driver: the external-provisioner watches for new PVCs and calls CreateVolume; the external-attacher watches for pods that need volumes and calls ControllerPublishVolume; the external-resizer watches for PVC size increases and calls ControllerExpandVolume.
>
> In practice as a DevOps engineer, I don't interact with these components directly — I install the CSI driver via Helm or an EKS addon, verify it's running, and then create StorageClasses pointing to it."

---

### Q13. What is the difference between AWS EBS CSI and AWS EFS CSI? When do you use each?

**Answer:**

> "EBS and EFS are fundamentally different storage types with different Kubernetes use cases.
>
> **AWS EBS CSI** creates Elastic Block Store volumes — these are block devices that attach to a single EC2 instance at a time. In Kubernetes terms, EBS only supports ReadWriteOnce. One node can mount the volume. If a pod is rescheduled to a different node, the EBS volume detaches from the old node and attaches to the new one — this attachment/detachment takes 10-30 seconds, causing brief downtime during pod migration. I use EBS for single-instance databases — PostgreSQL, MySQL, Redis — where you want dedicated, high-performance block storage.
>
> **AWS EFS CSI** creates Elastic File System mount points — these are managed NFS shares that any number of EC2 instances can mount simultaneously. In Kubernetes terms, EFS supports ReadWriteMany. Any number of nodes can mount the same EFS at the same time, all reading and writing. I use EFS when multiple pods across different nodes need to share the same files — shared configuration, user-uploaded files, ML model storage that multiple inference pods need to read.
>
> The tradeoff: EBS is faster (block device, lower latency), EFS is more flexible (multi-node access). For a banking transaction database — EBS. For shared report templates that 20 reporting pods need to read — EFS."

---

## SECTION D — STATEFULSET STORAGE

---

### Q14. ⚠️ How does storage work in a StatefulSet? What is volumeClaimTemplates?

**Answer:**

> "Regular Deployments with a PVC have a problem for stateful applications: all replicas share the same PVC. If you scale a PostgreSQL Deployment to 3 replicas and they all mount the same PVC, all three database processes are writing to the same files simultaneously — immediate data corruption.
>
> StatefulSets solve this with volumeClaimTemplates. Instead of referencing an existing PVC, you define a TEMPLATE for a PVC in the StatefulSet spec. Kubernetes creates one individual PVC from this template for each pod replica — dedicated, exclusive storage per pod.
>
> For a StatefulSet with 3 replicas and a volumeClaimTemplate named 'data', Kubernetes creates: data-postgres-0, data-postgres-1, data-postgres-2 — three separate PVCs, each bound to their own PV, each mounted exclusively by their respective pod.
>
> The critical behavior: when a StatefulSet pod is deleted and recreated, the new pod gets the SAME PVC — data-postgres-1 always goes to pod postgres-1. The storage identity follows the pod identity. This is why StatefulSets are essential for databases — postgres-1 always reconnects to its own data, not someone else's.
>
> Most important caveat: when you DELETE a StatefulSet, the PVCs created by volumeClaimTemplates are NOT deleted automatically. This is intentional — Kubernetes protects you from accidental data loss. You must manually delete the PVCs afterward if you want to clean up the storage."

---

### Q15. ⚠️ A StatefulSet has 3 replicas. You delete the StatefulSet. What happens to the PVCs?

**Answer:**

> "The PVCs are NOT deleted. This is deliberate protective behavior by Kubernetes.
>
> When you run `kubectl delete statefulset postgres`, the pods are terminated and the StatefulSet object is removed. But the three PVCs — data-postgres-0, data-postgres-1, data-postgres-2 — remain in the cluster in Bound state, each still holding their PV. The underlying storage (EBS volumes in AWS) is also untouched.
>
> This is a safety mechanism. StatefulSets often run databases. Accidentally deleting a StatefulSet should not cascade into destroying the database storage. The data is preserved.
>
> To fully clean up, you must manually delete the PVCs: `kubectl delete pvc data-postgres-0 data-postgres-1 data-postgres-2`. If the StorageClass has reclaimPolicy: Delete, deleting the PVCs will then delete the PVs and the underlying EBS volumes. If reclaimPolicy is Retain, the EBS volumes stay even after PVC deletion.
>
> This is an interview question people get wrong — they assume deleting the StatefulSet cleans everything up. It doesn't."

---

## SECTION E — TROUBLESHOOTING SCENARIOS

---

### Q16. ⚠️ A PVC is stuck in Pending state. What are the causes and how do you debug?

**Answer:**

> "PVC stuck in Pending means Kubernetes cannot find a suitable PV to bind it to. My debugging steps:
>
> First: `kubectl describe pvc <name>` — read the Events section. The event message tells you exactly why it's pending. Common messages: 'no persistent volumes available for this claim and no storage class is set' (no storageClassName and no default class), 'waiting for first consumer to be created before binding' (WaitForFirstConsumer mode, needs a pod first — this is normal, not an error).
>
> Second: check StorageClass. `kubectl get storageclass` — does the StorageClass named in the PVC exist? Is there a default if no class is specified? If the StorageClass doesn't exist, the PVC waits forever.
>
> Third: for static provisioning, check if matching PVs exist. `kubectl get pv` — is there a PV with matching storageClassName, sufficient capacity, and matching accessModes? If a PV has `storageClassName: fast` but PVC requests `storageClassName: standard` — no match.
>
> Fourth: check if the CSI driver is healthy. `kubectl get pods -n kube-system | grep csi` — if the CSI controller is crashing, dynamic provisioning fails. `kubectl describe pod <csi-controller-pod>` for error details.
>
> Fifth: for WaitForFirstConsumer — is there a pod trying to use this PVC? Without a pod, the PVC intentionally stays Pending. Create the pod and the PVC will bind.
>
> In our banking cluster, we had PVCs stuck Pending because someone created a StorageClass with a typo in the provisioner name. The CSI driver was running fine but Kubernetes was calling a non-existent provisioner. `kubectl describe pvc` showed 'no volume plugin matched' in the events."

---

### Q17. ⚠️ A pod is stuck in Pending with error 'had volume node affinity conflict'. What happened?

**Answer:**

> "This is the availability zone mismatch problem. The PV was created in one AZ — say us-east-1a — but the pod is being scheduled on a node in us-east-1b. EBS volumes can only be attached to EC2 instances in the same AZ. The Scheduler cannot place the pod where its volume lives.
>
> This happens when the StorageClass uses `volumeBindingMode: Immediate`. The PVC was bound and the EBS volume was created before any pod existed — Kubernetes picked an AZ arbitrarily. When the pod later gets scheduled, it might land in a different AZ.
>
> The fix: change StorageClass to `volumeBindingMode: WaitForFirstConsumer`. Delete the existing PVC and PV (the EBS volume will be deleted if reclaimPolicy is Delete). Recreate the PVC. Now wait — Kubernetes waits for the pod to be scheduled first. Once it knows the pod goes to us-east-1b, it creates the EBS volume in us-east-1b. Volume and node are in the same AZ. Pod starts.
>
> For existing stuck pods with existing PVCs: you need to either move the pod to a node in the same AZ (using node affinity), or delete the PVC, delete the PV (or make it available), recreate the PVC, and let WaitForFirstConsumer create the volume in the correct AZ."

---

### Q18. ⚠️ You deleted a PVC but the PV shows status 'Released' not 'Available'. Why? How do you fix it?

**Answer:**

> "Released means the PVC that was bound to this PV has been deleted, and the reclaim policy is Retain — so the PV was not deleted. The data is still there. But the PV has a claimRef field that still points to the old (now deleted) PVC.
>
> Released is NOT the same as Available. Kubernetes intentionally does not make a Released PV available for new PVCs automatically, even with Retain policy. This is to prevent a new workload from accidentally accessing data left by the previous workload.
>
> The fix — to make the PV available for a new PVC: `kubectl patch pv <pv-name> -p '{\"spec\":{\"claimRef\":null}}'`. This removes the claimRef, clearing the association with the old PVC. The PV status changes to Available. Now a new PVC with matching specs can claim it.
>
> If you want to use this PV with a SPECIFIC new PVC (to ensure only that PVC can claim it), create the PVC first and then patch the PV's claimRef to point to the new PVC explicitly — a technique called static binding or pre-binding."

---

### Q19. A pod was running with a PVC. The pod was deleted. The PVC was deleted. New pod needs fresh storage. What happened to the data and what do you do?

**Answer:**

> "What happened depends entirely on the reclaim policy of the underlying StorageClass.
>
> If reclaimPolicy was Delete: when the PVC was deleted, Kubernetes automatically deleted the PV and the underlying storage (EBS volume). The data is permanently gone. The new pod needs a new PVC which will dynamically provision a fresh empty volume.
>
> If reclaimPolicy was Retain: when the PVC was deleted, the PV went to Released state. The EBS volume still exists with all the old data. The admin needs to decide: recover the data, manually make the PV Available again, and bind the new PVC to it — OR delete the old PV and provision fresh storage.
>
> For the new pod in either case: create a new PVC with the same StorageClass. If Retain policy and you want a fresh start, also delete the old PV first (after backing up data if needed). If Delete policy, just create the new PVC — dynamic provisioning creates a fresh volume automatically.
>
> The lesson: in production, ALWAYS use Retain reclaim policy for databases. `kubectl delete pvc` should never silently delete your production database. With Retain, it's a two-step process — PVC deletion then manual PV cleanup — giving you a chance to catch accidental deletions."

---

## SECTION F — QUICK FIRE QUESTIONS

| Question | Answer |
|----------|--------|
| emptyDir data survives pod restart? | ✅ Yes (container restart), ❌ No (pod deletion) |
| emptyDir data survives pod deletion? | ❌ No — deleted with the pod |
| hostPath data survives pod deletion? | ✅ Yes — it's on the host node |
| hostPath works across different nodes? | ❌ No — tied to specific node |
| PV is namespaced? | ❌ No — cluster-level resource |
| PVC is namespaced? | ✅ Yes |
| What does PVC status Pending mean? | No matching PV found / waiting for first consumer |
| What does PV status Released mean? | PVC deleted, PV retained, not available for new PVCs |
| How to make Released PV available again? | `kubectl patch pv <name> -p '{"spec":{"claimRef":null}}'` |
| Can you shrink a PVC? | ❌ No — only expand |
| What does allowVolumeExpansion enable? | PVC size can be increased after creation |
| What StorageClass annotation marks it as default? | `storageclass.kubernetes.io/is-default-class: "true"` |
| RWO means one pod or one node? | One NODE (not one pod — common mistake) |
| Which access mode allows multi-node read+write? | ReadWriteMany (RWX) |
| Does EBS support ReadWriteMany? | ❌ No — only ReadWriteOnce |
| Does EFS support ReadWriteMany? | ✅ Yes |
| What is volumeBindingMode: WaitForFirstConsumer? | Create volume only after pod is scheduled — ensures correct AZ |
| What is volumeBindingMode: Immediate? | Create volume as soon as PVC is submitted (AZ risk) |
| What is CSI? | Container Storage Interface — standard for storage plugins |
| Are PVCs from volumeClaimTemplates deleted when StatefulSet is deleted? | ❌ No — must be manually deleted |
| What happens to PV with Delete policy when PVC is deleted? | PV and underlying storage are deleted |
| What happens to PV with Retain policy when PVC is deleted? | PV stays in Released state, data preserved |
| What is static provisioning? | Admin manually creates PV before PVC claims it |
| What is dynamic provisioning? | StorageClass auto-creates PV when PVC is submitted |
| Which CSI driver for EBS? | ebs.csi.aws.com |
| Which CSI driver for EFS? | efs.csi.aws.com |
| emptyDir medium: Memory means? | Volume backed by RAM (tmpfs) not disk — faster, lost on node reboot |
| What is RWOP access mode? | ReadWriteOncePod — only one pod in entire cluster can mount |

---

## SECTION G — THINGS THAT SOUND IMPRESSIVE IN AN INTERVIEW

1. **"The single most important storage design decision in production is reclaim policy. I always set Retain for databases and Delete for stateless storage. One accidental `kubectl delete pvc` with a Delete policy and your production database is gone. With Retain, it's a two-step process — PVC then PV — giving you a safety net."**

2. **"We had an incident where pods were stuck Pending with volume node affinity conflicts after migrating to a three-AZ setup. The StorageClass was set to volumeBindingMode: Immediate — it had always worked in a single-AZ cluster. Moving to three AZs exposed the problem. Switching to WaitForFirstConsumer fixed it in minutes. It's now the first thing I check when creating StorageClasses in AWS."**

3. **"In our banking project we separated storage tiers as StorageClasses: fast-ssd for transaction databases (gp3, 3000 IOPS, encrypted), standard for application storage (gp2), and archive for audit logs (st1 cold storage, cost-optimized). Developers just pick the class they need — the infrastructure is completely transparent to them. This is the right level of abstraction."**

4. **"People often confuse ReadWriteOnce with 'only one pod.' It's actually one NODE. If you have three pods of the same Deployment all scheduled on the same node, they can all mount an RWO volume. The restriction is at the node attachment level, not the pod level. This matters when designing storage for high-density node setups."**

5. **"For the StatefulSet volumeClaimTemplates behavior — PVCs not being deleted when StatefulSet is deleted — I actually think of this as Kubernetes being opinionated about data safety in the right way. Kubernetes is saying 'I created this database storage on your behalf, but I will not destroy it without explicit intent.' The extra manual step to delete PVCs is a deliberate speed bump against accidental data loss. I've seen this prevent real disasters."**

---

*File: K8s_Storage_Interview_Questions.md*
*Repository: Interview_Preparation_2026 → Kubernetes/*
*Companion file: K8s_Storage_Concept_and_Lab.md*
