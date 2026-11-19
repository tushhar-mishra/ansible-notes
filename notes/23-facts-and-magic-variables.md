# 23 · Facts and Magic Variables

> **Level:** ★ foundation
>
> **You will learn:** what **facts** are (information Ansible collects from each host: OS, IPs, memory, disks…), how and when they're gathered (`gather_facts`, the `setup` module, `gather_subset`), how to read them the current way (**`ansible_facts['distribution']`**, not the ⚠️ deprecated `ansible_distribution`), **custom facts** (`/etc/ansible/facts.d`), **`set_fact`**, **fact caching**, and Ansible's **magic variables** (`inventory_hostname`, `group_names`, `groups`, **`hostvars`**, `ansible_play_hosts`, `playbook_dir`…), including how to read **another host's** facts.
>
> **You need:** [22 · Jinja2 basics](22-jinja2-basics.md), [21 · Variables basics](21-variables-basics.md).
>
> **Part 3 · Variables, facts and logic** · Next: [24 · group_vars and host_vars](24-group-vars-and-host-vars.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** At the start of each play, Ansible runs the `setup` module on every host and stores what it finds in a per-host dictionary called **`ansible_facts`**. You use facts to write **one playbook for many kinds of servers** ("if RedHat, use dnf; listen on this host's own IP"). **Magic variables** are values Ansible itself provides: who this host is, which groups it's in, and the variables of **all other hosts** (`hostvars`).

**Picture it.** Before work starts, every new employee fills in a form (facts): name, languages spoken, desk number. The manager (Ansible) keeps all forms in a folder (`hostvars`) and can read anyone's.

**The one thing to know.**

```yaml
"{{ ansible_facts['os_family'] }}"                    # RedHat / Debian
"{{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}"
"{{ ansible_facts['default_ipv4']['address'] }}"      # this host's main IP
"{{ inventory_hostname }}"                            # web1 (its inventory name)
"{{ groups['web'] }}"                                 # ['web1', 'web2']
"{{ hostvars['db1']['ansible_facts']['default_ipv4']['address'] }}"   # db1's IP, from another host's play
```

**Try it now:**

```bash
cd ~/ansible-lab
ansible web1 -m ansible.builtin.setup | less          # everything (hundreds of lines)
ansible all -m ansible.builtin.setup -a 'filter=ansible_default_ipv4' -o
```

**You've got it if you can:** print each host's OS and IP, and from a play on `web`, print `db1`'s IP.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Facts

**Facts** = data about a managed host, discovered by the `ansible.builtin.setup` module. They're stored as host variables, under `ansible_facts`.

| Fact (key in `ansible_facts`) | Example | Use |
|---|---|---|
| `os_family` | `RedHat`, `Debian` | Choose package/service names |
| `distribution`, `distribution_version`, `distribution_major_version` | `AlmaLinux`, `10.1`, `10` | Version-specific tasks |
| `hostname`, `fqdn`, `nodename` | `web1` | Config files, banners |
| `default_ipv4.address`, `default_ipv4.interface` | `172.30.0.11`, `eth0` | Listen addresses, firewall rules |
| `all_ipv4_addresses`, `interfaces`, `eth0` | lists/dicts | Multi-NIC servers |
| `memtotal_mb`, `memfree_mb` | `7832` | Size JVM heaps, DB buffers |
| `processor_vcpus`, `processor_count`, `processor_cores` | `4` | Worker counts |
| `mounts` | list of dicts (mount, device, size_total, size_available) | Disk checks |
| `kernel`, `architecture` | `6.6.87-microsoft-standard-WSL2`, `x86_64` | Kernel/arch-specific downloads |
| `pkg_mgr`, `service_mgr` | `dnf`/`apt`, `systemd` | What the `package`/`service` modules use |
| `virtualization_type`, `virtualization_role` | `docker`, `guest` | Behave differently in containers/VMs |
| `selinux` | `{'status': 'enabled', 'mode': 'enforcing', ...}` | SELinux-aware tasks |
| `python`, `python_version` | `3.12.9` | Interpreter info (topic 07) |
| `date_time` | `{'date': '2026-11-19', 'epoch': ...}` | Timestamps (at gather time!) |
| `env` | the login user's environment | `HOME`, `PATH` on the target |
| `user_id`, `user_dir` | `root`, `/root` | Who ran `setup` (with `become`: root) |
| `ansible_local` | your custom facts | below |

### ⚠️ `ansible_facts['x']` vs `ansible_x`

For years, every fact was **also** copied into a top-level variable with an `ansible_` prefix: `ansible_distribution`, `ansible_default_ipv4`… That's controlled by the `INJECT_FACTS_AS_VARS` setting.

| Style | Status |
|---|---|
| `ansible_facts['distribution']` | ✅ Current. Works everywhere, no name clashes |
| `ansible_distribution` | ⚠️ **Deprecated since ansible-core 2.20**; the default flips to "not injected" in **2.24**, when these names stop existing unless you turn the setting back on |

Note the naming: **inside** `ansible_facts` the keys have **no** `ansible_` prefix (`ansible_facts['os_family']`). The `setup` module's `filter=` argument and its printed output still show the prefixed names (`ansible_os_family`).

### When facts are gathered

| Setting | Effect |
|---|---|
| `gather_facts: true` (play default) | Implicit "Gathering Facts" task at the start of the play |
| `gather_facts: false` | No facts (faster). Run `ansible.builtin.setup` yourself later if needed |
| `gather_subset: [min, network]` | Collect only some groups of facts (play keyword or `setup` argument) |
| `gather_timeout: 10` | Give up on slow fact collectors |
| `[defaults] gathering = smart` | With a fact cache: only gather if not already cached |

Subsets: `all`, `min` (always included: OS, hostname, Python, user…), `hardware`, `network`, `virtual`, `facter`, `ohai`. Exclude with `!`: `gather_subset: ['!hardware']`.

### Custom (local) facts

Put files in **`/etc/ansible/facts.d/`** on the managed host with the extension **`.fact`**. Each becomes `ansible_facts['ansible_local']['<file name>']`:

| File type | Content |
|---|---|
| Static INI or JSON | Read as data |
| **Executable** script | Must print JSON; run at fact-gathering time |

Use them to record things only the host knows: "which app version is deployed", "this host's rack", "date of last patch".

### set_fact

`ansible.builtin.set_fact` creates (or overwrites) **host variables at run time**, computed from other values:

```yaml
- name: Compute worker count from CPUs
  ansible.builtin.set_fact:
    app_workers: "{{ ansible_facts['processor_vcpus'] * 2 }}"
```

- The value is **per host**: each host computes its own.
- It lives until the end of the run (for later plays too), unless `cacheable: true` with a fact cache, which keeps it across runs.
- set_fact values have **high precedence** (topic 25): they override inventory and play vars.

### Fact caching

By default facts live only in memory for one run (`fact_caching = memory`). A **cache** stores them between runs:

```ini
[defaults]
gathering = smart
fact_caching = ansible.builtin.jsonfile
fact_caching_connection = ~/.ansible/facts_cache
fact_caching_timeout = 7200
```

- `jsonfile`: one JSON file per host. Other backends: `redis`, `memcached`, `mongodb`, `yaml`, `pickle` (in `community.general`).
- Benefits: faster runs with `gathering = smart`; you can use another host's facts **without contacting it** in this run.
- Risk: **stale** facts (an IP changed). Keep the timeout reasonable.

### Magic variables

Variables Ansible always provides. You can't set them (well, you shouldn't).

