# Kubernetes Controller Manager — Interview Questions & Answers
> Target: 4 Years DevOps Experience | Senior-Level Interviews
> Style: Easy to memorize + Professional to say out loud

---

## 💡 HOW TO USE THIS FILE

- Read the question first
- Try to answer in your own words
- Then read the answer — notice how it's simple but sounds professional
- The answer is written so you can **say it out loud** in an interview

---

## SECTION A — BASIC CONCEPT QUESTIONS

---

### Q1. What is the Kubernetes Controller Manager?

**Answer:**

> "The Controller Manager is a core control plane component that runs a set of controllers — each controller is a background loop that continuously watches the cluster state and takes action to make the actual state match the desired state. It's the automation engine of Kubernetes. Without it, Kubernetes would just store your YAML but never act on it.
>
> It's a single binary — one process — but it runs many controllers inside it simultaneously, like Deployment Controller, ReplicaSet Controller, Node Controller, and many others. In a high-availability setup, multiple instances run but only one is active at a time, using leader election."

---

### Q2. What is the Reconciliation Loop? Explain it simply.

**Answer:**

> "The reconciliation loop is the core pattern every controller follows. It has three steps: Watch, Compare, and Act.
>
> First, the controller watches the API Server for any changes. Second, it compares what the user WANTS — the desired state stored in etcd — versus what ACTUALLY exists in the cluster right now. Third, if there's a difference, it takes action to close that gap.
>
> Simple example: you say `replicas: 3` in your Deployment. Two pods are running. The ReplicaSet Controller detects the gap — desired is 3, actual is 2 — and creates one more pod. This loop runs forever, every few seconds, which is why Kubernetes is self-healing."

---

### Q3. What is the difference between Desired State and Actual State?

**Answer:**

> "Desired state is what you told Kubernetes you want — it's stored in etcd. For example, 'I want 3 replicas of nginx running.' Actual state is what's really happening in the cluster at this moment — maybe only 2 pods are running because one crashed.
>
> The controller's entire job is to detect when these two states don't match and take action to reconcile them. When they match — nothing happens. When they don't match — the controller fixes it. This model is what makes Kubernetes declarative — you declare what you want and Kubernetes figures out how to get there."

---

### Q4. Where does the Controller Manager run?

**Answer:**

> "The Controller Manager runs on the Control Plane — the master node. In a Kubernetes cluster managed by kubeadm, it runs as a static pod in the `kube-system` namespace. You can see it with `kubectl get pods -n kube-system | grep controller-manager`.
>
> It only talks to the API Server — it never reads from etcd directly. The API Server is the only component that reads and writes to etcd."

---

### Q5. Can there be multiple Controller Managers running at the same time?

**Answer:**

> "Yes — in a high-availability production cluster, you typically run 3 control plane nodes, so you have 3 instances of Controller Manager. But only ONE instance is active (the leader) at any time. The other two are on standby.
>
> This is handled by **leader election**. The active Controller Manager holds a Kubernetes Lease object and renews it every few seconds. If it fails to renew — which happens if the leader node crashes — the other two instances race to acquire the lease. Whichever one acquires it first becomes the new leader and starts processing. This prevents multiple controllers from fighting over the same resources and creating duplicate objects."

---

## SECTION B — TYPES OF CONTROLLERS

---

### Q6. What does the Deployment Controller do?

**Answer:**

> "The Deployment Controller manages the lifecycle of Deployments. When you create a Deployment, this controller creates a ReplicaSet. When you update the Deployment — for example, change the image — it creates a NEW ReplicaSet for the new version and slowly scales it up while scaling down the old ReplicaSet. This is the rolling update mechanism.
>
> It also handles rollbacks — if you run `kubectl rollout undo`, the Deployment Controller scales the old ReplicaSet back up and scales the new one down. It keeps a history of old ReplicaSets for exactly this purpose, controlled by `revisionHistoryLimit`."

---

### Q7. What is the difference between Deployment Controller and ReplicaSet Controller?

**Answer:**

