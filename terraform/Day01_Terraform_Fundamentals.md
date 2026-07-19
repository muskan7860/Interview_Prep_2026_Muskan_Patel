# Terraform Fundamentals — Complete Basics (Zero Prior Knowledge)

This file is deliberately **not tied to any specific script**. It covers Terraform as a subject on its own, the way you'd need to explain it to an interviewer starting from "what is it" all the way to "what happens on my screen when I press enter." Read this before looking at any actual `.tf` code — once these fundamentals are solid, any script becomes easy to read.

---

## 1. What problem does Terraform solve?

Before Terraform, if you needed a server, you'd log into the AWS Console (a website), click "Launch Instance," fill in a form — pick the OS, pick the size, pick the network — and click a button. Everything happens through clicking.

This has real problems once you're not doing it just once:

- **Not repeatable** — doing the exact same 20 clicks correctly, twice, in two different environments (dev and prod), is harder than it sounds. People forget a checkbox.
- **Not documented** — six months later, nobody remembers exactly what settings were chosen when that server was created.
- **Not reviewable** — you can't "review" someone's mouse clicks the way you review code before merging it.
- **Not version-controlled** — if something breaks, you can't "go back" to what it looked like last week, the way `git revert` lets you undo a code change.
- **Slow at scale** — clicking to create 1 server is fine. Clicking to create 50 servers, 10 databases, and a full network is a full day of manual, error-prone work.

**Infrastructure as Code (IaC)** is the general idea of solving all of this by describing your infrastructure as text files instead of manual actions. Terraform is the most widely used tool for doing this.

**One-line definition to say in an interview**: *"Terraform is an open-source Infrastructure as Code tool by HashiCorp that lets you define cloud and on-prem infrastructure in declarative configuration files, and it handles creating, updating, and destroying that infrastructure to match what you've defined."*

---

## 2. Declarative vs Imperative — the single most important concept

This distinction comes up constantly in interviews, so get it rock solid.

- **Imperative** = you specify the exact steps, in order. "First create a VPC. Then create a subnet inside it. Then create a security group. Then launch the instance using that security group." A shell script using AWS CLI commands is imperative — you're telling the computer *how* to do it, step by step.
- **Declarative** = you specify the end result you want, and the tool figures out the steps. "I want one VPC, one subnet inside it, one security group, and one EC2 instance using that security group." You never say "first do X, then do Y" — Terraform reads all of this and works out the correct order and actions on its own.

Terraform is **declarative**. This is exactly why you can run `terraform apply` five times in a row with no changes to your code, and nothing happens after the first time — because the current reality already matches what you declared. An imperative script, run five times, would try to create five VPCs unless you specifically coded logic to check "does this already exist" every single time.

---

## 3. Why Terraform specifically? (vs alternatives)

An interviewer may ask this to see if you understand the landscape, not just one tool.

| Tool | Category | Key difference |
|---|---|---|
| **AWS CloudFormation** | IaC | Native to AWS only, written in JSON/YAML. Terraform works across AWS, Azure, GCP, and hundreds of other providers with one consistent language. |
| **Ansible** | Configuration management | Designed to configure software *inside* servers that already exist (install packages, manage config files), not primarily to provision the servers themselves. Often used *together* with Terraform — Terraform builds the server, Ansible configures it. |
| **Pulumi** | IaC | Similar goal to Terraform, but you write actual programming languages (Python, TypeScript) instead of a dedicated configuration language like HCL. |
| **Manual console / custom scripts** | N/A | No state tracking, no plan-before-apply safety, no built-in dependency resolution. |

**Terraform's main strengths to mention**: cloud-agnostic (one tool for many providers), a huge ecosystem of providers and modules, a mature state management system, and the safety of always previewing a `plan` before anything actually changes.

---

## 4. Terraform's architecture — the four pieces that work together

Picture this as four separate components handing work off to each other:

```
Your .tf files  →  Terraform Core  →  Provider Plugin  →  Cloud API (AWS/Azure/GCP)
                          ↕
                     State File
```