| Variable | Contains |
|---|---|
| `inventory_hostname` | This host's name in the inventory (`web1`) |
| `inventory_hostname_short` | Up to the first dot (`web1` from `web1.example.com`) |
| `group_names` | Groups **this** host is in: `['alma', 'app', 'web']` (not `all`) |
| `groups` | Every group and its hosts: `groups['web']` → `['web1', 'web2']`, `groups['all']` |
| `hostvars` | **Every host's variables**: `hostvars['db1']['ansible_host']`, `hostvars['db1']['ansible_facts'][...]` |
| `ansible_play_hosts` | Hosts in the current play that are still active (not failed) |
| `ansible_play_hosts_all` | All hosts the play targeted |
| `ansible_play_batch` | Hosts in the current `serial` batch (topic 45) |
| `playbook_dir`, `inventory_dir`, `inventory_file`, `role_path` | Paths |
| `ansible_check_mode`, `ansible_diff_mode` | `true` during `--check` / `--diff` |
| `ansible_version` | `{'full': '2.22.0', 'major': 2, 'minor': 22, ...}` |
| `ansible_run_tags`, `ansible_skip_tags` | Tags given on the command line (topic 33) |
| `ansible_config_file` | Path of the ansible.cfg in use |
| `item`, `omit` | Loop item (topic 28), the omit placeholder (topic 22) |

