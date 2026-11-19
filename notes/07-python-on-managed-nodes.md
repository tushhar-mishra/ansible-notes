# 07 · Python on the Managed Node

> **Level:** ★ foundation
>
> **You will learn:** why most Ansible modules need **Python on the target**, which Python versions ansible-core 2.22 supports there (and which common distros are therefore too old), **interpreter discovery** and the warning it prints, how to pin `ansible_python_interpreter`, why some modules need **system Python bindings** (dnf, SELinux) and "respawn", how the **`raw`** module bootstraps hosts with no Python, and which hosts don't need Python at all (Windows, network devices).
>
> **You need:** [06 · Linux recap II](06-linux-recap-admin-tasks.md), [03 · What is Ansible?](03-what-is-ansible.md) (the "module is copied and run with Python" picture).
>
> **Part 0 · Foundations** · Next: [08 · Installing Ansible](08-installing-ansible.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** "Agentless" doesn't mean "needs nothing". Ansible copies each module (a small **Python** program) to the target and runs it there with the **target's own Python**. So Linux targets need Python **3.9 or newer** for ansible-core 2.22. Ansible finds the interpreter by itself (**interpreter discovery**), or you tell it with `ansible_python_interpreter`.

**Picture it.** Ansible sends a recipe card (module) to each kitchen. The card is written in Python, so every kitchen needs a cook who reads Python. If there's no cook, Ansible can still shout plain instructions through the door (`raw`), for example "hire a cook" (install Python).

**The one thing to know.**

```bash
ansible web1 -m ansible.builtin.raw -a "dnf install -y python3"      # no Python needed for raw
ansible web1 -m ansible.builtin.ping -e ansible_python_interpreter=/usr/bin/python3
```

(`-e name=value` sets a variable for this one run. Topic 21 covers variables.)

**Try it now:**

```bash
docker run --rm ubuntu:24.04 sh -c 'command -v python3 || echo "NO PYTHON"'
docker run --rm almalinux:10  sh -c 'command -v python3 || echo "NO PYTHON"'
```

The Ubuntu **container image** has no Python (a full Ubuntu server does). That's the classic surprise.

**You've got it if you can:** explain why a module fails with `/usr/bin/python3: not found`, and how to fix it on 100 hosts with one command.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Where Python runs

| | Control node | Managed node (Linux/Unix) |
|---|---|---|
| Runs | ansible-core itself (`ansible-playbook`, inventory, templating) | Each **module**, one at a time |
| Python needed for ansible-core 2.22 | **3.13 – 3.15** | **3.9 – 3.15** |
| Who installs it | You (topic 08) | Usually already there; else you bootstrap it |

The two Pythons are **independent**. Your control node can run Python 3.13 while a target runs 3.9.

### Which targets are supported today

Supported target Python for ansible-core 2.22: **3.9 to 3.15**. Common systems:

| Target OS | Its system Python | Works with ansible-core 2.22? |
|---|---|---|
| RHEL / AlmaLinux / Rocky **10** | 3.12 | ✅ |
| RHEL / AlmaLinux / Rocky **9** | 3.9 | ✅ |
| RHEL **8** | 3.6 (`/usr/libexec/platform-python`) | ⚠️ No. 3.6 support was dropped in ansible-core 2.17. Install a newer Python *and* see the bindings problem below, or manage these hosts with ansible-core 2.16 |
| Ubuntu **24.04** / **22.04** | 3.12 / 3.10 | ✅ |
| Ubuntu 20.04 | 3.8 | ❌ (3.8 dropped in 2.20) |
| Debian 12 / 13 | 3.11 / 3.13 | ✅ |
| Amazon Linux 2023 | 3.9 | ✅ |
| Amazon Linux 2 | 3.7 | ❌ |

Old targets are a real production problem: big companies often still have RHEL 8. The usual solutions are an older ansible-core for those hosts (pinned in a separate virtual environment or execution environment, topic 52) or upgrading the OS.

### Interpreter discovery

You don't normally tell Ansible where Python is. Before the first module runs on a host, Ansible **discovers** it:

1. It runs a tiny shell script over SSH (no Python needed) that reports the platform (`uname`) and the contents of `/etc/os-release`, plus which of a list of interpreter paths exist.
2. It picks an interpreter, preferring the distribution's standard one (`/usr/bin/python3` on most systems) and falling back through a list (`python3.15`, `python3.14`, ... , `/usr/bin/python3`, `/usr/libexec/platform-python`).
3. It stores the answer as the fact `discovered_interpreter_python` for that host, and uses it for every module.

The setting that controls this is `interpreter_python` in `ansible.cfg` (or the per-host variable `ansible_python_interpreter`):

| Value | Meaning |
|---|---|
| `auto` (default) | Discover; **warn** if the choice could change later (e.g. found a non-standard Python) |
| `auto_silent` | Discover, no warning |
| `auto_legacy`, `auto_legacy_silent` | Old behaviour (prefer `/usr/bin/python`). ⚠️ Legacy |
| `/usr/bin/python3.12` (a path) | **Pinned**: use exactly this, no discovery |

The warning you'll see sometimes:

```
[WARNING]: Platform linux on host web1 is using the discovered Python interpreter at
/usr/bin/python3.12, but future installation of another Python interpreter could change the
meaning of that path.
```

It's harmless information: "I guessed; if you install another Python later, I might guess differently". Silence it by pinning the interpreter for that group (topic 24) or setting `interpreter_python = auto_silent`.

### Bindings: modules that need the *system* Python

Some modules import Python libraries that only exist for the OS's **own** Python:

| Module | Needs | Package (RHEL / Ubuntu) |
|---|---|---|
| `ansible.builtin.dnf` | `dnf` Python bindings | `python3-dnf` (part of the OS) |
| `ansible.builtin.apt` | `apt` Python bindings | `python3-apt` |
| File modules with SELinux contexts | `selinux` bindings | `python3-libselinux` |
| `ansible.posix.seboolean`, `community.general.sefcontext` | SELinux management bindings | `python3-libsemanage`, `python3-policycoreutils` |
| `community.postgresql.*`, `community.mysql.*` | DB drivers | `python3-psycopg2`, `python3-PyMySQL` |

If you point Ansible at a *different* Python (say one you compiled into `/opt/python3.13`), those bindings aren't there. Several core modules (`dnf`, `apt`, `selinux`-aware file operations) handle this with **respawn**: the module notices the bindings are missing, finds the system Python that has them, and **re-launches itself** under that interpreter. That works only if the system Python is itself a supported version. That's exactly why RHEL 8 (system Python 3.6) breaks with new ansible-core even after you install Python 3.12 there.

**Rule:** on Linux targets, use the **OS's standard `python3`**. Don't point Ansible at random virtualenvs on targets.

### The `raw` module: no Python needed

`ansible.builtin.raw` sends a command straight through SSH, like `ssh host 'command'`. No module file, no Python, no JSON. Use it for exactly two things:

1. **Bootstrapping** Python on hosts that don't have it.
2. Devices that **can't** run Python (some appliances, very small systems).

It's **not idempotent** and gives no `changed` information. Never use it for normal work.

Related: `ansible.builtin.script` copies a local script and runs it with no Python on the target either. Same caveats.

### Targets that don't use Python at all

| Target | What runs the modules | Topic |
|---|---|---|
| **Windows** | PowerShell modules (`ansible.windows.*`) over WinRM or SSH. PowerShell 5.1, or 7 since ansible-core 2.21 | 44 |
| **Network devices** (Cisco IOS, Arista EOS, Juniper) | Modules run **on the control node** and send CLI/API commands to the device (`network_cli`, `httpapi`, `netconf` connections) | 44 |
| **Cloud APIs, Kubernetes** | Modules run **on the control node** (`localhost`) and call the API. Python libraries (e.g. `boto3`, `kubernetes`) must be on the **control node** | 43, 44 |

That last row catches people: for `amazon.aws.*` modules you install `boto3` **where Ansible runs**, not on the servers.

---

## Why it exists

Ansible could have used plain shell commands for everything. It chose Python because:

- Shell output is **text**, which differs between distros and versions. Python modules return **structured JSON** with the same keys everywhere.
- Python has real data structures and error handling, so modules can **check state carefully** (idempotency) and support **check mode**.
- Python was already on practically every Linux server (package managers like `dnf` are written in it).

The price is this topic: version support, discovery, bindings, and minimal images without Python.

---

## Analogy

**A travelling consultant with recipe cards.**

- The consultant (control node) mails each restaurant (managed node) a recipe card **written in Python**. The restaurant's own cook (the target's Python) follows it and mails back a report form (JSON).
- The cook must read **modern** Python (3.9+). A cook who only knows 2010-era Python (3.6 on RHEL 8) can't follow today's cards.
- Some recipes need **house equipment** (bindings). The `dnf` recipe needs the restaurant's own oven, so if a visiting chef (an extra Python) is asked, they pass the card back to the house cook (respawn).
- If there's **no cook at all**, the consultant phones the manager and says "hire a cook" (`raw`: `dnf install python3`).
- Some places, like a vending machine (network switch), will never have a cook; the consultant presses its buttons directly from outside (modules run on the control node).

