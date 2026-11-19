# 05 · Linux Recap I: SSH, Keys and Host Keys

> **Level:** ★ foundation
>
> **You will learn:** the SSH (Secure Shell) you need for Ansible: how a login works, **key pairs** (`ssh-keygen`, `ssh-copy-id`, `authorized_keys`), **ssh-agent**, **host keys** and `known_hosts` (and what "host key checking" really protects you from), `~/.ssh/config`, **jump hosts** (bastions), connection reuse (**ControlPersist**), and exactly how Ansible uses each of these. You'll set up key-based SSH to your own WSL2 machine.
>
> **You need:** [04 · YAML for Ansible](04-yaml-for-ansible.md). Full Linux background: [Linux 064 · Cryptography basics](../../linux-mastery-rhel/notes/064-cryptography-basics.md), [065 · SSH](../../linux-mastery-rhel/notes/065-ssh.md), [066 · SSH server hardening](../../linux-mastery-rhel/notes/066-ssh-server-hardening.md).
>
> **Part 0 · Foundations** · Next: [06 · Linux recap II: admin tasks](06-linux-recap-admin-tasks.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Ansible has no agent. **It logs in with plain SSH**, exactly like you do. If `ssh user@server` works without a password prompt, Ansible will work. If it doesn't, Ansible won't. So every Ansible connection problem is first an SSH problem.

**Picture it.** Two checks happen at the door:
- **You check the building** (host key): "Is this really server web1, or an impostor?" → `~/.ssh/known_hosts` on your side.
- **The building checks you** (user key): "Is this really alice?" → `~/.ssh/authorized_keys` on the server.

**The one thing to know.**

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ansible_key -C "ansible"   # make a key pair
ssh-copy-id -i ~/.ssh/ansible_key.pub user@server           # put the PUBLIC key on the server
ssh -i ~/.ssh/ansible_key user@server                       # test: must work with no password
```

Ansible then uses the same key: `ansible_ssh_private_key_file=~/.ssh/ansible_key`.

**Try it now:** run `ls -l ~/.ssh` in WSL2. If you see `id_ed25519` and `id_ed25519.pub`, you already have a key pair. `known_hosts` lists servers you've trusted before.

**You've got it if you can:** explain which file holds the server's identity and which holds yours, and what "Host key verification failed" means.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### SSH in one paragraph

**SSH** is the encrypted protocol for logging in to a remote machine and running commands there. The **client** (`ssh`, from the OpenSSH package) connects to the **server daemon** (`sshd`, a background service) on **TCP (Transmission Control Protocol) port 22**. They agree on encryption, the client checks the server's identity, then the server checks the client's identity. After that, everything sent is encrypted.

### The four files that matter

| File | Where | Contains | Who reads it |
|---|---|---|---|
| `~/.ssh/id_ed25519` | Control node | Your **private** key. Never share it. Mode `0600` | `ssh` client |
| `~/.ssh/id_ed25519.pub` | Control node | Your **public** key. Safe to share | You, to copy to servers |
| `~/.ssh/authorized_keys` | **Each server**, in the login user's home | Public keys allowed to log in as this user | `sshd` on the server |
| `~/.ssh/known_hosts` | Control node | Public **host keys** of servers you've trusted | `ssh` client |

And one on the server side: `/etc/ssh/ssh_host_ed25519_key` (+ `.pub`) is the **server's own key pair**, its identity. It's created when `sshd` is installed.

### Two kinds of keys, two directions of trust

```
  CONTROL NODE (you)                                SERVER (web1)
  ┌──────────────────────────┐                      ┌──────────────────────────────┐
  │ ~/.ssh/known_hosts        │ ◀── host key ──────  │ /etc/ssh/ssh_host_ed25519_key │
  │  "web1 = SHA256:abc..."   │  ① "Is this really   │  (server's identity)          │
  │                           │     web1?"           │                              │
  │ ~/.ssh/id_ed25519 (priv)  │ ── proof ─────────▶  │ ~deploy/.ssh/authorized_keys  │
  │                           │  ② "Is this really   │  "ssh-ed25519 AAAA... ansible"│
  │                           │     deploy's key?"   │                              │
  └──────────────────────────┘                      └──────────────────────────────┘
```

1. **Host key check (client checks server).** Protects you from a **man-in-the-middle** attack: someone pretending to be web1 to steal your session or secrets.
2. **User authentication (server checks client).** The client proves it holds the private key matching a public key in `authorized_keys`. The private key **never leaves** your machine; it only signs a challenge.

### Key types

| Type | Use it? | Notes |
|---|---|---|
| **ed25519** | ✅ Yes, default choice | Small, fast, secure |
| RSA 3072/4096 | ✅ OK | For very old servers that lack ed25519 |
| RSA 1024/2048 with SHA-1 | ❌ | Disabled by default in modern OpenSSH and on RHEL 9/10 |
| DSA | ❌ | Removed from modern OpenSSH |

### ssh-agent

A private key can be protected by a **passphrase** (a password for the key file). Typing it for every connection would make automation impossible. **ssh-agent** is a small background program that holds your **unlocked** key in memory. `ssh` (and therefore Ansible) asks the agent to sign challenges. You type the passphrase once per session.

```bash
eval "$(ssh-agent -s)"           # start an agent, set SSH_AUTH_SOCK
ssh-add ~/.ssh/ansible_key       # unlock the key once
ssh-add -l                       # list loaded keys
```

In CI pipelines (topics 53–54), the pipeline loads the key into an agent the same way, from a secret.

### `~/.ssh/config`: settings per host

Instead of typing options every time, put them in `~/.ssh/config`:

```
Host web1
    HostName 10.0.1.10
    User deploy
    IdentityFile ~/.ssh/ansible_key

Host 10.0.1.*
    ProxyJump bastion.example.com
```

Now `ssh web1` uses all of that. **Ansible uses the real `ssh` binary, so it reads this file too.**

### Jump hosts (bastions)

In production, servers in private networks aren't reachable from your laptop. You go **through** one hardened server, the **bastion** (or **jump host**):

```
  laptop ──SSH──▶ bastion (public IP) ──SSH──▶ web1 (private 10.0.1.10)
```

OpenSSH does this with `ProxyJump` (`ssh -J bastion web1`). The second connection is tunnelled through the first; your private key stays on your laptop. [Linux 067](../../linux-mastery-rhel/notes/067-ssh-tunnels.md) covers tunnels in depth.

### Connection reuse: ControlMaster / ControlPersist

Opening an SSH connection (handshake + key exchange + authentication) takes time, often 100–500 ms. Ansible runs **many** commands per task. OpenSSH can keep one **master connection** open and run new sessions through it:

- `ControlMaster=auto`: first connection becomes the master.
- `ControlPersist=60s`: keep the master open 60 s after the last use.
- `ControlPath=...`: the socket file used to reach the master.

**Ansible turns this on by default** (`-o ControlMaster=auto -o ControlPersist=60s`), which is why the second task to a host is much faster than the first. Topic 48 tunes it.

---

## Why it exists

Ansible chose SSH because **every Linux server already has it**, it's **encrypted**, it's **audited** (logins appear in `/var/log/secure` or `/var/log/auth.log`), and admins already know how to secure it. Nothing new to open in the firewall, nothing new to patch.

The flip side: Ansible inherits **all** of SSH's rules. Wrong key permissions, an unknown host key, a disabled user, a firewall on port 22: all of these become Ansible failures.

---

## Analogy

**A secure office building.**

- **Host key** = the building's official badge on the front door. The first time you visit, you note it down (`known_hosts`). Next time, if the badge looks different, you stop: maybe it's a fake building set up to steal your briefcase.
- **Your key pair** = a personal stamp (private key) and a sample print of it (public key). You give the sample to reception (`authorized_keys`). At the door, you stamp a fresh slip and they compare it with the sample. Your stamp never leaves your pocket.
- **ssh-agent** = a trusted assistant holding your stamp so you don't unlock your safe for every visit.
- **Bastion** = the guarded main gate of a business park. You can't drive to building 7 directly; you go through the gate first.
- **ControlPersist** = keeping the door propped open for a minute while you carry in several boxes.

---

## How it works inside

### One SSH login, step by step

1. **TCP connect** to `server:22`.
2. **Version exchange** (`SSH-2.0-OpenSSH_9.x`).
3. **Key exchange**: both sides agree a shared secret session key (Diffie–Hellman / Curve25519). Everything after this is encrypted.
4. **Server proves its identity**: it signs the exchange with its host private key. The client checks the host public key against `known_hosts`:
   - **Match** → continue.
   - **Unknown host** → ask "Are you sure you want to continue connecting?" (or fail, if not interactive).
   - **Different key** → **stop** with `REMOTE HOST IDENTIFICATION HAS CHANGED!`
5. **User authentication**: the client offers public keys (from the agent, `-i`, `IdentityFile`, defaults). The server checks `authorized_keys`; the client signs a challenge with the matching private key.
6. **Session**: the server starts the user's shell or runs the requested command.

### How Ansible uses SSH

Ansible's default **connection plugin** (the code that decides *how* to reach a host) is called `ssh`. It **runs the OpenSSH `ssh` program** for you, with options built from your inventory and settings:

| Ansible setting (variable) | Becomes the ssh option | Topic |
|---|---|---|
| `ansible_host` | the address to connect to | 11 |
| `ansible_user` | `-o User=...` | 11 |
| `ansible_port` | `-o Port=...` | 11 |
| `ansible_ssh_private_key_file` | `-o IdentityFile=...` | 11 |
| `ansible_ssh_common_args` | extra options for ssh/scp/sftp, e.g. `-o ProxyJump=bastion` | 11 |
| `host_key_checking = False` (ansible.cfg) | `-o StrictHostKeyChecking=no` | below |
| (defaults) | `-C -o ControlMaster=auto -o ControlPersist=60s` | 48 |

You can see the exact `ssh` command Ansible runs with `-vvv` (topic 13):

```
<web1> SSH: EXEC ssh -C -o ControlMaster=auto -o ControlPersist=60s
  -o 'IdentityFile="/home/you/.ssh/ansible_key"'
  -o KbdInteractiveAuthentication=no -o PreferredAuthentications=gssapi-with-mic,gssapi-keyex,hostbased,publickey
  -o PasswordAuthentication=no -o 'User="deploy"' -o ConnectTimeout=10
  -o 'ControlPath="/home/you/.ansible/cp/1a2b3c4d5e"' 10.0.1.10 '/bin/sh -c '"'"'echo ~deploy && sleep 0'"'"''
```

- `PasswordAuthentication=no`: Ansible assumes **key** login unless you ask for a password (`-k`).
- `ConnectTimeout=10`: give up after 10 seconds → `UNREACHABLE`.
- `ControlPath`: the socket for connection reuse, under `~/.ansible/cp/`.
- The last part is the remote command (here: "find deploy's home directory").

**Password logins** (`ansible ... -k` / `--ask-pass`) need extra help, because `ssh` reads passwords from a terminal. Since ansible-core 2.19 the default mechanism is `SSH_ASKPASS` (no extra package); older setups needed the `sshpass` program. In production, use **keys**, not passwords.

### Host key checking in Ansible

Ansible respects `known_hosts`. With a brand-new server that isn't in it yet:

- **Interactive terminal**: Ansible shows the ssh prompt and waits for you to type `yes`.
- **No terminal** (CI pipeline, cron): it fails with `Host key verification failed` → `UNREACHABLE`.

Your options, from best to worst:

| Option | How | Security |
|---|---|---|
| **Pre-load verified host keys** | `ssh-keyscan host >> known_hosts`, **then compare the fingerprint** with one from a trusted source (cloud console output, cloud-init logs, your provisioning tool) | ✅ Best |
| **Trust on first use, but never accept a changed key** | `StrictHostKeyChecking=accept-new` in `~/.ssh/config` or `ansible_ssh_common_args` | ✅ Good for fresh lab/cloud servers |
| **Turn checking off** | `host_key_checking = False` in `ansible.cfg`, or `ANSIBLE_HOST_KEY_CHECKING=False` | ⚠️ Labs only. Allows man-in-the-middle; secrets and `become` passwords could be stolen |

---

## Code / commands

### Create a key for automation

```bash
ssh-keygen -t ed25519 -f ~/.ssh/ansible_key -C "ansible@control" -N ""
```

- `-t ed25519`: key type.
- `-f`: file name (creates `ansible_key` and `ansible_key.pub`).
- `-C`: a comment, stored at the end of the public key, so you can recognise it in `authorized_keys`.
- `-N ""`: empty passphrase. Fine for a lab. In production use a passphrase + agent, or let your CI secret store protect it.

```
Generating public/private ed25519 key pair.
Your identification has been saved in /home/you/.ssh/ansible_key
Your public key has been saved in /home/you/.ssh/ansible_key.pub
The key fingerprint is:
SHA256:Qm3k8T...Zp0 ansible@control
```

### Install the public key on a server

```bash
ssh-copy-id -i ~/.ssh/ansible_key.pub deploy@10.0.1.10
```

- Logs in once (with a password), appends the public key to `~deploy/.ssh/authorized_keys` and sets correct permissions.

(Later Ansible itself will manage keys with the `ansible.posix.authorized_key` module, topic 17.)

### First connection: the host key prompt

```bash
ssh -i ~/.ssh/ansible_key deploy@10.0.1.10
```
```
The authenticity of host '10.0.1.10 (10.0.1.10)' can't be established.
ED25519 key fingerprint is SHA256:x7Rk2...LQ.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.0.1.10' (ED25519) to the list of known hosts.
```

- The fingerprint is a short hash of the server's host public key. Compare it with the real one: on the server, `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub`.
- `Permanently added`: written to `~/.ssh/known_hosts`. Next time: no prompt.

### Pre-load host keys (for CI and fresh servers)

```bash
ssh-keyscan -t ed25519 10.0.1.10 10.0.1.11 >> ~/.ssh/known_hosts
ssh-keygen -lf ~/.ssh/known_hosts          # list fingerprints to compare
ssh-keygen -R 10.0.1.10                    # remove an old entry (server rebuilt)
```

### Permissions that must be right

`sshd` refuses keys if files are too open (anyone else could have edited them):

| Path | Mode | Owner |
|---|---|---|
| `~` (home dir on server) | not writable by group/others (e.g. `0755` or `0700`) | the user |
| `~/.ssh` | `0700` | the user |
| `~/.ssh/authorized_keys` | `0600` | the user |
| `~/.ssh/id_ed25519` (private key, control node) | `0600` | you |

On RHEL, SELinux also needs the right label on `~/.ssh` (`restorecon -Rv ~/.ssh`, [Linux 073](../../linux-mastery-rhel/notes/073-selinux-file-contexts.md)).

### Debug a failing SSH login

```bash
ssh -v -i ~/.ssh/ansible_key deploy@10.0.1.10
```
```
debug1: Connecting to 10.0.1.10 [10.0.1.10] port 22.
debug1: Connection established.
debug1: Server host key: ssh-ed25519 SHA256:x7Rk2...LQ
debug1: Host '10.0.1.10' is known and matches the ED25519 host key.
debug1: Offering public key: /home/you/.ssh/ansible_key ED25519 SHA256:Qm3k8T... explicit
debug1: Server accepts key: /home/you/.ssh/ansible_key ED25519 SHA256:Qm3k8T... explicit
debug1: Authentication succeeded (publickey).
```

- `Connection established`: network and port are fine.
- `is known and matches`: host key check passed.
- `Offering public key` → `Server accepts key`: user authentication passed.
- If it ends with `Permission denied (publickey)` instead, the server didn't accept any key: check `authorized_keys` content and permissions, and the server log (`sudo journalctl -u sshd` on RHEL, `-u ssh` on Ubuntu).

---

## Hands-on lab

Set up key-based SSH **to your own WSL2 Ubuntu** (it acts as both client and server). This is exactly what Ansible will need in topic 09.

1. Install and start the SSH server:

   ```bash
   sudo apt update && sudo apt install -y openssh-server
   sudo systemctl enable --now ssh        # if systemd isn't enabled in WSL: sudo service ssh start
   ss -tlnp | grep ':22'                   # sshd listening on port 22
   ```

2. Create the automation key:

   ```bash
   ssh-keygen -t ed25519 -f ~/.ssh/ansible_key -C "ansible-lab" -N ""
   ```

3. Install it for your own user (you'll type your Linux password once):

   ```bash
   ssh-copy-id -i ~/.ssh/ansible_key.pub "$USER@localhost"
   ```

   Answer `yes` to the host key prompt. Then compare: `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` must show the same fingerprint you accepted.

4. Test: no password prompt should appear.

   ```bash
   ssh -i ~/.ssh/ansible_key "$USER@localhost" 'hostname; id'
   ```

5. Add a config alias and test it:

   ```bash
   cat >> ~/.ssh/config <<EOF
   Host labself
       HostName localhost
       User $USER
       IdentityFile ~/.ssh/ansible_key
   EOF
   chmod 600 ~/.ssh/config
   ssh labself 'echo it works'
   ```

6. Break it, read the error, fix it:
   - `chmod 0777 ~/.ssh/authorized_keys` → `ssh labself` asks for a password (key refused). `sudo journalctl -u ssh -n 5` shows `Authentication refused: bad ownership or modes`. Fix: `chmod 0600 ~/.ssh/authorized_keys`.
   - Simulate a "changed server" by trusting a **different** key for `localhost`:

     ```bash
     ssh-keygen -R localhost                                   # forget the real host key
     ssh-keygen -t ed25519 -f /tmp/fakehost -N "" -q           # a fake "server" key
     echo "localhost $(cut -d' ' -f1,2 /tmp/fakehost.pub)" >> ~/.ssh/known_hosts
     ssh labself                                               # → REMOTE HOST IDENTIFICATION HAS CHANGED!
     ```

     Fix: `ssh-keygen -R localhost`, reconnect, accept the real key, `rm /tmp/fakehost*`.

**Done when:** `ssh labself 'echo ok'` prints `ok` with no prompt.

---

## Where you'll use it

- **Every Ansible run**: Ansible is only as good as its SSH access.
- **New servers**: getting the automation user and key onto them (cloud-init, `key_name` in Terraform, or `ssh-copy-id`).
- **CI pipelines**: SSH private key from a secret + `known_hosts` from a secret or `ssh-keyscan` with verification (topics 53–54).
- **Bastion setups** in AWS/Azure private subnets: `ProxyJump` in `ansible_ssh_common_args`.
- **Key rotation**: replacing the automation key on 500 servers is itself an Ansible job.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `ssh: connect to host X port 22: Connection refused` | `sshd` not running, or listening on another port | `systemctl status sshd`; set `ansible_port` |
| `Connection timed out` | Firewall/security group blocks 22, wrong IP, host down | Check security group, `firewall-cmd`, route; try `nc -vz host 22` |
| `Permission denied (publickey)` | Public key not in `authorized_keys`, wrong user, wrong key offered | `ssh -v`; check user and `IdentityFile`; `ssh-copy-id` |
| Key exists but still asks for a password | Bad permissions on `~`, `~/.ssh` or `authorized_keys`; SELinux label | `chmod 700 ~/.ssh; chmod 600 ~/.ssh/authorized_keys`; `restorecon -Rv ~/.ssh` |
| `Host key verification failed` (often in CI) | Host not in `known_hosts` and no terminal to ask | Pre-load with `ssh-keyscan` (verify), or `accept-new` |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` | Server rebuilt with new host keys, **or** a real attack | Confirm the rebuild, then `ssh-keygen -R host`. Never just disable checking |
| `Too many authentication failures` | Agent offers many keys; server gives up after `MaxAuthTries` | `IdentitiesOnly yes` + the right `IdentityFile` |
| `UNPROTECTED PRIVATE KEY FILE!` | Private key mode too open | `chmod 600 ~/.ssh/ansible_key` |
| Works by hand, fails in Ansible | Different user/key/host than you think | Run Ansible with `-vvv` and compare its `SSH: EXEC` line with your manual command |
| Private key mounted from Windows (`/mnt/c/...`) refused | Windows drives in WSL show as `0777` | Keep keys in the Linux home (`~/.ssh`), not on `/mnt/c` |

---

## Quick recap

- Ansible logs in with **plain OpenSSH**. If `ssh user@host` works without prompts, Ansible will too. It reads `~/.ssh/config`.
- **Host key** (`known_hosts`) = you trust the server. **User key** (`authorized_keys`) = the server trusts you.
- Use **ed25519** keys, correct permissions (`700` `.ssh`, `600` keys), **ssh-agent** for passphrases.
- Host key checking protects against man-in-the-middle. Pre-load verified keys or use `accept-new`. Disabling it is for labs only.
- Ansible reuses connections with **ControlPersist** by default; bastions via **ProxyJump**.

### Interview questions

1. **Q: How does Ansible authenticate to Linux servers?**
   A: Over SSH, usually with key-based authentication: the control node's private key matches a public key in the remote user's `authorized_keys`. It runs the OpenSSH client, so `~/.ssh/config`, agents and jump hosts all work.
2. **Q: What is host key checking and should you disable it?**
   A: The SSH client verifying the server's identity against `known_hosts` to prevent man-in-the-middle attacks. Don't disable it in production: pre-populate verified host keys or use `StrictHostKeyChecking=accept-new`.
3. **Q: How do you run Ansible against servers in a private subnet?**
   A: Through a bastion with SSH `ProxyJump`, set in `~/.ssh/config` or `ansible_ssh_common_args: '-o ProxyJump=user@bastion'`. (Or run the control node inside the network, e.g. a CI runner or AWX there.)

### Scenario question (answer out loud first, then compare)

> *"Your GitHub Actions job running Ansible fails on 3 new servers with `Host key verification failed`, while the other 20 work. A teammate proposes `ANSIBLE_HOST_KEY_CHECKING=False`. What do you do?"*

**Model answer:**
1. Explain why: the 3 new servers aren't in the `known_hosts` the pipeline uses, and CI has no terminal to answer the prompt.
2. Turning checking off makes every run vulnerable to man-in-the-middle, and Ansible sends sudo passwords and secrets over that connection. Reject it for production.
3. Fix properly: keep a `known_hosts` file as a pipeline secret/variable and update it when servers are created, using fingerprints from a trusted source (e.g. Terraform/cloud-init output). Or use SSH certificates signed by an internal CA (Certificate Authority) so any host with a valid cert is trusted.
4. Acceptable middle ground for short-lived, freshly created cloud servers: `StrictHostKeyChecking=accept-new`, which still rejects **changed** keys.

**Likely follow-ups:**
- *What does `accept-new` still protect against?* A server whose key changes after the first contact (a hijack). It does not protect the very first connection.
- *How do you see the exact ssh command Ansible runs?* `-vvv` shows the `SSH: EXEC` line.