### Reading another host's facts

`hostvars['db1']['ansible_facts']` has data **only if** db1's facts were gathered **in this run** (or are in the cache). A play on `web` doesn't gather db1's facts. Two patterns:

```yaml
# Pattern 1: an earlier play gathers facts on everyone
- name: Gather facts on all hosts
  hosts: all
  gather_facts: true
  tasks: []

- name: Web servers need the DB address
  hosts: web
  tasks:
    - ansible.builtin.debug:
        msg: "DB is at {{ hostvars['db1']['ansible_facts']['default_ipv4']['address'] }}"
```

```yaml
# Pattern 2: gather db1's facts from inside the web play
- name: Get db facts
  ansible.builtin.setup:
  delegate_to: "{{ item }}"
  delegate_facts: true
  loop: "{{ groups['db'] }}"
  run_once: true
```

(`delegate_to` and `delegate_facts` are topic 46. Pattern 1 is simplest.)

Inventory variables (like `ansible_host`) of other hosts are **always** available in `hostvars`; only facts need gathering.

---

## Why it exists

Real fleets aren't uniform: different distros, versions, sizes, IPs. Without facts you'd hard-code `if web1 then ...` everywhere, or write separate playbooks per OS. With facts, the playbook adapts:

- package and service names from `os_family`,
- memory-based settings from `memtotal_mb`,
- listen addresses from `default_ipv4`,
- skip steps inside containers using `virtualization_type`.

Magic variables let hosts know about **each other**: a load balancer config needs the IPs of all `web` hosts; an app config needs the database host's address.

---

## Analogy

**A hospital admission.** When a patient arrives, a nurse measures height, weight, blood pressure and allergies (gather facts). Doctors then decide doses per patient (`memtotal_mb` → heap size). The patient can carry a card with extra notes the nurse can't measure, like "prefers morning appointments" (custom facts). The ward's **patient list** (`groups`) and **all charts** (`hostvars`) let a doctor caring for one patient look up another's blood type for a transfusion, as long as that patient's chart was filled in (facts gathered).

---

## How it works inside

```
 PLAY starts (gather_facts: true)
   │
   ├─ implicit task "Gathering Facts" = ansible.builtin.gather_facts
   │     → runs the setup module on each host (normal module run, topic 13)
   │         ├ reads /etc/os-release, /proc/meminfo, /proc/cpuinfo, `ip` output, mounts...
   │         ├ runs executable files in /etc/ansible/facts.d/*.fact, reads static ones
   │         └ returns {"ansible_facts": {...hundreds of keys...}}
   │
   ├─ control node stores them:  hostvars['web1']['ansible_facts'] = {...}
   │     (and, while INJECT_FACTS_AS_VARS is on, also as ansible_<name> variables)
   │     (and in the fact cache, if configured)
   │
   └─ later tasks read them like any variable, per host, on the control node
```

