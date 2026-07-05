# Kubernetes Imperative Commands, dry-run & YAML Generation
## Interview Questions & Answers
> Target: 4 Years DevOps Experience | Senior-Level Interviews
> Includes: Tricky questions, scenario-based, CKA exam style

---

## 💡 HOW TO USE THIS FILE

- Read the question
- Try answering yourself first
- Read the answer — written to say out loud in an interview
- Tricky questions are marked with ⚠️

---

## SECTION A — BASICS

---

### Q1. What is the difference between imperative and declarative approach in Kubernetes?

**Answer:**

> "Imperative means you tell Kubernetes what to do directly through commands — `kubectl create deployment nginx --image=nginx`. You're giving step-by-step instructions. It's fast and great for quick tasks but leaves no file record.
>
> Declarative means you write a YAML file describing the desired state and tell Kubernetes 'make reality match this file' using `kubectl apply -f`. Kubernetes figures out what changes are needed. The file is the source of truth — you can commit it to Git, review it in PRs, and reproduce it anywhere.
>
> In production I use declarative — YAML files in Git, applied through CI/CD. In interviews, exams, and quick debugging I use imperative. The best workflow combines both: use imperative commands with `--dry-run=client -o yaml` to generate the YAML base, edit it, then commit and apply declaratively."

---

### Q2. What does `--dry-run=client` do and why is it useful?

**Answer:**

> "`--dry-run=client` tells kubectl to build the Kubernetes object locally in memory and show you what it would send to the API Server — but without actually sending anything. Nothing is created in the cluster.
>
> It's useful for three things. First, YAML generation — combine it with `-o yaml` to get a perfect YAML template for any object type in seconds, instead of writing it from scratch with risk of indentation errors. Second, quick validation — confirm your command syntax is correct before running it for real. Third, learning — you can see exactly what fields Kubernetes uses for any object type without affecting anything.
>
> The output with `-o yaml` is a complete, valid YAML that you can save to a file, edit, and apply. This is the standard workflow for CKA exam and production template creation."

---

### Q3. What is the difference between `--dry-run=client` and `--dry-run=server`?

**Answer:**

> "Client dry-run simulates entirely on your local machine. kubectl builds the object in memory and prints it. The API Server is never contacted. No webhooks, no quota checks, no RBAC validation. It's purely local. Fast and works even when the cluster is unreachable. Use this for YAML generation.
>
> Server dry-run actually sends the request to the API Server. It runs through the complete admission pipeline — authentication, RBAC, mutating webhooks, schema validation, validating webhooks — but stops right before writing to etcd. So it checks real policies, real quotas, real OPA Gatekeeper rules. If it would fail in production, server dry-run tells you now.
>
> In our banking CI/CD pipeline, every deployment PR runs `kubectl apply -f --dry-run=server` against the staging cluster before the PR can merge. This catches policy violations and quota issues before anything reaches production."

---

### Q4. What does `-o yaml` do and what are the other output formats?

**Answer:**

> "`-o` means output format. By default kubectl prints a brief one-liner like `deployment.apps/nginx created`. With `-o yaml` it prints the complete YAML of the object — every field, every auto-generated value.
>
> The common output formats I use:
>
> `-o yaml` — full YAML, great for saving and editing.
> `-o json` — full JSON, useful for scripting and jsonpath extraction.
> `-o wide` — adds extra columns to list output like pod IP and node name.
> `-o name` — only prints the resource type and name, nothing else. Useful in scripts.
> `-o jsonpath='{.field.path}'` — extracts one specific field from the object. Powerful for automation.
> `-o custom-columns` — lets you define exactly which columns to show and what to call them.
>
> For generating templates I always use `-o yaml`. For scripting and extracting specific values I use `-o jsonpath`."

---

### Q5. How do you generate a Deployment YAML without writing it manually?

**Answer:**