> "These two work as a team but at different levels.
>
> The Deployment Controller is at the higher level — it manages ReplicaSets. It creates them, updates them during rolling updates, and uses them to implement rollbacks. It's concerned with the strategy of HOW updates happen.
>
> The ReplicaSet Controller is at the lower level — it manages pods directly. Its only job is to make sure the right number of pods are running at all times. If you need 3 pods and only 2 exist, it creates one more. If there are 4, it deletes one. It knows nothing about updates or rollbacks — that's the Deployment Controller's job.
>
> You almost never create a ReplicaSet directly — you create a Deployment, and the Deployment Controller creates the ReplicaSet for you."

---

### Q8. What does the Node Controller do? What are the timers involved?

**Answer:**

> "The Node Controller monitors the health of worker nodes. kubelet on each node sends heartbeats to the API Server every few seconds. If the Node Controller stops receiving heartbeats, it marks the node as `NotReady` after a grace period of **40 seconds**. If the node remains NotReady for **5 minutes**, the Node Controller evicts all pods from that node. Pods managed by Deployments or StatefulSets get rescheduled on healthy nodes. DaemonSet pods are not rescheduled because they're supposed to run on specific nodes.
>
> These timers are configurable via the `--node-monitor-grace-period` and `--pod-eviction-timeout` flags on the Controller Manager."

---

### Q9. What is a Job and what does the Job Controller do?

**Answer:**

> "A Job is a Kubernetes object for running a task that needs to complete — not run forever. Think database migrations, batch processing, sending reports. Unlike a Deployment which keeps pods running indefinitely, a Job creates pods, waits for them to complete successfully, and then considers itself done.
>
> The Job Controller watches Job objects. When a new Job appears, it creates pods to run the task. If a pod fails, it creates a new pod to retry, up to the `backoffLimit`. If a pod succeeds with exit code 0, the Job is marked as Complete. The pods are kept in Completed state afterward so you can check logs — they aren't deleted immediately."

---

### Q10. What is a CronJob and how does it differ from a Job?

**Answer:**

> "A CronJob is a Job that runs on a schedule — like a cron job in Linux. You define a schedule in cron format (for example, `0 2 * * *` for 2 AM daily), and the CronJob Controller creates a new Job object each time the schedule fires. That Job is then handled by the Job Controller.
>
> The key difference: a Job runs once — now or when created. A CronJob runs repeatedly on a schedule. The CronJob object is the schedule definition; each time it fires, it creates a fresh Job object."

---

### Q11. What does the Endpoint Controller do?

**Answer:**

> "The Endpoint Controller maintains the Endpoints object for each Service. A Service routes traffic to pods using a label selector, but pods come and go — their IPs change. The Endpoint Controller continuously watches all pods and updates the Endpoints object with the current IPs of pods that match the Service selector AND are in the Ready state.
>
> kube-proxy on every node reads these Endpoints and creates iptables rules accordingly. So if a pod's readiness probe starts failing, the Endpoint Controller removes its IP from the Endpoints → kube-proxy updates iptables → no more traffic is sent to that pod. When the pod becomes Ready again, its IP is re-added. This is the mechanism behind Service load balancing."

---

### Q12. What does the DaemonSet Controller do and when would you use a DaemonSet?

**Answer:**

> "The DaemonSet Controller ensures that exactly one pod runs on every node in the cluster — or every node matching a node selector. When a new node joins the cluster, the DaemonSet Controller automatically creates the specified pod on it. When a node is removed, that pod is cleaned up.
>
> DaemonSets are used for node-level agents — software that needs to run on every single node:
> - `kube-proxy` — must manage iptables rules on every node for Service routing
> - CNI plugins like Calico or Cilium — must configure pod networking on every node
> - Log collectors like Fluentd or Fluent Bit — log files are local to each node, so the collector must run where the logs are
> - Monitoring agents like node-exporter — must read CPU and memory metrics from each specific node"

---

### Q13. What is the HPA Controller and what does it need to work?

**Answer:**

> "The Horizontal Pod Autoscaler controller automatically scales the number of pod replicas based on observed metrics — typically CPU or memory usage. You define minimum replicas, maximum replicas, and a target metric threshold. For example: 'keep CPU usage at 50%, scale between 2 and 10 replicas.'
>
> The HPA Controller checks metrics every 15 seconds. If CPU goes above threshold, it scales up. If CPU drops well below, it scales down after a cooldown period to avoid thrashing.
>
> For it to work, **metrics-server must be installed** in the cluster. The HPA Controller reads from the metrics API, which metrics-server provides. Without metrics-server, HPA cannot read CPU/memory values and will show an error."

