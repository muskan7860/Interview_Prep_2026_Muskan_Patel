# Kubernetes Admission Controllers — Interview Questions & Answers
## Mutating Webhooks, Validating Webhooks, API Request Flow
> Target: 4 Years DevOps Experience | Senior-Level Interviews
> Style: Easy to memorize + Professional to say out loud

---

## 💡 HOW TO USE THIS FILE

- Read the question first
- Try to answer in your own words
- Then read the answer — it is written to be said out loud in an interview
- Pay attention to the real-world banking examples — use them to make your answer sound experienced

---

## SECTION A — BASIC CONCEPT QUESTIONS

---

### Q1. What is an Admission Controller in Kubernetes?

**Answer:**

> "An Admission Controller is a piece of code that intercepts every API request AFTER authentication and authorization but BEFORE the object is written to etcd. It sits inside the API Server pipeline as a gatekeeper.
>
> There are two types. A Mutating Admission Controller can read the incoming request and MODIFY it — adding fields, setting defaults, injecting containers. A Validating Admission Controller can only read the final object and either ALLOW or DENY it — no modifications.
>
> They're the enforcement layer for cluster policies. Without admission controllers, anyone who's authenticated and authorized could create any object regardless of whether it follows your organization's standards — no resource limits, no approved images, no required labels."

---

### Q2. What is the complete request flow when you run `kubectl apply`?

**Answer:**

> "When I run `kubectl apply`, the request goes through six steps inside the API Server before anything is saved.
>
> Step one: **Authentication** — the API Server verifies who I am by checking my client certificate against the cluster CA. If invalid — 401 Unauthorized.
>
> Step two: **Authorization** — RBAC check. Am I allowed to perform this operation on this resource in this namespace? If not — 403 Forbidden.
>
> Step three: **Mutating Admission Webhooks** — any registered mutating webhooks are called. They can modify the object — adding sidecars, injecting labels, setting default resource limits. The object may come out different from what I sent.
>
> Step four: **Schema Validation** — the API Server validates the MUTATED object against the schema for that resource type. Required fields present? Correct types? If invalid — 422 Unprocessable Entity.
>
> Step five: **Validating Admission Webhooks** — external validators see the final, mutated object and either allow or deny it. If any validator says no — request is rejected with the webhook's error message.
>
> Step six: **Written to etcd** — all checks passed. Object persisted. 201 Created returned to kubectl."

---

### Q3. Why do Mutating webhooks run BEFORE Validating webhooks? Can't it be the other way?

**Answer:**

> "The order is intentional and it has to be this way for a logical reason.
>
> The validator needs to see the FINAL version of the object — including everything the mutator added. If validation ran first, it might reject an object that was about to be fixed by mutation.
>
> Example: you have a policy 'all pods must have resource limits.' A pod comes in with no limits. The mutating webhook is supposed to ADD default limits. If validation ran first — it would reject the pod for having no limits, before the mutator got a chance to add them. The mutator never fires, the pod is always rejected even though it WOULD have been valid after mutation.
>
> By running mutators first: the limits get added, then the validator checks the COMPLETE object — limits are present — validation passes. This is the correct behavior."

---

### Q4. What is the difference between a Mutating and a Validating webhook?

**Answer:**

> "The key difference is what they can DO with the request.
>
> A Mutating webhook can READ the incoming object AND CHANGE IT. It returns a JSON Patch — a list of specific modifications to apply. The API Server applies these patches and the object going forward is the modified version. Think of it as a form-filler — you hand in an incomplete form and it fills in the blanks.
>
> A Validating webhook can only READ the final object and make a binary decision — allow or deny. It cannot change a single field. Think of it as an inspector — it can reject your form but it cannot fill it in.
>
> A third important point: some controllers are BOTH. LimitRanger, for example, first adds default limits (mutating behavior), then checks that the final limits are within the allowed min-max range (validating behavior)."

---

### Q5. What is `failurePolicy` in a webhook configuration? When would you use `Ignore` vs `Fail`?

**Answer:**

> "failurePolicy determines what happens when the webhook server itself is unreachable — it's DOWN, it timed out, or returned an error. The API Server has to make a decision: let the request through or block it?
>
> `failurePolicy: Ignore` means — if the webhook is unreachable, treat it as if the webhook said 'allowed.' The request goes through. This prioritizes AVAILABILITY — even if the webhook is down, the cluster keeps working.
>
> `failurePolicy: Fail` means — if the webhook is unreachable, block the request. This prioritizes SECURITY — no request gets through without being checked by the policy.
>
> Which to use depends on the use case. For a sidecar injector (Istio), `Ignore` is appropriate — if injection fails, you'd rather have a pod without a sidecar than have pod creation fail completely. For a security policy enforcer (OPA Gatekeeper blocking unapproved images), `Fail` is the right choice — a policy that can be bypassed by making the webhook unavailable is not a real policy.
>
> In our banking environment, all security-related validating webhooks were set to `Fail`. We had high availability for those webhook servers (3 replicas) to ensure the policy enforcement was always online."

