# 24 · Inventory Variables: group_vars and host_vars

> **Level:** ★ foundation
>
> **You will learn:** where per-group and per-host variables really belong: the **`group_vars/`** and **`host_vars/`** directories, file vs directory form (`group_vars/web.yml` or `group_vars/web/*.yml`), the special **`all`** group, inventory-side vs playbook-side locations, and **which group wins** when a host is in several groups (depth, then `ansible_group_priority`, then alphabetical). You'll refactor the lab so per-distro differences (package names, paths, service names) live in variables instead of duplicated `when:` tasks.
>
> **You need:** [23 · Facts and magic variables](23-facts-and-magic-variables.md), [10 · Inventory basics](10-inventory-basics.md), [21 · Variables basics](21-variables-basics.md).
>
> **Part 3 · Variables, facts and logic** · Next: [25 · Variable precedence](25-variable-precedence.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Put variables in **files named after groups and hosts**, next to your inventory. Ansible loads them automatically: every host gets the variables of every group it belongs to, plus its own `host_vars` file.

**Picture it.** A company handbook: a chapter for **everyone** (`all`), chapters per **department** (`web`, `db`), and a sticky note for **one person** (`host_vars/web1.yml`). The more specific page wins.

**The one thing to know.**

```
~/ansible-lab/
├── inventory.ini
├── group_vars/
│   ├── all.yml          ← every host
│   ├── web.yml          ← hosts in [web]
│   ├── alma.yml         ← hosts in [alma]
│   └── ubuntu.yml       ← hosts in [ubuntu]
└── host_vars/
    └── web1.yml         ← only web1 (beats all groups)
```

```bash
ansible-inventory --host web1          # every inventory variable web1 gets, merged
```

**Try it now:** create `group_vars/all.yml` with `ntp_server: pool.ntp.org` and `host_vars/web1.yml` with `ntp_server: time.example.com`, then compare `ansible-inventory --host web1` and `--host web2`.

**You've got it if you can:** predict which value a host gets when two of its groups set the same variable.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### The two directories

| Directory | Files inside | Applies to |
|---|---|---|
| `group_vars/` | `<group>.yml` (or `.yaml`, `.json`, no extension) **or** a directory `<group>/` with any number of YAML files | Every host in that group (and its child groups) |
| `host_vars/` | `<inventory_hostname>.yml` **or** a directory `<inventory_hostname>/` | Exactly that host |

The file name must match the **group name** or **inventory hostname** exactly (`web1`, not `172.30.0.11`).

**File or directory form?**

```
group_vars/web.yml                 group_vars/web/
                                   ├── main.yml        ← normal settings
                                   └── vault.yml       ← encrypted secrets (topic 40)
```

The directory form lets you split by topic and keep secrets in their own (encrypted) file. All files in the directory are loaded and merged (in alphabetical order).

### Where Ansible looks

Two locations, both loaded:

| Location | Relative to | Typical use |
|---|---|---|
| **Inventory-side** | The directory containing the inventory file (or the inventory directory itself) | ✅ Environment-specific values: `inventories/prod/group_vars/` |
| **Playbook-side** | The directory containing the playbook | Values shared by all environments that use this playbook |

If both define the same variable for the same group, **playbook-side wins**. Most teams use **inventory-side only**, so each environment's inventory folder carries its own values (topic 39).

In the lab, `inventory.ini` and the playbooks are in the same folder (`~/ansible-lab`), so `~/ansible-lab/group_vars/` is both at once. That's fine.

### `all`

`group_vars/all.yml` (or `all/`) applies to **every host**. Put global defaults here: NTP servers, DNS servers, the company domain, the automation user.

### Inline inventory variables vs files

```ini
[web:vars]
http_port=80            ← inline in the inventory file
```

Works, but:

- inline values in INI are (mostly) **strings**;
- mixing "who exists" with "how it's configured" makes the inventory hard to read;
- `group_vars/` files **beat** inline inventory group vars.

Use `group_vars/` and `host_vars/`. Keep the inventory for hosts and groups.

### When a host is in several groups

web1 is in `all`, `app`, `web`, `alma`. If two of those groups set the same variable, Ansible merges them in a fixed order; **the last one applied wins**:

```
1. all                           (always first: lowest)
2. other groups, sorted by:
     a. DEPTH in the group tree (parents before children → children win)
     b. ansible_group_priority  (higher number applied later → wins; default 1)
     c. NAME alphabetically      (later name wins)
3. host_vars                     (always last: highest)
```

For the lab inventory:

```
all            depth 0
├── app        depth 1
│   ├── web    depth 2   ← child of app, deeper
│   └── db     depth 2
├── lb         depth 1
├── alma       depth 1
└── ubuntu     depth 1
```

| Host | Groups (in merge order) | If `web` and `alma` both set `x` | If `lb` and `ubuntu` both set `x` |
|---|---|---|---|
| web1 | all → alma, app (depth 1, alphabetical) → web (depth 2) | **web** wins (deeper) | — |
| lb1 | all → lb, ubuntu (depth 1, alphabetical) | — | **ubuntu** wins (`u` after `l`) |

Alphabetical order is fragile ("we renamed a group and prod changed"). Two good ways out:

1. **Don't set the same variable in sibling groups.** Give each group its own job: OS groups set OS things (`pkg_cron`), role groups set role things (`http_port`).
2. If you must, set **`ansible_group_priority`** for a group **in the inventory** (it can't be set from `group_vars`):

```ini
[lb:vars]
ansible_group_priority=10
```

Now `lb` is applied after `ubuntu` and wins for lb1.

---

## Why it exists

- **Separation of concerns**: the inventory says *who*, group/host vars say *how they're configured*, playbooks say *what to do*.
- **Environments**: `inventories/staging/group_vars/web.yml` and `inventories/prod/group_vars/web.yml` hold different values for the **same** playbook.
- **No duplicated tasks**: instead of two tasks with `when: os_family == ...`, one task uses `{{ pkg_cron }}`, and the value comes from `group_vars/alma.yml` or `ubuntu.yml`.
- **Reviewable**: a change to prod's web settings is a one-line diff in one file.

---

## Analogy

**Dress code at a company.**

- `all` = company-wide rule: "smart casual".
- `group_vars/sales` = "sales wear suits" (overrides company-wide for salespeople).
- A person in two departments ("sales" and "IT")? HR has a rule: the more **specific sub-department** wins; if equal, the one with a higher **priority** flag; if still equal, the one listed **later in the alphabet**. That last rule works but nobody remembers it.
- `host_vars/priya` = "Priya has a medical exemption: trainers allowed". Always wins.

---

## How it works inside

```
 ansible-playbook -i inventory.ini web.yml
   │
   ├─ inventory plugin parses inventory.ini → hosts, groups, inline vars
   ├─ vars plugin "host_group_vars" (built in) runs for each host and group:
   │      <inventory dir>/group_vars/<group>.yml or <group>/*
   │      <inventory dir>/host_vars/<host>.yml or <host>/*
   │      <playbook dir>/group_vars/..., <playbook dir>/host_vars/...
   │   (loaded when needed; results cached for the run)
   │
   └─ for each host: merge
        all → groups (depth, priority, name) → host
        inline inventory vars < inventory-side files < playbook-side files
      → that host's inventory-level variables (then merged with play vars, facts, -e..., topic 25)
```

The thing that loads these files is a **vars plugin** (`host_group_vars`). Collections can add others (e.g. reading variables from a CMDB), but this built-in one is what you'll use.

---

## Code / commands

### Refactor the lab with OS variables

`~/ansible-lab/group_vars/all.yml`:

```yaml
---
ntp_server: pool.ntp.org
admin_group_name: admins
web_page_text: "Hello from {{ inventory_hostname }}"
```

`~/ansible-lab/group_vars/alma.yml`:

```yaml
---
pkg_cron: cronie
svc_cron: crond
svc_ssh: sshd
web_root: /usr/share/nginx/html
```

`~/ansible-lab/group_vars/ubuntu.yml`:

```yaml
---
pkg_cron: cron
svc_cron: cron
svc_ssh: ssh
web_root: /var/www/html
```

`~/ansible-lab/group_vars/web.yml`:

```yaml
---
http_port: 80
web_packages:
  - nginx
```

`~/ansible-lab/host_vars/web1.yml`:

```yaml
---
web_page_text: "web1 is the canary"
```

Now `web.yml` from topic 14 shrinks (no duplicated `when` tasks for paths):

`~/ansible-lab/web-vars.yml`:

```yaml
---
- name: Web servers run nginx (values from group_vars)
  hosts: web
  become: true
  tasks:
    - name: apt cache is fresh (Debian family only)
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600
      when: ansible_facts['os_family'] == 'Debian'

    - name: Web packages are installed
      ansible.builtin.package:
        name: "{{ web_packages }}"
        state: present

    - name: Home page is in place
      ansible.builtin.copy:
        dest: "{{ web_root }}/index.html"
        content: "{{ web_page_text }}\n"
        mode: "0644"

    - name: nginx is running and enabled
      ansible.builtin.systemd_service:
        name: nginx
        state: started
        enabled: true
```

- One home-page task instead of two: `web_root` comes from `alma.yml` or `ubuntu.yml`.
- `web_page_text` comes from `all.yml`, except on web1 where `host_vars/web1.yml` overrides it.
- The apt task keeps its `when`: refreshing the apt cache is an action, not a value.

### Check what each host gets

```bash
ansible-inventory --host web1
```
```json
{
    "admin_group_name": "admins",
    "ansible_host": "172.30.0.11",
    "http_port": 80,
    "ntp_server": "pool.ntp.org",
    "pkg_cron": "cronie",
    "svc_cron": "crond",
    "svc_ssh": "sshd",
    "web_packages": [
        "nginx"
    ],
    "web_page_text": "web1 is the canary",
    "web_root": "/usr/share/nginx/html"
}
```

```bash
ansible-inventory --host lb1
```
```json
{
    "admin_group_name": "admins",
    "ansible_host": "172.30.0.31",
    "ntp_server": "pool.ntp.org",
    "pkg_cron": "cron",
    "svc_cron": "cron",
    "svc_ssh": "ssh",
    "web_page_text": "Hello from {{ inventory_hostname }}",
    "web_root": "/var/www/html"
}
```

- lb1 has no `http_port` or `web_packages`: it's not in `web`.
- `web_page_text` shows the **raw** template: `ansible-inventory` doesn't render Jinja2. It's rendered when a task uses it.

Show variables in the group tree:

```bash
ansible-inventory --graph --vars web
```
```
@web:
  |--web1
  |  |--{ansible_host = 172.30.0.11}
  |  |--{web_page_text = web1 is the canary}
  |--web2
  |  |--{ansible_host = 172.30.0.12}
  |--{http_port = 80}
  |--{web_packages = ['nginx']}
```

### See group order decide a conflict

Add `motd_owner: ubuntu-team` to `group_vars/ubuntu.yml` and create `group_vars/lb.yml` with `motd_owner: lb-team`:

```bash
ansible-inventory --host lb1 | grep motd_owner     # "ubuntu-team" (u > l alphabetically)
```

Now give `lb` priority in `inventory.ini`:

```ini
[lb:vars]
ansible_group_priority=10
```

```bash
ansible-inventory --host lb1 | grep motd_owner     # "lb-team"
```

### Use the variables for the cron tasks too

```yaml
- name: cron is installed
  ansible.builtin.package:
    name: "{{ pkg_cron }}"
    state: present

- name: cron is running and enabled
  ansible.builtin.systemd_service:
    name: "{{ svc_cron }}"
    state: started
    enabled: true
```

Two tasks instead of four, and no `when:`. (Topic 36 shows another common pattern for roles: `include_vars` with a file per OS family.)

---

## Hands-on lab

1. Create the five variable files above and `web-vars.yml`. Run it twice: `changed=0` on the second run (the page text for web1 changes once).
2. `curl -s 172.30.0.11` → "web1 is the canary"; `curl -s 172.30.0.12` → "Hello from web2".
3. Rewrite the cron part of `base.yml` (topic 17) with `pkg_cron`/`svc_cron`. Remove the four `when` tasks. Run: `changed=0`.
4. Convert `group_vars/web.yml` into a directory `group_vars/web/main.yml` and add `group_vars/web/extra.yml` with `http_extra_header: X-Lab`. Check `ansible-inventory --host web2`.
5. Reproduce the `lb`/`ubuntu` conflict and fix it twice: once with `ansible_group_priority`, once by renaming the variable so the groups don't overlap. Decide which you'd prefer in a team (the second).
6. Put a variable in `[web:vars]` inside `inventory.ini` **and** in `group_vars/web.yml` with a different value. Which wins? (The file.)

**Done when:** your playbooks contain no per-distro path/name duplication and `ansible-inventory --host` shows exactly what you expect for every host.

---

## Where you'll use it

- **Per-environment config**: `inventories/prod/group_vars/all.yml` vs `inventories/staging/...` (topic 39).
- **Per-role settings**: `group_vars/db.yml` with database tuning, `group_vars/web.yml` with web settings.
- **OS differences**: `group_vars/<os_group>.yml` or dynamic OS groups from cloud tags or facts (topic 42).
- **One-off hosts**: `host_vars/legacy-db.yml` with special ports or an old Python path.
- **Secrets next to settings**: `group_vars/prod/vault.yml` (encrypted, topic 40).

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| Variables in `group_vars/web.yml` ignored | `group_vars/` is not next to the inventory **or** the playbook you're using | Move it, or check which inventory path is in use (`ansible-config dump`) |
| `host_vars/172.30.0.11.yml` ignored | File named after the IP, not the inventory hostname | Name it `web1.yml` |
| Unexpected value on a host in two groups | Sibling groups both set it; alphabetical order decided | Avoid overlaps; `ansible_group_priority` in the inventory |
| `ansible_group_priority` in `group_vars` has no effect | It's only honoured in the inventory source | Set it in the inventory file |
| A whole dict replaced instead of merged | Variables replace, they don't deep-merge (by default) | Split into separate variables, or `combine` explicitly (topic 32). ⚠️ `hash_behaviour = merge` is discouraged |
| Strings where you expected numbers | Inline INI inventory vars | Move to YAML `group_vars` |
| `ansible-inventory --host` shows `{{ ... }}` | Not rendered there (lazy) | Normal; check the rendered value with a `debug` task |
| Secret file in `group_vars/web/` shows as garbage in `ansible-inventory` | It's vault-encrypted | Provide the vault password (topic 40) |

---

## Quick recap

- `group_vars/<group>.yml` (or `<group>/*.yml`) and `host_vars/<host>.yml`, next to the **inventory** (preferred) or the **playbook** (wins on conflict).
- `group_vars/all` = every host. Host vars beat all group vars.
- Several groups: `all` first, then by **depth** (children win), **`ansible_group_priority`** (inventory only), then **name** (alphabetical); `host_vars` last.
- Replace duplicated `when:` tasks with variables per OS/role group.
- Check with `ansible-inventory --host X` and `--graph --vars`.

### Interview questions

1. **Q: Where do you put variables for a group of hosts?**
   A: In `group_vars/<groupname>.yml` (or a `group_vars/<groupname>/` directory), next to the inventory or the playbook; Ansible loads them automatically.
2. **Q: A host is in `web` and `eu`; both set `ntp_server`. Which wins?**
   A: Groups are merged by depth (children over parents), then `ansible_group_priority`, then alphabetically. If both are at the same depth with the same priority, `web` wins because it sorts after `eu`. `host_vars` would beat both.
3. **Q: How do you see the final inventory variables for a host?**
   A: `ansible-inventory -i <inv> --host <name>`.

### Scenario question (answer out loud first, then compare)

> *"A new group `pci` was added for servers in PCI (payment card) scope, with `ssh_allow_tcp_forwarding: false`. Some PCI web servers still allow forwarding after the run. What do you investigate?"*

**Model answer:**
1. Check what the hosts actually get: `ansible-inventory --host web7 | grep forwarding`.
2. Likely causes:
   - another group at the same or deeper level sets the variable too (e.g. `web` at depth 2 because it's a child of `app`, while `pci` is depth 1, so `web` wins);
   - alphabetical order (`web` > `pci`);
   - a `host_vars` file for those hosts sets it;
   - a playbook-side `group_vars` overrides the inventory-side one;
   - a higher-precedence source (play vars, role vars, `-e`) sets it (topic 25).
3. Fix: for security-critical settings, don't rely on group ordering. Either set `ansible_group_priority` for `pci` in the inventory, or (better) use a dedicated variable that only `pci` sets (`pci_mode: true`) and have the role enforce `false` when `pci_mode` is true. Add an `assert` in the play that fails if a PCI host ends up with forwarding enabled.

**Likely follow-ups:**
- *Why is "depth" surprising here?* Group depth comes from the group tree in the inventory, not from importance. A deeply nested functional group can outrank a flat compliance group.
- *How would you test this?* A CI job that runs `ansible-inventory --host` for representative hosts and checks critical values.
