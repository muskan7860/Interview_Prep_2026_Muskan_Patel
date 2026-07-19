# Terraform — Interview Questions & Answers

Spoken-answer style: each answer is written the way you'd actually say it out loud, not a bullet-point definition. Basics first, then intermediate, then troubleshooting.

---

## Section A — Foundational Concepts

**Q1. What is Terraform, in your own words?**

Terraform is an Infrastructure as Code tool — it lets me define my cloud infrastructure, like EC2 instances, VPCs, security groups, in configuration files instead of clicking around manually in a console. I write what I want the end state to look like, and Terraform figures out how to create, update, or delete real cloud resources to match that. It's provider-agnostic, meaning the same tool works across AWS, Azure, GCP, and many other platforms, through plugins called providers.

**Q2. Why do we need Terraform when we can just use the AWS Console or write our own scripts?**

The Console works for one-off, manual changes, but it doesn't scale and it leaves no audit trail — you can't version control a series of clicks. Custom scripts using the AWS CLI or SDK are procedural — you have to write the logic for "check if it exists, then create, then handle updates, then handle deletes" yourself. Terraform is declarative — I just describe the desired end state, and Terraform's engine handles diffing against the current state and figuring out the right actions. On top of that, Terraform configuration lives in Git, so it's versioned, reviewable, and repeatable across environments.

**Q3. What is the state file, and why is it so important?**

The state file, `terraform.tfstate`, is how Terraform keeps track of what it has already created and maps each resource block in my code to a real object in the cloud, along with its actual ID and attributes. Without it, every `terraform apply` would have no memory of what already exists and could try to create duplicate resources. It's essentially Terraform's source of truth for "what's real right now," separate from my `.tf` files, which describe "what I want."

**Q4. What's the difference between local and remote state, and why would a team use remote state?**

Local state just sits as a file on whichever machine ran `apply` — that's risky for teams because two people could run Terraform at the same time and corrupt or overwrite each other's state, and if that laptop is lost, the state is lost. Remote state stores the file somewhere shared, commonly an S3 bucket, combined with a DynamoDB table used purely for state locking — so if I'm running `apply`, DynamoDB locks the state, and if a teammate tries to run `apply` at the same time, Terraform blocks them until my run finishes. That prevents concurrent modification conflicts.

**Q5. Explain the Terraform workflow: init, plan, apply, destroy.**

`terraform init` sets up the working directory — it downloads the provider plugins referenced in my config and initializes the backend for state. `terraform plan` is a dry run — it compares my desired configuration against the current state and the real infrastructure, and shows me exactly what will be created, changed, or destroyed, without touching anything. `terraform apply` actually executes that plan — it calls the real cloud provider APIs to create or modify resources, and updates the state file as it goes. `terraform destroy` walks the same dependency graph in reverse and tears down everything Terraform is tracking in state.

**Q6. What is a provider in Terraform?**

A provider is a plugin that acts as the translator between Terraform's core engine and a specific platform's API — for example, the AWS provider knows how to turn my `aws_instance` resource block into the actual `RunInstances` API call AWS expects. Terraform Core itself has no built-in knowledge of AWS, Azure, or any cloud — all of that logic lives inside the provider plugin, which gets downloaded during `terraform init`.

**Q7. What's the difference between a resource block and a data block?**

A `resource` block tells Terraform to create and manage something new — Terraform owns its full lifecycle: create, update, destroy. A `data` block is read-only — it looks up information about something that already exists, which Terraform did not create and will not manage or destroy. For example, I might use a `data "aws_ami"` block to look up the latest Amazon Linux AMI ID dynamically, instead of hardcoding an AMI ID that goes stale.

**Q8. What is the dependency graph, and how does Terraform decide execution order?**

Terraform doesn't execute resource blocks in the order they're written in the file. It scans every resource for references to other resources — anytime one resource's argument points to another resource's attribute, like a security group referencing a VPC's ID, that creates what's called an implicit dependency. Terraform builds all of these relationships into a directed acyclic graph, then does a topological sort on it, so that every resource is only created after everything it depends on. Resources with no dependency relationship between them get created in parallel to speed things up.