---

## SECTION B — DEEP CONCEPT QUESTIONS

---

### Q6. What is an AdmissionReview object?

**Answer:**

> "The AdmissionReview is the JSON object that the API Server sends to a webhook and expects to receive back. It's the communication protocol between the API Server and webhook servers.
>
> The REQUEST side (API Server sends to webhook) contains: a unique UID for tracking, the kind and version of object being created, the operation type (CREATE, UPDATE, DELETE), the user who made the request, and the full object specification.
>
> The RESPONSE side (webhook sends back) contains: the same UID — must match so the API Server knows which request this responds to — an `allowed` boolean (true or false), and for mutating webhooks a `patch` field containing a base64-encoded JSON Patch with the modifications to make. For denials, a `status` object with a `message` field — this is the exact text the user will see in their kubectl error output.
>
> The UID matching is important — webhooks process requests asynchronously and the UID ensures the API Server correctly matches each response to its request."

---

### Q7. What is a JSON Patch and how does a mutating webhook use it?

**Answer:**

> "A JSON Patch is a standardized format (RFC 6902) for describing changes to a JSON document. Instead of sending back the entire modified object, the mutating webhook sends back a list of specific operations.
>
> Each operation has three parts: `op` which is the operation type (add, remove, replace, copy, move), `path` which is the location in the JSON document using slash notation like `/spec/containers/0/resources/limits`, and `value` which is the new value to set.
>
> The API Server applies these patches to the original object sequentially. This is more efficient than sending the whole object back — especially for large objects — and it makes the changes explicit and auditable.
>
> For example, Istio's sidecar injection patch adds an entire container definition at `/spec/containers/-` (the `-` means append to the array), adds init containers, and adds volume mounts. All of this in a single patch response."

---

### Q8. What is `namespaceSelector` in a webhook configuration and why is it important?

**Answer:**

> "namespaceSelector is a filter that controls which namespaces the webhook applies to. It uses label selectors — the same pattern as Service selectors and pod scheduling.
>
> Without a namespaceSelector, a webhook fires for EVERY request matching its rules across ALL namespaces. This creates two problems: performance (every pod creation across the whole cluster calls the webhook) and safety (the webhook fires for system namespaces like kube-system which might break critical components).
>
> Istio uses namespaceSelector with `istio-injection: enabled` — the webhook only fires for pods in namespaces that have this label. You control sidecar injection at the namespace level just by adding or removing that label. Other namespaces are untouched.
>
> There's also a safety recommendation: always add `namespaceSelector` to exclude `kube-system` from your webhooks. If a validating webhook is set to failurePolicy: Fail and it fires for system components during an upgrade, it could block critical system pods from being created — potentially breaking the control plane."

---

### Q9. What built-in admission controllers do you know? Explain three in detail.

**Answer:**

> "There are many built-in controllers compiled into the API Server. Three I use most in practice:
>
> **LimitRanger** — this is a mutating AND validating controller. When a pod is created without resource requests and limits, LimitRanger reads the namespace's LimitRange object and injects defaults. Then it validates that the final requests and limits are within the min-max range defined in the LimitRange. In our banking cluster, we used this so developers could create pods without thinking about resource limits — LimitRanger ensured sensible defaults, preventing any single pod from consuming unlimited resources.
>
> **ResourceQuota** — a validating controller that enforces namespace-level resource budgets. Before admitting any object, it checks the current usage plus the requested addition against the quota. If adding the new object would exceed the quota — pod count, total CPU, total memory — it rejects with a clear message. We used this in banking to give each team namespace a fixed resource budget and prevent any team from accidentally consuming the entire cluster.
>
> **NamespaceLifecycle** — validates that you're not creating objects in a namespace that is being deleted (in Terminating state), and prevents deletion of the built-in namespaces like `default`, `kube-system`, and `kube-public`. Small but critical — without it someone could accidentally delete kube-system."

---

### Q10. How does Istio sidecar injection work as a mutating webhook?

**Answer:**