1. **Configuration files (`.tf`)** — what *you* write. Plain text, in HCL, describing desired infrastructure.
2. **Terraform Core** — the actual `terraform` binary/engine. It parses your files, builds a dependency graph, decides what needs to happen, and orchestrates everything. It has zero built-in knowledge of AWS or any cloud by itself.
3. **Provider plugins** — separate downloadable binaries (one per cloud/platform: `aws`, `azurerm`, `google`, etc.) that Terraform Core talks to. Each provider knows how to translate a generic Terraform action ("create this resource") into the actual API calls that specific platform understands. This plugin architecture is *why* Terraform can support hundreds of platforms with one consistent workflow.
4. **State file (`terraform.tfstate`)** — a JSON file that is Terraform's record of what it has already created, mapping each resource in your code to a real object in the cloud along with its actual attributes (like its real ID).

**Simple analogy**: You (Terraform Core) don't personally speak every language in the world. When you need to talk to an AWS official, you bring a translator (AWS provider) who speaks fluent "AWS API." When you need to talk to an Azure official, you bring a different translator (azurerm provider). You, the coordinator, just decide *what* needs to be said and *in what order* — the translator handles the actual conversation. And you keep a personal notebook (state file) recording every conversation you've already had, so you don't repeat yourself.

---

## 5. What's inside a Terraform configuration — every block type explained

A `.tf` file is built from a small number of "block" types. Each one plays a distinct role:

### `terraform { }` block
Meta-configuration for Terraform itself — which provider versions to use, and which backend (where state is stored) to use.
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

### `provider { }` block
Configures a specific provider — commonly the region and credentials context.
```hcl
provider "aws" {
  region = "ap-south-1"
}
```

### `resource { }` block
The core building block — declares one real object you want Terraform to create and manage.
```hcl
resource "<resource_type>" "<local_name>" {
  argument1 = value1
  argument2 = value2
}
```
The **resource type** (e.g., `aws_instance`) tells Terraform + the provider exactly which kind of object and which API to use. The **local name** (e.g., `web_server`) is a nickname you invent, used only inside your own `.tf` files to refer back to this resource — it's never sent to the cloud provider.

### `variable { }` block
Declares an input — a placeholder value you fill in later, instead of hardcoding.
```hcl
variable "instance_type" {
  type        = string
  default     = "t3.micro"
  description = "EC2 instance size"
}
```
Used elsewhere as `var.instance_type`.

### `output { }` block
Exposes a value after resources are created — useful for printing something like a public IP, or for passing a value out of a module.
```hcl
output "instance_public_ip" {
  value = aws_instance.web_server.public_ip
}
```

### `data { }` block
Reads information about something that **already exists** and is not managed by this Terraform configuration — read-only, no create/update/destroy lifecycle.
```hcl
data "aws_ami" "latest_amazon_linux" {
  most_recent = true
  owners      = ["amazon"]
}
```

### `locals { }` block
Defines named values computed once and reused multiple times within the configuration, to avoid repeating an expression.
```hcl
locals {
  common_tags = {
    Project = "learning-terraform"
    Owner   = "muskan"
  }
}
```

### `module { }` block
Calls a reusable, packaged group of resources — like calling a function.
```hcl
module "network" {
  source = "./modules/vpc"
  cidr   = "10.0.0.0/16"
}
```

---

## 6. HCL — the language itself

HCL stands for **HashiCorp Configuration Language**. A few things worth knowing about it specifically:

- It's declarative and block-structured — not a general-purpose programming language, though it does support expressions, conditionals, and loops (`for`, `for_each`, `count`, `if`).
- **Interpolation** is how one resource references another's value — written as `resource_type.local_name.attribute`, e.g., `aws_vpc.main.id`. This is exactly how Terraform figures out dependencies between resources without you explicitly stating an order.
- It supports basic data types: string, number, bool, list, map, and object — same as most languages, just expressed in HCL's own syntax.
- Terraform can also read `.tf.json` files (pure JSON) instead of HCL, though almost nobody does this by hand — it exists mainly for machine-generated configuration.

---

## 7. The full workflow — every command, and what happens internally

This is the part interviewers dig into most, because it separates "I copy-pasted commands" from "I understand the tool."

### Step 1 — `terraform init`

**What you type**: `terraform init`

