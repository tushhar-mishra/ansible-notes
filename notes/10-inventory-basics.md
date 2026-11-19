# 10 · Inventory Basics

> **Level:** ★ foundation
>
> **You will learn:** what an **inventory** is, the two file formats (**INI** and **YAML**), **hosts**, **groups**, **child groups** (`children`), the built-in groups **`all`** and **`ungrouped`**, host **aliases** with `ansible_host`, **ranges** (`web[01:20]`), passing inventories with `-i` (file, directory, comma list), and checking what Ansible sees with **`ansible-inventory --graph / --list / --host`**. You'll write the inventory for your lab and ping all four nodes.
>
> **You need:** [09 · Lab setup](09-lab-setup.md) (nodes running), [04 · YAML](04-yaml-for-ansible.md).
>
> **Part 1 · Setup and first contact** · Next: [11 · Connection settings and ansible.cfg](11-connection-settings-and-ansible-cfg.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** The **inventory** is Ansible's address book: **which machines exist** and **how they're grouped**. Every command needs one. You target **groups** ("all web servers"), not individual IPs.

**Picture it.** A school register: every pupil (host) is listed once, and pupils belong to classes (groups) like "Year 9", "football team", "choir". A pupil can be in several groups. The teacher says "football team, go outside" instead of reading 15 names.

**The one thing to know.** `~/ansible-lab/inventory.ini`:

```ini
[web]
web1 ansible_host=172.30.0.11
web2 ansible_host=172.30.0.12

[db]
db1 ansible_host=172.30.0.21

[lb]
lb1 ansible_host=172.30.0.31
```

```bash
cd ~/ansible-lab
ansible-inventory -i inventory.ini --graph      # see the groups
ansible all -i inventory.ini -m ansible.builtin.ping   # reach every host
```

**Try it now:** create the file above and run both commands. All four hosts should answer `"ping": "pong"`.

**You've got it if you can:** add a new group `alma` containing `web1` and `db1`, and prove it with `--graph`.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

An **inventory** is a list of **managed nodes** (hosts) organised into **groups**, optionally with **variables** attached to hosts or groups. It can be:

| Kind | What it is | Topic |
|---|---|---|
| **Static** | A file you write: INI or YAML | This topic |
| **Directory** | A folder of inventory files (and `group_vars/`/`host_vars/`), all merged | 24, 39 |
| **Dynamic** | A **plugin** that asks a cloud/CMDB (Configuration Management Database) for hosts at run time, e.g. "all EC2 instances tagged role=web" | 42 |

### Hosts

A **host** is one entry: a name Ansible uses in output, optionally with an address to actually connect to.

```ini
web1 ansible_host=172.30.0.11
```

- `web1` = the **inventory hostname** (an alias). It's what you see in output and use in patterns.
- `ansible_host=172.30.0.11` = the real address (IP or DNS name). If you leave it out, Ansible connects to `web1` as a DNS name.

Using short aliases + `ansible_host` keeps output readable and lets you change IPs without renaming anything.

### Groups

```ini
[web]          ← group name in square brackets
web1           ← members, one per line
web2
```

- A host can be in **many** groups (`web` **and** `alma` **and** `production`).
- Groups usually represent **role** (web, db), **location** (dc1, eu-west-1), **environment** (staging, prod), or **OS** (alma, ubuntu).
- Group names: letters, digits and underscores. ⚠️ Avoid dashes and dots (`web-servers`): Ansible warns `Invalid characters were found in group names`, because group names also become variable-like names.

### The two built-in groups

| Group | Contains | Use |
|---|---|---|
| `all` | **Every** host in the inventory | `ansible all -m ping`; variables for everyone (`group_vars/all`) |
| `ungrouped` | Hosts not in any group except `all` | Rarely used directly |

Plus the special **implicit `localhost`**: if you target `localhost` and it's not in the inventory, Ansible still knows it and runs **locally** without SSH (you used this in topics 03 and 08).

### Child groups (groups of groups)

```ini
[app:children]     ← "app" contains GROUPS, not hosts
web
db
```

Now `app` = web1, web2, db1. Use it for layers:

```
all
├── prod
│   ├── prod_web     (web1, web2)
│   └── prod_db      (db1)
└── staging
    ├── staging_web
    └── staging_db
```

Child groups matter for **variables** too: a variable set on `prod` applies to everything inside it (topic 24).

### Ranges

```ini
[web]
web[01:20].example.com        # web01 ... web20 (keeps the leading zero)
[db]
db-[a:c].example.com          # db-a, db-b, db-c
[edge]
10.0.5.[10:19]                # 10 IPs
```

### INI vs YAML

The same inventory in both formats:

```ini
# inventory.ini
[web]
web1 ansible_host=172.30.0.11
web2 ansible_host=172.30.0.12

[db]
db1 ansible_host=172.30.0.21

[app:children]
web
db
```

```yaml
# inventory.yml
all:
  children:
    web:
      hosts:
        web1:
          ansible_host: 172.30.0.11
        web2:
          ansible_host: 172.30.0.12
    db:
      hosts:
        db1:
          ansible_host: 172.30.0.21
    app:
      children:
        web:
        db:
```

| | INI | YAML |
|---|---|---|
| Readability for small lists | ✅ Very easy | More lines |
| Variables with types (lists, numbers, booleans) | ⚠️ Inline values are strings (mostly) | ✅ Real YAML types |
| Nested groups | `[group:children]` | `children:` key |
| Common in | Labs, quick setups, old docs | Larger projects, generated inventories |

In YAML, `hosts:` and `children:` are **maps** (not lists). `web1:` with nothing after it is fine: a host with no variables (value = null). Same for `web:` under `app.children`.

### Variables in the inventory (a preview)

You can attach variables right in the inventory:

```ini
[web]
web1 ansible_host=172.30.0.11 http_port=8080     ← host variable

[web:vars]                                       ← group variables for every host in web
http_port=80
```

This works, but in real projects variables go into separate `group_vars/` and `host_vars/` files (topic 24). Keep the inventory for *who exists*, and variable files for *how they're configured*.

---

## Why it exists

Without an inventory, every command would need a list of IPs, and you'd repeat "these are the web servers" in every script. With an inventory:

- **One source of truth** for what machines exist.
- **Target by meaning**: `web`, `prod`, `eu_west`, not by IP.
- **Change once**: add a server to `[web]` and every playbook that targets `web` includes it.
- **Per-group settings**: different users, ports, variables per group.
- **Safety**: separate inventories per environment (`inventories/staging`, `inventories/prod`, topic 39) make "oops, I ran that on prod" much harder.

---

## Analogy

**Contacts on your phone.** Each contact (host) has a name you recognise ("Mum") and a real number underneath (`ansible_host`). You make groups ("Family", "Work", "Football"), and a contact can be in several. "Send to Family" reaches everyone in that group. "All contacts" is `all`. A group of groups ("Everyone I know in London" = "Work" + "Football") is `children`.

---

## How it works inside

What Ansible does with `-i inventory.ini` at the start of every run:

```
 -i inventory.ini
      │
      ▼ 1. pick an INVENTORY PLUGIN that can read this source
 enabled plugins tried in order: host_list, script, auto, yaml, ini, toml
      │   ("ini" accepts it; "yaml" would accept a .yml file; "auto" handles dynamic plugin configs)
      ▼ 2. parse → build an in-memory model
 groups {all, ungrouped, web, db, lb, app} ; hosts {web1, web2, db1, lb1}
 parent/child links ; variables attached to hosts and groups
      │
      ▼ 3. reconcile
 every group ends up under "all"; hosts in no group go to "ungrouped"
      │
      ▼ 4. later: your PATTERN ("web", "app:!db", "all") selects hosts from this model
```

- Several `-i` options (or a directory) = several sources merged into **one** model.
- Nothing is contacted during this step. The inventory is just data. A host that's down still appears; it fails only when Ansible tries to connect.

---

## Code / commands

### See the group tree

```bash
ansible-inventory -i inventory.ini --graph
```
```
@all:
  |--@ungrouped:
  |--@web:
  |  |--web1
  |  |--web2
  |--@db:
  |  |--db1
  |--@lb:
  |  |--lb1
  |--@app:
  |  |--@web:
  |  |  |--web1
  |  |  |--web2
  |  |--@db:
  |  |  |--db1
```

- `@` marks a **group**; plain names are hosts.
- `web` appears twice: under `all` and as a child of `app`. Same group, two parents.
- `ungrouped` is empty: every host is in at least one group.

### See everything Ansible knows (JSON)

```bash
ansible-inventory -i inventory.ini --list
```
```json
{
    "_meta": {
        "hostvars": {
            "db1":  { "ansible_host": "172.30.0.21" },
            "lb1":  { "ansible_host": "172.30.0.31" },
            "web1": { "ansible_host": "172.30.0.11" },
            "web2": { "ansible_host": "172.30.0.12" }
        }
    },
    "all": { "children": ["ungrouped", "web", "db", "lb", "app"] },
    "app": { "children": ["web", "db"] },
    "db":  { "hosts": ["db1"] },
    "lb":  { "hosts": ["lb1"] },
    "web": { "hosts": ["web1", "web2"] }
}
```

- `_meta.hostvars`: every variable for every host, after merging. Great for debugging "what value does web1 get?"
- This JSON format is also what dynamic inventory scripts must print (topic 42).

### One host's variables

```bash
ansible-inventory -i inventory.ini --host web1
```
```json
{
    "ansible_host": "172.30.0.11"
}
```

### Convert INI to YAML automatically

```bash
ansible-inventory -i inventory.ini --list --yaml > inventory.yml
```

`--yaml` prints the YAML form (with `--list`). Handy for migrating.

### List which hosts a pattern selects

```bash
ansible app --list-hosts -i inventory.ini
```
```
  hosts (3):
    web1
    web2
    db1
```

- `--list-hosts`: shows the selection **without** connecting. Use it before any risky command. Topic 12 covers patterns (`web:db`, `app:!db`, `web[0]`).

### Ping everything

```bash
ansible all -i inventory.ini -m ansible.builtin.ping
```
```
web1 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
web2 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3"
    },
    "changed": false,
    "ping": "pong"
}
db1 | SUCCESS => { ... }
lb1 | SUCCESS => { ... }
```

- Hosts may print in any order: they run **in parallel**.
- `discovered_interpreter_python`: interpreter discovery from topic 07, done on first contact.
- It worked without `ansible_user` or a key in the inventory because Ansible runs OpenSSH, which read your `~/.ssh/config` block for `172.30.0.*` (topic 09). Topic 11 puts these settings into Ansible itself, so they work on any machine.

### Ways to pass an inventory

| How | Example | Notes |
|---|---|---|
| A file | `-i inventory.ini` | Most common |
| A directory | `-i inventories/staging/` | All files inside are merged (ignores some extensions like `.retry`, `~`) |
| Several sources | `-i base.ini -i extra.yml` | Merged |
| A comma list | `-i '172.30.0.11,172.30.0.12,'` | Quick tests. ⚠️ The **trailing comma** tells Ansible it's a list, not a file name |
| Configured default | `inventory = ./inventory.ini` in `ansible.cfg` | Then you can drop `-i` (topic 11) |
| Nothing | `/etc/ansible/hosts` | The old global default. Usually absent in a venv install |

---

## Hands-on lab

1. In `~/ansible-lab`, create `inventory.ini`:

   ```ini
   [web]
   web1 ansible_host=172.30.0.11
   web2 ansible_host=172.30.0.12

   [db]
   db1 ansible_host=172.30.0.21

   [lb]
   lb1 ansible_host=172.30.0.31

   [app:children]
   web
   db

   [alma]
   web1
   db1

   [ubuntu]
   web2
   lb1
   ```

2. Explore it:

   ```bash
   ansible-inventory -i inventory.ini --graph
   ansible-inventory -i inventory.ini --graph alma        # just one group's tree
   ansible-inventory -i inventory.ini --host db1
   ansible app --list-hosts -i inventory.ini
   ```

3. Ping: `ansible all -i inventory.ini -m ansible.builtin.ping`. All four `SUCCESS`.

4. Generate the YAML version and diff the views:

   ```bash
   ansible-inventory -i inventory.ini --list --yaml > inventory.yml
   diff <(ansible-inventory -i inventory.ini --list) <(ansible-inventory -i inventory.yml --list) && echo "identical"
   ```

5. Mistakes to see once:
   - Add `[web-servers]` with `web1` under it → note the `Invalid characters were found in group names` warning. Remove it.
   - Run `ansible webb -i inventory.ini -m ansible.builtin.ping` (typo) → `Could not match supplied host pattern, ignoring: webb` and `No hosts matched, nothing to do`. Ansible **doesn't fail** on a pattern typo, so check `--list-hosts` in scripts.
   - Run `ansible all -i inventry.ini -m ansible.builtin.ping` (file typo) → `Unable to parse ... as an inventory source` and only `localhost` is available.

6. A host that doesn't exist: add `ghost1 ansible_host=172.30.0.99` to `[web]` and ping `web`. You'll get `UNREACHABLE!` for ghost1 only; the others still succeed. Remove it.

**Done when:** `--graph` shows `web`, `db`, `lb`, `app`, `alma`, `ubuntu`, and `ansible all -m ping` returns four `SUCCESS`.

---

## Where you'll use it

- **Every run**: `-i inventories/prod` vs `-i inventories/staging` is how teams separate environments.
- **Targeting**: "patch only `eu_db`", "deploy to `canary` first".
- **Cloud**: dynamic inventory builds groups from tags (topic 42), so new servers join `web` automatically.
- **Handover from Terraform**: an inventory file generated from `terraform output` (topic 43).
- **Debugging**: `ansible-inventory --host X` answers "what does Ansible think this host is?"

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `[WARNING]: provided hosts list is empty, only localhost is available` | No inventory found (no `-i`, no `ansible.cfg` setting) | Pass `-i` or set `inventory =` in `ansible.cfg` |
| `Unable to parse /path as an inventory source` | Wrong path, or a syntax error in the file | Check the path; run `ansible-inventory -i file --graph` to see the parse error |
| `Could not match supplied host pattern, ignoring: X` | Typo or group not in this inventory | `ansible-inventory --graph`; `--list-hosts` |
| `Invalid characters were found in group names` | Dashes/dots in group names | Use underscores: `web_servers` |
| `UNREACHABLE!` for one host | Wrong `ansible_host`, host down, SSH problem | `ssh` to it by hand (topic 05) |
| Output shows the IP instead of a name | Host listed by IP with no alias | `web1 ansible_host=10.0.0.5` |
| YAML inventory: `Invalid "hosts" entry for "web" group, requires a dictionary, found "<class 'list'>"` | Wrote hosts as a list (`- web1`) | `hosts:` is a **map**: `web1:` |
| A host gets an unexpected variable value | Same host in several groups with different values | `ansible-inventory --host`; topic 24 (group precedence) |

---

## Quick recap

- Inventory = **hosts** + **groups** (+ optional variables). Static (INI/YAML), directory, or dynamic (plugin).
- Built-ins: **`all`**, **`ungrouped`**, and implicit **`localhost`**.
- `name ansible_host=IP` gives a readable alias with a real address. `[group:children]` (or `children:`) nests groups.
- Check before you run: `ansible-inventory --graph`, `--list`, `--host X`, and `ansible PATTERN --list-hosts`.
- Keep variables out of the inventory file in real projects: `group_vars/`/`host_vars/` (topic 24).

### Interview questions

1. **Q: What is an Ansible inventory?**
   A: The list of managed hosts and their groups (and optionally variables). It can be a static INI/YAML file, a directory of sources, or a dynamic inventory plugin that queries a cloud or CMDB.
2. **Q: What are `all` and `ungrouped`?**
   A: Built-in groups: `all` contains every host; `ungrouped` contains hosts not in any other group.
3. **Q: How do you see what Ansible thinks the inventory looks like?**
   A: `ansible-inventory -i SRC --graph` for the tree, `--list` for full JSON with merged host variables, `--host NAME` for one host.

### Scenario question (answer out loud first, then compare)

> *"Someone ran a patch playbook 'against staging' but it hit production database servers too. How could that happen, and how do you structure inventories to prevent it?"*

**Model answer:**
1. Likely causes: one shared inventory containing both environments, and a pattern like `db` that matched **all** db hosts; or the wrong `-i` path; or a default `inventory =` in someone's `ansible.cfg` pointing at prod.
2. Structure: **one inventory directory per environment** (`inventories/staging/`, `inventories/prod/`), each with its own `group_vars`. A playbook can only touch prod if you explicitly pass `-i inventories/prod`.
3. Guardrails: no default inventory pointing at prod; CI pipelines run prod only from a protected branch with approval (topics 53–54); `--list-hosts` / `--check` step before applying; prod credentials (SSH key, vault password) only available to the prod pipeline.
4. In the playbook: an `assert` that the environment variable matches what was requested (topic 31).

**Likely follow-ups:**
- *How do you name groups for environment + role?* Either `prod_web` style groups, or separate inventories per environment with the same role groups (`web`, `db`) inside. The second keeps playbooks identical across environments.
- *How would a dynamic inventory help?* Groups come from cloud tags (`env=prod`, `role=db`), and separate credentials per account make crossing environments impossible.
