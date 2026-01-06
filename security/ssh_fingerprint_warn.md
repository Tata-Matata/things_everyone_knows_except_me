You ssh a server and see 

## The authenticity of host '1.2.3.4' can't be established.
#### ED25519 key fingerprint is SHA256:…


## Meaning

Your SSH client has never seen this server before, so it cannot yet verify its identity. So it prompts user to answer the question **Are you sure you are really connecting to the server you intended to connect to?**

**Fingerprint** is a hash of the server’s **public host key**. Every SSH server has a host key pair (public + private)

### ED25519
What kind of key is this? Public key algorithm to create the SSH host key

### SHA256
How is the fingerprint computed? Cryptographic hash function to create a short, fixed-length identifier of the public key

### Why do we need hash here?
Raw public key is too long (32 Bytes) when displayed to humans, manual comparison is error-prone, if not impossible

## If you pick Yes
SSH stores the server’s public key in:
``
~/.ssh/known_hosts
``

#### Next time you login:

- SSH checks the fingerprint
- If it matches → connection allowed silently
- If it differs → possible attack 

# What security problem does this address?
## Man in the middle attack
You are on public WiFi or corporate network. You ssh to a server. Attacker controls the network path. 
- Attacker intercepts traffic.
- Pretends to be the server by presenting their own ssh host key
- Relays traffic to the real server

#### If password auth
Attacker captures your password

#### If SSH key auth
Attacker can log and/or inject commands, capture agent forwarded sessions

# Similar concepts in other protocols

This exact interaction is SSH-specific, but the threat model is common for other protocols. The purpose is to ensure the server identity.

### HTTPS / TLS

Servers present certificates. Browsers verify certificate chain based on trusted CA.

##### Different trust anchors: 

- HTTPS:  CA decides trust ("I signed this certificate so you can trust it"). Binds domain name to certificate verified by CA signature

- SSH: user decides trust (“Do I want to forever bind this IP/hostname to this cryptographic identity?”). Binds hostname or IP to public key.

##### Different locations to store identity:
- SSH: ~/.ssh/known_hosts
- HTTPS: Browser / OS trust store

### Git over SSH

Same SSH host key mechanism: The fingerprint identifies GitHub’s SSH host key, so on first connect we see "The authenticity of host 'github.com' can't be established". 
In the next step, SSH also verifies you as a client - that you possess the private key that matches the public key stored on the account
After that, Git protocol runs inside encrypted SSH tunnel

##### Git hosting providers publish fingerprints
(GitHub, GitLab)
So you can verify out of band:
```
ssh-keyscan github.com
ssh -T git@github.com
```

### Git over HTTPS 
uses TLS instead: The server identity is verified via TLS certificate

# Naïve questions
## Why doesn't SSH use CAs by default
##### SSH assumes:

- Infrastructure is private 
- commonly connects to IPs
- Operators want full control

##### TLS assumes:
- Public internet
- Identity is domain based. IPs can change freely. 
- Centralized trust acceptable
- Frictionless UX (user doesn't have answer any prompts to decide trust every time they visit a website)

## Why doesn't TLS use SSH-style "public key anchoring"?
### SSH-style trust inside TLS = certificate pinning
- Client stores expected cert / fingerprint
- Any change → connection rejected
- Rejects even if signed by trusted CA: A Certificate Authority can no longer convince the client to trust a different key. 

### What pinning prevents
MITM via compromised CA (misused corporate proxy certs as an example)

### Scenarios where certificate pinning can be useful or even required
- Mobile apps calling backend (you control both client and server)
- High security internal services (financial systems)

### Examples where pinning is not welcome
- Public websites: Certificates rotate frequently 
- Browsers: If you pin incorrectly → you brick your site, users permanently locked out

# How does this affect infrastructure automation?

## Ansible 
If we provision virtual machines dynamically (for example, via Terraform) and then configure them using Ansible, every fresh VM has a new host key, unknown to us. That is why we set 

``
host_key_checking = False
``

#### Why this can be dangerous

##### With normal SSH:

Client ──(verify host key)──> Server

##### With host_key_checking = False:
Client ──> whoever answers on that IP

Encryption still exists, authentication does not.