---

## SECTION C — DEEP CONCEPT QUESTIONS

---

### Q14. How does a Controller communicate with the API Server? What is List-Watch?

**Answer:**

> "Controllers never talk to etcd directly — they only talk to the API Server. The communication pattern is called List-Watch.
>
> When a controller starts, it first does a **List** call — 'give me all existing Deployments right now' — to get the current state of the cluster. Then it issues a **Watch** call — 'now send me a notification whenever any Deployment is added, modified, or deleted.'
>
> The API Server sends event notifications back: ADDED, MODIFIED, or DELETED. The controller puts each event into a work queue and processes them. This is efficient because the controller doesn't have to poll repeatedly — changes are pushed to it in real time."

---

### Q15. What is an Informer in Kubernetes?

**Answer:**

> "An Informer is the code pattern in the Kubernetes Go client library that implements the List-Watch mechanism efficiently. It combines listing objects, watching for changes, caching the results locally in memory, and triggering event handlers when objects change.
>
> The local cache means that when a controller needs to look up an object, it reads from the in-memory cache instead of making an API call every time. This dramatically reduces API Server load, especially in large clusters with many controllers running simultaneously."

---

### Q16. How does leader election work in the Controller Manager?

**Answer:**

> "When running multiple control plane nodes for high availability, each node runs a Controller Manager instance. To prevent them from all running reconciliation loops simultaneously and creating conflicts, they use leader election.
>
> The mechanism uses a Kubernetes **Lease object** — essentially a distributed lock stored in etcd. The active Controller Manager holds this Lease and renews it every 2 seconds. If renewal stops — because the leader crashed or lost connectivity — the Lease expires after 10-15 seconds. The standby instances detect this and race to acquire the Lease. Whichever one acquires it first becomes the new leader and starts running. You can see the current leader by running `kubectl get lease kube-controller-manager -n kube-system`."

---

### Q17. What happens to running pods if the Controller Manager goes down?

**Answer:**

> "Pods that are already running will keep running. The Controller Manager doesn't directly run pods — kubelet on each worker node does. kubelet maintains pods locally and doesn't need the Controller Manager to keep them alive.
>
> However, the automatic healing stops. If a pod crashes while the Controller Manager is down, no replacement is created because the ReplicaSet Controller isn't running to detect the gap. If a node fails, no pods are rescheduled. If you change a Deployment, nothing happens.
>
> As soon as the Controller Manager comes back, it runs the reconciliation loop and catches up on everything that drifted while it was down. Kubernetes is designed to be eventually consistent — it converges to the desired state whenever controllers are running."

---

### Q18. What is the `--controllers` flag and how could you use it?

**Answer:**

> "The `--controllers` flag on the Controller Manager binary specifies which controllers to run. By default it's set to `*` which means run all built-in controllers. You could specify individual controllers to include or exclude.
>
> For example: `--controllers=*,-ttl` would run all controllers EXCEPT the TTL controller. The minus sign means 'exclude this one.'
>
> In practice, this is useful for advanced scenarios like running a custom controller externally while still using the built-in ones — or during troubleshooting to isolate a specific controller's behavior. In most production setups, you leave it as the default `*`."

---

### Q19. What is `revisionHistoryLimit` in a Deployment and which controller uses it?

**Answer:**

> "When you update a Deployment — say you change the image tag — the Deployment Controller creates a new ReplicaSet for the new version and keeps the old ReplicaSet around with 0 replicas. It does this so you can rollback to the previous version using `kubectl rollout undo`.
>
> `revisionHistoryLimit` controls how many of these old ReplicaSets to keep. Default is 10. So Kubernetes keeps the last 10 versions of your Deployment and you can rollback to any of them.
>
> In our banking project, we reduced this to 3 because keeping 10 old ReplicaSets was accumulating objects in etcd and increasing its size unnecessarily. Reducing to 3 saved significant etcd storage."

---

### Q20. What is a StatefulSet Controller and how is it different from a ReplicaSet Controller?

**Answer:**

