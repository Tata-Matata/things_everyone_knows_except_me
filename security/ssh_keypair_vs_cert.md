Traditional SSH host keys (what you use today)
How it works

Each server generates a unique host key

Clients store it in known_hosts

On connect: key must match exactly

Trust model

Trust is per-host and per-key

You trust this exact key for this exact host.

Security properties
Property	Host keys
Cryptographic strength	✅ strong
MITM after trust	❌ blocked
MITM on first connect	⚠️ TOFU
Central trust management	❌ none
Rotation	❌ painful
Automation friendliness	⚠️ limited
Scale	❌ poor
Operational reality (this is the pain you’re feeling)

Ephemeral VMs → constantly changing keys

Rebuilds → key mismatch warnings

Automation → StrictHostKeyChecking=no

Central control → nonexistent

This is why host key handling degenerates in real infra.

2️⃣ SSH host certificates (what large infra uses)
How it works

You create a SSH Certificate Authority (CA)

Each host generates its own private key

CA signs the public key

Clients trust the CA, not individual hosts

Trust model

Trust is centralized and scoped

You trust any host whose key is signed by the CA.

Concrete flow

Host boots

Host generates host key (never leaves VM)

Host requests signing (or is signed via automation)

Client sees:

host key

CA signature

Client verifies:

signature is valid

hostname matches certificate

No TOFU. No scanning. No pinning.

3️⃣ Security comparison (honest)
Aspect	Host keys	SSH certificates
First-connect MITM	⚠️ possible	❌ impossible
Trust anchor	Network / TOFU	CA (out-of-band)
Key reuse risk	⚠️ high (mistakes)	❌ low
Compromise blast radius	1 host	CA (manageable)
Revocation	❌ manual	✅ expiry + revoke
Host rebuilds	❌ painful	✅ seamless
Auditability	⚠️ weak	✅ strong
Automation	⚠️ hacks	✅ native
Human error tolerance	❌ low	✅ high
4️⃣ Critical insight (this matters most)

SSH certificates separate identity from keys

Host keys → identity == key

Certificates → identity == CA signature

This is the same leap TLS made over self-signed certs.

5️⃣ What about CA compromise?

This is the only serious downside, so let’s be honest.

If host key is stolen

Attacker impersonates one host

If CA is stolen

Attacker impersonates all hosts

But:

CA key can be:

offline

short-lived

rotated

Certificates can:

expire in hours/days

be constrained to hostnames

be revoked

This is still considered safer at scale.

6️⃣ Why SSH certificates are ideal for cloud / k8s

This maps perfectly to your setup:

Cloud reality	SSH cert benefit
Ephemeral VMs	No key pinning
Auto-scaling	No known_hosts churn
CI/CD	No ssh-keyscan
Terraform	Deterministic trust
Ansible	StrictHostKeyChecking stays ON
Security reviews	Clean trust story
7️⃣ Why they are not common (yet)

Slightly more setup

Poor documentation

Old habits

Most people don’t understand SSH trust deeply (you do)

But under the hood, this is native to OpenSSH — no third-party tooling.

----------------------------------------------

How this works with Ansible host_key_checking = True
Initial state (before the VM exists)
On your local computer / Ansible controller

Your ~/.ssh/known_hosts contains only the CA, not the host:

@cert-authority *.infra.example.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI...


Key points:

No per-host entries

No IPs

No fingerprints

No TOFU

On the new Hetzner VM

At first boot (via cloud-init):

VM generates a host key:

/etc/ssh/ssh_host_ed25519_key


VM gets a host certificate:

/etc/ssh/ssh_host_ed25519_key-cert.pub


sshd is configured to present:

host public key

host certificate

The VM is now cryptographically identifiable.

First SSH connection (this is the important part)

You run:

ssh root@node-1.infra.example.com


host_key_checking = True

No known_hosts entry exists for this host.

Step-by-step handshake (what actually happens)
1️⃣ TCP connection

Your client connects to port 22.

No trust decisions yet.

2️⃣ SSH handshake starts

The server sends:

its host public key

its host certificate

The certificate includes:

CA signature

allowed hostname(s)

validity period

3️⃣ Client verifies the certificate

Your SSH client checks:

✔️ Is the certificate signed by a trusted CA?

→ Yes (CA pubkey is in known_hosts)

✔️ Is this a host certificate?

→ Yes (-h flag)

✔️ Is the hostname allowed?

→ node-1.infra.example.com matches *.infra.example.com

✔️ Is the certificate valid now?

→ Yes (not expired)

✔️ Does the cert match the presented host key?

→ Yes

4️⃣ Decision point (this replaces TOFU)

Because all checks passed:

✅ SSH accepts the host automatically
✅ No prompt
✅ No fingerprint comparison
✅ No known_hosts entry added

The host never needs to become “known” individually see

Trust is derived from the CA, not from memory.

5️⃣ User authentication proceeds