---

## How it works inside

What happens the first time a task runs on a host, around Python:

```
 control node                                         web1
 ────────────                                         ────
 1. SSH: run discovery shell snippet ───────────────▶ uname; cat /etc/os-release;
                                                       command -v python3.15 ... /usr/bin/python3
 2. ◀─────────────────────────────── "Linux", "AlmaLinux 10", "/usr/bin/python3 exists"
 3. choose /usr/bin/python3, remember it as fact discovered_interpreter_python
 4. build module payload (AnsiballZ zip, topic 13) ─▶ ~/.ansible/tmp/.../AnsiballZ_dnf.py
 5. SSH: /usr/bin/python3 .../AnsiballZ_dnf.py ─────▶ module runs
                                                       ├─ import dnf  ✔ (system bindings)
                                                       └─ checks/install, prints JSON
 6. ◀──────────────────────────────────── {"changed": true, "results": [...]}
```

If step 1 finds **no** Python, discovery warns and falls back to `/usr/bin/python3`; step 5 then fails with exit code **127** ("command not found").

With `raw`, steps 1–5 are skipped: it just runs your command over SSH and returns its output.

---

## Code / commands

You'll run these against real lab hosts from topic 09 onwards. For now, read the output shapes.

### See which Python a host uses

```bash
ansible web1 -m ansible.builtin.setup -a 'filter=ansible_python*'
```
```
web1 | SUCCESS => {
    "ansible_facts": {
        "ansible_python": {
            "executable": "/usr/bin/python3",
            "version": { "major": 3, "minor": 12, "micro": 9, ... }
        },
        "ansible_python_version": "3.12.9",
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false
}
```