**Q9. What's the difference between implicit and explicit dependencies?**

An implicit dependency happens automatically whenever one resource references an attribute of another, like `vpc_id = aws_vpc.main.id` — Terraform can see this in the code and builds the graph edge on its own. An explicit dependency is when I manually add `depends_on = [...]` because there's a dependency Terraform can't detect through attribute references — commonly used when an IAM role needs a few seconds for its permissions to propagate across AWS, and my resource doesn't directly reference any attribute of that role.

**Q10. Difference between `count` and `for_each`?**

Both let me create multiple copies of a resource from one block. `count` takes a number and creates resources indexed numerically, like `aws_instance.web[0]`, `aws_instance.web[1]`. The problem is if I remove an item from the middle of a list driving `count`, Terraform can shift all the indexes and end up destroying and recreating resources that didn't actually need to change. `for_each` takes a map or a set of strings and creates one resource per key, addressed by that key instead of a number, like `aws_instance.web["frontend"]`. That makes it much safer for anything where individual identity matters, because removing one entry doesn't disturb the others.

**Q11. What are input variables and output values used for?**

Input variables, declared with `variable` blocks, let me parametrize my configuration instead of hardcoding values — like instance type or AMI ID — so the same code can be reused across environments by just changing variable values, often through a `.tfvars` file. Output values, declared with `output` blocks, expose specific attributes after `apply` completes — like printing an instance's public IP — and they're also how a parent module reads values from a child module.

**Q12. What is a Terraform module?**

A module is just a reusable, self-contained group of resources — think of it like a function in programming. Instead of copy-pasting the same VPC-plus-subnet-plus-route-table code across five projects, I write it once as a module with input variables and outputs, then call that module from each project, passing in different values each time. The root directory I run `apply` from is technically the "root module," and any module I call from it is a "child module."

**Q13. What is HCL?**

HCL stands for HashiCorp Configuration Language — it's the declarative language Terraform configuration files are written in. It's designed to be both human-readable and machine-parseable, structured around blocks, arguments, and expressions, and it also supports interpolation, so I can reference one resource's attributes from within another resource's arguments.

**Q14. What's the difference between Terraform and configuration management tools like Ansible?**

Terraform is primarily for provisioning — creating and managing the infrastructure itself: the servers, networks, load balancers. Ansible is primarily for configuration management — installing software, managing files and services on servers that already exist. In practice they're often used together: Terraform stands up the EC2 instance, then Ansible connects to it afterward to configure Nginx, deploy the application, and so on. Terraform does have `user_data` and provisioners for basic bootstrapping, but that's not a replacement for a full configuration management tool.

**Q15. What is `terraform.tfvars` and how is it different from `variables.tf`?**

`variables.tf` is where I declare which variables exist, along with their types, descriptions, and optional default values — it's the definition. `terraform.tfvars` is where I actually assign values to those variables for a specific run or environment — it's the assignment. Terraform automatically loads a file literally named `terraform.tfvars` without me needing to pass any flag; for anything named differently, like `prod.tfvars`, I'd pass it explicitly with `-var-file`.

**Q16. What is workspace in Terraform?**

A workspace lets me maintain multiple, separate state files for the same configuration — commonly used to manage environments like dev, staging, and prod from the same codebase without duplicating files. Each workspace gets its own state, so resources created in the dev workspace are completely isolated from prod. That said, for anything beyond simple environment separation, most teams prefer separate directories or separate backend configurations per environment instead of workspaces, because workspaces can get confusing at scale and don't isolate variables by default, only state.

**Q17. What is `terraform import` used for?**