> "Istio deploys a mutating webhook called the sidecar injector. It registers a MutatingWebhookConfiguration that matches Pod CREATE operations in namespaces labeled with `istio-injection: enabled`.
>
> When you create a pod in such a namespace, the API Server calls Istio's webhook server (the istiod service). Istio reads the incoming pod spec and returns a JSON Patch that adds: an init container called `istio-init` that configures iptables rules to intercept all network traffic, a second container called `istio-proxy` (the Envoy sidecar) that handles all inbound and outbound traffic, and the necessary volume mounts and environment variables for both.
>
> The pod that gets created has these containers even though the developer never wrote them. From the developer's perspective they wrote one container. In reality the pod runs three.
>
> This is the foundation of a service mesh — every pod automatically gets a proxy without developer involvement. The proxies then form the mesh, handling mTLS, traffic routing, and observability transparently."

---

### Q11. What is OPA Gatekeeper and how does it work?

**Answer:**

> "OPA Gatekeeper is an open-source policy engine for Kubernetes that works as a validating webhook. OPA stands for Open Policy Agent — it's a general-purpose policy engine that uses a language called Rego to express policies.
>
> The architecture: Gatekeeper deploys a validating webhook that intercepts all CREATE and UPDATE operations. Policies are defined as ConstraintTemplates (which define the Rego logic and the shape of the policy) and Constraints (which are instances of a template with specific configuration — like 'require resource limits in the banking namespace').
>
> When a request comes in, Gatekeeper evaluates ALL active Constraints against the incoming object. If any Constraint is violated, Gatekeeper returns `allowed: false` with the violation message. The user sees that message.
>
> In our banking project, we used Gatekeeper for: requiring resource limits on all pods, enforcing that all images came from our internal ECR registry, requiring specific labels (team, cost-center, environment), and blocking privileged containers. All policies as code, stored in Git, applied via GitOps pipeline."

---

## SECTION C — SCENARIO-BASED QUESTIONS

---

### Q12. A developer runs `kubectl apply` and gets a 403 error. How do you determine if it's RBAC or a validating webhook?

**Answer:**

> "This is a great troubleshooting question because both RBAC denials and validating webhook denials can show up as 403 errors.
>
> First, look at the error message carefully. An RBAC denial looks like: `Error from server (Forbidden): pods is forbidden: User 'dev-user' cannot create resource 'pods' in API group '' in the namespace 'production'` — it mentions the user, the resource, and the namespace explicitly. A webhook denial looks like: `Error from server (Forbidden): admission webhook 'validate-resources.company.com' denied the request: Resource limits are required` — it mentions the WEBHOOK NAME and shows your custom policy message.
>
> If the message mentions `admission webhook` — it's the webhook. If it mentions the user and resource without a webhook name — it's RBAC.
>
> To investigate further: try `kubectl auth can-i create pods -n production` — if this returns `no` → RBAC problem. If it returns `yes` but apply still fails → webhook is blocking it.
>
> Then check which webhooks are active: `kubectl get validatingwebhookconfigurations` and `kubectl describe validatingwebhookconfiguration <name>` to see the rules and which namespaces they apply to."

---

### Q13. Your cluster is having performance issues. You trace it to admission webhooks. What would you do?

**Answer:**

> "Webhook performance issues happen when webhooks add latency to every request they intercept. Every matched request waits for the webhook to respond before proceeding.
>
> First, I'd identify which webhooks are slow. I'd check API Server logs for webhook call durations. Kubernetes logs slow webhook calls with a warning. I'd also look at webhook server metrics if they expose Prometheus metrics — request duration histogram.
>
> Second, I'd review the webhook rules for scope creep. A common mistake is setting `resources: ['*']` or `apiGroups: ['*']` — this makes the webhook fire for EVERY object creation, including system objects that don't need policy checks. I'd narrow the rules to only the resources that actually need this webhook.
>
> Third, I'd add a `namespaceSelector` if missing — don't call the webhook for kube-system and other system namespaces.
>
> Fourth, I'd check the `timeoutSeconds` setting. Keep it at 5-10 seconds maximum. A webhook that takes 30 seconds to respond adds 30 seconds to every pod creation.
>
> Fifth, I'd ensure the webhook server has enough replicas and resources. A single-replica webhook that's OOMKilling adds latency for every request that hits the failurePolicy timeout.
>
> In our banking cluster, we had a webhook that was validating ALL resources including Lease renewals (which happen every 2 seconds from controller managers). This was adding 50-100ms to every Lease update — causing leader election instability. Narrowing the rules to only pods and deployments fixed it."

---

### Q14. How would you implement a policy that blocks all pods using `latest` image tag in production namespace?