> "I use `kubectl create deployment` with `--dry-run=client -o yaml` and redirect to a file:
>
> `kubectl create deployment web-app --image=nginx:1.25 --replicas=3 --dry-run=client -o yaml > deployment.yaml`
>
> This gives me a complete, syntactically correct Deployment YAML in seconds. Then I open the file and add what the imperative command can't set — resource requests and limits, readiness and liveness probes, environment variables from ConfigMaps or Secrets, volume mounts, node affinity, tolerations.
>
> The generated YAML has a `resources: {}` placeholder that I replace with actual values. After editing I run `kubectl apply -f deployment.yaml --dry-run=server` to validate it passes all cluster policies, then apply for real."

---

## SECTION B — EDITING AND MODIFYING OBJECTS

---

### Q6. ⚠️ You lost your deployment YAML file. The deployment is still running in Kubernetes. How do you change the replica count?

**Answer:**

> "Losing the YAML file doesn't matter — the object exists in etcd and I have multiple ways to change it without the file.
>
> Fastest way: `kubectl scale deployment web-app --replicas=5`. Done in one command, no editor needed.
>
> If I need to change other fields too: `kubectl edit deployment web-app`. This opens the full current YAML in vi. I find the `replicas` field, change the number, save and quit with `:wq`. Changes apply immediately.
>
> If I want to recover the YAML file for future use: `kubectl get deployment web-app -o yaml > web-app-recovered.yaml`. This gives me the full YAML from etcd. The recovered file has extra auto-generated fields like `resourceVersion`, `uid`, `creationTimestamp` — that's fine, `kubectl apply` handles them correctly.
>
> Going forward I'd commit this recovered YAML to Git so I don't lose it again."

---

### Q7. ⚠️ How do you change a container image on a running deployment without modifying any file?

**Answer:**

> "`kubectl set image deployment/web-app nginx=nginx:1.25`
>
> Breaking this down: `set image` is the subcommand. `deployment/web-app` is what to target — the format is resource-type/resource-name. `nginx=nginx:1.25` is container-name equals new-image-tag — you must know the container name (which is inside the deployment spec, usually same as the deployment name for simple deployments).
>
> This immediately triggers a rolling update — the Deployment Controller creates a new ReplicaSet with the updated image and gradually shifts pods from old to new. I can watch it with `kubectl rollout status deployment/web-app`.
>
> If the new image causes problems — `kubectl rollout undo deployment/web-app` rolls back to the previous version. Kubernetes kept the old ReplicaSet exactly for this purpose."

---

### Q8. What is `kubectl patch` and how is it different from `kubectl edit`?

**Answer:**

> "`kubectl patch` lets you change one specific field by providing just the changed portion as a JSON object. You don't open an editor, you don't see the full YAML — you surgically target one field.
>
> For example: `kubectl patch deployment web-app -p '{\"spec\":{\"replicas\":5}}'`. The `-p` flag takes a JSON string. Only the replicas field is changed. Everything else stays exactly as it is.
>
> `kubectl edit` opens the ENTIRE current YAML in a text editor. You can change multiple fields in one session. Better for complex changes where you need to see the full context.
>
> When to use which: I use `patch` for scripting and automation — when I know exactly which field to change and want to do it in a one-liner without human interaction. I use `edit` when I'm troubleshooting and need to see the full object and possibly change multiple things.
>
> Patch also supports JSON Merge Patch and Strategic Merge Patch types with `--type` flag, which are important for arrays — like adding a container to a pod spec without replacing the entire containers array."

---

### Q9. ⚠️ How do you change CPU and memory limits on a running deployment without editing any YAML file?

**Answer:**

> "`kubectl set resources deployment web-app --requests=cpu=200m,memory=256Mi --limits=cpu=500m,memory=512Mi`
>
> This updates the resource requirements directly on the live deployment. It triggers a rolling update because changing resources is a pod template change — existing pods need to be replaced with the new spec.
>
> If the deployment has multiple containers and I only want to change one: add `-c=container-name` to target a specific container.
>
> After running this, I'd also want to update my YAML file so the file stays in sync with reality: `kubectl get deployment web-app -o yaml > updated-deployment.yaml` — then commit to Git."