> "The StatefulSet Controller manages stateful applications — databases, message queues, anything where each pod needs a unique, stable identity. The key differences from the ReplicaSet Controller:
>
> First, **pod identity is stable** — pods are named postgres-0, postgres-1, postgres-2 — and if postgres-1 is recreated, it comes back with the same name and same DNS hostname.
>
> Second, **each pod gets its own PersistentVolumeClaim** — its own dedicated storage. This storage survives pod restarts and is reattached to the same pod when it comes back.
>
> Third, **ordering is enforced** — pods are created in sequence (0 must be Running before 1 starts) and deleted in reverse order. This is critical for databases where a primary must be up before replicas connect to it.
>
> The ReplicaSet Controller treats all pods as identical and interchangeable — it doesn't care which specific pod runs, just that the count is correct."

---

## SECTION D — SCENARIO-BASED QUESTIONS

---

### Q21. A pod crashes. Walk me through exactly what happens — which controllers are involved?

**Answer:**

> "When a pod crashes:
>
> 1. **kubelet** (on the worker node) detects the container has exited. It tries to restart it based on `restartPolicy`.
>
> 2. If the pod keeps crashing (CrashLoopBackOff), **kubelet** updates the pod's status in the API Server — sets phase to Failed and reports the restart count.
>
> 3. The **ReplicaSet Controller** is watching pods via List-Watch. It receives the notification: 'pod is gone or Failed.' It checks: 'Desired replicas = 3, actual Running pods = 2. Gap of 1.'
>
> 4. ReplicaSet Controller calls the API Server: 'Create a new pod object with this spec.'
>
> 5. The new pod object is written to etcd, with no node assigned yet.
>
> 6. The **Scheduler** detects the unscheduled pod, finds the best node, and writes the node name into the pod object.
>
> 7. **kubelet** on the assigned node detects 'there's a pod assigned to me.' It pulls the image and starts the container.
>
> 8. Pod becomes Running. **Endpoint Controller** adds the new pod's IP to the Service endpoints.
>
> Total time: typically 10-30 seconds end to end."

---

### Q22. You notice old ReplicaSets piling up in your cluster. What created them and how do you manage this?

**Answer:**

> "Old ReplicaSets are created by the Deployment Controller each time you update a Deployment. Every time you change the image, environment variable, or any pod template field, the Deployment Controller creates a new ReplicaSet for the new version and keeps the old one (with 0 replicas) for rollback purposes.
>
> If you deploy frequently without managing this, you'll accumulate dozens of old ReplicaSets in etcd, increasing its database size.
>
> The solution is `revisionHistoryLimit` in the Deployment spec. Set it to a low value like 3 or 5 — this tells the Deployment Controller to keep only the last N ReplicaSets and delete older ones automatically.
>
> You can also clean up existing old ones with: `kubectl delete replicaset -l app=<name> --field-selector 'status.replicas=0'` — this deletes all ReplicaSets for that app that currently have 0 replicas (the old unused ones)."

---

### Q23. You delete a Namespace and it's stuck in "Terminating" forever. What is happening?

**Answer:**

> "When you delete a namespace, the Namespace Controller marks it as 'Terminating' and then starts deleting all resources inside it. It waits for EVERY resource inside to be fully deleted before removing the namespace itself.
>
> If it's stuck in Terminating, it means some resource inside the namespace has a **finalizer** that isn't being cleaned up. A finalizer is a protection mechanism — it says 'don't delete me until this specific cleanup action is done first.' If the controller responsible for that cleanup is gone or not running, the finalizer never gets removed, and the namespace waits forever.
>
> Common causes: custom resources (CRDs) whose controller is not installed, PersistentVolumes waiting for external cleanup, or API services that have been removed.
>
> Emergency fix (use with caution in production): patch the namespace to remove finalizers manually: `kubectl patch namespace <name> -p '{"spec":{"finalizers":[]}}' --type=merge`. This forces the namespace to delete without waiting for finalizers. Make sure the resources inside are actually safe to abandon before doing this."

---

### Q24. What happens if you have `replicas: 5` in your Deployment but also manually create pods with the same labels?

**Answer:**

