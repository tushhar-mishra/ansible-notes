# 03 · What Is Ansible?

> **Level:** ★ foundation
>
> **You will learn:** what Ansible is and where it came from, the **control node** and **managed nodes**, the core building blocks (inventory, module, task, play, playbook, role, collection, plugin) at a glance, the difference between **ansible-core**, the **`ansible` package**, **AWX** and **Ansible Automation Platform**, how versions work, and the mental model every later topic builds on. You'll run your first Ansible command inside a throw-away container.
>
> **You need:** [02 · The tool landscape](02-tool-landscape.md)
>
> **Part 0 · Foundations** · Next: [04 · YAML for Ansible](04-yaml-for-ansible.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Ansible is an open-source automation tool. You install it on **one** machine (the **control node**). It connects over **SSH** (Secure Shell) to the servers you want to manage (**managed nodes**), runs small programs called **modules** there, and reports what changed. Nothing is installed on the managed nodes: they only need SSH and Python.

**Picture it.** A conductor (control node) with a sheet of music (playbook) and a list of musicians (inventory). Each musician already knows how to play (has SSH + Python); the conductor tells each one what to play, in order.

**The one thing to know.** The vocabulary:

```
inventory  = WHO     (list of servers and groups)
module     = WHAT    (one unit of work: "install a package", "copy a file")
task       = one module call with its arguments
play       = a list of tasks + which hosts they run on
playbook   = a YAML file with one or more plays
```

**Try it now** (Docker in WSL2, nothing installed on your machine):

```bash
docker run --rm -it python:3.13-slim bash -c \
  "pip install -q ansible-core && ansible localhost -m ansible.builtin.ping"
```

You'll see `localhost | SUCCESS` and `"ping": "pong"`. You just ran Ansible.

**You've got it if you can:** say what runs on the control node, what runs on managed nodes, and what "inventory", "module", "task", "play" and "playbook" mean.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

**Ansible** is an **agentless**, **push-based** automation tool written in Python. You describe what you want in **YAML** files (YAML Ain't Markup Language, topic 04), and Ansible makes it happen on many machines at once, over SSH.

It does three kinds of work:

| Kind of work | Example |
|---|---|
| **Configuration management** | "Every server has chrony installed, this `chrony.conf`, and the service running" |
| **Application deployment** | "Copy version 2.4 of the app, update its config, restart it" |
| **Orchestration** (doing steps across many machines in a set order) | "Take web1 out of the load balancer, upgrade it, check health, put it back, then web2…" |

Plus a lot of everyday **automation**: patching, user management, gathering information from 300 servers, configuring network switches, calling cloud APIs.

### A short history

- **2012**: Michael DeHaan (who earlier co-wrote the Cobbler provisioning tool) releases Ansible. The name comes from science fiction: an "ansible" is a device that sends messages instantly across space.
- **2013–2015**: the company AnsibleWorks (renamed Ansible, Inc.) grows it fast because it's simple and agentless.
- **2015**: **Red Hat** buys Ansible.
- **2019**: **IBM** buys Red Hat. Ansible stays open source; Red Hat sells the enterprise version.
- **2021**: the project splits into **ansible-core** (the engine) and **collections** (the modules, released separately). This is why you see names like `ansible.builtin.copy` today (topic 16).

### The two kinds of machines

| | **Control node** | **Managed node** (also "target" or "host") |
|---|---|---|
| What it is | The machine where Ansible is installed and where you run `ansible` / `ansible-playbook` | Any server/device Ansible manages |
| What it needs | Linux or macOS (on Windows: **WSL2**, Windows Subsystem for Linux), Python 3.13+ for ansible-core 2.22 | SSH server + a login user + **Python 3.9+** (for Linux targets). Windows targets: WinRM or SSH + PowerShell |
| How many | One (your laptop, a CI runner, a "bastion" admin server, or AWX) | 1 to many thousands |
| Ansible installed? | Yes | **No** |

**Native Windows can't be a control node.** That's why your lab uses WSL2 Ubuntu (topic 08).

### The building blocks at a glance

You'll learn each one properly later. Here's the whole map so nothing feels random:

| Block | What it is | Example | Full topic |
|---|---|---|---|
| **Inventory** | The list of managed nodes, in **groups** | `[web]` → `web1`, `web2` | 10 |
| **Module** | A small program that does one job **idempotently** (topic 01: re-running changes nothing if already correct) | `ansible.builtin.user`, `ansible.builtin.copy` | 16–18 |
| **Task** | One call to one module, with arguments and a name | "user deploy exists" | 14 |
| **Play** | Tasks + the hosts they run on + settings (like sudo) | "on group `web`, run these 5 tasks" | 14 |
| **Playbook** | A YAML file containing one or more plays | `site.yml` | 14 |
| **Variables** | Values that change per host/group/environment | `http_port: 8080` | 21–25 |
| **Facts** | Information Ansible collects from each host automatically | OS name, IP addresses, memory | 23 |
| **Template** | A config file with placeholders filled from variables, written in **Jinja2** (a Python templating language) | `nginx.conf.j2` | 29 |
| **Handler** | A task that runs only when something changed | "restart nginx" after its config changed | 30 |
| **Role** | A reusable, standard folder of tasks + templates + defaults | `roles/nginx/` | 35–36 |
| **Collection** | A distributable package of modules, plugins and roles | `community.general`, `amazon.aws` | 37 |
| **Plugin** | Code that extends Ansible itself (how to connect, how to print output, where to find hosts) | `ssh` connection plugin, `aws_ec2` inventory plugin | 38 |

### ansible-core vs `ansible` vs AWX vs AAP

These four names confuse everyone. Here's the difference:

| Name | What it is | What you get | Cost |
|---|---|---|---|
| **ansible-core** | The **engine**: the `ansible*` commands + the `ansible.builtin` collection (~70 core modules: `copy`, `user`, `dnf`, `apt`, `template`…) | Minimal. You add the collections you need | Free, open source |
| **`ansible`** (the community package) | ansible-core **plus ~85 curated collections** (`community.general`, `amazon.aws`, `ansible.posix`, `kubernetes.core`…) | "Batteries included". Big (hundreds of MB) | Free, open source |
| **AWX** | A **web UI and API** to run Ansible centrally: who may run what, on which servers, with which secrets, on a schedule | A control plane for teams | Free, open source (upstream of AAP) |
| **Ansible Automation Platform (AAP)** | Red Hat's supported product: Automation Controller (from AWX) + Automation Hub (certified collections) + Event-Driven Ansible + execution environments + support | Enterprise platform | Paid subscription |

**Versions today** (checked November 2026):

| Thing | Version | Notes |
|---|---|---|
| ansible-core | **2.22** (2.22.0, 2 Nov 2026) | New release every ~6 months (May, Nov). Each supported ~18 months |
| `ansible` package | **15** (15.0.0, 17 Nov 2026) | Built on ansible-core 2.22. `ansible` 14 = core 2.21, `ansible` 13 = core 2.20 |
| AAP | 2.6 (2.7 announced as container-install only) | Ships its own tested ansible-core |

Don't mix up the two version numbers: `ansible 15.0.0` *contains* `ansible-core 2.22.0`. When someone says "Ansible 2.9", they mean the old all-in-one release from 2019, before the split. You will still meet it on old servers; it's long out of support. Treat any blog post that uses it with suspicion.

**Which one should you learn on?** ansible-core + the few collections you need (topic 08). It's what production teams pin in CI and what execution environments (topic 52) are built from. The `ansible` package is fine for learning too.

---

## Why it exists

Before Ansible, the main CM (configuration management) tools were Puppet and Chef (topic 02). Both need:

1. an **agent** installed and running on every server,
2. a **central server** with certificates for every agent,
3. a **custom language** (Puppet DSL, Ruby).

That's a lot of setup before you can automate anything, and it doesn't help on a brand-new server that has no agent yet. Network devices (switches, routers) can't run agents at all.

Ansible's idea: **use what's already there.** Every Linux server has SSH and almost every one has Python. So:

- Install Ansible on **one** machine. Done.
- Write YAML, which ops people can read without learning a programming language.
- Run it. Changes happen in the order you wrote, which is easy to reason about.

---

## Analogy

**An orchestra.**

| Orchestra | Ansible |
|---|---|
| Conductor | Control node |
| Musicians | Managed nodes |
| List of musicians by section (violins, brass) | Inventory with groups |
| The musicians' existing skills (they already know how to play) | SSH + Python already on servers. **No agent** |
| One instruction: "play bars 1–8 forte" | A task calling a module |
| One movement for the string section | A play (tasks for one group of hosts) |
| The whole score | A playbook |
| A standard piece every orchestra can play ("Happy Birthday" arrangement) | A role |
| A music publisher's catalogue of arrangements | A collection |

The conductor doesn't live inside the violinist. They just give instructions, and the violinist plays. That's agentless.

---

## How it works inside

The mental model you'll use for the rest of these notes. Topic 13 and topic 15 trace it in full detail.

```
 CONTROL NODE (your WSL2 Ubuntu)                         MANAGED NODES
 ┌──────────────────────────────────────┐
 │ ansible-playbook site.yml             │
 │                                       │
 │ 1. read inventory  → which hosts?     │
 │ 2. read playbook   → which tasks?     │
 │ 3. work out variables for each host   │
 │                                       │   SSH    ┌────────────────────────┐
 │ 4. for each task, for each host ──────┼────────▶ │ web1                    │
 │    (up to 5 hosts at once = "forks")  │          │ a. temp dir in ~/.ansible/tmp
 │                                       │          │ b. module (Python) copied in
 │                                       │          │ c. python3 runs it      │
 │                                       │ ◀────────┤ d. prints JSON result   │
 │                                       │   JSON   │ e. temp dir deleted     │
 │ 5. print ok / changed / failed        │          └────────────────────────┘
 │ 6. after all tasks: PLAY RECAP        │          (same for web2, web3...)
 └──────────────────────────────────────┘
```

Key points:

1. **Everything starts on the control node.** Managed nodes never talk to each other or to the control node on their own.
2. **Each task is a separate mini-program.** The module is copied, run with Python, and deleted. Nothing stays behind.
3. **The result is JSON** (JavaScript Object Notation, a simple text format for data). It always says at least whether the task `changed` something and whether it `failed`.
4. **Hosts run in parallel, tasks run in order.** By default task 1 runs on (up to 5 at a time) all hosts, then task 2 on all hosts, and so on.
5. **No state is kept.** Every run checks the real servers again.

---

## Code / commands

### The commands that come with ansible-core

| Command | What it's for | Topic |
|---|---|---|
| `ansible` | Run **one** module on hosts ("ad hoc") | 12 |
| `ansible-playbook` | Run a playbook | 14 |
| `ansible-inventory` | Show/check the inventory | 10 |
| `ansible-doc` | Read module documentation offline | 16 |
| `ansible-config` | See configuration settings and where they came from | 11 |
| `ansible-galaxy` | Install/create roles and collections | 35, 37 |
| `ansible-vault` | Encrypt secrets | 40 |
| `ansible-console` | Interactive shell for ad hoc commands | 58 |
| `ansible-pull` | Pull mode: a host pulls a Git repo and runs it on itself | 55 |
| `ansible-test` | Test collections (for collection developers) | 38 |

Separate tools you'll meet later: `ansible-lint` (50), `molecule` (51), `ansible-navigator` and `ansible-builder` (52).

### A first look at an Ansible run

```bash
ansible localhost -m ansible.builtin.ping
```

- `ansible`: the ad hoc command.
- `localhost`: the host to target. With no inventory, Ansible still knows `localhost` (it uses it without SSH).
- `-m ansible.builtin.ping`: run the `ping` module. It's **not** network ping (ICMP, Internet Control Message Protocol). It checks "can Ansible log in and run Python there?"

```
[WARNING]: No inventory was parsed, only implicit localhost is available
localhost | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

- `[WARNING]`: you gave no inventory file. Harmless here.
- `localhost | SUCCESS`: host name and result.
- `"changed": false`: ping changes nothing. Every module reports `changed`.
- `"ping": "pong"`: the module's own return value.

Collect facts about the machine:

```bash
ansible localhost -m ansible.builtin.setup -a 'filter=ansible_distribution*'
```

- `-a '...'`: arguments for the module. Here: only show facts whose names start with `ansible_distribution`.

```
localhost | SUCCESS => {
    "ansible_facts": {
        "ansible_distribution": "Debian",
        "ansible_distribution_major_version": "13",
        "ansible_distribution_release": "trixie",
        "ansible_distribution_version": "13.2"
    },
    "changed": false
}
```

- `ansible_facts`: data Ansible discovered. Later you'll use it to write one playbook that works on both RHEL and Ubuntu (topic 23).
- The `python:3.13-slim` image is built on Debian, which is why it says Debian.

---

## Hands-on lab

Run Ansible without installing anything on your machine.

1. Start a throw-away Python container and install ansible-core inside it:

   ```bash
   docker run --rm -it python:3.13-slim bash
   pip install -q ansible-core
   ansible --version
   ```

   Look at the first line: `ansible [core 2.22.x]`. Topic 08 explains every line.

2. Run the ping module and the setup module:

   ```bash
   ansible localhost -m ansible.builtin.ping
   ansible localhost -m ansible.builtin.setup -a 'filter=ansible_memtotal_mb'
   ```

3. Make a change and see `changed`:

   ```bash
   ansible localhost -m ansible.builtin.file -a 'path=/tmp/hello state=directory'
   ansible localhost -m ansible.builtin.file -a 'path=/tmp/hello state=directory'
   ```

   - First run: `CHANGED` (yellow) and `"changed": true`. The directory was created.
   - Second run: `SUCCESS` (green) and `"changed": false`. Already there: **idempotent**.

4. Compare with the non-idempotent way:

   ```bash
   ansible localhost -m ansible.builtin.command -a 'mkdir /tmp/hello2'
   ansible localhost -m ansible.builtin.command -a 'mkdir /tmp/hello2'
   ```

   - First run: `CHANGED`. The `command` module just runs a command; it can't know whether anything changed, so it always says `changed`.
   - Second run: `FAILED` with `mkdir: cannot create directory '/tmp/hello2': File exists`.

   This is the whole idea of topic 27 in miniature: **use a real module, not a command**.

5. `exit` (the container disappears).

---

## Where you'll use it

- **Server baselines** for every new VM: users, SSH, time sync, monitoring agents, hardening.
- **Patching** fleets in batches.
- **App deploys** to VMs, with rolling restarts.
- **Kubernetes node setup** (kubeadm clusters are often installed with Ansible, e.g. the Kubespray project).
- **Network automation**: Cisco, Arista, Juniper switches with no agent possible.
- **From CI/CD**: Jenkins or GitHub Actions run `ansible-playbook` after Terraform (topics 53–54).
- **At scale with a UI**: AWX/AAP (topic 56).

---

## When it breaks

You'll meet these first. Each has full coverage later; for now, recognise the words.

| Message | Means | Topic |
|---|---|---|
| `UNREACHABLE! ... Failed to connect to the host via ssh` | Ansible couldn't log in over SSH | 05, 11, 59 |
| `Permission denied (publickey,password)` | SSH key not accepted | 05 |
| `Missing sudo password` / `Timeout waiting for privilege escalation prompt` | sudo needs a password Ansible wasn't given | 19 |
| `/usr/bin/python3: not found` / `module_stdout ... python: not found` | No Python on the target | 07 |
| `couldn't resolve module/action 'xyz'` | Module name wrong or its collection not installed | 16, 37 |
| `'my_var' is undefined` | Variable used but never set | 21 |
| `Syntax Error while loading YAML` | YAML indentation/quoting mistake | 04 |
| `Host key verification failed` | SSH doesn't trust this server yet | 05 |
| `ansible: command not found` | Not installed, or installed in a venv that isn't active | 08 |

---

## Quick recap

- Ansible = agentless, push-based automation over SSH. Install it on the **control node** only; **managed nodes** need SSH + Python.
- Inventory = who. Module = what (idempotent unit of work). Task = one module call. Play = tasks + hosts. Playbook = YAML file of plays.
- Each task: module copied over SSH → run with Python → JSON result (`changed`, `failed`) → deleted.
- **ansible-core** = engine + `ansible.builtin`. **`ansible`** = core + ~85 collections. **AWX** = free web UI/API. **AAP** = Red Hat's paid platform.
- Today: ansible-core 2.22 / `ansible` 15. "Ansible 2.9" is the old pre-split release.

### Interview questions

1. **Q: What are the control node and managed nodes?**
   A: The control node is where Ansible is installed and run (Linux/macOS/WSL). Managed nodes are the targets; they need only SSH and Python (or WinRM/PowerShell for Windows). No agent.
2. **Q: Difference between ansible-core and the ansible package?**
   A: ansible-core is the engine plus the `ansible.builtin` modules. The `ansible` package bundles a specific ansible-core with many community collections. Production teams usually pin ansible-core plus only the collections they need.
3. **Q: What's the difference between a task, a play and a playbook?**
   A: A task calls one module. A play maps a set of tasks onto a group of hosts with settings like `become`. A playbook is a YAML file containing one or more plays.

### Scenario question (answer out loud first, then compare)

> *"A colleague says: 'Ansible must install something on the servers, otherwise how does it run Python code there?' Explain what really happens."*

**Model answer:**
1. Ansible connects with normal SSH, using an existing user and key.
2. For each task it copies a self-contained Python program (the module, bundled with its helper code) into a temporary directory on the target.
3. It runs that program with the target's own Python interpreter. The program checks the current state, changes it only if needed, and prints a JSON result.
4. Ansible reads the JSON over SSH and deletes the temp directory.
5. So nothing is **installed** or left running. The requirements are SSH access, Python, and usually sudo for admin tasks.

**Likely follow-ups:**
- *What if the target has no Python?* Use the `raw` module (plain SSH command, no Python) to install Python first (topic 07). Network devices use special connection plugins instead.
- *Isn't copying code for every task slow?* A bit. Pipelining and SSH connection reuse fix most of it (topic 48).