It's for bringing existing infrastructure — something created manually in the console, or by another tool — under Terraform's management, without destroying and recreating it. I write a resource block matching what I want to import, then run `terraform import <resource_address> <real_resource_id>`, and Terraform links that real resource into my state file. It does not generate the `.tf` configuration for me automatically in older versions — I still have to write the resource block myself and make sure it matches, or the next `plan` will show a big diff.

**Q18. What does `terraform taint` do, and is it still used?**

`terraform taint` used to mark a specific resource as needing to be destroyed and recreated on the next apply, without it actually being changed in the configuration — useful if a resource is in a broken state and needs a hard reset. In newer Terraform versions, `taint` is deprecated in favor of `terraform apply -replace="<resource_address>"`, which does the same thing but is more explicit and works as part of a normal plan/apply flow.

---

## Section B — Troubleshooting Questions

**Q19. Terraform apply fails halfway through — some resources got created, some didn't. What do you do?**

First, I don't panic and I don't manually delete things from the AWS console. Terraform's state file will accurately reflect exactly which resources succeeded and which didn't — it doesn't roll back what already succeeded. I run `terraform plan` again to see the current diff, fix whatever caused the failure — commonly a naming conflict, a quota limit, or a missing permission — and run `apply` again. Terraform will only create what's still missing, because everything that succeeded is already recorded in state and won't be recreated.

**Q20. You see "Error: state lock" when running apply. What's happening and how do you fix it?**

This means the state is locked, usually because a previous `apply` or `plan` didn't exit cleanly — maybe it was interrupted, or a CI pipeline crashed mid-run — and the lock, typically held in a DynamoDB table for an S3 backend, was never released. First, I check whether another apply is actually still running somewhere before doing anything — force-unlocking while a real run is in progress can corrupt state. If I've confirmed nothing else is running, I use `terraform force-unlock <lock-id>` with the lock ID from the error message to manually release it.

**Q21. Someone manually changed a resource in the AWS console after Terraform created it. What happens on the next apply?**

This is called drift — the real infrastructure no longer matches what's recorded in state. On the next `terraform plan`, Terraform refreshes its view of real infrastructure and will show a diff trying to bring that resource back in line with what's defined in the `.tf` files, since Terraform's `.tf` code is always treated as the source of truth, not the console. If the manual change was actually correct and I want to keep it, I need to update my `.tf` configuration to match, or the next apply will silently revert someone's manual fix.

**Q22. Terraform wants to destroy and recreate a resource I didn't expect. Why would that happen?**

Certain attributes are marked as "force new resource" in the provider — changing them can't be done in-place on the cloud side, so Terraform's only option is to destroy the old one and create a new one. A common example is changing the AMI ID on an EC2 instance, or renaming an S3 bucket. Before applying, I always read the plan output carefully — it explicitly marks these as `-/+` (destroy and recreate) versus `~` (in-place update), and for anything stateful like a database, an unexpected destroy-and-recreate is exactly the kind of thing you want to catch in `plan` before it happens in `apply`.

**Q23. You get a "cyclic dependency" error. What does that mean and how do you fix it?**

It means two or more resources reference each other in a way that creates a loop — resource A depends on resource B, and resource B also depends on resource A, so Terraform can't determine which one to create first. This usually happens by accident, often through indirect chains rather than something obvious. The fix is to break the cycle — usually by removing one of the references, restructuring so the dependency only flows one direction, or in some cases splitting a resource into two steps using something like `depends_on` more deliberately instead of a circular attribute reference.

**Q24. Your team's state file gets corrupted or accidentally deleted. What now?**

This is why remote state with versioning matters — if the backend is an S3 bucket with versioning enabled, I can restore a previous version of the state file from S3's version history. If there's genuinely no backup at all, the recovery path is painful: I'd have to reconstruct state using `terraform import` for every single existing resource, one at a time, matching each to a resource block in my configuration. This is exactly why production setups should always use a versioned remote backend, never local state.

**Q25. `terraform plan` shows changes even though you didn't touch the `.tf` files. Why?**