---

### Q10. ⚠️ What happens if you try to `kubectl edit` a Pod and change its container image?

**Answer:**

> "Pods are largely immutable after creation. Most fields in a pod's spec — including the container image, container name, resources, and volumes — cannot be changed on a running pod. If you try to edit the image in `kubectl edit pod nginx-pod`, kubectl will reject the save with an error: 'The Pod is invalid: spec.containers[0].image: Forbidden: pod updates may not change fields.'
>
> The correct approach: if you're using a Deployment (which you should be for production workloads), edit the Deployment — not the pod. The Deployment's pod template IS mutable. Change the image in the Deployment and it creates new pods with the new image through a rolling update.
>
> If you genuinely need to 'change' a pod — the only way is delete and recreate it. For a standalone pod: `kubectl delete pod nginx-pod` then recreate with the new image. But this causes downtime, which is exactly why you use Deployments instead of standalone pods."

---

### Q11. ⚠️ You have a running deployment but NO YAML file. How do you get the YAML back?

**Answer:**

> "`kubectl get deployment web-app -o yaml > web-app.yaml`
>
> This fetches the complete current specification from etcd and writes it to a file. The recovered YAML will include auto-generated fields that weren't in your original file — things like `resourceVersion`, `uid`, `creationTimestamp`, `generation`, and `managedFields`. These are fine to keep — `kubectl apply` is designed to handle them.
>
> However, if I want a cleaner file closer to what I originally wrote, I can filter these out. A common approach: pipe through `kubectl neat` (a plugin) which strips the auto-generated fields. Or manually remove the `status` section, `resourceVersion`, `uid`, `creationTimestamp`, and `managedFields` sections — these are all auto-managed by Kubernetes.
>
> The important thing: the object spec — `containers`, `replicas`, `selector`, `labels`, `resources`, `probes` — is all there and accurate. This is genuinely how you recover a lost YAML in production."

---

## SECTION C — GENERATING YAML FOR DIFFERENT OBJECTS

---

### Q12. How do you generate a Job YAML using imperative command?

**Answer:**

> "`kubectl create job db-migration --image=flyway --dry-run=client -o yaml -- sh -c 'flyway migrate'`
>
> The `--` separator is important. Everything after `--` is the command to run inside the container. Without the `--`, kubectl would try to interpret those words as its own flags.
>
> I save this to a file, then edit it to add things the imperative command can't set — like `backoffLimit` (how many times to retry on failure), `activeDeadlineSeconds` (max job duration before force-stopping), `ttlSecondsAfterFinished` (auto-cleanup delay after completion), and any environment variables or volume mounts the job needs.
>
> A common mistake: forgetting `restartPolicy: Never` on the pod template inside a Job. Jobs require either `Never` or `OnFailure` — not `Always` (which is the default for Deployments). The generated YAML sets this correctly, which is another reason to generate rather than write from scratch."

---

### Q13. How do you generate a Secret YAML with multiple key-value pairs?

**Answer:**

> "`kubectl create secret generic db-credentials --from-literal=username=admin --from-literal=password=SecurePass123 --dry-run=client -o yaml`
>
> Each `--from-literal` adds one key-value pair. The generated YAML automatically base64 encodes the values — you'll see them encoded in the `data` section. This is important: the values in a Secret YAML are always base64 encoded, not plain text.
>
> If I need to add a value manually to the YAML later, I must base64 encode it first: `echo -n 'mypassword' | base64`. The `-n` flag is critical — without it `echo` adds a newline character which gets encoded too, and the decoded value will have a hidden newline that breaks database connections.
>
> In production I wouldn't store plain credentials in YAML at all — I'd use External Secrets Operator pulling from AWS Secrets Manager. But for lab and exam scenarios this is the standard approach."

---

### Q14. ⚠️ How do you generate a Pod YAML that runs a specific command, not the default image command?

**Answer:**