Facts are only a **snapshot**: `ansible_facts['memfree_mb']` is the value at gather time. If a task changes something (installs a package, adds an IP), facts don't update until you run `setup` again.

---

## Code / commands

### Explore facts

```bash
ansible web1 -m ansible.builtin.setup -a 'filter=ansible_distribution*'
ansible web1 -m ansible.builtin.setup -a 'gather_subset=!all,!min,network filter=ansible_default_ipv4'
ansible all -m ansible.builtin.setup -a 'filter=ansible_memtotal_mb' -o
```
```
web1 | SUCCESS => {"ansible_facts": {"ansible_memtotal_mb": 15872, "discovered_interpreter_python": "/usr/bin/python3"}, "changed": false}
...
```

- In containers, memory and CPU facts show the **WSL2 VM's** resources (shared kernel).

Save all facts to files for browsing:

```bash
ansible all -m ansible.builtin.setup --tree /tmp/facts >/dev/null
ls /tmp/facts                      # one JSON file per host
python3 -m json.tool /tmp/facts/web2 | grep -A3 '"ansible_default_ipv4"'
```

### The lab playbook

`~/ansible-lab/facts.yml`:

```yaml
---
- name: Gather facts everywhere (so hostvars has them)
  hosts: all
  gather_facts: true
  tasks: []

- name: Facts and magic variables
  hosts: web
  become: true
  tasks:
    - name: Who and what am I
      ansible.builtin.debug:
        msg: >-
          {{ inventory_hostname }} is {{ ansible_facts['distribution'] }}
          {{ ansible_facts['distribution_version'] }} ({{ ansible_facts['os_family'] }}),
          {{ ansible_facts['processor_vcpus'] }} vCPUs, {{ ansible_facts['memtotal_mb'] }} MB,
          IP {{ ansible_facts['default_ipv4']['address'] }}, groups {{ group_names }}

    - name: The other web servers and the database address
      ansible.builtin.debug:
        msg: >-
          Web peers: {{ groups['web'] | difference([inventory_hostname]) }};
          DB at {{ hostvars['db1']['ansible_facts']['default_ipv4']['address'] }}
          (inventory address {{ hostvars['db1']['ansible_host'] }})

    - name: Compute a value per host
      ansible.builtin.set_fact:
        app_workers: "{{ ansible_facts['processor_vcpus'] * 2 }}"

    - name: Facts directory exists
      ansible.builtin.file:
        path: /etc/ansible/facts.d
        state: directory
        mode: "0755"

    - name: Static custom fact
      ansible.builtin.copy:
        dest: /etc/ansible/facts.d/app.fact
        content: |
          [deploy]
          version=1.0
          owner=team-shop
        mode: "0644"

    - name: Executable custom fact (must print JSON)
      ansible.builtin.copy:
        dest: /etc/ansible/facts.d/disk.fact
        content: |
          #!/bin/sh
          printf '{"root_used_pct": %s}\n' "$(df --output=pcent / | tail -1 | tr -dc '0-9')"
        mode: "0755"

    - name: Re-read facts so the new local facts appear
      ansible.builtin.setup:
        filter: ansible_local

    - name: Use custom facts and set_fact
      ansible.builtin.debug:
        msg: >-
          app version {{ ansible_facts['ansible_local']['app']['deploy']['version'] }},
          root disk {{ ansible_facts['ansible_local']['disk']['root_used_pct'] }}% used,
          workers {{ app_workers }}
```

- `msg: >-`: a folded string (topic 04), so long messages stay readable. `-` removes the final newline.
- `groups['web'] | difference([inventory_hostname])`: the web hosts except me.
- `hostvars['db1']['ansible_facts']`: works because play 1 gathered facts on **all** hosts.
- After installing custom facts, we run `setup` again (filtered) to refresh `ansible_local`.
- INI fact → `ansible_local.app.deploy.version` (file → section → key). Values from INI are **strings**.

