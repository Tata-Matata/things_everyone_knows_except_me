In ansible.cfg we can specify

``
host_key_checking = False
``

For the cases when nodes are frequently recreated, SSH host key change and we don't want automation blocked by a trust prompt 
``
The authenticity of host '1.2.3.4' can't be established.
``

Supposed to be acceptable risk if you trust VM provider. 

# Risks

## Background
Every machine that runs SSH server has a host key pair. The public key is presented to clients during SSH handshake. On first SSH connection, the host or rather its public key is unknown to the client so the client is prompted to confirm the authenticity of the host. If confirmed - the key is stored in 
``
~/.ssh/known_hosts
``
Client is not presented with interactive prompt any more.
This first check should prevent MITM attacks. However, if you roll out new VMs frequently, you have to confirm each time.

## What happens when host_key_checking = True (default)
Ansible uses SSH under the hood. Connects to 1.2.3.4 SSH compares host key to known_hosts. Key mismatch → hard failure
Ansible then:

- Aborts execution
- Skips all tasks
- Leaves cluster half-configured

## What changes with host_key_checking = False
Tells Ansible: Accept whatever host key the server presents

Equivalent to 
``
ssh -o StrictHostKeyChecking=no
``

You lose MITM protection. For interactive human SSH it is a bad idea. For controlled infra automation may be acceptable if you are connecting to IPs you just created yourself

# Alternative 1
## ssh-keyscan after creation

``
ssh-keyscan -H <ips> >> ~/.ssh/known_hosts
``
ssh-keygen fetches SSH host public keys from remote servers without making an interactive SSH connection and stores them in ~/.ssh/known_hosts

#### What it actually does
- Opens a TCP connection to port 22 (or another SSH port)
- Performs **only the SSH handshake**
- Requests the server’s host public key
- Prints the key to stdout
- Does not authenticate
- Does not verify identity
- Does not establish a shell
- -H hashes the hostname
- with -t ed25519,rsa we can specify key types

### Still some risks
ssh-keyscan does not verify that the key belongs to the intended host.

That means: If a Man-in-the-Middle is present at scan time, you will store the attacker’s key permanently.

So it is safe only if:

-  You run it on a trusted network
- Or you compare fingerprints via a trusted channel
- Or the infrastructure is created inside the same private network

### Compared to host_key_checking=False
Completely disabling host key checking  does not just allow first-connect MITM — it allows permanent MITM. For MITM to succeed during ssh-keyscan, Attacker must be present at scan time. So ssh-keygen, while not completely eliminating the risks, reduces the window of attack

# Alternative 2
## Retrieving generated host keys via cloud provider API and store in known_hosts

- API is authenticated (TLS + IAM)
- MITM during provisioning or on SSH network impossible
- If your cloud API is compromised, you have much bigger problems.

Hetzner via Terraform does not expose host keys

# Alternative 3
## SSH host certificates

- host generates its own key and cert
- cert gets signed by a CA
- hosts prove they were signed by CA, so identity asserted by CA signature
- clients trust the CA
- no host pinning in known_hosts




