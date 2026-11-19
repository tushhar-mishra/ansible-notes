# 32 · Advanced Jinja2: Data Wrangling and Lookups

> **Level:** ★★ intermediate
>
> **You will learn:** the filters that reshape data (`map`, `select`/`reject`, **`selectattr`**/`rejectattr`, **`combine`**, `extract`, `zip`, `groupby`, `dict2items`/`items2dict`, `from_json`/`from_yaml`, `regex_findall`, `community.general.json_query`, `ansible.utils.ipaddr`), how to chain them into one-line answers ("the IPs of all web hosts", "users with sudo", "merge defaults with overrides"), and **lookup plugins** that fetch data **on the control node** (`file`, `env`, `template`, `pipe`, **`password`**, `first_found`, `fileglob`, `vars`, `url`, `csvfile`), plus where they're dangerous.
>
> **You need:** [31 · Error handling](31-error-handling.md), [22 · Jinja2 basics](22-jinja2-basics.md), [28 · Loops](28-loops.md), [23 · hostvars](23-facts-and-magic-variables.md).
>
> **Part 3 · Variables, facts and logic** · Next: [33 · Tags](33-tags.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Real data is lists of dictionaries (users, hosts, mounts, API results). A few filters, chained with `|`, let you **pick**, **reshape** and **merge** that data without writing loops. **Lookups** pull data from the control node: files, environment variables, generated passwords.

**Picture it.** A production line: `selectattr` is the inspector who keeps only items matching a rule, `map` is the machine that takes one part off each item, `join` packs them into a box.

**The one thing to know.**

```yaml
web_ips:   "{{ groups['web'] | map('extract', hostvars, 'ansible_host') | list }}"
sudo_names: "{{ people | selectattr('sudo', 'defined') | selectattr('sudo') | map(attribute='name') | list }}"
settings:  "{{ defaults | combine(overrides, recursive=true) }}"
api_data:  "{{ result.stdout | from_json }}"
db_pass:   "{{ lookup('ansible.builtin.password', 'credentials/db_pass length=24') }}"
home:      "{{ lookup('ansible.builtin.env', 'HOME') }}"
```

**Try it now:**

```bash
cd ~/ansible-lab
ansible localhost -m ansible.builtin.debug -a "msg={{ groups['web'] | map('extract', hostvars, 'ansible_host') | list }}"
```

**You've got it if you can:** get a list of one attribute from a list of dicts, filtered by another attribute, in one expression.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Selecting and transforming lists

| Filter | Does | Example → result |
|---|---|---|
| `map(attribute='x')` | Take attribute `x` from each item | `users \| map(attribute='name')` → `['alice', 'bob']` |
| `map('filter', args)` | Apply a filter to each item | `['A','B'] \| map('lower')` → `['a','b']` |
| `map('extract', container)` | Look each item up in a dict/list | `['web1','web2'] \| map('extract', hostvars, 'ansible_host')` → IPs |
| `select('test', arg)` | Keep items passing a test | `[1,5,9] \| select('gt', 4)` → `[5, 9]` |
| `reject('test', arg)` | Drop items passing a test | `['a','','b'] \| reject('equalto', '')` → `['a','b']` |
| `selectattr('x', 'test', arg)` | Keep dicts whose attribute passes a test | `users \| selectattr('shell', 'equalto', '/bin/zsh')` |
| `selectattr('x')` | Keep dicts whose attribute is **true** | `users \| selectattr('sudo')` |
| `selectattr('x', 'defined')` | Keep dicts that **have** attribute `x` | (do this first, or missing keys error) |
| `rejectattr(...)` | Opposite of selectattr | |
| `unique`, `sort(attribute='x')`, `reverse` | Clean up | `users \| sort(attribute='uid')` |
| `groupby('x')` | Group dicts by an attribute → list of `[key, items]` | `hosts \| groupby('os')` |
| `zip(other)` | Pair two lists | `['a','b'] \| zip([1,2]) \| list` → `[['a',1],['b',2]]` |
| `batch(n)` / `slice(n)` | Chunks | `range(10) \| batch(3) \| list` |
| `sum(attribute='x')` | Total | `disks \| sum(attribute='size_gb')` |

Note: `map`, `select`, `selectattr`, `zip`... return **generators**; add `| list` when you need an actual list (to print, index or store it). In `loop:`, Ansible accepts them directly, but `| list` never hurts.

**Tests** commonly used with `select`/`selectattr`: `equalto`, `eq`, `ne`, `gt`, `ge`, `lt`, `le`, `in`, `match`, `search`, `defined`, `undefined`, `none`, `truthy`, `falsy`, `version`.

### Dictionaries

| Filter | Does | Example |
|---|---|---|
| `combine(other)` | Merge: keys of `other` win (shallow) | `{'a':1,'b':2} \| combine({'b':3})` → `{'a':1,'b':3}` |
| `combine(other, recursive=true)` | Deep merge nested dicts | Merge nested settings |
| `combine(..., list_merge='append')` | How to merge lists inside (`replace` default, `append`, `prepend`, `keep`, `append_rp`, `prepend_rp`) | |
| `dict2items` / `items2dict` | Dict ⇄ list of `{key, value}` (topic 28) | |
| `dict(list_of_pairs)` | Build a dict | `dict(['a','b'] \| zip([1,2]))` → `{'a':1,'b':2}` |
| `community.general.dict_kv('k')` | Wrap a value as `{k: value}` | |
| `.keys()`, `.values()`, `.items()` | Python dict methods (fine in expressions and templates) | `settings.keys() \| list` |

`combine` is how you get the "merge, don't replace" behaviour from topic 25's trap:

```yaml
nginx_defaults:  { worker_processes: auto, gzip: true, limits: { body: 1m, rate: 10r/s } }
nginx_overrides: { gzip: false, limits: { body: 50m } }
nginx: "{{ nginx_defaults | combine(nginx_overrides, recursive=true) }}"
# → { worker_processes: auto, gzip: false, limits: { body: 50m, rate: 10r/s } }
```

### Parsing text and data

| Filter | Use |
|---|---|
| `from_json`, `from_yaml`, `from_yaml_all` | Turn command/API output into data |
| `to_json`, `to_nice_json`, `to_yaml`, `to_nice_yaml(indent=2)` | Write data into files |
| `split(sep)` | String → list (Python `.split()` also works) |
| `regex_search(pat, '\\1')`, `regex_findall(pat)`, `regex_replace(pat, rep)` | Extract/replace with regexes |
| `community.general.json_query(expr)` | JMESPath queries on nested data (needs the `jmespath` Python library on the **control node**) |
| `ansible.utils.ipaddr`, `ipv4`, `ipaddr('network')`, `ipaddr('host/prefix')`, `nthhost` | IP address maths (needs the `netaddr` library on the control node) |
| `to_datetime(fmt)`, `strftime(fmt)` | Dates (`'%Y-%m-%d' \| strftime` uses "now" if no epoch given) |
| `community.general.version_sort` | Sort version strings properly |

### Lookups

A **lookup plugin** fetches data **on the control node** when the expression is evaluated. Call with `lookup('name', args...)` (returns a string, comma-joined if several) or `query('name', args...)` (always a list, topic 28).

| Lookup (FQCN `ansible.builtin.`) | Returns | Example |
|---|---|---|
| `file` | Contents of a local file | `lookup('ansible.builtin.file', 'files/id.pub')` |
| `template` | A rendered local template (topic 29) | `lookup('ansible.builtin.template', 'snippet.j2')` |
| `env` | A local environment variable | `lookup('ansible.builtin.env', 'BUILD_NUMBER', default='0')` |
| `pipe` | Output of a **local** command | `lookup('ansible.builtin.pipe', 'git rev-parse --short HEAD')` |
| `password` | A random password, **saved to a local file** the first time and re-read afterwards | `lookup('ansible.builtin.password', 'credentials/' ~ inventory_hostname ~ '/app length=24 chars=ascii_letters,digits')` |
| `first_found` | The first file that exists from a list | OS-specific vars files (topic 36) |
| `fileglob` | Local files matching a pattern | `query('ansible.builtin.fileglob', 'files/conf.d/*.conf')` |
| `vars` / `varnames` | A variable by (computed) name / names matching a regex | `lookup('ansible.builtin.vars', 'port_' ~ env)` (topic 22) |
| `url` | Content of a URL (fetched by the control node) | `lookup('ansible.builtin.url', 'https://example.com/keys', split_lines=False)` |
| `csvfile`, `ini` | A value from a local CSV/INI file | |
| `unvault` | Decrypted contents of a vaulted file (topic 40) | |
| `community.hashi_vault.hashi_vault`, `amazon.aws.aws_secret` | Secrets from external stores (topic 41) | |

Key facts about lookups:

- They run on the **control node**, never on the managed host. `lookup('file', '/etc/hostname')` reads **your** machine's file. To read a file on the target, use `slurp` or `command` + `register`.
- They're evaluated **every time** the expression is used (lazily). A `pipe` lookup in a variable used by 50 tasks runs 50 times per host. Store results with `set_fact` if expensive.
- `password` is idempotent only because it stores the value in a file. **Protect or vault that file** (or use a secret manager, topic 41).

---

## Why it exists

Infrastructure data comes in nested shapes: facts, inventory, API responses, CSV exports. Without these filters you'd write loops with `set_fact` accumulating lists, which is slow, verbose and hard to read. With them, each "question about the data" is one expression you can test in the playground first.

Lookups exist because some inputs live **next to the code**, not on the servers: public keys in the repo, the Git commit being deployed, CI environment variables, generated secrets.

---

## Analogy

**A spreadsheet's filter-and-formula tools.**

- `selectattr` = the column filter ("show only rows where Sudo = Yes").
- `map(attribute=...)` = copying one column.
- `combine` = "paste values, overwriting cells that differ" onto a template sheet.
- `groupby` = a pivot table.
- `json_query` = an advanced query box.
- Lookups = importing data from **another file on your own desk** (not from the remote office's files).

---

## How it works inside

```
 "{{ people | selectattr('sudo', 'defined') | selectattr('sudo') | map(attribute='name') | list }}"
      │
      ▼ evaluated left to right, lazily, on the control node
 people                          [ {name: alice, sudo: true}, {name: bob}, {name: carol, sudo: false} ]
 | selectattr('sudo','defined')  [ alice, carol ]          (bob has no 'sudo' → removed)
 | selectattr('sudo')            [ alice ]                 (carol's sudo is false)
 | map(attribute='name')         generator → 'alice'
 | list                          ['alice']
```

Lookups plug into the same evaluation: when Jinja2 reaches `lookup(...)`, Ansible loads the lookup plugin, runs it on the control node, and inserts its return value.

---

## Code / commands

### A data-wrangling playbook

`~/ansible-lab/data.yml`:

```yaml
---
- name: Gather facts everywhere
  hosts: all
  gather_facts: true
  tasks: []

- name: Data wrangling on the control node
  hosts: localhost
  gather_facts: false
  vars:
    people:
      - { name: alice, uid: 2001, sudo: true,  team: shop }
      - { name: bob,   uid: 2002,              team: shop }
      - { name: carol, uid: 3001, sudo: false, team: data }
      - { name: dave,  uid: 3002, sudo: true,  team: data }
    defaults:  { port: 8080, workers: 2, tls: { enabled: false, protocols: [TLSv1.3] } }
    overrides: { workers: 8, tls: { enabled: true } }

  tasks:
    - name: Web server IPs from the inventory
      ansible.builtin.debug:
        msg: "{{ groups['web'] | map('extract', hostvars, 'ansible_host') | list }}"

    - name: Distribution of every host (from gathered facts)
      ansible.builtin.debug:
        msg: "{{ dict(groups['all'] | zip(groups['all'] | map('extract', hostvars, ['ansible_facts', 'distribution']))) }}"

    - name: Names of people with sudo
      ansible.builtin.debug:
        msg: "{{ people | selectattr('sudo', 'defined') | selectattr('sudo') | map(attribute='name') | list }}"

    - name: People grouped by team
      ansible.builtin.debug:
        msg: "{% for team, members in people | groupby('team') %}{{ team }}: {{ members | map(attribute='name') | join(', ') }}; {% endfor %}"

    - name: Highest UID
      ansible.builtin.debug:
        msg: "{{ people | map(attribute='uid') | max }}"

    - name: Deep merge of settings
      ansible.builtin.debug:
        msg: "{{ defaults | combine(overrides, recursive=true) }}"

    - name: Free bytes on / for web1 (from facts)
      ansible.builtin.debug:
        msg: >-
          {{ hostvars['web1']['ansible_facts']['mounts']
             | selectattr('mount', 'equalto', '/')
             | map(attribute='size_available') | first | human_readable }}

    - name: Lookups (run HERE, on the control node)
      ansible.builtin.debug:
        msg:
          home: "{{ lookup('ansible.builtin.env', 'HOME') }}"
          git_or_none: "{{ lookup('ansible.builtin.pipe', 'git -C ' ~ playbook_dir ~ ' rev-parse --short HEAD 2>/dev/null || echo none') }}"
          my_pubkey_start: "{{ lookup('ansible.builtin.file', 'nodes/ansible_key.pub')[:30] }}"
          conf_files: "{{ query('ansible.builtin.fileglob', 'files/*.conf') }}"
          generated: "{{ lookup('ansible.builtin.password', 'credentials/demo_pass length=16 chars=ascii_letters,digits') }}"
```

New bits:

- `map('extract', hostvars, ['ansible_facts', 'distribution'])`: a **list** as the third argument walks nested keys: `hostvars[h]['ansible_facts']['distribution']`.
- `dict(a | zip(b))`: build `{host: distro}` from two parallel lists.
- `groupby('team')` inside a one-line Jinja2 `for` (templates syntax works in `msg` too).
- `human_readable`: bytes → `"15.2 GB"`.
- `[:30]`: Python slicing on a string (first 30 characters).
- `msg:` can be a dictionary; `debug` prints it as structured data.

```bash
ansible-playbook data.yml
```
```
TASK [Web server IPs from the inventory] ***********************************************
ok: [localhost] => { "msg": ["172.30.0.11", "172.30.0.12"] }

TASK [Distribution of every host (from gathered facts)] ********************************
ok: [localhost] => { "msg": { "db1": "AlmaLinux", "lb1": "Ubuntu", "web1": "AlmaLinux", "web2": "Ubuntu" } }

TASK [Names of people with sudo] *******************************************************
ok: [localhost] => { "msg": ["alice", "dave"] }

TASK [People grouped by team] **********************************************************
ok: [localhost] => { "msg": "data: carol, dave; shop: alice, bob; " }

TASK [Highest UID] *********************************************************************
ok: [localhost] => { "msg": 3002 }

TASK [Deep merge of settings] **********************************************************
ok: [localhost] => { "msg": { "port": 8080, "tls": { "enabled": true, "protocols": ["TLSv1.3"] }, "workers": 8 } }

TASK [Free bytes on / for web1 (from facts)] *******************************************
ok: [localhost] => { "msg": "912.41 GB" }

TASK [Lookups (run HERE, on the control node)] *****************************************
ok: [localhost] => {
    "msg": {
        "conf_files": ["/home/you/ansible-lab/files/app.conf"],
        "generated": "q7XkPz2LmV9aT4cB",
        "git_or_none": "none",
        "home": "/home/you",
        "my_pubkey_start": "ssh-ed25519 AAAAC3NzaC1lZDI1NT"
    }
}
```

- Run it again: `generated` is the **same** value, because the password lookup saved it in `~/ansible-lab/credentials/demo_pass`. Look at that file. That's where the "idempotency" comes from, and why the file must be protected.
- The disk size is the WSL2 virtual disk (containers share it).

### Parse JSON from a command

```yaml
- name: List block devices as JSON
  ansible.builtin.command: lsblk --json -o NAME,SIZE,TYPE
  register: lsblk
  changed_when: false

- name: Disk names
  ansible.builtin.debug:
    msg: "{{ (lsblk.stdout | from_json).blockdevices | selectattr('type', 'equalto', 'disk') | map(attribute='name') | list }}"
```

### json_query (JMESPath)

```bash
uv pip install jmespath        # on the control node, in Ansible's venv
```

```yaml
- ansible.builtin.debug:
    msg: "{{ people | community.general.json_query('[?sudo==`true`].name') }}"     # ['alice', 'dave']
```

JMESPath is powerful for deep API results; for simple cases `selectattr`/`map` are clearer and need no extra library.

### IP maths

```bash
uv pip install netaddr
ansible-galaxy collection install ansible.utils
```

```yaml
- ansible.builtin.debug:
    msg:
      network: "{{ '172.30.0.11/24' | ansible.utils.ipaddr('network') }}"        # 172.30.0.0
      tenth_host: "{{ '172.30.0.0/24' | ansible.utils.nthhost(10) }}"           # 172.30.0.10
      is_private: "{{ '172.30.0.11' | ansible.utils.ipaddr('private') }}"       # 172.30.0.11 (truthy) or false
```

---

## Hands-on lab

1. Run `data.yml`. Then, in the playground, answer each in **one** expression:
   - the UIDs of team `data`,
   - people **without** a `sudo` key,
   - a comma-separated string of all inventory hostnames sorted in reverse,
   - a dict `{name: uid}` for everyone (`dict(people | map(attribute='name') | zip(people | map(attribute='uid')))`).
2. Merge `{a: {x: 1, y: [1]}}` with `{a: {y: [2], z: 3}}` three ways: shallow, `recursive=true`, and `recursive=true, list_merge='append'`. Explain the three results.
3. Parse `lsblk --json` on web1 (task above) and print only disk names.
4. Use `lookup('ansible.builtin.pipe', 'date +%s')` in a variable used by two debug tasks; observe it's evaluated twice (values differ). Then `set_fact` it once and use the fact.
5. Prove lookups are local: `lookup('ansible.builtin.file', '/etc/hostname')` in a task on web1 prints **your WSL2** hostname, not `web1`. Then read web1's file properly with `slurp` + `b64decode`.
6. Delete `credentials/demo_pass` and re-run: a new password appears. Add `credentials/` to `.gitignore` (or vault it, topic 40).

**Done when:** you can answer "give me X from Y where Z" questions about your inventory and facts in one line.

---

## Where you'll use it

- **Templates and configs** built from inventory: backend lists, allow-lists, cluster members (`map('extract', hostvars, ...)`).
- **Defaults + overrides** for roles (`combine(recursive=true)`).
- **Parsing** command and API output (`from_json`, `json_query`, `regex_findall`).
- **Validation**: `assert` with `selectattr` over facts (free disk, mounted filesystems, NIC names).
- **CI context**: `lookup('env', 'BUILD_NUMBER')`, `lookup('pipe', 'git rev-parse HEAD')` for version stamps.
- **Generated secrets** (`password` lookup) in labs and small setups; secret managers in production (topic 41).

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `'dict object' has no attribute 'sudo'` inside `selectattr` | Some items lack the attribute | `selectattr('sudo', 'defined')` first |
| Output shows `<generator object ...>` | `map`/`select` not turned into a list | `\| list` |
| `No filter named 'json_query'` / `ipaddr` | Collection missing or short name | Install `community.general` / `ansible.utils`; use FQCN |
| `You need to install "jmespath"` / `netaddr` | Python library missing **on the control node** | `uv pip install jmespath netaddr` in Ansible's venv |
| `combine` lost nested keys | Shallow merge | `recursive=true` |
| `lookup('file', ...)` reads the wrong machine's file | Lookups are local | `slurp`/`fetch`/`command` for remote files |
| `lookup` used in `loop` gives one string | `lookup` joins results | `query(...)` |
| A `pipe`/`url` lookup makes runs slow | Evaluated every use, per host | `set_fact` once (with `run_once` if it's the same for all) |
| Generated password changes each run | `password` lookup path depends on something that changes, or the file is deleted (e.g. CI workspace) | Stable path; persistent/vaulted storage, or a secret manager |
| `from_json` error: `Expecting value` | Output isn't pure JSON (warnings, banners) | Command options for quiet JSON output; check `stdout` |

---

## Quick recap

- Pick and reshape: **`selectattr`** (filter dicts), **`map(attribute=)`** (take a field), `map('extract', hostvars, ...)` (look up hosts), `select`/`reject`, `groupby`, `zip`, `sort(attribute=)`, `sum`; add **`| list`**.
- Merge: **`combine`** (shallow) / `combine(recursive=true, list_merge=...)` (deep).
- Parse: `from_json`/`from_yaml`, `regex_findall`, `community.general.json_query` (needs `jmespath`), `ansible.utils.ipaddr` (needs `netaddr`).
- **Lookups run on the control node**: `file`, `env`, `pipe`, `template`, `password` (stores a file), `first_found`, `fileglob`, `vars`, `url`; `query` returns lists.
- Lookups are lazy and re-evaluated per use: cache with `set_fact` if expensive.

### Interview questions

1. **Q: How do you get the list of IP addresses of all hosts in the `web` group?**
   A: `{{ groups['web'] | map('extract', hostvars, 'ansible_host') | list }}` (or use facts like `['ansible_facts','default_ipv4','address']` if gathered).
2. **Q: How do you merge two dictionaries in Ansible?**
   A: With the `combine` filter; `recursive=true` for nested dicts, `list_merge` to control how lists inside are merged.
3. **Q: Where do lookup plugins run?**
   A: On the control node. They read local files, environment variables, command output or external services; they don't read the managed host.

### Scenario question (answer out loud first, then compare)

> *"You must generate a firewall allow-list on every database server containing the private IPs of all app servers in the same datacenter, from inventory and facts. App servers have several NICs. How would you build that list?"*

**Model answer:**
1. Inputs: `groups['app']`, each app host's datacenter variable (e.g. `dc` in `group_vars`), and their facts (gathered in an earlier play on `app`, or a fact cache).
2. Expression (in a `set_fact` on the db hosts):
   - start from `groups['app'] | map('extract', hostvars)` (list of hostvars dicts),
   - keep those with `selectattr('dc', 'equalto', dc)` (same datacenter as this db host),
   - take each host's addresses: `map(attribute='ansible_facts') | map(attribute='all_ipv4_addresses') | flatten`,
   - keep private addresses: `select('ansible.utils.ipaddr', 'private')`, maybe restrict to the app subnet with `ansible.utils.ipaddr('10.20.0.0/16')`,
   - `unique | sort` for a stable result (no needless changes).
3. Render it with a template or loop into `ansible.posix.firewalld` rich rules, with a handler to reload.
4. Guard with `assert` that the list isn't empty before applying (an empty allow-list could lock out the app).

**Likely follow-ups:**
- *Why sort?* A stable order keeps the rendered file identical between runs (idempotency).
- *What if an app server's facts weren't gathered?* `map(attribute='ansible_facts')` fails or yields nothing; gather on `app` first or use a fact cache, and assert.