```bash
ansible-playbook facts.yml
```
```
TASK [Who and what am I] ***************************************************************
ok: [web1] => {
    "msg": "web1 is AlmaLinux 10.1 (RedHat), 8 vCPUs, 15872 MB, IP 172.30.0.11, groups ['alma', 'app', 'web']"
}
ok: [web2] => {
    "msg": "web2 is Ubuntu 24.04 (Debian), 8 vCPUs, 15872 MB, IP 172.30.0.12, groups ['app', 'ubuntu', 'web']"
}

TASK [The other web servers and the database address] **********************************
ok: [web1] => {
    "msg": "Web peers: ['web2']; DB at 172.30.0.21 (inventory address 172.30.0.21)"
}
...
TASK [Use custom facts and set_fact] ***************************************************
ok: [web1] => {
    "msg": "app version 1.0, root disk 23% used, workers 16"
}
```

### What happens without play 1

Comment out the first play and run again:

```
fatal: [web1]: FAILED! => {"msg": "The task includes an option with an undefined variable. ... 'dict object' has no attribute 'default_ipv4' ..."}
```

`hostvars['db1']['ansible_facts']` exists but is **empty**: nobody gathered db1's facts this run. (Or turn on the fact cache, run play 1 once, and it works from the cache.)

### Deprecated style (to recognise in old code)

```yaml
when: ansible_distribution == "Ubuntu"          # ⚠️ deprecated injected fact
when: ansible_facts['distribution'] == "Ubuntu" # ✅
```

Find old usages in a repo: `grep -rnE 'ansible_(distribution|os_family|default_ipv4|hostname|memtotal)' .`

### Turn on a fact cache

Add to `ansible.cfg`:

```ini
[defaults]
gathering = smart
fact_caching = ansible.builtin.jsonfile
fact_caching_connection = ~/.ansible/facts_cache
fact_caching_timeout = 3600
```

Run `facts.yml` twice with `ANSIBLE_CALLBACKS_ENABLED=ansible.posix.profile_tasks`: on the second run, Gathering Facts is near 0 s.

---

## Hands-on lab

1. `ansible all -m ansible.builtin.setup --tree /tmp/facts`, then find in the JSON: each host's `os_family`, `pkg_mgr`, `virtualization_type`, `selinux.status`.
2. Run `facts.yml`. Then log in to web1 and run `/etc/ansible/facts.d/disk.fact` by hand: it prints JSON.
3. Remove play 1 and see the `hostvars` failure. Restore it.
4. Add a `set_fact` that sets `app_tier: "{{ 'big' if ansible_facts['memtotal_mb'] > 8000 else 'small' }}"` and print it.
5. Turn on the jsonfile fact cache, run once, then `ls ~/.ansible/facts_cache` and `python3 -m json.tool ~/.ansible/facts_cache/db1 | head`. Run a web-only play that reads db1's IP **without** play 1. It works from the cache.
6. Use `ansible_play_hosts` in a debug task with `run_once: true` (prints once instead of per host): `msg: "This play runs on {{ ansible_play_hosts | join(', ') }}"`.

**Done when:** you can get any fact for any host, from any play, and know when it might be stale or missing.

---

## Where you'll use it