- `executable`: the interpreter that ran the module.
- `discovered_interpreter_python`: what discovery chose.

### What "no Python" looks like

```bash
ansible ubuntu1 -m ansible.builtin.ping
```
```
[WARNING]: No python interpreters found for host ubuntu1 (tried ['python3.15', 'python3.14',
'python3.13', ..., '/usr/bin/python3', '/usr/libexec/platform-python'])
ubuntu1 | FAILED! => {
    "changed": false,
    "module_stderr": "Shared connection to 172.20.0.12 closed.\r\n",
    "module_stdout": "/bin/sh: 1: /usr/bin/python3: not found\r\n",
    "msg": "The module failed to execute correctly, you probably need to set the interpreter.\nSee stdout/stderr for the exact error",
    "rc": 127
}
```

- `[WARNING]: No python interpreters found`: discovery found nothing, and lists every path it tried.
- `module_stdout: ... python3: not found`: the shell couldn't start Python.
- `rc: 127`: the shell's exit code for "command not found".
- Note: SSH **worked** (this is `FAILED`, not `UNREACHABLE`). The problem is Python, not the connection.

### Bootstrap Python with `raw`

```bash
ansible ubuntu1 -m ansible.builtin.raw -a "apt-get update -qq && apt-get install -y -qq python3" --become
```
```
ubuntu1 | CHANGED | rc=0 >>
Selecting previously unselected package python3.12-minimal.
...
Setting up python3 (3.12.3-0ubuntu2) ...
Shared connection to 172.20.0.12 closed.
```

