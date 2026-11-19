# 08 · Installing Ansible

> **Level:** ★ foundation
>
> **You will learn:** the ways to install Ansible (distro packages, `pip` in a **virtual environment**, `pipx`/`uv tool`), why production teams **pin** versions in a `requirements.txt`, how to get the **Python 3.13** that ansible-core 2.22 needs on WSL2 Ubuntu 24.04 (which ships 3.12) using **uv**, how to add collections, what every line of `ansible --version` means, and where Ansible keeps its files.
>
> **You need:** [07 · Python on the managed node](07-python-on-managed-nodes.md). Your WSL2 Ubuntu with Docker Engine from [Docker topic 04](../../Docker/notes/04-installing-docker.md).
>
> **Part 1 · Setup and first contact** · Next: [09 · Lab setup](09-lab-setup.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Ansible is a Python program. Install it into its **own isolated Python environment** (a *virtual environment*, or "venv") so it never fights with the system's Python packages, and so you can pin an exact version.

**Picture it.** A venv is a **separate toolbox** just for Ansible. The system toolbox stays untouched; you can throw Ansible's box away and rebuild it any time.

**The one thing to know.** On WSL2 Ubuntu 24.04:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh        # uv: fast Python/venv manager
source ~/.local/bin/env                               # put uv on PATH (new shells do this automatically)
uv venv --python 3.13 ~/.venvs/ansible                # venv with its own Python 3.13
source ~/.venvs/ansible/bin/activate                  # use it in this shell
uv pip install "ansible-core==2.22.*"                 # install Ansible
ansible --version                                     # first line: ansible [core 2.22.x]
```

**Try it now:** run the block above, then `ansible localhost -m ansible.builtin.ping`.

**You've got it if you can:** explain why you install Ansible in a venv instead of with `sudo pip`, and read the first 4 lines of `ansible --version`.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### The install options

| Method | Command (example) | Version you get | Use it for |
|---|---|---|---|
| **Distro package** | `sudo apt install ansible` / `sudo dnf install ansible-core` | Whatever the distro froze (Ubuntu 24.04: ansible-core **2.16**; RHEL: Red Hat's supported build) | Quick start; RHEL shops with Red Hat support |
| **venv + pip** (or `uv pip`) | `uv pip install "ansible-core==2.22.*"` | Any version you choose | ✅ **Projects and CI**: pinned, reproducible |
| **pipx / `uv tool`** | `uv tool install --python 3.13 ansible-core` | Latest (or pinned) | Personal "one global Ansible" on your laptop |
| **Execution environment** (container image) | `ansible-navigator run ...` | Whatever the image contains | Teams, AWX/AAP, "works the same everywhere" (topic 52) |
| ❌ `sudo pip install ansible` | | | **Never**: mixes with system Python packages, can break `apt`/`dnf`. Modern distros refuse it (`externally-managed-environment`) |

### Words used here

- **pip**: Python's package installer. Downloads packages from **PyPI** (the Python Package Index, pypi.org).
- **Virtual environment (venv)**: a folder with its own Python interpreter link and its own installed packages. **Activating** it (`source .../bin/activate`) puts its `bin/` first on your `PATH`, so `ansible` means *this* Ansible.
- **uv**: a fast tool (from the company Astral) that creates venvs, installs packages, and can **download a Python version** your OS doesn't have. That's why we use it here: Ubuntu 24.04 has Python 3.12, and ansible-core 2.22 needs **3.13+** on the control node.
- **pipx / `uv tool`**: install a Python *application* in its own hidden venv and expose its commands globally.

### ansible-core or the `ansible` package?

| | `ansible-core` | `ansible` |
|---|---|---|
| Size | Small (~a few MB) | Large (hundreds of MB, ~85 collections) |
| Collections | Only `ansible.builtin` | Many, frozen at a tested version set |
| You add collections | Yes, with `ansible-galaxy` + `requirements.yml` (topic 37) | Rarely |
| Best for | Production, CI, execution environments | Learning, "try everything" |

These notes use **ansible-core + the collections we need**: `ansible.posix`, `community.general`, and later `community.docker`, `amazon.aws`, `kubernetes.core`. That's what you'll do at work.

### Pinning for production

In a real project, the repo contains:

```
requirements.txt          # Python packages: exact versions
collections/requirements.yml   # Ansible collections: exact versions (topic 37)
```

```text
# requirements.txt
ansible-core==2.22.0
ansible-lint==25.11.0     # (topic 50; example version)
```

Everyone and every pipeline runs `uv pip install -r requirements.txt`, so the whole team uses **identical** versions. "It works on my laptop" problems go away.

---

## Why it exists

Ansible versions matter more than people expect:

- Behaviour changes between releases (ansible-core 2.19 made conditionals stricter; 2.20 deprecated injected fact variables).
- Target Python support changes (topic 07).
- Collections require minimum ansible-core versions.

If Ansible is installed "however" on each machine, the same playbook behaves differently on your laptop, your colleague's laptop and CI. A venv + pinned requirements makes the toolchain part of the code.

---

## Analogy

**Power tools in a workshop.** The **system Python** is the building's electrical wiring: you don't rewire it to plug in a drill. A **venv** is a **toolbox with its own battery pack**: everything Ansible needs is inside, at the versions you chose. `requirements.txt` is the **packing list** taped inside the lid, so anyone can pack an identical toolbox. `sudo pip install` is like soldering your drill straight into the building's wiring.

---

## How it works inside

What happens when you type `ansible` after activating the venv:

```
 $ ansible --version
     │
     ▼  your shell searches $PATH from left to right
 ~/.venvs/ansible/bin/ansible     ← found first because activate prepended ~/.venvs/ansible/bin
     │  (a small script; its first line points to the venv's python)
     ▼
 ~/.venvs/ansible/bin/python  →  Python 3.13 (downloaded by uv into ~/.local/share/uv/python/)
     │
     ▼  imports the ansible package from
 ~/.venvs/ansible/lib/python3.13/site-packages/ansible/
```

Where Ansible keeps things on the control node:

| Path | What's there |
|---|---|
| `~/.venvs/ansible/` | Your Ansible install (this topic) |
| `~/.ansible/collections/ansible_collections/` | Collections you install with `ansible-galaxy` (default location) |
| `~/.ansible/tmp/` | Local temp files for module runs |
| `~/.ansible/cp/` | SSH ControlPersist sockets (topic 05) |
| `~/.ansible/galaxy_cache/` | Cache for Galaxy downloads |
| `./ansible.cfg` | Settings for the project you're in (topic 11) |

---

## Code / commands

### Install uv and create the venv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.local/bin/env
uv --version
```
```
uv 0.9.x
```

- The installer puts `uv` in `~/.local/bin` and adds it to your shell startup files.
- Piping a script from the internet into `sh` trusts that website. If your company forbids it: `pipx install uv` or download the release from GitHub and check its checksum.

```bash
uv venv --python 3.13 ~/.venvs/ansible
```
```
Using CPython 3.13.9
Creating virtual environment at: /home/you/.venvs/ansible
Activate with: source /home/you/.venvs/ansible/bin/activate
```

- `--python 3.13`: if no 3.13 exists on the system, uv **downloads** a standalone build. Your system Python 3.12 is untouched.

### Install Ansible into it

```bash
source ~/.venvs/ansible/bin/activate
uv pip install "ansible-core==2.22.*"
```
```
Resolved 9 packages in 412ms
Installed 9 packages in 95ms
 + ansible-core==2.22.0
 + cffi==2.0.0
 + cryptography==46.0.3
 + jinja2==3.1.6
 + markupsafe==3.0.3
 + packaging==25.0
 + pycparser==2.23
 + pyyaml==6.0.3
 + resolvelib==1.2.1
```

- `"ansible-core==2.22.*"`: any 2.22 patch release (bug fixes only). Quote it so the shell doesn't treat `*` as a wildcard.
- The dependencies tell you what Ansible is made of: **Jinja2** (templating, topic 22), **PyYAML** (reading YAML, topic 04), **cryptography** (Vault, topic 40), **resolvelib** (dependency solving for `ansible-galaxy`).

Make it automatic in new terminals:

```bash
echo 'source ~/.venvs/ansible/bin/activate' >> ~/.bashrc
```

### Read `ansible --version`

```bash
ansible --version
```
```
ansible [core 2.22.0]
  config file = None
  configured module search path = ['/home/you/.ansible/plugins/modules', '/usr/share/ansible/plugins/modules']
  ansible python module location = /home/you/.venvs/ansible/lib/python3.13/site-packages/ansible
  ansible collection location = /home/you/.ansible/collections:/usr/share/ansible/collections
  executable location = /home/you/.venvs/ansible/bin/ansible
  python version = 3.13.9 (main, Oct 14 2026, ...) [Clang 20.1.4 ] (/home/you/.venvs/ansible/bin/python3)
  jinja version = 3.1.6
  pyyaml version = 6.0.3 (with libyaml v0.2.5)
```

| Line | Meaning | Why you care |
|---|---|---|
| `ansible [core 2.22.0]` | ansible-core version | The number every bug report and porting guide refers to |
| `config file = None` | Which `ansible.cfg` is in effect (topic 11) | "Why is my setting ignored?" starts here |
| `configured module search path` | Extra places Ansible looks for your **own** modules | Custom modules (topic 38) |
| `ansible python module location` | Where the Ansible code lives | Proves which install you're using (venv vs system) |
| `ansible collection location` | Where collections are searched, in order | "couldn't resolve module" problems (topic 37) |
| `executable location` | The `ansible` binary that ran | Two installs fighting? This line tells you |
| `python version` | Python running **the control node side** | Must be 3.13+ for 2.22 |
| `jinja version` / `pyyaml version` | Templating and YAML library versions | `with libyaml` = the fast C parser is used |

### Add the collections these notes use

```bash
ansible-galaxy collection install ansible.posix community.general
```
```
Starting galaxy collection install process
Process install dependency map
Starting collection install process
Downloading https://galaxy.ansible.com/api/v3/plugin/ansible/content/published/collections/artifacts/ansible-posix-2.1.0.tar.gz to /home/you/.ansible/tmp/...
Installing 'ansible.posix:2.1.0' to '/home/you/.ansible/collections/ansible_collections/ansible/posix'
ansible.posix:2.1.0 was installed successfully
...
community.general:12.0.1 was installed successfully
```

```bash
ansible-galaxy collection list
```
```
# /home/you/.ansible/collections/ansible_collections
Collection        Version
----------------- -------
ansible.posix     2.1.0
community.general 12.0.1
```

- `ansible.builtin` isn't listed: it's part of ansible-core itself.
- Versions shown are examples. Topic 37 shows how to pin them in `requirements.yml`.

### Shell completion (optional, nice)

```bash
uv pip install argcomplete
activate-global-python-argcomplete --user
```

Tab-completes options like `ansible-playbook --li<TAB>`.

### The alternative: one global install with `uv tool`

```bash
uv tool install --python 3.13 ansible-core
ansible --version
```

Good for a personal laptop. For project work, prefer a venv per project with a `requirements.txt`.

### On RHEL as a control node

```bash
sudo dnf install ansible-core          # Red Hat's supported build (version depends on the RHEL minor release)
```

RHEL's `ansible-core` package is supported by Red Hat and is what RHEL System Roles are tested with. It's usually a bit older than upstream. Same choice as on Ubuntu: distro package for "supported", venv for "exact version I chose".

---

## Hands-on lab

1. In WSL2 Ubuntu, install uv, create `~/.venvs/ansible` with Python 3.13, activate it, install `ansible-core==2.22.*` (Core section).

2. Prove which Ansible and which Python you're using:

   ```bash
   command -v ansible
   ansible --version | head -3
   python3 --version              # inside the venv: 3.13.x
   /usr/bin/python3 --version     # system: 3.12.x (untouched)
   ```

3. Deactivate and see the difference:

   ```bash
   deactivate
   command -v ansible || echo "not on PATH outside the venv"
   source ~/.venvs/ansible/bin/activate
   ```

4. Install the two collections and list them (commands above).

5. Write a project `requirements.txt` and rebuild a second venv from it, to feel how CI does it:

   ```bash
   mkdir -p ~/ansible-lab && cd ~/ansible-lab
   printf 'ansible-core==2.22.0\n' > requirements.txt
   uv venv --python 3.13 /tmp/ci-venv
   uv pip install --python /tmp/ci-venv -r requirements.txt
   /tmp/ci-venv/bin/ansible --version | head -1
   rm -rf /tmp/ci-venv
   ```

6. First run against your own machine:

   ```bash
   ansible localhost -m ansible.builtin.setup -a 'filter=ansible_distribution*'
   ```

   You should see `Ubuntu` and `24.04`.

**Done when:** `ansible --version` shows `core 2.22.x` with `python version = 3.13.x`, and `ansible-galaxy collection list` shows `ansible.posix` and `community.general`.

---

## Where you'll use it

- **Every laptop and CI runner** that runs Ansible: same `requirements.txt` everywhere.
- **CI pipelines** (topics 53–54): a job step creates the venv and installs pinned versions, or uses an execution environment image.
- **Upgrades** (topic 63): change one line in `requirements.txt`, test, merge.
- **Debugging**: `ansible --version` is the first thing to paste in any bug report or team chat.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `ansible: command not found` | venv not activated in this shell | `source ~/.venvs/ansible/bin/activate` (or add it to `~/.bashrc`) |
| `error: externally-managed-environment` | `pip install` into the **system** Python (blocked by Ubuntu 24.04/RHEL, PEP 668) | Use a venv; never `--break-system-packages` |
| `ERROR: Could not find a version that satisfies the requirement ansible-core==2.22...` / `requires Python >=3.13` | venv uses Python 3.12 | Recreate with `uv venv --python 3.13` |
| `ansible --version` shows `core 2.16.x` | The **apt** package is first on `PATH` | `sudo apt remove ansible ansible-core` or fix `PATH`; check `executable location` |
| Old version in CI, new on laptop | Unpinned `pip install ansible-core` | Pin in `requirements.txt` |
| `couldn't resolve module/action 'ansible.posix.firewalld'` | Collection not installed for this user/venv | `ansible-galaxy collection install ansible.posix`; check `ansible collection location` |
| `ERROR! Unexpected Exception ... No module named 'boto3'` | A cloud module's Python library missing from **Ansible's venv** | `uv pip install boto3` in the venv |
| `locale` error: `Ansible requires the locale encoding to be UTF-8` | Minimal environment without a UTF-8 locale | `export LC_ALL=C.UTF-8` |

---

## Quick recap

- Install Ansible in a **venv**, never with `sudo pip`. Activate it to use it.
- ansible-core 2.22 needs **Python 3.13+**; on Ubuntu 24.04 use **uv** to get it without touching the system Python.
- Prefer **ansible-core + chosen collections**, pinned in `requirements.txt` and `requirements.yml`.
- `ansible --version` tells you the core version, config file, collection paths, executable and Python: read it first when debugging.
- Collections go to `~/.ansible/collections` by default.

### Interview questions

1. **Q: How do you install Ansible for a team project?**
   A: In a Python virtual environment, with exact versions of ansible-core (and tools like ansible-lint) pinned in `requirements.txt`, and collections pinned in `requirements.yml`. CI uses the same files, or an execution environment image built from them.
2. **Q: What's the difference between `ansible` and `ansible-core` on PyPI?**
   A: `ansible-core` is the engine plus `ansible.builtin`. `ansible` is a bundle of a specific ansible-core plus ~85 community collections.
3. **Q: How do you find which Ansible config file is being used?**
   A: `ansible --version` (the `config file` line) or `ansible-config dump --only-changed`.

### Scenario question (answer out loud first, then compare)

> *"A playbook works on your laptop but fails in the Jenkins pipeline with `conditional result was not a boolean`. What's your first move?"*

**Model answer:**
1. Compare versions: run `ansible --version` in both places. That error appears with ansible-core 2.19+, which made conditionals strict. The laptop likely has an older core, or CI has a newer one.
2. Pin the toolchain: commit `requirements.txt` with the exact ansible-core version, and make both the laptop and CI install from it.
3. Fix the playbook itself to be compatible with the newer version (make the `when:` expression return a real boolean), because older versions will go out of support.
4. Prevent it: CI runs `ansible-lint` and a check-mode run on every PR with the pinned versions.

**Likely follow-ups:**
- *How would you upgrade Ansible safely later?* Change the pin on a branch, run lint + Molecule + a check-mode run against staging, read the porting guide, then merge (topic 63).
- *Why not use the distro package in CI?* Its version depends on the runner image; you lose control of upgrades.