- **Multi-distro roles**: `ansible_facts['os_family']` selects names and paths (topic 36).
- **Sizing**: workers from vCPUs, memory settings from `memtotal_mb`.
- **Cluster configs**: a load balancer template looping over `groups['web']` with `hostvars[h]['ansible_facts']['default_ipv4']['address']` (topic 29); database replicas finding the primary.
- **Reporting**: collect facts from the fleet into a CSV/HTML report (topic 29 templates, `run_once` + `delegate_to: localhost`).
- **Inventory of record**: custom facts storing deployed versions, read by later automation or AWX.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `'ansible_facts' is undefined` / `'dict object' has no attribute 'distribution'` | `gather_facts: false` | Enable it or run `ansible.builtin.setup` first |
| `hostvars['db1']['ansible_facts']` is empty | db1 not in any play that gathered facts this run | Gather on `all` first, `delegate_facts`, or fact cache |
| `[DEPRECATION WARNING]: INJECT_FACTS_AS_VARS ...` | Using `ansible_distribution`-style names | Switch to `ansible_facts['distribution']` |
| Gathering Facts is slow | Many hosts, slow hardware/network collectors, hung NFS mounts | `gather_subset`, `gather_timeout`, `gathering = smart` + cache |
| Wrong/old IP used | Stale fact cache | Lower `fact_caching_timeout`; `--flush-cache` |
| Custom fact missing | File not ending in `.fact`, wrong permissions, script printed invalid JSON or isn't executable | Check name, `chmod 0755`, run it by hand |
| Custom INI fact compared as a number fails | INI values are strings | `| int` |
| `ansible_facts['env']['HOME']` is `/root` | Facts gathered with `become: true` | Expected: facts reflect the user that ran `setup` |
| `memtotal_mb` huge on containers | Containers report the host's memory | Use cgroup limits if needed; know your environment |

---

## Quick recap

- **Facts** = per-host data from the `setup` module, gathered at play start (`gather_facts`, `gather_subset`). Read them as **`ansible_facts['key']`**; the `ansible_key` style is ⚠️ deprecated (2.20, default off in 2.24).
- **Custom facts**: `/etc/ansible/facts.d/*.fact` (INI/JSON or executable printing JSON) → `ansible_facts['ansible_local']`.
- **set_fact** = runtime host variables (high precedence). **Fact cache** (`jsonfile`, `gathering = smart`) = faster runs and other hosts' facts without contacting them.
- **Magic variables**: `inventory_hostname`, `group_names`, `groups`, `hostvars`, `ansible_play_hosts`, `playbook_dir`, `ansible_check_mode`, `ansible_version`.
- Another host's facts need to be gathered in this run (or cached).

### Interview questions

1. **Q: What are facts and how are they collected?**
   A: Information about each managed host (OS, network, hardware...), collected by the `setup` module, automatically at the start of a play unless `gather_facts: false`. They're stored per host under `ansible_facts`.
2. **Q: How do you use one host's data in another host's tasks?**
   A: Through `hostvars['otherhost']`. Inventory variables are always there; facts only if that host's facts were gathered in the run or are in the fact cache.
3. **Q: What are custom facts?**
   A: Files in `/etc/ansible/facts.d/` ending in `.fact`, static INI/JSON or executables printing JSON, exposed as `ansible_facts['ansible_local']`.

### Scenario question (answer out loud first, then compare)

> *"Your nginx load-balancer template lists the IPs of all web servers using `hostvars`. When you run the playbook with `--limit lb1`, the template fails with an undefined attribute. Why, and how do you fix it without dropping the limit?"*

**Model answer:**
1. With `--limit lb1`, only lb1 is in any play, so facts are gathered only for lb1. `hostvars['web1']['ansible_facts']` is empty, and `default_ipv4` is undefined.
2. Options:
   - Use **inventory** addresses instead of facts (`hostvars[h]['ansible_host']`), which are always available. Simplest when the inventory has the right IPs.
   - Enable a **fact cache** with a sensible timeout, so web facts from earlier runs are available.
   - Explicitly gather web facts from the lb play: `ansible.builtin.setup` with `delegate_to: "{{ item }}"`, `delegate_facts: true`, `loop: "{{ groups['web'] }}"`, `run_once: true`. That contacts the web hosts despite the limit, so be sure that's acceptable.
3. Also guard the template with a clear error (`| mandatory` or an `assert`) so the failure message says "web facts missing" rather than a cryptic attribute error.

**Likely follow-ups:**
- *Does `--limit` stop `delegate_to` from contacting other hosts?* No; delegation targets any host in the inventory.
- *What's the risk of the cache?* Stale data: a replaced web server's old IP could stay in the LB config until the cache expires or is flushed.
