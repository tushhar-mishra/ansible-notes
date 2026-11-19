# 39 · Project Layout for Production

> **Level:** ★★ intermediate
>
> **You will learn:** how to lay out a real Ansible repository so it scales to many environments, roles and people: **one inventory directory per environment** (with its own `group_vars`/`host_vars`), playbooks in a `playbooks/` folder, roles (or an internal collection), pinned dependencies, a committed `ansible.cfg`, lint configs and a `Makefile`; the **path gotchas** when playbooks live in a subfolder; monorepo vs multi-repo; naming conventions; and what goes in `.gitignore` and the README. You'll restructure your lab into this layout.
>
> **You need:** [38 · Plugins](38-plugins-and-custom-modules.md), [37 · Collections](37-collections-and-galaxy.md), [24 · group_vars](24-group-vars-and-host-vars.md), [11 · ansible.cfg](11-connection-settings-and-ansible-cfg.md), [34 · import_playbook](34-import-vs-include.md).
>
> **Part 4 · Organizing and reusing code** · Next: [40 · Ansible Vault](40-ansible-vault.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Separate **what** you do (playbooks and roles: identical for every environment) from **where and with which values** (an inventory directory per environment). Pin every tool and dependency. Choose the environment only with `-i`.

**Picture it.** A franchise restaurant chain: one recipe book (roles/playbooks) for every branch; each branch has its own folder with staff list, opening hours and supplier (inventory + group_vars).

**The one thing to know.**

```
infra-ansible/
├── ansible.cfg
├── requirements.txt                  # ansible-core, ansible-lint... pinned
├── collections/requirements.yml      # collections pinned
├── inventories/
│   ├── staging/
│   │   ├── hosts.yml
│   │   ├── group_vars/  all/{main.yml,vault.yml}  web.yml  db.yml
│   │   └── host_vars/
│   └── prod/
│       ├── hosts.yml
│       ├── group_vars/ ...
│       └── host_vars/
├── playbooks/
│   ├── site.yml   web.yml   db.yml
│   └── ops/  restart-app.yml  patch.yml
└── roles/  baseline/  nginx_web/  postgres/
```

```bash
ansible-playbook -i inventories/staging playbooks/site.yml --check --diff
ansible-playbook -i inventories/prod    playbooks/site.yml
```

**Try it now:** sketch this tree for your lab (one environment called `lab`).

**You've got it if you can:** explain why `group_vars` must live inside the inventory folder in this layout.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### The full recommended layout

```
infra-ansible/
├── README.md                     how to install, run, add a host, add a role
├── ansible.cfg                   project settings (no default prod inventory!)
├── requirements.txt              Python: ansible-core==2.22.x, ansible-lint, molecule...
├── collections/
│   └── requirements.yml          collections (+ standalone roles) pinned
├── inventories/
│   ├── dev/
│   │   ├── hosts.yml             hosts and groups only
│   │   ├── group_vars/
│   │   │   ├── all/
│   │   │   │   ├── main.yml      environment-wide settings
│   │   │   │   └── vault.yml     encrypted secrets (topic 40)
│   │   │   ├── web.yml
│   │   │   └── db.yml
│   │   └── host_vars/
│   │       └── db1.yml
│   ├── staging/  (same shape)
│   └── prod/     (same shape)
├── playbooks/
│   ├── site.yml                  import_playbook of everything
│   ├── baseline.yml
│   ├── web.yml
│   ├── db.yml
│   └── ops/                      operational, action playbooks (restart, rotate, patch)
├── roles/                        your roles (or move them into an internal collection)
│   ├── baseline/
│   ├── nginx_web/
│   └── postgres/
├── .ansible-lint                 lint profile and rules (topic 50)
├── .yamllint
├── .pre-commit-config.yaml       run linters before each commit
├── .gitignore
└── Makefile                      common commands (deps, lint, check, apply)
```

### Why each piece is where it is

| Choice | Reason |
|---|---|
| **One inventory dir per environment** | `-i inventories/prod` is the **only** way to touch prod. Each environment's values live together |
| `group_vars` **inside** each inventory dir (inventory-side) | Playbooks are in `playbooks/`, so playbook-side `group_vars` would have to be in `playbooks/group_vars` and would apply to **every** environment. Inventory-side keeps values per environment |
| Same group names in every environment (`web`, `db`) | Playbooks don't need to know the environment |
| `playbooks/` subfolder | Clean root; many playbooks |
| `roles/` at the root + `roles_path` in `ansible.cfg` | Roles aren't next to the playbooks any more, so tell Ansible where they are |
| `collections/requirements.yml` + `collections_path` | Pinned, project-local dependencies (topic 37) |
| `ops/` playbooks | Actions (restart, rotate, patch) separate from convergent config (topic 27) |
| No default `inventory =` pointing at prod | Prevents accidents |
| Linters + pre-commit + Makefile | Same checks for everyone, locally and in CI |

### ansible.cfg for this layout

```ini
[defaults]
roles_path = ./roles
collections_path = ./collections:~/.ansible/collections
# No 'inventory =' line: always pass -i explicitly (or set inventories/dev as a safe default)
interpreter_python = auto_silent
forks = 20
callbacks_enabled = ansible.posix.profile_tasks
retry_files_enabled = False
nocows = True

[ssh_connection]
pipelining = True
```

(`pipelining` is explained in topic 48. Paths in `ansible.cfg` are relative to the file's directory.)

### Path gotchas when playbooks are in a subfolder

| Thing | Looked up relative to | Consequence |
|---|---|---|
| `group_vars/`, `host_vars/` | The **inventory** dir **and** the **playbook** dir | Put them in `inventories/<env>/` |
| `roles/` | The playbook dir, then `roles_path` | Set `roles_path = ./roles` |
| `collections/` (adjacent) | The playbook dir | Set `collections_path` |
| `files/`, `templates/` for tasks in playbooks | The playbook dir (and role dirs for role tasks) | Keep files/templates inside roles |
| `vars_files: [...]`, `include_tasks: x.yml` | The playbook (or the including file) | Use paths relative to the playbook, or `{{ playbook_dir }}/..` |
| `ansible.cfg` | The **current working directory** | Run commands from the repo root (or set `ANSIBLE_CONFIG`) |

Rule: **run everything from the repository root.**

### Inventory: hosts only

```yaml
# inventories/prod/hosts.yml
all:
  children:
    web:
      hosts:
        web-p01: { ansible_host: 10.20.1.11 }
        web-p02: { ansible_host: 10.20.1.12 }
    db:
      hosts:
        db-p01: { ansible_host: 10.20.2.11 }
```

Values go to `group_vars`/`host_vars`. In the cloud, `hosts.yml` is replaced by a **dynamic inventory** file (topic 42) in the same folder.

### Monorepo or multi-repo?

| Layout | Pros | Cons | Fits |
|---|---|---|---|
| **One repo** (roles + playbooks + inventories) | Simple, atomic changes, easy onboarding | Everyone's changes in one place; access control is all-or-nothing | Small/medium teams |
| **Content repo(s) + inventory repo** | Roles/collections versioned and shared; config owned per team/env | Changes span repos; version pinning needed | Larger orgs, platform teams |
| **Roles as collections** in their own repos, consumed via `requirements.yml` | Real versioning, reuse across products | More release process | Platform/shared content |

A common evolution: start with a monorepo; when roles are shared by several teams, move them into a versioned internal collection.

### Naming conventions

| Item | Convention | Example |
|---|---|---|
| Roles | lowercase, underscores, one job | `nginx_web`, `postgres_primary` |
| Role variables | role prefix | `nginx_web_port` |
| Groups | role/function, underscores | `web`, `db`, `lb`, `eu_west` |
| Environments | inventory folder names | `dev`, `staging`, `prod` |
| Playbooks | what they converge or do | `site.yml`, `web.yml`, `ops/restart-app.yml` |
| Task names | desired state, sentence case | "nginx is installed" |
| Tags | role name + phase | `nginx_web`, `nginx_web_config` |

### .gitignore

```gitignore
# installed dependencies
collections/ansible_collections/
roles/external/
# local artifacts
*.retry
fetched/
credentials/          # password-lookup output (or vault it)
.vault_pass*          # never commit vault passwords
.venv/
__pycache__/
```

### Makefile

```makefile
ENV ?= dev
PB  ?= playbooks/site.yml
INV  = inventories/$(ENV)

deps:
	uv pip install -r requirements.txt
	ansible-galaxy collection install -r collections/requirements.yml -p ./collections

lint:
	yamllint .
	ansible-lint

check:
	ansible-playbook -i $(INV) $(PB) --check --diff

apply:
	ansible-playbook -i $(INV) $(PB) --diff

.PHONY: deps lint check apply
```

```bash
make deps
make lint
make check ENV=staging
make apply ENV=staging PB=playbooks/web.yml
```

---

## Why it exists

A lab can live in one folder. A production repo serves several environments, dozens of roles, CI pipelines and many people. Without a clear structure you get: values for prod mixed with staging, playbooks that only work from one directory, roles copied between repos, and "which collection version are we on?". A conventional layout lets anyone (including AWX/AAP and CI) run the code the same way, and makes it hard to hit the wrong environment by accident.

---

## Analogy

**A franchise operations manual.** The **central kitchen manual** (roles and playbooks) is identical for every branch. Each **branch binder** (inventory folder) has its own address list, staff and local settings (group_vars). Head office checks which supplier versions every branch uses (requirements files). New staff read the binder's front page (README) and use the same checklists (Makefile, linters).

---

## How it works inside

What Ansible resolves for `ansible-playbook -i inventories/prod playbooks/web.yml` run from the repo root:

```
 cwd = infra-ansible/
   ├─ ansible.cfg            ← found in cwd (topic 11)
   ├─ inventory              ← inventories/prod/ (directory: hosts.yml + plugins)
   │     inventory-side vars ← inventories/prod/group_vars/, host_vars/
   ├─ playbook               ← playbooks/web.yml
   │     playbook-side vars  ← playbooks/group_vars/ (avoid using it)
   │     adjacent roles      ← playbooks/roles/ (absent) → roles_path ./roles
   │     adjacent collections← playbooks/collections/ (absent) → collections_path
   └─ roles' files/templates ← roles/<role>/files, roles/<role>/templates
```

---

## Code / commands

### Restructure the lab

```bash
mkdir -p ~/infra && cd ~/infra
mkdir -p inventories/lab/{group_vars,host_vars} playbooks/ops roles collections
cp ~/ansible-lab/requirements.txt .
cp ~/ansible-lab/collections/requirements.yml collections/
cp -r ~/ansible-lab/roles/* roles/
cp ~/ansible-lab/group_vars/*.yml inventories/lab/group_vars/
cp ~/ansible-lab/host_vars/*.yml inventories/lab/host_vars/
```

`inventories/lab/hosts.yml`:

```yaml
all:
  children:
    app:
      children:
        web:
          hosts:
            web1: { ansible_host: 172.30.0.11 }
            web2: { ansible_host: 172.30.0.12 }
        db:
          hosts:
            db1: { ansible_host: 172.30.0.21 }
    lb:
      hosts:
        lb1: { ansible_host: 172.30.0.31 }
    alma:
      hosts: { web1: {}, db1: {} }
    ubuntu:
      hosts: { web2: {}, lb1: {} }
```

`inventories/lab/group_vars/all/main.yml` (move `all.yml` into `all/main.yml`), plus connection settings for this environment:

```yaml
ansible_user: ansible
ansible_ssh_private_key_file: ~/.ssh/ansible_key
ansible_ssh_common_args: "-o UserKnownHostsFile=~/.ssh/known_hosts_lab -o StrictHostKeyChecking=accept-new"
```

- Putting connection settings in the **environment's** `group_vars/all` means prod can use a different user/key/bastion without touching `ansible.cfg`.

`ansible.cfg` (as above). `playbooks/site.yml`:

```yaml
- ansible.builtin.import_playbook: web.yml
- ansible.builtin.import_playbook: lb.yml
```

`playbooks/web.yml`:

```yaml
- name: Web tier
  hosts: web
  become: true
  roles:
    - nginx_web
```

`playbooks/lb.yml`: copy from topic 30, but move its template into a role (`roles/lb_nginx/templates/lb-nginx.conf.j2`, tasks and handler into the role) so no playbook-level `templates/` folder is needed.

Install and run:

```bash
cd ~/infra
make deps
ansible-inventory -i inventories/lab --graph
ansible-playbook -i inventories/lab playbooks/site.yml --check --diff
ansible-playbook -i inventories/lab playbooks/site.yml
```

### Prove the environment separation

```bash
cp -r inventories/lab inventories/lab2
sed -i 's/Hello from/Greetings from/' inventories/lab2/group_vars/all/main.yml   # if your page text var lives there
ansible-inventory -i inventories/lab  --host web2 | grep -i text
ansible-inventory -i inventories/lab2 --host web2 | grep -i text
rm -rf inventories/lab2
```

Same playbooks, different values, chosen only by `-i`.

---

## Hands-on lab

1. Build `~/infra` as above and get `site.yml` running with `changed=0` on the second run.
2. Run a playbook from **inside** `playbooks/` (`cd playbooks && ansible-playbook -i ../inventories/lab web.yml`). What breaks? (No `ansible.cfg` in cwd → no `roles_path`.) Go back to the root.
3. Turn `lb.yml` into a role-based playbook so `playbooks/` has no `templates/` folder.
4. Create `playbooks/ops/restart-web.yml` (restart nginx with `serial: 1`, topic 45 preview) and keep it out of `site.yml`.
5. Add `.gitignore`, `Makefile`, and initialise Git: `git init && git add -A && git commit -m "Initial infra layout"`. Check that `collections/ansible_collections/` and `credentials/` aren't committed.
6. Write the README: prerequisites, `make deps`, how to run check/apply per environment, how to add a host, where secrets live.

**Done when:** a new teammate can clone the repo and run `make deps && make check ENV=lab` without asking you anything.

---

## Where you'll use it

- **Every team repo**; AWX/AAP **projects** point at this repo and expect this shape (topic 56).
- **CI/CD** jobs: `make lint`, `make check ENV=staging` on PRs, `make apply ENV=prod` on approval (topics 53–54).
- **Onboarding** and audits: the layout itself documents what exists per environment.
- **Scaling**: dynamic inventories per environment (topic 42), internal collections for shared roles.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `the role 'x' was not found` | Playbooks in a subfolder, no `roles_path` | `roles_path = ./roles` in `ansible.cfg`; run from the root |
| Prod values used in staging | Variables in `playbooks/group_vars/` (playbook-side, all environments) | Move them to `inventories/<env>/group_vars/` |
| `ansible.cfg` ignored | Command run from another directory | Run from the repo root, or `ANSIBLE_CONFIG` |
| Template not found for a play-level task | `templates/` expected next to the playbook | Move the task and template into a role |
| Different collection versions in CI | Not installed from the project's `requirements.yml` | `make deps` in CI; `collections_path` |
| Someone ran against prod by mistake | A default `inventory =` pointing to prod, or prod credentials on laptops | No prod default; prod credentials only in the CI/AAP |
| Secrets committed | No vault, no `.gitignore` rules | Vault (topic 40), `.gitignore`, pre-commit secret scanning |
| Inventory directory parsing errors from README files | Non-inventory files in the inventory dir | Keep only inventory sources, `group_vars`, `host_vars` there (or set `inventory_ignore_extensions`) |

---

## Quick recap

- **Code** (playbooks, roles) is identical for every environment; **config** lives in `inventories/<env>/` with its own `group_vars`/`host_vars`.
- Playbooks in `playbooks/` → set `roles_path` and `collections_path`; keep files/templates inside roles; run from the repo root.
- Pin everything: `requirements.txt`, `collections/requirements.yml`. Commit `ansible.cfg`, lint configs, `Makefile`, README; ignore installed collections and secrets.
- Separate **ops/action** playbooks from convergent config playbooks.
- Monorepo first; versioned internal collections when content is shared.

### Interview questions

1. **Q: How do you structure an Ansible repo for multiple environments?**
   A: One inventory directory per environment (`inventories/dev|staging|prod`), each with its own `group_vars`/`host_vars`; shared playbooks and roles; select the environment with `-i`; pinned dependencies and a committed `ansible.cfg`.
2. **Q: Why put `group_vars` inside the inventory directory rather than next to the playbooks?**
   A: Playbook-side `group_vars` apply to every inventory you run against, so environment-specific values would leak across environments. Inventory-side keeps them per environment.
3. **Q: What belongs in `.gitignore` for an Ansible repo?**
   A: Installed dependencies (`collections/ansible_collections`, external roles), retry files, fetched files, generated credentials, vault password files, virtualenvs.

### Scenario question (answer out loud first, then compare)

> *"Three teams share one Ansible repo. Team A changed a shared role default and broke Team B's production. How would you restructure ownership and releases so this can't happen silently?"*

**Model answer:**
1. Move shared roles into a **versioned internal collection** (own repo, semantic versioning, changelog, Molecule tests, CODEOWNERS for review).
2. Each team's playbooks/inventory repo pins the collection version in `collections/requirements.yml`. A role change only reaches Team B when **they** bump the pin, after their own check-mode run against staging.
3. Changes to defaults are treated as potentially breaking: documented in the changelog; major version bump if behaviour changes.
4. In the monorepo (if kept): CODEOWNERS on `roles/<shared>` require the owning and consuming teams' review; CI runs every team's check mode against staging for any role change.

**Likely follow-ups:**
- *Isn't that slower?* A bit, but upgrades become deliberate and testable; urgent fixes are a patch release + pin bump.
- *Where do environment values live then?* In each team's inventory directories, unchanged.
