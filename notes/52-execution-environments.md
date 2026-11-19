# 52 · Execution Environments and ansible-navigator

> **Level:** ★★★ advanced
>
> **You will learn:** why "it works on my laptop" happens with Ansible (different ansible-core, collections, Python libraries and system tools on each machine), what an **execution environment (EE)** is (a container image holding the whole Ansible toolchain), how to build one with **ansible-builder** (`execution-environment.yml` v3: base image, ansible-core, ansible-runner, collections, Python and system dependencies via `bindep.txt`), how to run playbooks inside it with **ansible-navigator** (modes, settings file, network and SSH key access, artifacts and replay), and how EEs are used in CI and AWX/AAP. You'll build an EE for `~/infra` and run your site playbook in it.
>
> **You need:** [51 · Molecule](51-testing-with-molecule.md), [37 · Collections](37-collections-and-galaxy.md), [08 · Installing Ansible](08-installing-ansible.md), Docker images and Dockerfiles ([Docker topics 06](../../Docker/notes/06-images-tags-and-registries.md), [14](../../Docker/notes/14-dockerfile-basics.md), [22](../../Docker/notes/22-tagging-pushing-registries.md)).
>
> **Part 8 · Quality and testing** · Next: [53 · Ansible in GitHub Actions](53-ansible-in-github-actions.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Your playbooks depend on a **toolchain**: an ansible-core version, collections, Python libraries (`boto3`, `kubernetes`, `hvac`, `jmespath`...) and system tools (`ssh`, `git`, `rsync`). An **execution environment** puts that whole toolchain into **one container image** with a version tag. Everyone (laptops, CI, AWX/AAP) runs playbooks **inside the same image**, so results are identical.

**Picture it.** A **lunchbox** packed in the morning with exactly what you need. Wherever you eat (laptop, CI, AWX), the lunch is the same, instead of hoping each canteen has the right ingredients.

**The one thing to know.**

```yaml
# execution-environment.yml
version: 3
images:
  base_image:
    name: quay.io/fedora/fedora:42
dependencies:
  python_interpreter: { package_system: python3, python_path: /usr/bin/python3 }
  ansible_core:   { package_pip: ansible-core==2.22.0 }
  ansible_runner: { package_pip: ansible-runner }
  galaxy: collections/requirements.yml
  python: requirements-ee.txt
  system: bindep.txt
```

```bash
ansible-builder build -t infra-ee:1.0 --container-runtime docker
ansible-navigator run playbooks/site.yml -i inventories/lab --eei infra-ee:1.0 --mode stdout
```

**Try it now:** follow the Hands-on lab to build `infra-ee:1.0` and run `site.yml` through it.

**You've got it if you can:** list the four kinds of dependencies an EE contains and explain why AWX/AAP need EEs.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### What's inside an EE

| Layer | Examples | Defined in |
|---|---|---|
| Base OS + Python | Fedora/RHEL UBI with Python 3.13 | `images.base_image` + `python_interpreter` |
| **ansible-core** | `ansible-core==2.22.0` | `dependencies.ansible_core` |
| **ansible-runner** | The library AWX/navigator use to run playbooks and collect events | `dependencies.ansible_runner` |
| **Collections** | `ansible.posix`, `community.general`, `amazon.aws`... (pinned) | `dependencies.galaxy` → `requirements.yml` |
| **Python libraries** | `boto3`, `kubernetes`, `hvac`, `jmespath`, `netaddr`, `pywinrm` | `dependencies.python` → `requirements.txt` |
| **System packages** | `openssh-clients`, `git-core`, `rsync`, `sshpass`, `krb5-workstation` | `dependencies.system` → `bindep.txt` |

Collections often **declare** their own Python/system needs (in their `requirements.txt`/`bindep.txt`); ansible-builder collects and installs those too.

### ansible-builder

Turns `execution-environment.yml` into a **Containerfile** (Dockerfile) plus a build context, then builds the image with Docker or Podman.

| Command | Does |
|---|---|
| `ansible-builder create` | Only generate `context/Containerfile` (inspect or build elsewhere) |
| `ansible-builder build -t name:tag` | Generate and build |
| `--container-runtime docker` | Use Docker instead of Podman |
| `-v 3` | Verbose output |

Key sections of the v3 file:

| Section | Purpose |
|---|---|
| `images.base_image.name` | Starting image. Community: `quay.io/fedora/fedora:42`, `quay.io/centos/centos:stream10`. Red Hat: `registry.redhat.io/ansible-automation-platform-2x/ee-minimal-rhel9` (subscription) |
| `dependencies` | As in the table above (`python_interpreter`, `ansible_core`, `ansible_runner`, `galaxy`, `python`, `system`) |
| `options.package_manager_path` | `/usr/bin/dnf` (or `microdnf` on minimal images) |
| `options.tags` | Default tags for the image |
| `additional_build_steps` | Extra Containerfile lines `prepend_base`, `append_final`, ... (custom CA certs, extra tools) |
| `additional_build_files` | Files copied into the build context (certs, configs) |

### bindep.txt

One system package per line, with **platform selectors**:

```
openssh-clients [platform:rpm]
openssh-client [platform:dpkg]
git-core [platform:rpm]
rsync
sshpass [platform:rpm]
```

### ansible-navigator

A command-line tool (with an optional text UI) that runs Ansible **inside an EE**:

| Command | Like |
|---|---|
| `ansible-navigator run site.yml -i inv` | `ansible-playbook` |
| `ansible-navigator inventory -i inv --graph` (or TUI) | `ansible-inventory` |
| `ansible-navigator doc ansible.posix.firewalld` | `ansible-doc` |
| `ansible-navigator collections` | `ansible-galaxy collection list` (what's **in the image**) |
| `ansible-navigator images` | Inspect EE images (versions, collections, Python packages) |
| `ansible-navigator config` | `ansible-config` (inside the EE) |
| `ansible-navigator replay artifact.json` | Re-open a past run's results |
| `ansible-navigator exec -- <cmd>` | Run any command inside the EE |

Important options:

| Option / setting | Meaning |
|---|---|
| `--eei` / `--execution-environment-image` | Which EE image |
| `--mode stdout` / `--mode interactive` (`-m`) | Plain output (like ansible-playbook, good for CI) vs the TUI |
| `--pp never` / `--pull-policy` | Don't pull the image (for local builds) |
| `--ce docker` / `--container-engine` | Docker or Podman |
| `--ee false` | Run **without** an EE (local ansible) |
| `--co '--net=host'` / `--container-options` | Extra `docker run` options (e.g. host networking to reach lab IPs) |
| `--penv VAR` / `--senv VAR=value` | Pass / set environment variables into the EE |
| `--lf`, `--pae`, `--pas` | Log file, playbook artifact enable, artifact save path |

Settings live in **`ansible-navigator.yml`** in the project (committed), so you don't type them every time.

What navigator does automatically: mounts the **current directory** into the EE (your playbooks, inventories, `ansible.cfg`), mounts **`~/.ssh`** and forwards the **SSH agent** socket, and passes your vault password file if configured.

### Where EEs are used

| Place | How |
|---|---|
| Laptops | `ansible-navigator run ...` |
| CI (GitHub Actions, Jenkins, GitLab) | `container: infra-ee:1.0` job image, or navigator, or `docker run ... ansible-playbook` (topics 53–54) |
| **AWX / AAP** | **Every job runs in an EE** chosen per job template/organization (topic 56) |
| Molecule | Can run converge inside an EE (advanced) |

### Default and published EEs

- `ghcr.io/ansible/community-ansible-dev-tools`: community image with the dev tools (navigator's default in recent versions).
- `quay.io/ansible/awx-ee`: AWX's default EE.
- Red Hat ships supported EEs (`ee-minimal`, `ee-supported`) for AAP.

For production, **build your own** from your pinned requirements, tag it with a version, push it to your registry, and scan it like any container image.

---

## Why it exists

Ansible itself is just Python, but real playbooks need: the right ansible-core (2.19 changed templating!), the right collection versions, `boto3` of a compatible version, `kubectl`-free Kubernetes libraries, `sshpass` for that one legacy host, Kerberos for Windows... Installing all that consistently on every laptop and every CI runner is fragile. And AWX/AAP need to run jobs for many teams with different dependencies on the same cluster. Container images solve both: **immutable, versioned, identical everywhere**, and each team can have its own.

---

## Analogy

**A field engineer's sealed toolkit.** Instead of every engineer collecting their own tools (some with an old screwdriver, some missing a part), the company issues **numbered toolkits** (EE images `v1.0`, `v1.1`) packed from one checklist (`execution-environment.yml`). Any engineer, any site (laptop, CI, AWX), opens toolkit v1.1 and has exactly the same tools. Upgrading = issuing toolkit v1.2 after testing it.

---

## How it works inside

```
 BUILD (ansible-builder)
 execution-environment.yml ─▶ context/Containerfile (multi-stage):
     base   : base image + python + ansible-core + ansible-runner
     galaxy : ansible-galaxy collection install -r requirements.yml
     builder: collect python/system deps from you + every collection → wheels
     final  : base + collections + installed deps
 ─▶ docker build ─▶ image infra-ee:1.0

 RUN (ansible-navigator run site.yml)
 docker run --rm -v $PWD:$PWD -w $PWD -v ~/.ssh:/home/runner/.ssh -v $SSH_AUTH_SOCK:... [--net=host] \
            infra-ee:1.0  ansible-runner ... ansible-playbook site.yml ...
     │  inside the container: ansible-core 2.22 + pinned collections + libs
     ├─ SSH from the container to the managed nodes (same as before)
     └─ events streamed back → navigator prints them / saves an artifact JSON
```

---

## Code / commands

### Install the tools

```bash
source ~/.venvs/ansible/bin/activate
uv pip install ansible-builder ansible-navigator
ansible-builder --version
ansible-navigator --version
```

### Define the EE (in `~/infra`)

`execution-environment.yml`:

```yaml
---
version: 3

images:
  base_image:
    name: quay.io/fedora/fedora:42

dependencies:
  python_interpreter:
    package_system: python3
    python_path: /usr/bin/python3
  ansible_core:
    package_pip: ansible-core==2.22.0
  ansible_runner:
    package_pip: ansible-runner
  galaxy: collections/requirements.yml
  python: requirements-ee.txt
  system: bindep.txt

options:
  package_manager_path: /usr/bin/dnf
  tags:
    - infra-ee:1.0
```

`requirements-ee.txt`:

```text
jmespath
netaddr
hvac
docker
kubernetes
```

`bindep.txt`:

```text
openssh-clients [platform:rpm]
git-core [platform:rpm]
rsync [platform:rpm]
```

(Fedora 42's Python is 3.13, which ansible-core 2.22 requires on the control node, topic 08.)

### Build it

```bash
cd ~/infra
ansible-builder build --container-runtime docker -v 3
```
```
Running command:
  docker build -f context/Containerfile -t infra-ee:1.0 context
...
#18 [final 4/6] RUN /output/scripts/install-from-bindep && rm -rf /output/wheels
...
Complete! The build context can be found at: /home/you/infra/context
```

```bash
docker images infra-ee
ls context/                     # Containerfile + _build/ (requirements, scripts)
```

Add `context/` to `.gitignore` (it's generated).

### Inspect what's inside

```bash
ansible-navigator images --eei infra-ee:1.0 --mode stdout --pp never --ce docker
docker run --rm infra-ee:1.0 ansible --version | head -1
docker run --rm infra-ee:1.0 ansible-galaxy collection list
```
```
ansible [core 2.22.0]
# /usr/share/ansible/collections/ansible_collections
Collection        Version
----------------- -------
ansible.posix     2.1.0
community.docker  5.0.0
community.general 12.0.1
...
```

### Navigator settings for the project

`~/infra/ansible-navigator.yml`:

```yaml
---
ansible-navigator:
  execution-environment:
    enabled: true
    image: infra-ee:1.0
    pull:
      policy: never
    container-engine: docker
    container-options:
      - "--net=host"                       # reach the lab's 172.30.0.x network from the container
  mode: stdout
  playbook-artifact:
    enable: true
    save-as: "{playbook_dir}/../artifacts/{playbook_name}-{time_stamp}.json"   # navigator's own {placeholders}, not Jinja2
  logging:
    level: warning
    file: /tmp/ansible-navigator.log
```

- `--net=host`: the EE container uses the WSL2 host's network, so it can reach the lab containers (otherwise Docker isolates networks from each other).
- Artifacts: a JSON file per run with every event, re-openable with `ansible-navigator replay`.

### Run the site playbook in the EE

```bash
cd ~/infra
mkdir -p artifacts
ansible-navigator run playbooks/site.yml -i inventories/lab
```
```
PLAY [Web tier] ************************************************************************
TASK [Gathering Facts] *****************************************************************
ok: [web1]
...
PLAY RECAP *****************************************************************************
web1   : ok=14  changed=0  ...
```

Same output as `ansible-playbook`, but every module, collection and library came from `infra-ee:1.0`.

Replay later:

```bash
ansible-navigator replay artifacts/site-*.json --mode interactive
```

(Interactive mode: browse plays → tasks → per-host results with arrow keys and numbers; `:q` or Esc to go back/quit.)

### Version and publish

```bash
docker tag infra-ee:1.0 registry.example.com/platform/infra-ee:1.0
docker push registry.example.com/platform/infra-ee:1.0
```

Bump the tag when **any** requirement changes (`requirements.yml`, `requirements-ee.txt`, `bindep.txt`, ansible-core). CI and AWX reference the exact tag.

---

## Hands-on lab

1. Write the three files, build `infra-ee:1.0`, and list its collections and Python packages.
2. Add `ansible-navigator.yml` and run `site.yml` through the EE. Confirm the recap matches a normal run.
3. Run `ansible-navigator run playbooks/site.yml -i inventories/lab --ee false`: same run without the container (uses your venv). Compare `ansible-navigator config` output in both modes.
4. Prove the toolchain is in the image: deactivate your venv (`deactivate`), and run `docker run --rm infra-ee:1.0 python3 -c "import hvac, jmespath; print('ok')"`.
5. Bump `community.general` in `collections/requirements.yml`, rebuild as `infra-ee:1.1`, and run with `--eei infra-ee:1.1`. Now you can run any past release's toolchain by tag.
6. Replay a saved artifact in interactive mode and find one task's full result for web2.
7. Remove `--net=host` from the settings and run again: read the `UNREACHABLE` error, then restore it.

**Done when:** your playbooks run identically from the EE image, and changing the toolchain means building a new tagged image.

---

## Where you'll use it

- **AWX/AAP**: required; each job template picks an EE (topic 56).
- **CI**: jobs run in the EE image, so pipelines don't install Ansible every time (topics 53–54).
- **Teams with different needs**: network team EE (`cisco.*`, `paramiko`), cloud team EE (`amazon.aws`, `boto3`), each versioned.
- **Upgrades**: test `infra-ee:2.0` with a new ansible-core against staging before switching production.
- **Audits/reproducibility**: "which exact toolchain ran last Tuesday's change?" = the EE tag in the job log.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `UNREACHABLE` from inside the EE | Container can't reach the hosts' network | `container-options: [--net=host]` (labs), proper networking/VPN in CI |
| `Permission denied (publickey)` in the EE | SSH key/agent not available inside | Run `ssh-agent` + `ssh-add`; check navigator mounts `~/.ssh`; key file permissions |
| `couldn't resolve module/action` inside the EE | Collection not in the image | Add it to `requirements.yml`, rebuild, bump the tag |
| `Failed to import the required Python library` inside the EE | Python lib not in `requirements-ee.txt` | Add, rebuild |
| Build fails: `python3 >= 3.13 required` / ansible-core install fails | Base image Python too old for ansible-core 2.22 | Use a base with Python 3.13+ (e.g. Fedora 42) |
| `Error: Unable to find a match: <pkg>` during build | `bindep.txt` package name wrong for the base OS | Fix names/selectors |
| navigator keeps pulling an image you built locally | Pull policy | `pull.policy: never` (or `missing`) |
| `ansible.cfg` ignored inside the EE | Not run from the project root, or world-writable mount | Run from the repo root; check permissions |
| Huge images, slow builds | Many collections/libs, no layer caching | Only needed content; base image with common deps; CI build cache |

---

## Quick recap

- **Execution environment** = container image with ansible-core + ansible-runner + pinned **collections** + **Python** libs + **system** packages: the same toolchain everywhere.
- **ansible-builder** builds it from `execution-environment.yml` (v3) + `requirements.yml` + `requirements.txt` + `bindep.txt`.
- **ansible-navigator** runs playbooks (and doc/inventory/collections/config/replay) inside the EE; settings in `ansible-navigator.yml`; mounts the project and `~/.ssh`.
- Tag EE images by version; bump on any dependency change; push to a registry; scan them.
- AWX/AAP always run jobs in EEs; CI can use the same image.

### Interview questions

1. **Q: What is an Ansible execution environment?**
   A: A container image that packages ansible-core, ansible-runner, collections, Python dependencies and system packages, so playbooks run with an identical, versioned toolchain on laptops, CI and AWX/AAP.
2. **Q: How do you build one?**
   A: Write `execution-environment.yml` (base image, ansible-core, runner, `requirements.yml`, Python `requirements.txt`, `bindep.txt`) and run `ansible-builder build -t name:tag`.
3. **Q: What is ansible-navigator?**
   A: A CLI/TUI that runs playbooks and other Ansible commands inside an execution environment, with artifacts you can replay.

### Scenario question (answer out loud first, then compare)

> *"Three teams (Linux, network, cloud) share one AWX/AAP. The network team needs old `paramiko` and specific `cisco.*` versions; the cloud team needs the newest `amazon.aws` and `boto3`. Their pinned versions conflict. How do you handle this?"*

**Model answer:**
1. Build **separate EEs** per team (`linux-ee`, `network-ee`, `cloud-ee`), each from its own pinned `requirements.yml`/`requirements.txt`/`bindep.txt`, built and tested in CI, tagged by version and pushed to the internal registry.
2. In AWX/AAP, assign each team's job templates (or organization default) to its EE; teams can't break each other.
3. Shared basics (company CA certs, base packages) come from a common **base image** used by all three builds.
4. Upgrades are per team: build `cloud-ee:2.0`, run the cloud playbooks' check mode/Molecule against it, then switch the job template.
5. Scan images for CVEs and rebuild on a schedule; keep old tags for reproducing past runs.

**Likely follow-ups:**
- *Why not one big EE with everything?* Conflicting versions, huge images, and every upgrade affects every team.
- *How does a laptop run the same toolchain?* `ansible-navigator run --eei registry.example.com/network-ee:3.2 ...`.