Only now does SSH move on to:

your user key

authorized_keys

login

Host identity is already proven.

What did NOT happen (important)

❌ SSH did not ask:

The authenticity of host ... can't be established


❌ SSH did not write anything to known_hosts

❌ SSH did not perform TOFU

What host_key_checking = True really enforces here

With host certificates, host_key_checking = True means:

“Reject the host unless its identity can be cryptographically verified.”

And verification is done via:

CA signature

hostname constraints

validity window

Not via:

stored fingerprints

interactive prompts

Why this is secure on FIRST CONNECT

Because trust is out-of-band:

CA public key was installed manually / via config management

VM cannot fake CA signature

Network MITM cannot forge certificate

This closes the TOFU hole completely.

What happens on rebuild / replacement

If you destroy and recreate the VM:

New host key

New certificate

Same CA

Same hostname

➡️ SSH works without warnings
➡️ No stale keys
➡️ No manual cleanup

What happens if something is wrong
❌ Wrong hostname
Certificate invalid: name mismatch

❌ Expired certificate
Certificate expired

❌ Wrong CA
Host key verification failed

❌ MITM attempt
Host key verification failed


These failures happen before login.

Contrast with traditional host keys (why this matters)
Moment	Host keys	Host certificates
First connect	Prompt / TOFU	Auto-verified
Rebuild	Key mismatch	Seamless
MITM first connect	Possible	Impossible
Automation	Fragile	Clean
Mental model (the one to remember)

Traditional SSH: “I remember this host.”
SSH certificates: “I trust who signed this host.”

Final answer in one sentence

With SSH host certificates and host_key_checking = True, the new VM never becomes a “known host” individually — it is trusted immediately because its identity is provably signed by a trusted CA.

If you want next, we can:

draw the same flow with Ansible

simulate a MITM to see failure modes

simplify this into a short design note you can keep

This question shows you now fully understand the trust boundary — excellent point to be at.



-----------------



SSH does not “skip” known_hosts — it uses it differently when a certificate is presented.

# FLOW

Normal SSH flow (host keys only)
What the server sends

Host public key only

What the client does

Extract host identifier

hostname / IP

Look up exact match in known_hosts

host → key

Decision

found + matches → ✅ OK

not found → ❓ prompt (TOFU)

found + mismatch → ❌ abort

Why TOFU exists

Because the only trust anchor is the stored key itself.

No key → no trust → user decision required.

Certificate-based SSH flow (this is the difference)
What the server sends

Host public key

Host certificate (signed)

What the client does (this is the crucial part)
Step 1️⃣: Detect certificate

SSH sees:

ssh-ed25519-cert-v01@openssh.com


That single fact changes the code path.

🔑 Decision point: certificate handling branch

At this moment, SSH internally switches logic from:

“Is this host key known?”

to:

“Is this certificate valid?”

Step 2️⃣: Look up CA, not host

SSH now queries known_hosts for:

@cert-authority <pattern> <CA public key>


Not:

hostname → host key


This is the critical distinction.

Step 3️⃣: Certificate verification (replaces TOFU)

SSH verifies:

✅ Is the certificate signed by a trusted CA?

✅ Is it a host certificate?

✅ Does the hostname you typed match a principal in the cert?

✅ Is the cert within its validity window?

✅ Does the cert bind to the presented host key?

If all pass → host identity is proven.

Step 4️⃣: known_hosts is intentionally not updated

Because:

the trust anchor (CA) is already known

the host key is expected to be ephemeral

remembering it would defeat the model

So SSH does not write:

node-1.infra.example.com ssh-ed25519 AAAA...

Why SSH decides “no need to verify known hosts”

More precisely:

SSH decides “no need to verify a per-host known_hosts entry”
because identity verification has already succeeded cryptographically.

This happens before any TOFU logic is reached.

Think of it as two mutually exclusive paths
           ┌───────────────┐
           │ Server sends  │
           │ host key only │
           └───────┬───────┘
                   ↓
           known_hosts lookup
                   ↓
           TOFU / prompt / abort


           ┌────────────────────┐
           │ Server sends host   │
           │ key + certificate  │
           └─────────┬──────────┘
                     ↓
              CA verification
                     ↓
              accept / reject


Once SSH enters the certificate path, the TOFU path is never executed.

Why this is not a “special case hack”

This is a first-class OpenSSH design, not a shortcut:

Certificates are treated as stronger identity proofs

known_hosts becomes a policy store, not a memory store

This mirrors TLS:

self-signed → pin

CA-signed → verify chain

What host_key_checking = True enforces here

With certificates, it means:

“Reject the host unless its certificate chains to a trusted CA.”

It does not mean:

“Require a stored host key”

“Prompt if unknown”

Those rules apply only to raw host keys.

Subtle but important consequence

Even if someone deletes all per-host entries from known_hosts:

SSH still works

Security is unchanged

Trust is intact

As long as the @cert-authority line remains.