- `CHANGED | rc=0`: `raw` always reports `changed` when the command succeeds. It can't know better.
- `--become`: run it through sudo (topic 19).

The same in a playbook. Facts must be **off** for this play, because gathering facts itself needs Python:

```yaml
- name: Bootstrap Python where it's missing
  hosts: all
  gather_facts: false            # fact gathering runs a Python module: would fail
  become: true
  tasks:
    - name: Python 3 is installed (Debian/Ubuntu)
      ansible.builtin.raw: test -e /usr/bin/python3 || (apt-get update -qq && apt-get install -y -qq python3)
      register: bootstrap
      changed_when: "'Setting up python3' in bootstrap.stdout"
```

- `test -e /usr/bin/python3 || (...)`: install only if missing. A simple way to make `raw` idempotent.
- `register` + `changed_when`: report `changed` only when something was installed (topics 26–27 explain these).

### Pin the interpreter

In an inventory (topic 10):

```ini
[legacy]
old1 ansible_host=10.0.3.5 ansible_python_interpreter=/usr/bin/python3.11
```

Or for all hosts, in `ansible.cfg` (topic 11):

```ini
[defaults]
interpreter_python = auto_silent
```

---

## Hands-on lab

Your SSH lab nodes arrive in topic 09. This lab uses `localhost` inside a container to *see* interpreter selection and `raw`.

1. Start a container with ansible-core:

   ```bash
   docker run --rm -it python:3.13-slim bash
   pip install -q ansible-core
   ```