> "I use `kubectl run` with `--command` flag:
>
> `kubectl run task-pod --image=busybox --command --dry-run=client -o yaml -- sh -c 'echo hello && sleep 3600'`
>
> The `--command` flag is what tells kubectl that everything after `--` overrides the container's ENTRYPOINT — the main command. Without `--command`, what comes after `--` is treated as ARGS passed TO the existing command — a different thing.
>
> So to be precise: ENTRYPOINT in Docker = `command` in Kubernetes pod spec. CMD in Docker = `args` in Kubernetes pod spec.
>
> With `--command`: `-- sh -c 'sleep 3600'` → pod spec has `command: [sh, -c, sleep 3600]`
> Without `--command`: `-- sleep 3600` → pod spec has `args: [sleep, 3600]` — these are passed to whatever the image's default ENTRYPOINT is."

---

### Q15. ⚠️ How do you create a pod that runs in a specific namespace, with a specific label, without writing a YAML file?

**Answer:**

> "`kubectl run nginx-pod --image=nginx --labels='app=web,env=prod' -n banking`
>
> Breaking this down:
> - `run nginx-pod` → create a Pod named nginx-pod
> - `--image=nginx` → use nginx image
> - `--labels='app=web,env=prod'` → add multiple labels, comma-separated
> - `-n banking` → create in the banking namespace
>
> To verify it was created correctly:
> `kubectl get pod nginx-pod -n banking --show-labels`
>
> If I also need environment variables: add `--env='DB_HOST=mysql' --env='DB_PORT=3306'`
>
> If I need to generate the YAML first and then edit: add `--dry-run=client -o yaml > pod.yaml` and add more complex fields like volumes, resource limits, probes before applying."

---

## SECTION D — TRICKY SCENARIOS

---

### Q16. ⚠️ You need to quickly test if a Kubernetes cluster can pull images from your private registry. What single command do you run and immediately clean up?

**Answer:**

> "`kubectl run registry-test --image=your-private-registry/app:latest --restart=Never && kubectl delete pod registry-test`
>
> `--restart=Never` creates a standalone Pod (not a Deployment). It runs once and stops. The `&&` means run the delete only if the create succeeded.
>
> Better version with auto-cleanup: `kubectl run registry-test --image=your-private-registry/app:latest --restart=Never --rm -it -- echo 'image pulled successfully'`
>
> `--rm` deletes the pod automatically when the command finishes. `--restart=Never` with `--rm` and a quick command is the perfect combination for ephemeral test pods.
>
> To watch if it's pulling or failing: `kubectl get pod registry-test -w` in another terminal — watch for `ErrImagePull` or `ImagePullBackOff` in the STATUS column."

---

### Q17. ⚠️ How do you see EVERY Kubernetes object in a namespace — including ConfigMaps, Secrets, ServiceAccounts — in one command?

**Answer:**

> "`kubectl get all -n banking` only shows the most common types — pods, services, deployments, replicasets, statefulsets, daemonsets, jobs. It does NOT show ConfigMaps, Secrets, ServiceAccounts, PersistentVolumeClaims, Ingresses.
>
> For truly everything: `kubectl get $(kubectl api-resources --verbs=list --namespaced -o name | tr '\n' ',' | sed 's/,$//') -n banking 2>/dev/null`
>
> Breaking this down: `kubectl api-resources --verbs=list --namespaced -o name` lists all namespaced resource types that support the list operation. `tr '\n' ','` converts newlines to commas making it a comma-separated list. `sed 's/,$//'` removes the trailing comma. The outer `kubectl get <all-types>` fetches all of them at once. `2>/dev/null` suppresses errors for resource types that aren't present.
>
> For day-to-day use I break it down: `kubectl get all,cm,secret,sa,pvc,ingress -n banking` — listing the types I care about explicitly."

---

### Q18. ⚠️ You need to change a field that `kubectl apply` says is immutable. What do you do?

**Answer:**