**Answer:**

> "I'd implement this as a validating webhook. Two approaches depending on the tooling available.
>
> **Option 1: OPA Gatekeeper (preferred in enterprise)**
> I'd create a ConstraintTemplate with Rego logic that checks all containers and init containers in a pod for image tags. If any image ends with `:latest` or has no tag at all (implying latest), the constraint is violated. I'd then create a Constraint that applies this template to the `production` namespace. All of this goes in Git and gets applied via CI/CD.
>
> **Option 2: Kyverno (simpler, Kubernetes-native)**
> Kyverno lets you write policies in YAML without learning Rego. A ClusterPolicy with a `deny` rule that checks `request.object.spec.containers[].image` for the `:latest` pattern — two or three lines of YAML.
>
> **Option 3: Custom webhook**
> If neither tool is available, I'd write a small HTTPS server (Python/Go) that receives the AdmissionReview, loops through containers checking the image tag, and returns allowed/denied accordingly. Deploy it as a Deployment with a Service in the cluster, get the CA cert, and register it as a ValidatingWebhookConfiguration.
>
> In our banking project, we used Kyverno for most policies because writing Rego has a steep learning curve and Kyverno policies are readable by anyone who knows Kubernetes YAML."

---

### Q15. A validating webhook is `failurePolicy: Fail` and the webhook server crashes. What happens to the cluster?

**Answer:**

> "If the webhook server crashes and failurePolicy is Fail, the impact depends on what resources that webhook covers.
>
> For any request matching the webhook's rules — the API Server tries to call the webhook, gets a connection error or timeout, applies failurePolicy: Fail, and REJECTS the request. This means: no new pods can be created if pods match the rules, no deployments can be updated, no scaling operations. Anything that matches the webhook rules is blocked.
>
> EXISTING running pods are unaffected — admission webhooks only fire on CREATE and UPDATE operations, not on already-running things.
>
> The cluster doesn't fall apart but it becomes unable to change. This is actually the INTENDED behavior for security webhooks — better to block changes than let unvalidated changes through.
>
> Recovery: restart the webhook server Deployment. Because the webhook is deployed as a Kubernetes Deployment, Kubernetes itself will restart it — but if the webhook rules cover pods and the webhook is down, the replacement pods might not start. This is the classic catch-22. Solution: use `namespaceSelector` to EXCLUDE the webhook's own namespace from its rules, so the webhook can restart itself even when the cluster is in a blocked state.
>
> This is why in production we always run webhook servers with at least 3 replicas spread across different nodes. Single replica failurePolicy:Fail webhooks are an availability risk."

---

### Q16. What is the difference between `--dry-run=client` and `--dry-run=server`?

**Answer:**

> "`--dry-run=client` means kubectl validates the YAML structure LOCALLY before sending it. It checks if the YAML syntax is correct and if the field names exist — but it does this using kubectl's local knowledge of the Kubernetes API schema. It does NOT contact the API Server. No webhooks are called. No quota is checked. It's fast but shallow.
>
> `--dry-run=server` sends the request to the API Server and runs it through the FULL pipeline — authentication, authorization, mutating webhooks, schema validation, validating webhooks — but stops before writing to etcd. This is a true test of whether your request would be accepted.
>
> The critical difference in practice: `--dry-run=client` will tell you 'your YAML looks valid.' `--dry-run=server` will tell you 'your YAML is valid AND OPA Gatekeeper would allow it AND you haven't exceeded your quota.'
>
> Before deploying to production, I always use `--dry-run=server` as part of the CI pipeline. It catches policy violations, quota issues, and webhook rejections before the actual deployment — fail fast, fail in the pipeline not in production."

---

## SECTION D — ARCHITECTURE QUESTIONS

---

### Q17. How does a webhook server prove its identity to the API Server?

**Answer:**

> "The webhook server uses TLS — it presents a certificate when the API Server connects to it. The API Server needs to verify this certificate is legitimate, not from an attacker who has hijacked the webhook's IP.
>
> The verification uses the `caBundle` field in the MutatingWebhookConfiguration or ValidatingWebhookConfiguration. This field contains a base64-encoded CA certificate. When the API Server connects to the webhook server over HTTPS, the webhook presents its TLS cert. The API Server checks: is this certificate signed by the CA in caBundle? If yes — trusted connection. If no — connection rejected, failurePolicy applied.
>
> This means the webhook operator must: generate a CA keypair, sign the webhook server's certificate with that CA, put the webhook server's cert and key on the server, and put the CA cert (base64 encoded) in the caBundle field of the webhook configuration.
>
> Tools like cert-manager automate this entire process — they generate the certs, rotate them before expiry, and can automatically inject the caBundle into webhook configurations using annotations. In production, using cert-manager for webhook certs is standard practice."

