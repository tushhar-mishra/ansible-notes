# 37 · Collections and Galaxy

> **Level:** ★★ intermediate
>
> **You will learn:** what a **collection** is (the package format for modules, plugins, roles and playbooks), its layout (`galaxy.yml`, `plugins/`, `roles/`, `meta/runtime.yml`), where collections come from (**Ansible Galaxy**, **Red Hat Automation Hub**, a private hub, Git, tarballs), how to install and **pin** them with **`requirements.yml`**, where they're installed and **which copy wins** (`collections_path`), standalone Galaxy roles, configuring Galaxy servers, verifying and signing content, and how to judge whether third-party content is trustworthy.
>
> **You need:** [36 · Roles II](36-roles-advanced.md), [16 · FQCNs](16-modules-fqcn-and-ansible-doc.md), [08 · Installing Ansible](08-installing-ansible.md) (requirements.txt pinning).
>
> **Part 4 · Organizing and reusing code** · Next: [38 · Plugins and custom modules](38-plugins-and-custom-modules.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Since 2020, almost all Ansible content (modules, plugins, roles) ships in **collections**: versioned packages named `namespace.collection`. ansible-core contains only `ansible.builtin`; everything else you install with **`ansible-galaxy`**, pinned in a **`requirements.yml`** committed to your repo.

**Picture it.** ansible-core is a phone's operating system; collections are apps from an app store (Galaxy). `requirements.yml` is the list of apps (with exact versions) every phone in the company must have.

**The one thing to know.** `~/ansible-lab/collections/requirements.yml`:

```yaml
---
collections:
  - name: ansible.posix
    version: "==2.1.0"
  - name: community.general
    version: ">=12.0.0,<13.0.0"
  - name: community.docker
    version: "==5.0.0"
```

```bash
ansible-galaxy collection install -r collections/requirements.yml -p ./collections
ansible-galaxy collection list
```

(Versions are examples: pick the current ones and pin them.)

**Try it now:** run the two commands, then `ansible-doc community.general.ufw` to prove the collection is found.

**You've got it if you can:** pin a collection version, install it into the project, and explain which copy Ansible uses if two versions are installed.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### A collection

A **collection** is a directory (or a `.tar.gz` of it) named `<namespace>/<collection>` with a fixed layout:

```
ansible_collections/
└── community/
    └── general/
        ├── galaxy.yml or MANIFEST.json   ← name, version, dependencies, authors, license
        ├── meta/runtime.yml              ← requires_ansible, redirects, deprecations, tombstones, action groups
        ├── plugins/
        │   ├── modules/                  ← modules (community.general.ufw, ...)
        │   ├── module_utils/             ← shared code for modules
        │   ├── filter/  lookup/  test/   ← Jinja2 plugins
        │   ├── inventory/  callback/  connection/  become/  cache/  vars/
        │   └── action/                   ← action plugins
        ├── roles/                        ← roles (community.general.<role>)
        ├── playbooks/                    ← playbooks (run as namespace.collection.playbook)
        ├── docs/  tests/  changelogs/
        └── README.md
```

- Everything inside is addressed by **FQCN**: `community.general.ufw` (module), `community.general.json_query` (filter), `redhat.rhel_system_roles.timesync` (role), `ansible-playbook mycompany.platform.site` (playbook).
- Collections use **semantic versioning** (`MAJOR.MINOR.PATCH`): a major bump may remove or change things.
- `meta/runtime.yml` declares `requires_ansible: ">=2.17.0"`: installing on an older core gives a warning/error.

### Where collections come from

| Source | What | Notes |
|---|---|---|
| **Ansible Galaxy** (galaxy.ansible.com) | Community hub, anyone can publish | Default server. Quality varies |
| **Red Hat Automation Hub** (console.redhat.com) | **Certified** (supported by Red Hat + partners) and **Validated** content | Needs an AAP subscription token |
| **Private Automation Hub** | Your company's own hub (part of AAP), or Galaxy NG | Mirror approved content, publish internal collections |
| Git repo | `git+https://github.com/org/repo.git,v1.2.0` | Handy for internal/unreleased content |
| Tarball / URL / directory | `./mycompany-platform-1.0.0.tar.gz` | Air-gapped installs, CI artifacts |
| The `ansible` PyPI package | ~85 collections bundled in `site-packages/ansible_collections` | Convenient, but versions are fixed by the package |

### Where they're installed, and which copy wins

`ansible-galaxy collection install` puts collections in the **first** path of `collections_path` (default `~/.ansible/collections`), or where you say with `-p`.

When a playbook uses a collection, Ansible searches, **first match wins**:

| Order | Location |
|---|---|
| 1 | `./collections/ansible_collections/` **next to the playbook** (automatically) |
| 2 | Each path in `collections_path` (ansible.cfg / `ANSIBLE_COLLECTIONS_PATH`), in order (default `~/.ansible/collections:/usr/share/ansible/collections`) |
| 3 | The Python environment's `site-packages/ansible_collections` (where the `ansible` package bundles its collections) |

Only **one version** of a collection is used in a run: the first one found. This is the #1 source of "works on my machine": one person has `community.general` 9 in `~/.ansible/collections`, another has 12 bundled in the `ansible` package.

`ansible-galaxy collection list` shows **every** installed copy, grouped by path, so you can see the shadowing.

**Recommendation:** install project collections into **`./collections`** inside the repo (`-p ./collections`, and add `collections/ansible_collections/` to `.gitignore`), or set `collections_path = ./collections` in the project's `ansible.cfg`. Then the project uses exactly its pinned versions.

### requirements.yml

One file for collections **and** standalone roles:

```yaml
---
collections:
  - name: ansible.posix
    version: "==2.1.0"                  # exact
  - name: community.general
    version: ">=12.0.0,<13.0.0"         # range: patch/minor updates, no breaking major
  - name: amazon.aws
    version: "==10.1.2"
    source: https://galaxy.ansible.com  # explicit server (optional)
  - name: https://github.com/mycompany/ansible-platform.git
    type: git
    version: v1.4.0                     # tag, branch or commit
  - name: ./vendor/mycompany-security-2.0.1.tar.gz
    type: file

roles:
  - name: geerlingguy.postgresql       # standalone Galaxy role (legacy format)
    version: "4.0.1"
```

```bash
ansible-galaxy install -r requirements.yml -p ./collections      # collections + roles (roles go to roles_path)
ansible-galaxy collection install -r requirements.yml            # collections only
ansible-galaxy role install -r requirements.yml                  # roles only
```

Version specifiers: `==`, `!=`, `>=`, `>`, `<=`, `<`, combined with commas; `*` = latest.

### Standalone roles vs collections

Before collections, Galaxy distributed **roles** individually (`geerlingguy.nginx`). They still exist and work (`ansible-galaxy role install`), installed into `roles_path`. New content is published as collections (a collection can contain roles). Both are fine to consume; for your own shared content, prefer a **collection**.

### Configuring Galaxy servers (Automation Hub / private hub)

```ini
# ansible.cfg
[galaxy]
server_list = automation_hub, galaxy

[galaxy_server.automation_hub]
url = https://console.redhat.com/api/automation-hub/content/published/
auth_url = https://sso.redhat.com/auth/realms/redhat-external/protocol/openid-connect/token
token = <from an env var or secret store, not committed>

[galaxy_server.galaxy]
url = https://galaxy.ansible.com/
```

- Servers are tried in `server_list` order.
- Pass tokens through environment variables (`ANSIBLE_GALAXY_SERVER_AUTOMATION_HUB_TOKEN`), never commit them.
- Since ansible-core 2.20, `ansible-galaxy` only talks to servers with the **v3** API.

### Trust and supply chain

Installing a collection means running its code **with your credentials, on your control node and servers**. Treat it like any dependency:

| Practice | How |
|---|---|
| Prefer certified/vendor-maintained content | `amazon.aws`, `kubernetes.core`, `ansible.posix`, Red Hat Certified on Automation Hub |
| Check maintenance | Recent releases, open issues, CI badges, number of maintainers |
| **Pin** exact versions (or tight ranges) | `requirements.yml`, reviewed upgrades |
| Verify integrity | `ansible-galaxy collection verify <name>` compares installed files with the server's checksums; signed collections (GPG) on Automation Hub with `--keyring` and `--required-valid-signature-count` |
| Mirror | A private hub with only approved collections; CI installs from there |
| Read the code | Especially for small community collections and anything with `shell`/download logic |
| Lock it into an image | Execution environments (topic 52) |

---

## Why it exists

The old "Ansible 2.9" shipped **3,000+ modules** in one package: every bug fix to one cloud module waited for a full Ansible release, and nobody could maintain all of it. Collections split content by owner (AWS team, network vendors, community groups), each releasing on its own schedule. You get faster fixes and **only the content you need**, at the price of managing versions yourself.

---

## Analogy

**Libraries in a software project** (like `package.json` or `requirements.txt`). ansible-core is the language runtime; collections are libraries; `requirements.yml` is the lock file; Galaxy is the public package registry (like npm or PyPI); Automation Hub is a curated, supported registry; a private hub is your company's internal registry mirror.

---

## How it works inside

```
 ansible-galaxy collection install -r requirements.yml -p ./collections
   │
   ├─ read requirements; for each: ask servers (server_list order) for matching versions
   ├─ resolve DEPENDENCIES (galaxy.yml "dependencies") with resolvelib → one version each
   ├─ download tar.gz (+ signatures if configured), verify checksums
   └─ extract to ./collections/ansible_collections/<ns>/<name>/  (+ MANIFEST.json, FILES.json)

 ansible-playbook site.yml  (task: community.general.ufw)
   ├─ collection loader searches: ./collections → collections_path → site-packages
   ├─ first community/general found → read meta/runtime.yml (redirects, requires_ansible)
   └─ load plugins/modules/ufw.py (topic 16/13)
```

---

## Code / commands

### Install pinned collections into the project

`~/ansible-lab/collections/requirements.yml`: as in the Core section.

```bash
cd ~/ansible-lab
ansible-galaxy collection install -r collections/requirements.yml -p ./collections
```
```
Starting galaxy collection install process
Process install dependency map
Starting collection install process
Downloading https://galaxy.ansible.com/api/v3/plugin/ansible/content/published/collections/artifacts/ansible-posix-2.1.0.tar.gz to /home/you/.ansible/tmp/ansible-local-.../tmp...
Installing 'ansible.posix:2.1.0' to '/home/you/ansible-lab/collections/ansible_collections/ansible/posix'
ansible.posix:2.1.0 was installed successfully
...
community.docker:5.0.0 was installed successfully
[WARNING]: The specified collections path '/home/you/ansible-lab/collections' is not part of the configured Ansible collections paths
'/home/you/.ansible/collections:/usr/share/ansible/collections'. The installed collection will not be picked up in an Ansible run, unless within a playbook-adjacent collections directory.
```

- The warning is about **ad hoc** commands and `ansible-doc`, which don't have a "playbook directory". Playbooks in `~/ansible-lab` find `./collections` automatically. To make everything find it, add to `ansible.cfg`:

```ini
[defaults]
collections_path = ./collections:~/.ansible/collections
```

### See all copies and which wins

```bash
ansible-galaxy collection list
```
```
# /home/you/ansible-lab/collections/ansible_collections
Collection        Version
----------------- -------
ansible.posix     2.1.0
community.docker  5.0.0
community.general 12.0.1

# /home/you/.ansible/collections/ansible_collections
Collection        Version
----------------- -------
ansible.posix     2.1.0
community.general 12.0.1
```

- Two copies of `community.general`: the **first path** (the project) wins.
- Check one collection: `ansible-galaxy collection list community.general`.
- Prove which file a module comes from: `ansible-playbook ... -vvv | grep 'Using module file'`.

### Upgrade on purpose

```bash
# change the pin in requirements.yml, then:
ansible-galaxy collection install -r collections/requirements.yml -p ./collections --upgrade
```

- Without `--upgrade` (or `--force`), an already-installed version that satisfies the requirement is kept.
- Read the collection's **changelog** for breaking changes before bumping a major version.

### Verify installed content

```bash
ansible-galaxy collection verify -p ./collections ansible.posix
```
```
Verifying 'ansible.posix:2.1.0'.
Installed collection found at '/home/you/ansible-lab/collections/ansible_collections/ansible/posix'
Successfully verified that checksums for 'ansible.posix:2.1.0' match the remote collection.
```

Edit a file in the installed collection and run it again: it reports the modified file.

### Air-gapped installs

```bash
# on a machine with internet:
ansible-galaxy collection download -r collections/requirements.yml -p ./offline
# copy ./offline to the air-gapped machine, then:
ansible-galaxy collection install -r ./offline/requirements.yml -p ./collections --offline
```

`download` also writes a `requirements.yml` pointing at the downloaded tarballs.

### Use a role from a collection

```yaml
- hosts: alma
  become: true
  roles:
    - role: community.general.some_role      # FQCN of a role inside a collection (example)
```

Real example you might use on RHEL: `redhat.rhel_system_roles.timesync` from the `redhat.rhel_system_roles` collection (Automation Hub) or `fedora.linux_system_roles.timesync` (Galaxy, the upstream).

### Create your own collection (skeleton)

```bash
ansible-galaxy collection init mycompany.platform --init-path ./collections/ansible_collections
```

Produces `galaxy.yml`, `plugins/`, `roles/`, `meta/runtime.yml`, `README.md`. Move `roles/nginx_web` into `.../mycompany/platform/roles/nginx_web` and call it as `mycompany.platform.nginx_web`. Build and publish:

```bash
cd collections/ansible_collections/mycompany/platform
ansible-galaxy collection build              # → mycompany-platform-1.0.0.tar.gz
ansible-galaxy collection publish mycompany-platform-1.0.0.tar.gz --server my_private_hub
```

(Topic 38 adds a custom module and filter to it.)

---

## Hands-on lab

1. Write `collections/requirements.yml` with pinned `ansible.posix`, `community.general`, `community.docker`, install into `./collections`, add `collections_path` to `ansible.cfg`, and add `collections/ansible_collections/` to `.gitignore`.
2. `ansible-galaxy collection list` and identify every shadowed copy. Remove the one in `~/.ansible/collections` (or keep it and confirm the project's wins with `-vvv`).
3. Install `community.general` at an **older** major version into `~/.ansible/collections` (`ansible-galaxy collection install 'community.general:<10' -p ~/.ansible/collections`). From a **different directory without an ansible.cfg**, run an ad hoc module from it and check which version is used. Then run from `~/ansible-lab`. Understand the difference.
4. `ansible-galaxy collection verify` one collection; tamper with one file; verify again; reinstall with `--force`.
5. Create `mycompany.platform` with `collection init`, move your `nginx_web` role into it, and run it by FQCN from a playbook.
6. Read `meta/runtime.yml` of `community.general` and find one redirect and one tombstone.

**Done when:** a fresh clone of your project gets exactly the same collections with one command.

---

## Where you'll use it

- **Every project**: `collections/requirements.yml` pinned and installed in CI and on laptops.
- **Cloud/containers/network**: `amazon.aws`, `azure.azcollection`, `google.cloud`, `kubernetes.core`, `community.docker`, `cisco.ios`, `arista.eos`...
- **Company platform content**: internal collections (`mycompany.platform`) with roles and plugins, versioned and published to a private hub.
- **AAP/AWX**: projects with `collections/requirements.yml` are installed automatically; execution environments bake them in (topics 52, 56).
- **Security reviews**: dependency pinning, mirroring and verification.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `couldn't resolve module/action 'x.y.z'` | Collection not installed in any searched path | Install it; check `collections_path`; `ansible-galaxy collection list` |
| Works in playbooks, not in ad hoc/ansible-doc | Collection only playbook-adjacent | Add `./collections` to `collections_path` |
| Different behaviour on two machines | Different versions; the first found wins | Pin in `requirements.yml`; install into the project; check `collection list` |
| `Collection x does not support Ansible version 2.22.0` | `requires_ansible` in `meta/runtime.yml` excludes your core | Use a compatible collection version |
| `ERROR! Failed to resolve the requested dependencies map` | Conflicting version ranges between collections | Relax/align pins; read the conflict message |
| `ERROR! Galaxy server ... does not support the v3 API` | Old server (ansible-core 2.20+ requires v3) | Upgrade/replace the server |
| `HTTP Error 401` from Automation Hub | Missing/expired token | Refresh the offline token; pass it via env var |
| `--upgrade` didn't upgrade | Pin still allows only the old version, or a playbook-adjacent copy shadows it | Change the pin; check all paths |
| Role installed with `ansible-galaxy install` not found | Roles go to `roles_path`, not `collections_path` | Check `roles_path`; use `-p roles` |
| Tampered/unknown content | No verification | `collection verify`, signatures, private hub |

---

## Quick recap

- **Collection** = versioned package (`namespace.collection`) of modules, plugins, roles, playbooks; layout with `galaxy.yml`, `plugins/`, `roles/`, `meta/runtime.yml`.
- Sources: **Galaxy** (community), **Automation Hub** (certified/validated), **private hub**, Git, tarballs; the `ansible` package bundles many.
- **Pin** in `requirements.yml`; install with `ansible-galaxy collection install -r ... -p ./collections`.
- Search order: **playbook-adjacent `./collections`** → `collections_path` → site-packages. **First found wins**; `ansible-galaxy collection list` shows shadowing.
- Supply chain: certified content, pins, `collection verify`, signatures, private mirror, execution environments.

### Interview questions

1. **Q: What is an Ansible collection?**
   A: A versioned distribution format bundling modules, plugins, roles and playbooks under a `namespace.collection` name, installed with `ansible-galaxy` from Galaxy, Automation Hub, a private hub, Git or tarballs.
2. **Q: How do you make sure everyone uses the same collection versions?**
   A: Pin them in a committed `requirements.yml`, install into a project-local path (`-p ./collections` + `collections_path`), and in CI/AAP use the same file or an execution environment built from it.
3. **Q: Two versions of a collection are installed. Which one runs?**
   A: The first one found in the search order: playbook-adjacent `collections/`, then `collections_path` entries in order, then site-packages. Only one version is used per run.

### Scenario question (answer out loud first, then compare)

> *"Security wants control over which third-party Ansible content runs in production, and the ability to rebuild any past release exactly. Design the collection supply chain."*

**Model answer:**
1. **Approved sources only**: a private Automation Hub (or Galaxy NG) that mirrors only reviewed collections (prefer certified/vendor content). CI and AAP use it as the only Galaxy server.
2. **Pinning**: every repo has `collections/requirements.yml` with exact versions; upgrades happen in PRs with changelog review.
3. **Integrity**: signed collections where available, `ansible-galaxy collection verify` in CI, checksums recorded.
4. **Reproducibility**: build an **execution environment** image per release containing ansible-core, pinned collections and Python dependencies (topic 52); tag it with the release and store it in the registry. Re-running an old release = using its old image.
5. **Review**: code review for new collections (especially community ones), dependency scanning for their Python requirements.

**Likely follow-ups:**
- *What about internal collections?* Publish them to the same private hub with semantic versions and changelogs; consumers pin them like any other.
- *How do you handle an urgent CVE fix in a collection?* Bump the pin, run tests/Molecule, rebuild the EE, roll out through the normal pipeline quickly.