> "Some fields are truly immutable — like a Service's `clusterIP`, a StatefulSet's `selector`, or a PVC's storage class. Once created, they cannot be changed with `apply` or `edit` — you get an error like 'field is immutable.'
>
> The options:
>
> Option 1 — `kubectl replace --force -f updated.yaml`. This deletes the existing object and recreates it from the file. Brief downtime — the object doesn't exist for a second.
>
> Option 2 — Delete manually and recreate: `kubectl delete deployment web-app && kubectl apply -f updated-deployment.yaml`. Same effect as replace --force but more controlled.
>
> Option 3 — For Services with immutable clusterIP: delete the service, recreate with the new spec. The clusterIP will be reassigned (or you can request a specific one in the spec).
>
> For PVCs — you generally cannot resize the storage class or reduce size. You can increase size (if the StorageClass supports it) using `kubectl patch pvc my-pvc -p '{\"spec\":{\"resources\":{\"requests\":{\"storage\":\"20Gi\"}}}}'` — but only upward and only if the provisioner supports it."

---

### Q19. ⚠️ How do you run a one-time command inside a running pod to debug something, then clean up?

**Answer:**

> "For a running pod: `kubectl exec -it <pod-name> -n <namespace> -- bash` — this opens an interactive shell inside the running pod. When I'm done debugging, `exit` closes the shell. The pod keeps running.
>
> If the pod doesn't have bash: `kubectl exec -it <pod-name> -- sh` — sh is available in almost all images.
>
> If I need a fresh debug pod that doesn't affect running workloads: `kubectl run debug-pod --image=busybox --rm -it -n banking -- sh`
>
> For very specific commands without interactive shell: `kubectl exec <pod-name> -- curl http://payment-svc:8080/health`. No `-it` needed — just runs the command and prints output.
>
> For network debugging I use the `nicolaka/netshoot` image — it has all networking tools: `kubectl run netdebug --image=nicolaka/netshoot --rm -it -- sh`. From inside I can run `nslookup`, `curl`, `ping`, `traceroute`, `netstat`, `tcpdump` — everything needed for network troubleshooting."

---

### Q20. ⚠️ You want to create a deployment and immediately expose it as a service — all in one go without writing any YAML. How?

**Answer:**

> "Two commands back to back:
>
> `kubectl create deployment web-app --image=nginx --replicas=3`
> `kubectl expose deployment web-app --port=80 --type=ClusterIP --name=web-svc`
>
> Or as a one-liner using `&&` (run second only if first succeeds):
> `kubectl create deployment web-app --image=nginx --replicas=3 && kubectl expose deployment web-app --port=80 --type=ClusterIP --name=web-svc`
>
> To verify both: `kubectl get deployment web-app && kubectl get service web-svc && kubectl get endpoints web-svc`
>
> If I want to save the combined YAML for Git:
> ```
> kubectl create deployment web-app --image=nginx --replicas=3 --dry-run=client -o yaml > app.yaml
> echo '---' >> app.yaml
> kubectl expose deployment web-app --port=80 --dry-run=client -o yaml >> app.yaml
> kubectl apply -f app.yaml
> ```
> The `---` separator is the YAML document separator — multiple objects in one file are separated by it."

---

### Q21. ⚠️ How do you find which pods are using a specific image across all namespaces?

**Answer:**

> "`kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{\"\\t\"}{.metadata.name}{\"\\t\"}{range .spec.containers[*]}{.image}{\"\\n\"}{end}{end}' | grep nginx`
>
> Breaking this down:
> - `-A` → all namespaces
> - `-o jsonpath` → extract specific fields
> - `{range .items[*]}...{end}` → loop through all pods
> - `{.metadata.namespace}` → print namespace
> - `{\"\\t\"}` → tab separator
> - `{.metadata.name}` → print pod name
> - `{range .spec.containers[*]}{.image}` → loop through containers and print each image
> - `| grep nginx` → filter to only show lines with 'nginx' in the image name
>
> Simpler alternative using custom-columns:
> `kubectl get pods -A -o custom-columns='NAMESPACE:.metadata.namespace,POD:.metadata.name,IMAGE:.spec.containers[0].image'`"