---

### Q18. Can a single webhook serve both mutating and validating functions?

**Answer:**

> "Yes — the webhook server is just an HTTPS server with endpoints. Nothing stops you from having `/mutate` and `/validate` endpoints on the same server, registered separately in a MutatingWebhookConfiguration and a ValidatingWebhookConfiguration.
>
> Many tools do this. OPA Gatekeeper, for example, runs as a single deployment but can be configured to both mutate objects (adding default policy annotations) and validate them (enforcing constraints) — two separate webhook registrations pointing to the same service but different paths.
>
> The API Server treats them completely independently. The mutating registration fires in step 3, the validating registration fires in step 5 — even if they call the same server. The server just handles each request based on the path."

---

## SECTION E — QUICK FIRE QUESTIONS

| Question | Answer |
|----------|--------|
| At what step do mutating webhooks run? | Step 3 — after AuthZ, before schema validation |
| At what step do validating webhooks run? | Step 5 — after schema validation, after mutation |
| Why do mutating webhooks run before validating? | Validator must see the final (mutated) object |
| Can a validating webhook modify the object? | NO — it can only allow or deny |
| Can a mutating webhook reject a request? | Technically yes, but by convention no — that's the validator's job |
| What format does a mutating webhook use to send changes? | JSON Patch (RFC 6902) |
| What is the `uid` field in AdmissionReview? | Unique ID to match request with its response |
| What happens when a webhook times out with `failurePolicy: Ignore`? | Request is allowed through |
| What happens when a webhook times out with `failurePolicy: Fail`? | Request is rejected |
| Which built-in controller adds default resource limits? | LimitRanger |
| Which built-in controller enforces namespace resource budgets? | ResourceQuota |
| Which built-in controller prevents objects in deleting namespaces? | NamespaceLifecycle |
| What object registers a mutating webhook? | MutatingWebhookConfiguration |
| What object registers a validating webhook? | ValidatingWebhookConfiguration |
| What does `sideEffects: None` mean? | Webhook has no external side effects — needed for dry-run |
| What does `namespaceSelector` do in a webhook? | Limits which namespaces trigger the webhook |
| Which dry-run mode tests webhooks? | `--dry-run=server` |
| What port does the API Server use to call webhooks? | HTTPS (443 typically, configurable) |
| What is the `caBundle` field? | Base64-encoded CA cert to verify the webhook server's TLS cert |
| What is OPA Gatekeeper? | Policy engine that runs as a validating webhook using Rego policies |
| What is Kyverno? | Kubernetes-native policy engine that runs as validating/mutating webhook using YAML policies |

---

## SECTION F — THINGS THAT SOUND IMPRESSIVE IN AN INTERVIEW

Use these naturally — understand the idea first:

1. **"The admission controller pipeline is why Kubernetes is called a platform — not just an orchestrator. Anyone can deploy containers. A platform has guardrails, defaults, and policies that make deployments safe and consistent automatically. Admission controllers are those guardrails."**

2. **"In our banking environment, we had a rule: no pod could run without CPU and memory limits. Instead of training every developer to always write resource limits, we used LimitRanger to inject defaults and OPA Gatekeeper to reject anything that somehow slipped through without them. Policy enforcement that developers don't have to think about is the best kind."**

3. **"One thing people miss about failurePolicy: Fail webhooks is the bootstrapping problem. If your webhook covers pods and the webhook itself crashes, it can't restart because its own pod creation is blocked by the policy it enforces. The fix: use namespaceSelector to exclude the webhook's own namespace from its rules. The webhook is always exempt from itself."**

4. **"We treat webhook policies as code — all ConstraintTemplates and Constraints are in Git, applied via ArgoCD. This means policy changes go through code review, have an audit trail, and can be rolled back if a new policy breaks deployments. Policy-as-code is as important as infrastructure-as-code."**

5. **"The `--dry-run=server` flag is one of the most underused features. In our CI/CD pipeline, every PR runs a server-side dry run against the staging cluster before merging. We catch OPA violations, quota exhaustion, and webhook rejections at code review time — not at 2 AM during a production deployment."**

---

*File: K8s_AdmissionControllers_Interview_Questions.md*
*Repository: Interview_Preparation_2026 → Kubernetes/*
*Companion file: K8s_AdmissionControllers_Concept_and_Lab.md*