A few common causes: someone else on the team applied a change and I haven't pulled the latest state; something changed manually in the console, causing drift; a data source is returning a different value now than it did before, like an AMI lookup resolving to a newer AMI ID because a new version was published; or a provider version got upgraded and its default behavior for some attribute changed slightly between versions. The way to diagnose it is to actually read the plan output line by line — it tells you exactly which attribute is changing and from what value to what value, which almost always points straight at the cause.

**Q26. Two engineers run `apply` at the same time on the same infrastructure. What stops this from causing damage?**

State locking. With a proper remote backend like S3 plus DynamoDB, the moment the first `apply` starts, it acquires a lock in DynamoDB. The second engineer's `apply` will fail immediately with a "state locked" error instead of proceeding, because Terraform checks for the lock before making any changes. This is exactly why local state, which has no locking at all, is unsafe for team environments — two people applying at once against a local or unlocked state can genuinely corrupt it.

**Q27. An EC2 instance created by Terraform is unreachable over SSH. How do you debug it, from a Terraform/infrastructure angle?**

I'd check it in layers, from the network inward. First, does the security group actually allow inbound port 22 from my IP — not from `0.0.0.0/0` if it was intentionally restricted. Second, is the instance actually in a subnet with a route to an Internet Gateway, and does it have a public IP assigned. Third, was a key pair actually attached via `key_name`, and do I have the matching private key. Fourth, I'd check whether the instance even finished booting — via the AWS console's instance status checks, or by checking `user_data`/cloud-init logs through Session Manager if SSH itself is the problem. It's rarely a Terraform bug at this point — it's almost always one of these four things missing or misconfigured in the resource definition itself.

**Q28. What's the difference between "refresh-only" apply and a normal apply?**

`terraform apply -refresh-only` updates the state file to match real-world infrastructure without changing any actual resources and without trying to reconcile against the `.tf` configuration — it's purely syncing Terraform's memory to reality. This is useful specifically when you know drift happened, you've decided the manual change should be accepted as the new truth, and you want state updated to reflect that before your next real apply, instead of Terraform trying to revert it.

**Q29. Your provider version got upgraded and now `terraform plan` shows unexpected changes across many resources. How do you handle it?**

I'd check the provider's changelog for that version bump — provider maintainers do occasionally change default values or attribute behavior between versions, and that shows up as unexpected diffs across resources that use that attribute. The safe fix is to pin the provider version explicitly in the `terraform` block's `required_providers` section, using a lock file, rather than always pulling the latest. If the new version's diffs are actually fine, I still review each one instead of blindly applying, because a version bump touching every resource at once is exactly the kind of change that should be tested in a non-production environment first.

**Q30. How would you handle a situation where you need to rename a resource in code without destroying and recreating it in AWS?**

Simply renaming the resource block, like changing `resource "aws_instance" "my_instance"` to `resource "aws_instance" "web_server"`, would make Terraform think the old one should be destroyed and a completely new one created, because from Terraform's perspective the address changed. The correct way is `terraform state mv <old_address> <new_address>`, which tells Terraform's state file directly "this real resource is now referred to by this new address," without touching the actual AWS resource at all. I'd then update the `.tf` file to match that new name so the next plan shows no changes.

---

## Section C — Quick-Fire Conceptual Checks

**Q31. Is Terraform declarative or imperative?**

Declarative — I describe what I want the end result to be, not the sequence of steps to get there. Terraform's engine figures out the steps.

**Q32. Can Terraform manage infrastructure it didn't create?**

Yes, through `terraform import`, or by using `data` sources to reference it read-only without managing its lifecycle at all.

**Q33. What happens if you run `apply` with no changes to make?**

Terraform reports "No changes. Infrastructure is up-to-date." and exits — nothing is touched, since the desired state already matches state and real infrastructure.

**Q34. Is the state file safe to store in Git?**

No — it can contain sensitive data in plaintext, like database passwords or private keys, depending on what resources are managed. It should live in a remote backend, ideally encrypted, never committed to version control.