2. See discovery's result:

   ```bash
   ansible localhost -m ansible.builtin.setup -a 'filter=*python*'
   ```

   Note `executable`. (For `localhost`, Ansible uses the Python it's running on.)

3. Point it at a Python that doesn't exist and read the error:

   ```bash
   ansible localhost -m ansible.builtin.ping -e ansible_python_interpreter=/usr/bin/python9
   ```

   Find `rc` and `module_stdout` in the output. Is it `FAILED` or `UNREACHABLE`? (FAILED: the "connection" worked.)

4. Now `raw`, with the same broken interpreter:

   ```bash
   ansible localhost -m ansible.builtin.raw -a 'echo hello from raw; uname -s' -e ansible_python_interpreter=/usr/bin/python9
   ```

   It works: `raw` never uses Python.

5. Check the target-Python problem in images you'll meet:

   ```bash
   exit
   for img in ubuntu:24.04 debian:13 almalinux:10 almalinux:9 alpine:3.22; do
     printf '%-14s ' "$img"; docker run --rm "$img" sh -c 'python3 --version 2>/dev/null || echo "no python3"'
   done
   ```

   Expect: Ubuntu, Debian and Alpine images have **no** python3; the AlmaLinux images do (because `dnf` needs it). Topic 09's Dockerfile installs Python for this reason.

---

## Where you'll use it

- **New cloud images** or minimal installs: a bootstrap play with `raw` (or Python installed by cloud-init) before anything else.
- **Mixed fleets** with old systems: pin interpreters per group, or keep an older ansible-core for legacy hosts.
- **Upgrading ansible-core** in CI: check the target support matrix first, or old hosts suddenly fail (topic 63).
- **Cloud and Kubernetes automation**: install `boto3` / `kubernetes` libraries on the **control node** (or in the execution environment).
- **Containers as targets** (Molecule tests, topic 51): images often need Python added first.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `/usr/bin/python3: not found`, `rc: 127`, task `FAILED` | No Python on the target | Bootstrap with `raw` (+ `gather_facts: false`) or cloud-init |
| `[WARNING]: No python interpreters found for host` | Same | Same |
| `SyntaxError` / `ansible-core requires a minimum of Python version 3.9` on the target | Target Python too old (RHEL 8's 3.6, Ubuntu 20.04's 3.8) | Upgrade the OS, or use an older ansible-core for those hosts |
| `Could not import the dnf python module using /opt/py/bin/python3` | Interpreter without system bindings, and respawn couldn't find a usable system Python | Use the OS's `/usr/bin/python3` |
| `Aborting, target uses selinux but python bindings (libselinux-python) aren't installed!` | Missing `python3-libselinux` | Install it (it's normally present on RHEL) |
| `Failed to import the required Python library (boto3)` | Library missing **on the control node** (cloud modules run locally) | `pip install boto3` in Ansible's environment |
| Discovery warning on every run | Non-standard Python found | Pin `ansible_python_interpreter`, or `interpreter_python = auto_silent` |
| `raw` task says `changed` every run | `raw` can't detect changes | Guard with `test -e ... ||` and `changed_when` |

---

## Quick recap

- Linux targets run modules with **their own Python**; ansible-core 2.22 needs **3.9–3.15** there and **3.13–3.15** on the control node.
- **Interpreter discovery** picks the OS's standard Python and stores it as `discovered_interpreter_python`. Pin with `ansible_python_interpreter`.
- Some modules need **system bindings** (`dnf`, `apt`, SELinux) and respawn under the system Python: use the OS's `/usr/bin/python3`.
- **`raw`** needs no Python. Use it only to bootstrap (with `gather_facts: false`).
- Windows uses PowerShell; network and cloud modules run **on the control node**, so their libraries go there.

### Interview questions

1. **Q: Does Ansible need anything on managed nodes?**
   A: SSH access and Python (3.9+ for current ansible-core) on Linux targets; WinRM/SSH and PowerShell on Windows. No agent. Network devices need nothing, since modules run on the control node.
2. **Q: How do you manage a host that has no Python?**
   A: Use the `raw` module (with `gather_facts: false`) to install Python, then continue normally. Or install it with cloud-init at first boot.
3. **Q: What is `ansible_python_interpreter`?**
   A: A per-host variable that tells Ansible exactly which Python to use on the target, bypassing interpreter discovery.

### Scenario question (answer out loud first, then compare)

> *"You upgraded ansible-core in CI from 2.16 to 2.22. Now every task fails on your RHEL 8 hosts, but works on RHEL 9. What's going on and what are your options?"*

**Model answer:**
1. RHEL 8's system Python is 3.6. Target support for 3.6 ended in ansible-core 2.17, so modules no longer run there.
2. Installing Python 3.12 on RHEL 8 and pinning `ansible_python_interpreter` fixes simple modules, but `dnf` and SELinux operations need the system Python bindings (3.6 only), so they still fail.
3. Options:
   - keep a **second, older** toolchain (ansible-core 2.16) for the RHEL 8 group, in its own virtualenv or execution environment, run as a separate pipeline job;
   - prioritise **upgrading those hosts** to RHEL 9/10;
   - only as a stop-gap: pin CI back to 2.16 for everything (you lose fixes and security updates; 2.16 is out of support).
4. Prevent: check the support matrix in the porting guide before upgrading, and test upgrades against every OS in your fleet first (Molecule, topic 51).

**Likely follow-ups:**
- *Why not just use `raw`/`shell` for RHEL 8?* You'd lose idempotency, check mode and structured results everywhere. It doesn't scale.
- *Where would you record which toolchain manages which hosts?* In the repo: separate inventories/pipelines, and a pinned `requirements.txt` per toolchain.