---

### Q22. ⚠️ What is the difference between `kubectl create` and `kubectl apply`?

**Answer:**

> "`kubectl create` is purely imperative — it creates the object if it doesn't exist and FAILS if it already exists. No idempotency.
>
> `kubectl apply` is declarative and idempotent — if the object doesn't exist it creates it; if it already exists it compares the desired state (your file) with the current state (etcd) and makes only the necessary changes. Running `kubectl apply` twice with the same file is safe — second run sees no diff, does nothing.
>
> `kubectl apply` also tracks what it previously applied using an annotation called `kubectl.kubernetes.io/last-applied-configuration` stored in the object's metadata. It uses this to detect fields that were in the previous apply but removed in the new one — those are deleted. `kubectl create` doesn't have this intelligence.
>
> In production: always use `kubectl apply`. In quick scripts that run once: `kubectl create` is fine. In CI/CD pipelines: `kubectl apply` because the pipeline may run multiple times and idempotency is essential."

---

### Q23. ⚠️ How do you add a new environment variable to a running deployment without downtime?

**Answer:**

> "`kubectl set env deployment/web-app NEW_VAR=new_value -n banking`
>
> This adds the environment variable `NEW_VAR` with value `new_value` to ALL containers in the deployment. It triggers a rolling update — new pods get the new env var, old pods are gradually replaced. Since it's a rolling update, there's no downtime — pods are replaced one at a time while others keep serving traffic.
>
> To update an existing variable: same command — `kubectl set env` replaces the value if the key already exists.
>
> To remove an environment variable: `kubectl set env deployment/web-app NEW_VAR-` — the trailing minus `-` means remove this key.
>
> To see current environment variables: `kubectl set env deployment/web-app --list`
>
> After doing this imperatively, don't forget to update your YAML file: `kubectl get deployment web-app -o yaml > updated.yaml` — otherwise your file and cluster are out of sync, and the next `kubectl apply -f` from the old file would remove the new env var."

---

## SECTION E — CKA EXAM STYLE QUESTIONS

---

### Q24. Create a deployment named `web` using image `nginx:1.24` with 3 replicas in namespace `prod`. Save the YAML to `/tmp/web-deploy.yaml` and apply it.

**Answer (commands to run):**

```bash
# Step 1: Create namespace if it doesn't exist
kubectl create namespace prod

# Step 2: Generate YAML
kubectl create deployment web \
  --image=nginx:1.24 \
  --replicas=3 \
  -n prod \
  --dry-run=client \
  -o yaml > /tmp/web-deploy.yaml

# Step 3: Apply
kubectl apply -f /tmp/web-deploy.yaml

# Step 4: Verify
kubectl get deployment web -n prod
```

---

### Q25. A deployment `api-server` in namespace `banking` needs its replica count changed to 5. You have no YAML file. Do it.

**Answer (commands to run):**

```bash
# Fastest way
kubectl scale deployment api-server --replicas=5 -n banking

# Verify
kubectl get deployment api-server -n banking
```

---

### Q26. Create a ConfigMap named `app-settings` with `DB_HOST=mysql.banking.svc`, `DB_PORT=3306`, `LOG_LEVEL=INFO` in namespace `banking`. Generate the YAML and apply.

**Answer (commands to run):**

```bash
kubectl create configmap app-settings \
  --from-literal=DB_HOST=mysql.banking.svc \
  --from-literal=DB_PORT=3306 \
  --from-literal=LOG_LEVEL=INFO \
  -n banking \
  --dry-run=client \
  -o yaml | kubectl apply -f -
```

---

### Q27. Get all pod names and their images in namespace `banking` in format: `podname → image`

**Answer (commands to run):**

```bash
kubectl get pods -n banking \
  -o jsonpath='{range .items[*]}{.metadata.name}{" → "}{.spec.containers[0].image}{"\n"}{end}'
```

---

## SECTION F — QUICK FIRE QUESTIONS

