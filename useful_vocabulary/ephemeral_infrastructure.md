Ephemeral infrastructure means:

Servers and resources are temporary, disposable, and expected to be created, destroyed, and recreated frequently.

They are treated like cattle, not pets.

Intuition first
Traditional (“pets”)

Named servers (db-prod-1)

Long-lived

Manually fixed

You care which server it is

Ephemeral (“cattle”)

Short-lived

Auto-created and destroyed

Replaced, not repaired

You care only that some instance exists

What makes infrastructure “ephemeral”?

A resource is ephemeral if its identity does not matter and it is expected to disappear.

Typical characteristics:

Lifespan: minutes → days

Created by automation

No manual changes

Rebuilt from code

State lives outside the instance

Common examples
1. Kubernetes Pods

Pods die and respawn constantly

New IP, new node, new filesystem

Same role, different instance

2. Auto-scaling VM instances

Instances added/removed based on load

Same image, same config

Identity is irrelevant

3. CI/CD runners

Created per pipeline

Destroyed after job

No persistence expected

4. Preview / test environments

Created per PR

Deleted after merge

Fully reproducible

What ephemeral infrastructure is not

❌ Long-lived database with manual tweaks
❌ Snowflake servers (“works only on this machine”)
❌ SSHing in to debug and fixing things permanently

You may SSH for debugging — but you never rely on it.

Why ephemeral infrastructure exists
1️⃣ Reliability

If something breaks:

destroy → recreate


Faster and safer than debugging in place.

2️⃣ Security

Compromised instances don’t live long

Secrets rotate naturally

Fewer long-term footholds

3️⃣ Scalability

Horizontal scaling assumes disposability

Load increases → add instances

Load drops → remove instances

4️⃣ Reproducibility

If you can’t recreate it:

it doesn’t exist

Infrastructure is defined as code:

Terraform

Cloud-init

Ansible

Helm

Ephemeral infrastructure and SSH (important link)

This connects directly to your earlier SSH questions.

Problem

Every rebuild generates a new SSH host key

Same IP / hostname, different key

SSH screams:

REMOTE HOST IDENTIFICATION HAS CHANGED!

This is normal in ephemeral setups.
How ephemeral infra handles identity

Common solutions:

Pre-populate known_hosts

Disable strict checking (last resort)

Use SSH host certificates

Use bastion hosts

Prefer short-lived credentials

The instance identity is temporary — trust is shifted elsewhere.

Ephemeral vs immutable (related but distinct)
Concept	Focus
Ephemeral	Short-lived resources
Immutable	Never modified after creation

Most modern systems are both:

Immutable images

Ephemeral instances

Mental model (one sentence)

Ephemeral infrastructure assumes failure and replacement as the normal operating mode.

Summary
Aspect	Ephemeral infrastructure
Lifespan	Short
Replacement	Expected
Manual fixes	Discouraged
State	External
Identity	Disposable
Automation	Required

If you want, next we can:

Tie this directly to Kubernetes node identity

Show how SSH host certificates solve ephemerality

Explain why databases are usually not ephemeral

Design a clean trust model for ephemeral VMs

Just tell me where you want to go next.