**What happens internally**:
1. Terraform reads your `terraform { required_providers {} }` block to see which providers and versions you need.
2. It reaches out to the Terraform Registry (or a private registry if configured) and downloads the matching provider plugin binaries into a hidden `.terraform/` folder in your working directory.
3. It reads your backend configuration (if any) and initializes the connection to wherever state will be stored — locally by default, or remotely if you've configured S3, Terraform Cloud, etc.
4. It writes a `.terraform.lock.hcl` file — this pins the *exact* provider versions used, so that everyone on the team, and every CI pipeline, uses identical provider versions instead of silently drifting to newer ones.

You run this once per project folder, and again any time you add a new provider, change the backend, or upgrade a provider version constraint.

### Step 2 — `terraform validate` (optional)

Purely local syntax and internal-consistency checking — no network calls, no talking to any cloud. Catches things like a missing closing brace, a wrong argument name, or a type mismatch, before you waste time on a `plan`.

### Step 3 — `terraform plan`

**What happens internally**:
1. Terraform parses all `.tf` files into an internal representation of your **desired state**.
2. It reads the current `terraform.tfstate` — its record of the **last known state**.
3. For every resource already in state, it makes read-only API calls through the provider to check the **real, current state** in the actual cloud account — this catches drift (manual changes made outside Terraform).
4. It computes the difference between desired state and real/last-known state, and prints exactly what it would do:
   - `+` = create
   - `~` = update in place
   - `-` = destroy
   - `-/+` = destroy and recreate (for changes that can't be done in-place)
5. **Nothing is actually changed on the cloud side during `plan`.** It's entirely a preview.

### Step 4 — `terraform apply`

**What happens internally**:
1. Runs the same comparison as `plan` (or accepts a previously saved plan file via `terraform apply <planfile>`).
2. Prompts for confirmation (`yes`) unless run with `-auto-approve`.
3. Builds a **dependency graph** (see section 8) from all resource references.
4. Walks that graph, calling the appropriate provider function for each resource — this is where the provider plugin translates each block into a real API call, like `RunInstances` for an EC2 instance or `CreateSecurityGroup` for a security group.
5. Resources with no dependency relationship between each other are processed **in parallel** by Terraform's internal engine, which speeds up applies significantly for larger configurations.
6. As each resource is successfully created/updated, its real attributes (like a generated ID) are written into `terraform.tfstate` immediately — not just at the very end. This is important: if something later in the run fails, everything that already succeeded stays recorded correctly, it isn't rolled back or lost.
7. Once the whole graph completes, any `output` values are printed to the terminal.

### Step 5 — `terraform destroy`

Same graph, walked in **reverse dependency order** — resources that depend on others are destroyed first, and the resources they depended on are destroyed last (since AWS won't let you delete a VPC while something still lives inside it, for example). Reads state to know exactly what it's responsible for tearing down.

### Other commonly used commands

- **`terraform fmt`** — auto-formats `.tf` files to Terraform's canonical style (indentation, alignment). Purely cosmetic, but expected in any team codebase.
- **`terraform show`** — prints the current state file in a human-readable format.
- **`terraform state list`** — lists every resource address Terraform currently tracks.
- **`terraform state mv`** — renames or moves a resource's address in state without destroying/recreating the real object.
- **`terraform refresh`** *(now built into `plan`/`apply` by default in modern versions)* — reconciles state with real infrastructure without changing configuration.
- **`terraform output`** — prints output values after apply, without re-running the whole plan.

---

## 8. The dependency graph — how Terraform decides *order*, deeply

This is the part that actually explains "how it works in the backend," which is usually the vaguest part of people's understanding.

1. Terraform scans every resource block for references to other resources' attributes. Any time resource B's argument reads a value from resource A (like `subnet_id = aws_subnet.main.id`), that's an **implicit dependency** — Terraform now knows "B needs A to exist first."
2. All of these relationships across the whole configuration get assembled into a **Directed Acyclic Graph (DAG)** — "directed" meaning each edge has a clear "before/after" direction, "acyclic" meaning no circular loops are allowed (A can't need B if B also needs A — that's a **cyclic dependency error**).
3. Terraform performs a **topological sort** on this graph — producing a valid execution order where every resource appears only after everything it depends on.
4. During `apply`, independent branches of this graph (resources with no relationship to each other) are executed **concurrently**, using Terraform's internal worker pool — not one at a time, and not in the order they're written in the file. Only resources that actually depend on each other are forced to wait in sequence.
5. **`depends_on`** is the manual escape hatch — used only when a real dependency exists that Terraform *cannot* see through attribute references (a common example: an IAM role needing time for its permissions to propagate across AWS, where the dependent resource doesn't reference any actual attribute of that IAM role).

**Why this matters practically**: this is exactly why you never need to "order" your resource blocks in a `.tf` file. You can write your EC2 instance block above your VPC block in the file, and Terraform will still create the VPC first, because it's reading *relationships*, not *file order*.

---

## 9. The state file, in depth

- It's JSON, and it records, for every managed resource: its resource address, every attribute AWS (or another provider) returned about it (including sensitive-looking values sometimes, like generated IDs), and metadata like dependencies.
- It is the **only** way Terraform knows what it's responsible for. If you manually delete a resource from state (`terraform state rm`) without destroying it in the cloud, Terraform "forgets" about it — the real object keeps existing, but Terraform will no longer track or manage it.
- **State locking**: when using a remote backend that supports locking (S3 + DynamoDB is the classic combination), Terraform acquires a lock before any write operation (`plan` that refreshes, or `apply`), so two people running Terraform against the same state at the same time can't corrupt it — the second person's run waits or fails with a lock error.
- **Sensitive data**: state files can contain plaintext sensitive values (like a database password set via a resource argument), which is exactly why state should never be committed to Git, and should ideally be stored encrypted in a remote backend with restricted access.

---

## 10. Providers, in depth

- A provider is a separate, versioned plugin binary — Terraform Core and the AWS provider are updated and released independently of each other.
- One Terraform configuration can use **multiple providers** at once — for example, `aws` for cloud resources and `cloudflare` for DNS, in the same apply.
- You can also configure **multiple instances of the same provider** using **provider aliases** — commonly used to manage resources across multiple AWS regions or multiple AWS accounts from one configuration.
- Providers expose both `resource` types (manageable) and `data` types (read-only lookups) — e.g., the AWS provider offers `aws_instance` (resource) and `data "aws_ami"` (data source).

---

## 11. A minimal, generic example (not tied to any real project)

To see all of this working together in the smallest possible form:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-south-1"
}

variable "bucket_name" {
  type = string
}

resource "aws_s3_bucket" "example" {
  bucket = var.bucket_name
}

output "bucket_arn" {
  value = aws_s3_bucket.example.arn
}
```

Walking through this with everything covered above: `terraform init` downloads the `aws` provider. `terraform plan` shows one S3 bucket will be created, using whatever value you pass for `bucket_name`. `terraform apply` calls the AWS provider, which calls AWS's `CreateBucket` API, gets back a real ARN, writes it into state, and prints it because of the `output` block. `terraform destroy` calls `DeleteBucket` using the ID recorded in state. There is exactly one resource here, so there's no dependency graph complexity to speak of — but the moment you add a second resource that references `aws_s3_bucket.example.id`, the graph concept kicks in.

---

## 12. How to summarize all of this in one interview answer

If asked cold, "explain Terraform end to end," a solid structured answer sounds like:

*"Terraform is a declarative Infrastructure as Code tool. You describe desired infrastructure in HCL configuration files using resource blocks, and Terraform's core engine reads those files, builds a dependency graph based on references between resources, and figures out the correct order to create things — running independent resources in parallel. It talks to actual cloud platforms through provider plugins, which translate generic actions into real API calls. Every change goes through `init` to set up providers, `plan` to preview exactly what will change, and `apply` to execute it — and every resource it creates gets recorded in a state file, which is how Terraform tracks what already exists so it never tries to recreate the same thing twice. For teams, that state is stored remotely, usually in S3 with DynamoDB for locking, so multiple people can safely collaborate without corrupting each other's changes."*

That one paragraph, said confidently, covers: definition, language, engine, dependency graph, providers, full workflow, state, and team collaboration — the entire subject in one breath.
