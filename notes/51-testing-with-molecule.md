# 51 · Testing with Molecule

> **Level:** ★★★ advanced
>
> **You will learn:** how to **test a role automatically** with **Molecule**: spin up fresh test instances (Docker/Podman containers with systemd, or VMs), **converge** (apply the role), run it **again** to prove **idempotence** (`changed=0`), **verify** the result with assertions, and destroy everything; the files in a scenario (`molecule.yml`, `converge.yml`, `verify.yml`, `prepare.yml`), testing **several distros** at once, the commands for the development loop (`converge`, `login`, `verify`) vs the full `test`, and running it in CI. You'll test `nginx_web` on AlmaLinux 10 and Ubuntu 24.04.
>
> **You need:** [50 · Linting](50-linting.md), [36 · Roles II](36-roles-advanced.md) (`nginx_web`), [27 · Idempotency](27-idempotency-in-depth.md), [09 · systemd in containers](09-lab-setup.md).
>
> **Part 8 · Quality and testing** · Next: [52 · Execution environments and ansible-navigator](52-execution-environments.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Linting reads code; **Molecule runs it**. For each test it creates **brand-new** instances, applies your role, applies it **again** and fails if anything changed (idempotence), runs your **checks**, then throws the instances away. Same result every time, on every distro you support.

**Picture it.** A car crash-test lab: a new car (fresh container) for every test, drive the test course (converge), drive it again (idempotence), inspect the dummies (verify), scrap the car (destroy).

**The one thing to know.**

```
roles/nginx_web/molecule/default/
├── molecule.yml     ← which platforms (images), driver, test steps
├── converge.yml     ← playbook that applies the role
└── verify.yml       ← playbook with assertions
```

```bash
cd ~/infra/roles/nginx_web
molecule test          # create → converge → idempotence → verify → destroy
molecule converge      # dev loop: apply (keeps instances)
molecule verify        # run only the checks
molecule login -h alma10
molecule destroy
```

**Try it now:** follow the Hands-on lab; the first `molecule test` should end with no failures.

**You've got it if you can:** name Molecule's test steps and say which one proves idempotency.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Molecule's building blocks

| Term | Meaning |
|---|---|
| **Scenario** | One test setup (folder under `molecule/`). `default` is the usual one; add others (`ubuntu-only`, `tls`, `upgrade`) |
| **Driver** | What creates the instances: **Docker**, **Podman**, delegated (your own create/destroy playbooks: VMs, cloud) |
| **Platforms** | The instances to create: name, image, settings (privileged, systemd) |
| **Provisioner** | Ansible: runs `converge.yml`, `prepare.yml`, `verify.yml`... with options (inventory vars, env) |
| **Verifier** | How checks run: **Ansible** (a `verify.yml` playbook, default) or **testinfra** (Python tests) |

### The test sequence (`molecule test`)

| Step | Does |
|---|---|
| `dependency` | Install collections/roles from `requirements.yml` |
| `cleanup` / `destroy` | Remove leftovers from previous runs |
| `syntax` | `ansible-playbook --syntax-check` on converge |
| `create` | Start the platforms (containers/VMs) |
| `prepare` | Optional `prepare.yml`: things your role expects to already exist (e.g. Python, a repo) |
| **`converge`** | Run `converge.yml` (apply the role) |
| **`idempotence`** | Run converge **again**; **fail if any task reports `changed`** |
| `side_effect` | Optional: simulate an event (restart, failover) |
| **`verify`** | Run `verify.yml` (assertions about the end state) |
| `cleanup`, `destroy` | Tear down |

During development you use the individual commands and keep instances around:

| Command | Use |
|---|---|
| `molecule create` | Start instances |
| `molecule converge` | Apply the role (re-run after each code change) |
| `molecule idempotence` | Run the second-converge check |
| `molecule verify` | Run checks only |
| `molecule login -h <name>` | Shell into an instance to look around |
| `molecule list` | Show instances and their state |
| `molecule destroy` | Remove instances |
| `molecule test` | The full sequence (what CI runs) |
| `molecule test --destroy=never` | Full sequence but keep instances for debugging |

### What makes a good verify

Test **behaviour and end state**, not the tasks themselves:

| ✅ Check | ❌ Don't just check |
|---|---|
| The service is **running and enabled** | "the systemd task ran" |
| The site **answers** with the expected content on the expected port | "the template task changed" |
| Config files have the right **owner/mode** | |
| A disallowed thing is **absent** (default site removed, port closed) | |
| Validation rejects bad input (a separate scenario expecting failure) | |

### Testing several distros

List several platforms; Molecule converges and verifies all of them in one run. Your role's multi-OS logic (topic 36) is exercised for real.

### Molecule versions and installation

Molecule is maintained by the Ansible community and versioned by date (e.g. 25.x). Drivers other than the built-in default come from the **`molecule-plugins`** package (`molecule-plugins[docker]`, `[podman]`, `[vagrant]`...). Install both into Ansible's venv (or use the **ansible-dev-tools** bundle, topic 50). The Docker driver also needs the `community.docker` collection and the `docker` Python SDK.

---

## Why it exists

Without automated tests, the only way to know a role still works on Ubuntu after you changed something for RHEL is to try it on a real server, often production. Manual testing is slow and inconsistent, and nobody tests idempotency by hand. Molecule makes "does it work on every supported OS, from scratch, and is it idempotent?" a single command that CI runs on every pull request.

---

## Analogy

**A driving test centre.** Every candidate (role version) gets a **brand-new test car** (fresh container), drives the same course twice (converge + idempotence: the second lap must be calm, no surprises), and the examiner ticks a checklist of outcomes (verify): "parked within the lines", not "turned the steering wheel". Then the car is reset for the next candidate (destroy).

---

## How it works inside

```
 molecule test   (in roles/nginx_web)
   │
   ├─ read molecule/default/molecule.yml
   ├─ create: driver (docker) starts containers from the platform images
   │          builds an inventory: alma10, ubuntu2404 (connection: community.docker.docker)
   ├─ converge: ansible-playbook -i <molecule inventory> converge.yml
   │            (ANSIBLE_ROLES_PATH points at the role under test)
   ├─ idempotence: same command again → parse recap → any "changed" → FAIL
   ├─ verify: ansible-playbook verify.yml against the same instances
   └─ destroy: remove containers
```

Molecule talks to containers through the Docker API (no SSH needed), using the same systemd-in-a-container settings as your lab (topic 09).

---

## Code / commands

### Install

```bash
source ~/.venvs/ansible/bin/activate
uv pip install molecule "molecule-plugins[docker]" docker
cd ~/infra
ansible-galaxy collection install community.docker -p ./collections
molecule --version
```

### Create the scenario

```bash
cd ~/infra/roles/nginx_web
molecule init scenario default --driver-name docker
```

Replace the generated files with the following.

`molecule/default/molecule.yml`:

```yaml
---
dependency:
  name: galaxy
  options:
    requirements-file: ../../collections/requirements.yml
driver:
  name: docker
platforms:
  - name: alma10
    image: ansible-lab/alma10            # the lab images: systemd + python already inside
    pre_build_image: true
    command: /usr/sbin/init
    privileged: true
    cgroupns_mode: host
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    tmpfs:
      - /run
      - /tmp
  - name: ubuntu2404
    image: ansible-lab/ubuntu2404
    pre_build_image: true
    command: /lib/systemd/systemd
    privileged: true
    cgroupns_mode: host
    volumes:
      - /sys/fs/cgroup:/sys/fs/cgroup:rw
    tmpfs:
      - /run
      - /tmp
provisioner:
  name: ansible
  config_options:
    defaults:
      interpreter_python: auto_silent
      callbacks_enabled: ansible.posix.profile_tasks
  inventory:
    group_vars:
      all:
        nginx_web_sites:
          - { name: test, title: "Molecule test" }
        nginx_web_remove_default_site: true
verifier:
  name: ansible
scenario:
  test_sequence:
    - dependency
    - destroy
    - syntax
    - create
    - converge
    - idempotence
    - verify
    - destroy
```

- `pre_build_image: true`: use the image as is (don't build a Dockerfile). In CI you'd use public systemd-ready images or build them in a `create` step.
- systemd settings mirror topic 09's Compose file.
- `inventory.group_vars`: the role's inputs for this scenario.
- `dependency`: installs collections from your project's requirements file.

`molecule/default/converge.yml`:

```yaml
---
- name: Converge
  hosts: all
  gather_facts: true
  tasks:
    - name: Apply nginx_web
      ansible.builtin.include_role:
        name: nginx_web
```

`molecule/default/verify.yml`:

```yaml
---
- name: Verify
  hosts: all
  gather_facts: false
  tasks:
    - name: Collect service state
      ansible.builtin.service_facts:

    - name: nginx is running and enabled
      ansible.builtin.assert:
        that:
          - ansible_facts['services']['nginx.service']['state'] == 'running'
          - ansible_facts['services']['nginx.service']['status'] == 'enabled'
        fail_msg: "nginx is not running/enabled"

    - name: Site answers on 8081 with the expected title
      ansible.builtin.uri:
        url: http://127.0.0.1:8081/
        headers: { Host: "test.{{ inventory_hostname }}" }
        return_content: true
      register: page
      failed_when: "'Molecule test' not in page.content"

    - name: Health endpoint answers
      ansible.builtin.uri:
        url: http://127.0.0.1:8081/health
        return_content: true
      register: health
      failed_when: health.content | trim != 'ok'

    - name: Config file has safe permissions
      ansible.builtin.stat:
        path: /etc/nginx/conf.d/nginx_web.conf
      register: conf
      failed_when: conf.stat.mode != '0644' or conf.stat.pw_name != 'root'

    - name: Debian default site is gone
      ansible.builtin.stat:
        path: /etc/nginx/sites-enabled/default
      register: default_site
      failed_when: default_site.stat.exists
      when: inventory_hostname is search('ubuntu')
```

- `uri` runs **inside** each container (targets 127.0.0.1 there).
- Each check is about the **outcome**: service state, HTTP response, file permissions, absence of the default site.

### Run the full test

```bash
cd ~/infra/roles/nginx_web
molecule test
```
```
INFO     default scenario test matrix: dependency, destroy, syntax, create, converge, idempotence, verify, destroy
INFO     Running default > create
PLAY [Create] ...
changed: [localhost] => (item=alma10)
changed: [localhost] => (item=ubuntu2404)
...
INFO     Running default > converge
PLAY [Converge] ************************************************************************
TASK [nginx_web : Validating arguments against arg spec 'main' ...] *********************
ok: [alma10]
ok: [ubuntu2404]
...
PLAY RECAP *****************************************************************************
alma10      : ok=12  changed=6  unreachable=0  failed=0  skipped=2  ...
ubuntu2404  : ok=14  changed=8  unreachable=0  failed=0  skipped=0  ...

INFO     Running default > idempotence
PLAY RECAP *****************************************************************************
alma10      : ok=12  changed=0  ...
ubuntu2404  : ok=13  changed=0  ...
INFO     Idempotence completed successfully.

INFO     Running default > verify
TASK [nginx is running and enabled] ****************************************************
ok: [alma10] => { "changed": false, "msg": "All assertions passed" }
ok: [ubuntu2404] => { "changed": false, "msg": "All assertions passed" }
...
INFO     Verifier completed successfully.

INFO     Running default > destroy
```

- **Idempotence completed successfully** = the second converge reported `changed=0` on both distros.
- If any task changed on the second run, Molecule prints it and fails: `Idempotence test failed because of the following tasks: * [ubuntu2404] => nginx_web : ...`.

### The development loop

```bash
molecule converge                     # create (if needed) + apply
# edit the role...
molecule converge                     # apply again
molecule idempotence                  # second-run check
molecule verify
molecule login -h ubuntu2404          # look around: systemctl status nginx, cat config...
molecule destroy
```

### Break it on purpose

1. In `tasks/configure.yml`, add `force: true` and a run-time value to the index page content (e.g. `{{ now() }}`): `molecule test` fails at **idempotence**.
2. In `verify.yml`, expect the wrong title: fails at **verify**.
3. Remove `vars/Debian.yml`: fails at **converge** on Ubuntu (first_found can't find a file).

Each failure points to a different class of bug: non-idempotent code, wrong behaviour, missing OS support.

### In CI (preview of topic 53)

```yaml
# .github/workflows/molecule.yml (sketch)
jobs:
  molecule:
    runs-on: ubuntu-24.04
    strategy:
      matrix:
        role: [nginx_web, shop_app]
    steps:
      - uses: actions/checkout@v5
      - run: pip install ansible-core==2.22.* molecule "molecule-plugins[docker]" docker
      - run: docker build -t ansible-lab/alma10 -f ci/Dockerfile.alma ci/   # systemd test images
      - run: cd roles/${{ matrix.role }} && molecule test
```

---

## Hands-on lab

1. Install Molecule and create the scenario for `nginx_web`. Run `molecule test` until it passes on both distros.
2. Use the dev loop: `molecule converge`, `molecule login -h alma10`, check `/etc/nginx/conf.d/nginx_web.conf`, `exit`, `molecule destroy`.
3. Do the three "break it" experiments and match each failure to its step.
4. Add a second scenario `molecule/tls` (copy `default`) that sets different inputs (e.g. two sites, port 8443 without real TLS) and verifies both sites answer: `molecule test -s tls`.
5. Add a **negative** test: a scenario whose converge passes `nginx_web_port: "eighty"` and expects failure. (Tip: wrap the role in `block`/`rescue` in converge and `assert` the failure happened.)
6. Create a Molecule scenario for `shop_app` (topic 49) that deploys version 1.0 then 1.1 (`side_effect.yml`) and verifies `/version` returns 1.1 and `current` points to `releases/1.1`.

**Done when:** every role you own has a Molecule scenario that passes `molecule test` on all supported distros.

---

## Where you'll use it

- **Every shared role/collection**: required quality gate in CI before merging and releasing.
- **Upgrades**: test roles against the new ansible-core or new OS versions before production.
- **Refactors**: change internals with confidence; idempotence + verify catch regressions.
- **Certified content**: Automation Hub and Galaxy best practices expect tested roles.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Failed to find driver docker` | `molecule-plugins[docker]` not installed | `uv pip install "molecule-plugins[docker]"` |
| `Failed to import the required Python library (Docker SDK for Python)` | `docker` library missing | `uv pip install docker` |
| Containers exit immediately / `System has not been booted with systemd` | Missing systemd settings (command, privileged, cgroup, tmpfs) | Mirror topic 09's settings in `platforms` |
| `couldn't resolve module/action` in converge | Collection not installed for Molecule | `dependency` step with your `requirements.yml`, or `collections_path` |
| `the role 'nginx_web' was not found` | Molecule not run from the role directory, or role path differs | Run in the role dir; Molecule adds it to `ANSIBLE_ROLES_PATH` |
| Idempotence fails | A task changes on the second run | Fix the task (topic 27); the output names it |
| Verify `uri` fails with connection refused | Service not listening (or wrong port/host) inside the container | `molecule login`, check `ss -tlnp`, logs |
| Slow tests | Image pulls/package installs each run | Pre-built images with dependencies; cache in CI |
| Works locally, fails in CI | Different images, cgroup v1 on the runner, missing privileges | Use the same images; runner with cgroup v2 and Docker privileges |

---

## Quick recap

- **Molecule** = automated role testing: **create → converge → idempotence → verify → destroy** on fresh instances.
- Scenario files: `molecule.yml` (driver, **platforms**, provisioner, verifier, sequence), `converge.yml`, `verify.yml` (+ `prepare.yml`, `side_effect.yml`).
- **Idempotence** step fails if the second converge changes anything. **Verify** checks outcomes (service running, HTTP answers, permissions).
- Multi-distro: several platforms; systemd containers need the topic 09 settings.
- Dev loop: `converge` / `login` / `verify` / `destroy`; CI runs `molecule test`.

### Interview questions

1. **Q: How do you test Ansible roles?**
   A: With Molecule: it creates fresh instances (containers or VMs), applies the role, re-applies it to check idempotence, runs verification checks, and destroys the instances, typically across all supported OSes in CI. Plus ansible-lint for static checks.
2. **Q: What does Molecule's idempotence step do?**
   A: Runs the converge playbook a second time and fails if any task reports `changed`.
3. **Q: What should `verify.yml` test?**
   A: The resulting behaviour and state (services running, ports answering with expected content, file ownership and modes, things that must be absent), not the fact that tasks ran.

### Scenario question (answer out loud first, then compare)

> *"Your company's `baseline` role is used on 2,000 servers across RHEL 8/9/10 and Ubuntu 22.04/24.04. A change last month broke SSH on Ubuntu 22.04 only. Design a testing strategy so that can't happen again."*

**Model answer:**
1. **Molecule scenarios** for every supported OS (one platform each, systemd-capable images), run in CI on every PR touching the role. Matrix over OS versions; for RHEL 8 (old Python, topic 07) use the toolchain version that manages it.
2. **Verify** the critical outcomes, especially access: sshd running, config passes `sshd -t`, the automation user can still log in (e.g. an SSH connection test from the converge host), sudo works.
3. **Idempotence** on all platforms.
4. **Lint** at production profile.
5. **Staged rollout** after merge: check mode against a canary group per OS, then `serial` batches with `max_fail_percentage: 0` and a post-change SSH check per batch (topics 45, 49).
6. For changes Molecule can't fully model (kernel, SELinux enforcing), a VM-based scenario (delegated driver) or a staging environment per OS.

**Likely follow-ups:**
- *Isn't testing all OSes on every PR slow?* Run in parallel jobs; cache images; full matrix on PRs touching the role, nightly full run for everything.
- *Testinfra or Ansible verifier?* Ansible verifier keeps one language; testinfra (pytest) is handy for complex assertions. Either is fine; be consistent.