> "The ReplicaSet Controller doesn't care HOW pods were created. It only cares about the COUNT of pods matching the selector labels. If your ReplicaSet selector is `app: nginx` and you manually create 2 extra pods with label `app: nginx`, the controller now counts 7 pods (5 from ReplicaSet + 2 manual) and sees a gap: 'Desired=5, Actual=7 → too many.'
>
> The ReplicaSet Controller will **delete 2 pods** to bring the count back to 5. It will pick which ones to delete (usually the newest or the extra ones). Your manually created pods are not safe — they will be deleted to maintain the desired count.
>
> This is why you should never manually create pods with labels that match an existing ReplicaSet selector — the controller will clean them up automatically."

---

### Q25. Explain what happens during a rolling update — which controllers are involved and in what order?

**Answer:**

> "A rolling update is triggered when you change any field in the pod template of a Deployment — typically the image tag.
>
> **Deployment Controller** detects the change. It creates a new ReplicaSet with the new pod template. Both old and new ReplicaSets now exist.
>
> Based on `maxSurge` (how many extra pods can exist above desired count) and `maxUnavailable` (how many pods can be down at once), the Deployment Controller orchestrates the shift:
>
> 1. New ReplicaSet scaled up by 1 (new pod created by ReplicaSet Controller)
> 2. Wait for new pod to become Ready (readiness probe passes, Endpoint Controller adds it)
> 3. Old ReplicaSet scaled down by 1 (old pod deleted)
> 4. Repeat until all pods are on the new version
>
> The old ReplicaSet is kept with 0 replicas for rollback purposes. If at any point the new pods fail to become Ready, the Deployment Controller stops the rollout — it won't continue deleting old pods while the new ones are broken. This is the protective behavior that prevents a bad deployment from taking down all your pods."

---

## SECTION E — QUICK FIRE QUESTIONS (For Rapid-Fire Rounds)

| Question | Answer |
|----------|--------|
| Which namespace does Controller Manager run in? | `kube-system` |
| How many Controller Managers can be ACTIVE at once? | Only 1 (leader election) |
| What object is used for leader election? | Kubernetes Lease object |
| How long before a node is marked NotReady? | 40 seconds |
| How long before pods are evicted from a NotReady node? | 5 minutes |
| Which controller creates ReplicaSets? | Deployment Controller |
| Which controller creates pods? | ReplicaSet Controller (and StatefulSet, DaemonSet, Job controllers) |
| Does Controller Manager talk directly to etcd? | No — only via API Server |
| What pattern does every controller follow? | Reconciliation Loop (Watch → Compare → Act) |
| What does HPA need to work? | metrics-server installed |
| Which controller handles namespace deletion cleanup? | Namespace Controller |
| What is `backoffLimit` in a Job? | Maximum retries before Job is marked Failed |
| What flag controls how many old ReplicaSets to keep? | `revisionHistoryLimit` |
| Which controller removes pod IPs from Service when pod is not Ready? | Endpoint Controller |
| What is the cron expression for "every day at midnight"? | `0 0 * * *` |

---

## SECTION F — THINGS THAT SOUND IMPRESSIVE IN AN INTERVIEW

Use these naturally — don't memorize word for word, understand the idea:

1. **"Kubernetes is declarative — you declare what you want and the Controller Manager's reconciliation loops continuously enforce it."**

2. **"Every controller follows the same Watch-Compare-Act pattern. This consistency is why Kubernetes is so extensible — you can write your own custom controllers using the same pattern with tools like kubebuilder or operator-sdk."**

3. **"In our banking project, we reduced revisionHistoryLimit from the default 10 to 3 because we deploy multiple times a day and the old ReplicaSets were accumulating in etcd. After 3 months, this would have been hundreds of stale ReplicaSet objects."**

4. **"Leader election via Lease objects means zero-downtime control plane upgrades — when you drain a leader node, the other standby Controller Managers automatically elect a new leader within 15 seconds."**

5. **"The Endpoint Controller is the bridge between the Service abstraction and actual pod network addresses. Understanding this is key to debugging service connectivity issues — if endpoints are empty, your traffic will always fail no matter how the pods look."**

---

*File: K8s_ControllerManager_Interview_Questions.md*
*Repository: Interview_Preparation_2026 → Kubernetes/*
*Companion file: K8s_ControllerManager_Concept_and_Lab.md*