| Question | Answer |
|----------|--------|
| Command to generate Deployment YAML without creating it? | `kubectl create deployment name --image=img --dry-run=client -o yaml` |
| How to save generated YAML to a file? | Add `> filename.yaml` at the end |
| How to append YAML to existing file? | Use `>>` instead of `>` |
| How to apply YAML piped from a command? | `... -o yaml \| kubectl apply -f -` |
| How to change replicas without YAML file? | `kubectl scale deployment name --replicas=N` |
| How to change image without YAML file? | `kubectl set image deployment/name container=image:tag` |
| How to change env var without YAML file? | `kubectl set env deployment/name KEY=value` |
| How to change resource limits without YAML file? | `kubectl set resources deployment name --limits=cpu=200m,memory=256Mi` |
| How to open a live object in editor? | `kubectl edit deployment name` |
| How to save editor changes? | `:wq` in vi / Ctrl+S in nano |
| How to change editor from vi to nano? | `KUBE_EDITOR="nano" kubectl edit deployment name` |
| How to patch one field via command? | `kubectl patch deployment name -p '{"spec":{"replicas":5}}'` |
| How to recover YAML of live object? | `kubectl get deployment name -o yaml > recovered.yaml` |
| What does `kubectl get all` NOT show? | ConfigMaps, Secrets, ServiceAccounts, PVCs, Ingresses |
| What is YAML document separator? | `---` (three dashes) |
| How to delete a pod immediately without graceful shutdown? | `kubectl delete pod name --force --grace-period=0` |
| How to run a temporary debug pod that auto-deletes? | `kubectl run debug --image=busybox --rm -it -- sh` |
| How to target a specific container in set commands? | Add `-c=container-name` flag |
| How to run a Job from a CronJob immediately? | `kubectl create job manual-now --from=cronjob/cronjob-name` |
| What field tracks what kubectl apply previously set? | `kubectl.kubernetes.io/last-applied-configuration` annotation |
| Can you edit a pod's image with kubectl edit? | No — pod spec is immutable after creation |
| What do you use instead? | Edit the Deployment (not the pod directly) |
| How to see rollout progress? | `kubectl rollout status deployment/name` |
| How to rollback a deployment? | `kubectl rollout undo deployment/name` |
| What does `--current-replicas` flag do in scale? | Safety check — only scale if current count equals this value |

---

## SECTION G — THINGS THAT SOUND IMPRESSIVE IN AN INTERVIEW

1. **"In the CKA exam and in day-to-day work, I never write YAML from scratch. I generate the base with `--dry-run=client -o yaml`, save it, then add only what's missing — resources, probes, volumes. This approach is faster and produces correct indentation every time."**

2. **"When someone says they lost their YAML file, I ask them — is the object still running? If yes, `kubectl get <object> -o yaml` gives you everything back. etcd is the ground truth. The file is just a representation of what's there."**

3. **"We put `--dry-run=server` in our CI pipeline as a mandatory gate before any production deployment. It validates the real admission webhooks, real quota limits, and real RBAC — things `--dry-run=client` completely ignores. We've caught OPA Gatekeeper violations in PRs that would have failed silently at 2 AM."**

4. **"The `kubectl set image` command is my preferred way to do a manual hotfix in production — faster than editing files, immediately triggers a rolling update, and if something is wrong `kubectl rollout undo` brings the old version back in under 30 seconds. The whole cycle takes less than a minute."**

5. **"One thing that catches junior engineers: `kubectl get all` does not actually get ALL. It skips ConfigMaps, Secrets, Ingresses, ServiceAccounts, and PVCs. I always tell my team — if something seems missing from the namespace, run `kubectl get all,cm,secret,sa,pvc,ingress -n namespace` to see the complete picture."**

---

*File: K8s_ImperativeCommands_DryRun_YAML_Interview_Questions.md*
*Repository: Interview_Preparation_2026 → Kubernetes/*
*Companion file: K8s_ImperativeCommands_DryRun_YAML_Concept_and_Lab.md*
