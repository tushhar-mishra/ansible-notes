# 11 · Connection Settings and ansible.cfg

> **Level:** ★ foundation
>
> **You will learn:** the **connection variables** that tell Ansible how to reach each host (`ansible_host`, `ansible_port`, `ansible_user`, `ansible_ssh_private_key_file`, `ansible_ssh_common_args`, `ansible_connection`), the **connection plugins** (`ssh`, `local`, `community.docker.docker`, `winrm`, `network_cli`), the **`ansible.cfg`** file (where Ansible looks for it, why only one is used, the world-writable-directory trap on WSL), and which setting wins when the same thing is set in several places. You'll write the `ansible.cfg` for your lab so you never type `-i` again.
>
> **You need:** [10 · Inventory basics](10-inventory-basics.md), [05 · SSH recap](05-linux-recap-ssh.md).
>
> **Part 1 · Setup and first contact** · Next: [12 · Ad hoc commands](12-ad-hoc-commands.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Two kinds of settings control Ansible:

1. **Per-host connection variables** in the inventory: *how to reach this host* (`ansible_user=ansible`, `ansible_port=2222`).
2. **`ansible.cfg`**: *how Ansible itself behaves* for this project (default inventory, default user, parallelism, SSH options).

**Picture it.** Connection variables are the **address and door code on each house**. `ansible.cfg` is the **delivery company's rulebook**: which van, which route, how many deliveries at once.

**The one thing to know.** Put an `ansible.cfg` in your project folder. Ansible uses the one in the **current directory** (if no `ANSIBLE_CONFIG` is set):

```ini
# ~/ansible-lab/ansible.cfg
[defaults]
inventory = ./inventory.ini
remote_user = ansible
private_key_file = ~/.ssh/ansible_key
interpreter_python = auto_silent
```

```bash
cd ~/ansible-lab
ansible --version | grep 'config file'       # → config file = /home/you/ansible-lab/ansible.cfg
ansible all -m ansible.builtin.ping          # no -i needed any more
```

**Try it now:** create that file, then run `ansible-config dump --only-changed` to see exactly which settings you changed.

**You've got it if you can:** say where Ansible looks for `ansible.cfg`, in order, and explain why it ignores one in a world-writable folder.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Connection variables

Special inventory variables that start with `ansible_` and control **how** Ansible connects. (They're also called "behavioural inventory parameters".)

| Variable | Meaning | Default | Example |
|---|---|---|---|
| `ansible_host` | Address to connect to | the inventory hostname | `172.30.0.11` |
| `ansible_port` | SSH port | `22` | `2222` |
| `ansible_user` | Login user | your local username (or SSH config's `User`) | `ansible`, `ec2-user` |
| `ansible_ssh_private_key_file` | Private key to use | SSH defaults / agent | `~/.ssh/ansible_key` |
| `ansible_ssh_common_args` | Extra options for `ssh`, `scp` **and** `sftp` | none | `-o ProxyJump=bastion` |
| `ansible_ssh_extra_args` | Extra options for `ssh` only | none | `-o ServerAliveInterval=30` |
| `ansible_connection` | Which **connection plugin** | `ssh` | `local`, `community.docker.docker`, `winrm` |
| `ansible_password` | SSH password | none | ⚠️ Avoid. If you must, keep it in Vault (topic 40) |
| `ansible_python_interpreter` | Python on the target | discovered | `/usr/bin/python3` (topic 07) |
| `ansible_become*` | Privilege escalation settings | | topic 19 |

Where you can set them (best place first):

```ini
# 1. For a group or all hosts (best kept in group_vars/, topic 24)
[all:vars]
ansible_user=ansible
ansible_ssh_private_key_file=~/.ssh/ansible_key

# 2. For one host, inline
[legacy]
old1 ansible_host=10.0.3.5 ansible_port=2222 ansible_user=admin
```

### Connection plugins

A **connection plugin** is the code that gets a module onto a host and runs it.

| Plugin | How it reaches the target | Use for |
|---|---|---|
| `ansible.builtin.ssh` (default) | Runs OpenSSH `ssh`/`scp`/`sftp` | Linux/Unix servers |
| `ansible.builtin.local` | Runs on the control node itself, no SSH | `localhost`, API calls (cloud, Kubernetes) |
| `ansible.builtin.paramiko_ssh` | Pure-Python SSH library | Rare: old systems where OpenSSH can't be used |
| `community.docker.docker` | `docker exec` into a container | Configuring containers without sshd |
| `ansible.builtin.winrm` / `ansible.builtin.psrp` | WinRM (Windows Remote Management) / PowerShell Remoting | Windows (topic 44) |
| `ansible.netcommon.network_cli`, `httpapi`, `netconf` | Device CLI over SSH, or device APIs | Switches, routers, firewalls (topic 44) |
| `amazon.aws.aws_ssm` | AWS SSM (Systems Manager) Session Manager | EC2 without open SSH ports |

### ansible.cfg

An INI file of **settings for Ansible itself**, grouped in sections:

| Section | Common settings |
|---|---|
| `[defaults]` | `inventory`, `remote_user`, `private_key_file`, `forks`, `interpreter_python`, `host_key_checking`, `roles_path`, `collections_path`, `stdout_callback`, `callbacks_enabled`, `log_path`, `timeout` |
| `[privilege_escalation]` | `become`, `become_method`, `become_user`, `become_ask_pass` (topic 19) |
| `[ssh_connection]` | `ssh_args`, `pipelining`, `control_path_dir`, `scp_if_ssh`/`transfer_method` (topics 13, 48) |
| `[inventory]` | `enable_plugins`, `cache` (topic 42) |
| `[galaxy]` | Galaxy/Automation Hub servers (topic 37) |

### Where Ansible finds ansible.cfg

Ansible uses **the first one it finds**, in this order, and **ignores all the others** (they are not merged):

| Order | Location | Typical use |
|---|---|---|
| 1 | File named in the **`ANSIBLE_CONFIG`** environment variable | CI pipelines, special cases |
| 2 | **`./ansible.cfg`** in the **current directory** | ✅ Per project (commit it to Git) |
| 3 | **`~/.ansible.cfg`** (note the dot) in your home | Personal defaults |
| 4 | **`/etc/ansible/ansible.cfg`** | System-wide (distro packages) |

⚠️ **Security rule:** Ansible **ignores** `./ansible.cfg` if the current directory is **world-writable**, because anyone could have planted a malicious config there. You'll see:

```
[WARNING]: Ansible is being run in a world writable directory (/mnt/c/Users/you/ansible-lab),
ignoring it as an ansible.cfg source.
```

This bites **WSL users** whose projects live under `/mnt/c/...`: Windows drives appear as mode `0777`. Keep Ansible projects in your Linux home (`~/ansible-lab`), or set `ANSIBLE_CONFIG` explicitly.

### Environment variables

Almost every setting has an environment variable form: `ANSIBLE_` + the setting's name in capitals.

```bash
ANSIBLE_FORKS=20 ansible-playbook site.yml
ANSIBLE_HOST_KEY_CHECKING=False ansible all -m ping     # ⚠️ labs only (topic 05)
ANSIBLE_STDOUT_CALLBACK=json ansible-playbook site.yml
```

`ansible-config list` shows every setting with its `ini` key and `env` name.

### Which value wins

When the same thing is set in several places, Ansible applies these rules (lowest → highest):

```
 LOWEST                                                                    HIGHEST
 built-in default → ansible.cfg → ANSIBLE_* env var → command-line option → play keyword → variable
                                                         (-u, -f, -i ...)   (remote_user:)  (ansible_user)
```

Example for the login user:

| Where | Setting | Wins over |
|---|---|---|
| ansible.cfg | `remote_user = ansible` | the default (your local username) |
| Environment | `ANSIBLE_REMOTE_USER=ops` | ansible.cfg |
| Command line | `ansible ... -u admin` | env var and config |
| Play keyword | `remote_user: deploy` (in a playbook) | the command line |
| Variable | `ansible_user=ec2-user` (inventory, `group_vars`, `-e`) | **everything above** |

The surprise for most people: **`ansible_user` in the inventory beats `-u` on the command line.** Settings are defaults; variables are specific to a host, so they win. Full variable precedence is topic 25.

---

## Why it exists

- **Projects need repeatable behaviour.** Without a committed `ansible.cfg`, every team member's Ansible behaves differently (different inventory, forks, callbacks).
- **Hosts differ.** Some use port 2222, some are reached through a bastion, some are Windows, some are network switches. Per-host connection variables handle that without changing playbooks.
- **Separation:** playbooks say *what* to do; inventory + config say *where* and *how to connect*. The same playbook runs against the lab, staging and prod.

---

## Analogy

**A courier company.**

- **Connection variables** = what's written on each parcel: street address (`ansible_host`), flat number (`ansible_port`), who signs for it (`ansible_user`), the door code (`ansible_ssh_private_key_file`), "go through the side gate" (`ProxyJump`).
- **Connection plugin** = the vehicle: van (ssh), walking next door (local), drone into the container (docker), boat to the island (winrm).
- **ansible.cfg** = the depot's rulebook: how many vans leave at once (`forks`), which address book to use (`inventory`).
- **Precedence** = the parcel label beats the depot's default instructions, because it's more specific.

---

## How it works inside

How Ansible builds the SSH command for one host, combining all sources:

```
 ansible.cfg              remote_user=ansible, private_key_file=~/.ssh/ansible_key, ssh_args=...
      +
 environment              ANSIBLE_* overrides
      +
 command line             -u, --private-key, -c (connection), -T (timeout)
      +
 play keywords            remote_user:, connection:
      +
 host variables           ansible_host, ansible_user, ansible_port, ansible_ssh_common_args
      │
      ▼  resolved per host
 connection plugin "ssh":
   ssh -C -o ControlMaster=auto -o ControlPersist=60s
       -o 'IdentityFile="/home/you/.ssh/ansible_key"'
       -o Port=22 -o 'User="ansible"' -o ConnectTimeout=10
       -o 'ControlPath="/home/you/.ansible/cp/%C"'
       172.30.0.11 '/bin/sh -c ...'
```

Then OpenSSH itself also applies `~/.ssh/config`. Options Ansible passes **explicitly** (like `User=`) beat what `~/.ssh/config` says for that host; options Ansible doesn't pass are taken from `~/.ssh/config`.

---

## Code / commands

### The lab's ansible.cfg

```ini
# ~/ansible-lab/ansible.cfg
[defaults]
inventory = ./inventory.ini
remote_user = ansible
private_key_file = ~/.ssh/ansible_key
interpreter_python = auto_silent
forks = 10
host_key_checking = True

[ssh_connection]
ssh_args = -C -o ControlMaster=auto -o ControlPersist=60s -o UserKnownHostsFile=~/.ssh/known_hosts_lab -o StrictHostKeyChecking=accept-new
```

- `inventory = ./inventory.ini`: default inventory, relative to the config file. You can still override with `-i`.
- `remote_user`, `private_key_file`: login defaults for every host. Now the lab works even without the `~/.ssh/config` block (for example in CI).
- `interpreter_python = auto_silent`: discover Python, no warning (topic 07).
- `forks = 10`: up to 10 hosts in parallel (default 5; topic 45).
- `host_key_checking = True`: the default. Written here to make the decision visible.
- `ssh_args`: **replaces** Ansible's default SSH options, so keep `-C -o ControlMaster=auto -o ControlPersist=60s` and add the lab's known_hosts file and `accept-new`.

### Generate a full, commented config

```bash
ansible-config init --disabled > ansible.cfg.example
```

Produces every setting, all commented out, with descriptions. Good for reading; don't commit all of it, only what you change.

### See what's really in effect

```bash
ansible-config dump --only-changed
```
```
CONFIG_FILE() = /home/you/ansible-lab/ansible.cfg
DEFAULT_FORKS(/home/you/ansible-lab/ansible.cfg) = 10
DEFAULT_HOST_LIST(/home/you/ansible-lab/ansible.cfg) = ['/home/you/ansible-lab/inventory.ini']
DEFAULT_PRIVATE_KEY_FILE(/home/you/ansible-lab/ansible.cfg) = /home/you/.ssh/ansible_key
DEFAULT_REMOTE_USER(/home/you/ansible-lab/ansible.cfg) = ansible
HOST_KEY_CHECKING(/home/you/ansible-lab/ansible.cfg) = True
INTERPRETER_PYTHON(/home/you/ansible-lab/ansible.cfg) = auto_silent
SSH_ARGS(/home/you/ansible-lab/ansible.cfg) = -C -o ControlMaster=auto -o ControlPersist=60s -o UserKnownHostsFile=~/.ssh/known_hosts_lab -o StrictHostKeyChecking=accept-new
```

- Each line: **internal setting name**, **(where it came from)**, **= value**.
- `(env: ANSIBLE_FORKS)` would appear instead if an environment variable set it.
- This is the fastest answer to "why does Ansible behave differently on my machine?"

Look up one setting:

```bash
ansible-config list | grep -A12 '^DEFAULT_FORKS'
```
```
DEFAULT_FORKS:
  default: 5
  description: Maximum number of forks Ansible will use to execute tasks on target hosts.
  env:
  - name: ANSIBLE_FORKS
  ini:
  - key: forks
    section: defaults
  name: Number of task forks
  type: integer
```

### Per-host connection examples

```ini
[web]
web1 ansible_host=172.30.0.11
web2 ansible_host=172.30.0.12

[private]
app7 ansible_host=10.0.1.7 ansible_ssh_common_args='-o ProxyJump=ops@bastion.example.com'

[containers]
build1 ansible_connection=community.docker.docker ansible_host=ansible-lab-web1-1

[local]
localhost ansible_connection=local
```

- `ProxyJump` in `ansible_ssh_common_args`: reach a private server through a bastion (topic 05). It goes in *common* args so file transfers (`scp`/`sftp`) use the jump too.
- `community.docker.docker` + `ansible_host=<container name>`: no SSH at all; Ansible uses `docker exec` (needs `ansible-galaxy collection install community.docker` and access to the Docker socket).
- `localhost ansible_connection=local`: an explicit local host. Without this line, the implicit localhost behaves the same way.

### Command-line equivalents

| Option | Same as |
|---|---|
| `-i SRC` | `inventory` |
| `-u USER` | `remote_user` |
| `--private-key FILE` | `private_key_file` |
| `-c PLUGIN` | `ansible_connection` / `connection:` |
| `-f N` | `forks` |
| `-T SECONDS` | `timeout` (SSH connect timeout, default 10) |
| `-k` | ask for the SSH password |
| `-e 'var=value'` | an extra variable (highest precedence, topic 25) |

---

## Hands-on lab

1. In `~/ansible-lab`, create the `ansible.cfg` shown above. Then:

   ```bash
   ansible --version | grep 'config file'
   ansible-config dump --only-changed
   ansible all -m ansible.builtin.ping          # no -i
   ```

2. Prove Ansible no longer depends on your `~/.ssh/config` block: temporarily move it away.

   ```bash
   mv ~/.ssh/config ~/.ssh/config.bak
   ansible all -m ansible.builtin.ping          # still 4 x SUCCESS
   mv ~/.ssh/config.bak ~/.ssh/config
   ```

3. See precedence in action:

   ```bash
   ansible web1 -m ansible.builtin.command -a whoami                  # ansible
   ansible web1 -m ansible.builtin.command -a whoami -u root          # -u beats ansible.cfg → root login fails
   ansible web1 -m ansible.builtin.command -a whoami -u root -e ansible_user=ansible   # variable beats -u → ansible
   ```

   The second command fails with `Permission denied` (no key for root on the nodes). The third works: a variable beats the command-line option.

4. Find the config file the hard way:

   ```bash
   cd /tmp && ansible --version | grep 'config file'       # None: no ./ansible.cfg here
   ANSIBLE_CONFIG=~/ansible-lab/ansible.cfg ansible --version | grep 'config file'
   cd ~/ansible-lab
   ```

5. The world-writable trap (simulated):

   ```bash
   mkdir -p /tmp/ww && cp ansible.cfg inventory.ini /tmp/ww/ && chmod 0777 /tmp/ww && cd /tmp/ww
   ansible all -m ansible.builtin.ping       # WARNING ... world writable ... + "No inventory was parsed"
   cd ~/ansible-lab && rm -rf /tmp/ww
   ```

6. Optional: the Docker connection plugin, no SSH involved:

   ```bash
   ansible-galaxy collection install community.docker
   ansible all -i 'ansible-lab-web1-1,' -c community.docker.docker -m ansible.builtin.ping
   ```

   (Run it as a user who can use `docker`. Note the trailing comma: it's a host list, not a file.)

**Done when:** `ansible all -m ansible.builtin.ping` works from `~/ansible-lab` with no `-i` and no `~/.ssh/config`.

---

## Where you'll use it

- **Every project repo** has an `ansible.cfg` at its root, committed to Git.
- **CI pipelines** set `ANSIBLE_CONFIG` or run from the repo root, and pass secrets through env vars.
- **Bastions** in private clouds: `ansible_ssh_common_args` with `ProxyJump` in `group_vars`.
- **Mixed fleets**: `ansible_connection: winrm` for Windows groups, `network_cli` for switches, `local` for API-only plays.
- **Debugging**: `ansible-config dump --only-changed` in a bug report.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Ansible is being run in a world writable directory ... ignoring it as an ansible.cfg source` | Project under `/mnt/c` in WSL, or a `0777` dir | Move the project to `~`; or `export ANSIBLE_CONFIG=/path/ansible.cfg` |
| `config file = None` | You're not in the project directory | `cd` to it, or set `ANSIBLE_CONFIG` |
| A setting in `~/.ansible.cfg` is ignored | A `./ansible.cfg` exists, and only the **first** file found is used | Put the setting in the project file |
| `-u admin` seems ignored | `ansible_user` is set in the inventory/group_vars and wins | Change the variable, or use `-e ansible_user=admin` |
| `Failed to connect to the host via ssh: ... Permission denied` only in CI | CI has no `~/.ssh/config`; key/user weren't in Ansible's own settings | Set `remote_user`, `private_key_file` (or agent) in config/inventory |
| Added `ssh_args` and connections got slower | `ssh_args` **replaced** the defaults, dropping ControlPersist | Include `-C -o ControlMaster=auto -o ControlPersist=60s` |
| `Unsupported parameters` / `connection plugin not found: docker` | Short or wrong plugin name, or the collection is missing | Use the FQCN `community.docker.docker` and install the collection |
| `Timeout (12s) waiting for privilege escalation prompt` | Not a connection issue: sudo is waiting for a password | Topic 19 |

---

## Quick recap

- **Connection variables** (`ansible_host`, `ansible_port`, `ansible_user`, `ansible_ssh_private_key_file`, `ansible_ssh_common_args`, `ansible_connection`) say how to reach each host.
- **Connection plugins**: `ssh` (default), `local`, `community.docker.docker`, `winrm`/`psrp`, `network_cli`, `aws_ssm`.
- **ansible.cfg** lookup: `ANSIBLE_CONFIG` → `./ansible.cfg` → `~/.ansible.cfg` → `/etc/ansible/ansible.cfg`. **First found wins**, no merging. World-writable dirs are ignored.
- Precedence: default < `ansible.cfg` < env var < command line < play keyword < **variable**.
- `ansible-config dump --only-changed` shows what's really in effect and where it came from.

### Interview questions

1. **Q: In what order does Ansible look for its configuration file?**
   A: `ANSIBLE_CONFIG`, then `ansible.cfg` in the current directory, then `~/.ansible.cfg`, then `/etc/ansible/ansible.cfg`. It uses only the first one it finds.
2. **Q: You pass `-u admin` but Ansible still logs in as `ec2-user`. Why?**
   A: `ansible_user` is defined as a variable (inventory or group_vars), and variables take precedence over command-line options.
3. **Q: How do you make Ansible connect through a bastion host?**
   A: Set `ansible_ssh_common_args: '-o ProxyJump=user@bastion'` for those hosts (or configure ProxyJump in the SSH config Ansible uses).

### Scenario question (answer out loud first, then compare)

> *"A new engineer clones the Ansible repo on Windows (WSL2) into `/mnt/c/Users/sam/repos/infra`. Every run says no inventory was parsed, even though `ansible.cfg` sets it. What's wrong and what's the right fix?"*

**Model answer:**
1. Files under `/mnt/c` appear with mode `0777` in WSL. Ansible ignores `ansible.cfg` in a world-writable current directory for security (a planted config could run arbitrary code). The warning says so in the output.
2. Right fix: clone the repo inside the WSL Linux filesystem (`~/repos/infra`). That also gives correct permissions for SSH keys and much faster file access.
3. Workaround if they must stay on `/mnt/c`: set `ANSIBLE_CONFIG` to the file explicitly, or mount the drive with metadata (`/etc/wsl.conf` → `[automount] options = "metadata"`) and fix permissions. Not recommended.
4. Prevention: the repo README says "clone into WSL's home", and a check in CI/pre-commit doesn't help here, so onboarding docs matter.

**Likely follow-ups:**
- *Why is a world-writable config dangerous?* `ansible.cfg` can point to plugin and module paths, so whoever writes it can make Ansible run their code with your credentials.
- *How do you check which config CI used?* Print `ansible --version` and `ansible-config dump --only-changed` at the start of the job.
