# 28 · Loops

> **Level:** ★ foundation
>
> **You will learn:** how to repeat a task over a list with **`loop`** and **`item`**, looping over **lists of dictionaries** and over **dictionaries** (`dict2items`), nested combinations (`product`, `subelements`), **`loop_control`** (`label` for readable output, `index_var`, `loop_var`, `pause`, `extended`), what **`register`** holds after a loop (`results`), how **`when`** works per item, **retry loops** with `until` / `retries` / `delay`, the legacy **`with_*`** forms (and when you still meet them), `query` vs `lookup`, and when **not** to loop (pass a list to the module instead).
>
> **You need:** [27 · Idempotency](27-idempotency-in-depth.md), [26 · register and when](26-register-and-when.md), [22 · Jinja2](22-jinja2-basics.md).
>
> **Part 3 · Variables, facts and logic** · Next: [29 · Templates](29-templates.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** `loop:` runs the same task once for **each element** of a list. Inside the task, the current element is **`item`**.

**Picture it.** A mail merge: one letter template, one row per recipient.

**The one thing to know.**

```yaml
- name: Users exist
  ansible.builtin.user:
    name: "{{ item.name }}"
    shell: "{{ item.shell | default('/bin/bash') }}"
    groups: "{{ item.groups | default(omit) }}"
    append: true
  loop:
    - { name: carol, groups: devs }
    - { name: dave, shell: /bin/sh }
  loop_control:
    label: "{{ item.name }}"          # print just the name, not the whole dict

- name: Wait until nginx answers (retry loop)
  ansible.builtin.uri:
    url: "http://{{ ansible_host }}/"
  register: page
  until: page.status == 200
  retries: 10
  delay: 3
  delegate_to: localhost
```

**Try it now:** run the lab playbook below and look at the `(item=...)` lines.

**You've got it if you can:** loop over a list of dicts, over a dict, and write a retry loop.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### `loop` basics

```yaml
- name: Directories exist
  ansible.builtin.file:
    path: "{{ item }}"
    state: directory
    mode: "0755"
  loop:
    - /opt/app
    - /opt/app/logs
```

- The task runs **twice per host**, once with `item = "/opt/app"`, once with `item = "/opt/app/logs"`.
- `loop` takes a **list**. It can come from a variable: `loop: "{{ app_dirs }}"`.
- Items are processed **in order**, one by one, for each host. Hosts still run in parallel.

### Looping over a list of dictionaries

The most common real case:

```yaml
vars:
  app_users:
    - { name: carol, uid: 3001, groups: devs }
    - { name: dave,  uid: 3002 }
tasks:
  - name: Users exist
    ansible.builtin.user:
      name: "{{ item.name }}"
      uid: "{{ item.uid }}"
      groups: "{{ item.groups | default(omit) }}"
      append: true
    loop: "{{ app_users }}"
```

`default(omit)` (topic 22) handles keys some items don't have.

### Looping over a dictionary

`loop` needs a list, so convert with **`dict2items`**: each key/value pair becomes `{key: ..., value: ...}`.

```yaml
vars:
  sysctl_settings:
    vm.swappiness: 10
    net.core.somaxconn: 4096
tasks:
  - name: Kernel settings are in the drop-in file
    ansible.builtin.lineinfile:
      path: /etc/sysctl.d/90-app.conf
      regexp: '^{{ item.key | regex_escape }}\s*='
      line: "{{ item.key }} = {{ item.value }}"
      create: true
      mode: "0644"
    loop: "{{ sysctl_settings | dict2items }}"
```

- `item.key` = `vm.swappiness`, `item.value` = `10`.
- `regex_escape`: escape the dots so they're literal in the regexp.
- The reverse filter, `items2dict`, turns such a list back into a dictionary.

### Nested loops

| Need | Filter | Example |
|---|---|---|
| Every combination of two lists | `product` | `loop: "{{ ['alice','bob'] \| product(['db1','db2']) \| list }}"` → `item[0]`, `item[1]` |
| For each item, each element of its sub-list | `subelements` | `loop: "{{ users \| subelements('keys') }}"` → `item.0` = user, `item.1` = one key |
| Flatten lists of lists | `flatten` | `loop: "{{ [list_a, list_b] \| flatten }}"` |
| A range of numbers | `range` | `loop: "{{ range(1, 4) \| list }}"` → 1, 2, 3 (2.19+: add `\| list`) |

`subelements` example (one task per user per SSH key):

```yaml
vars:
  people:
    - name: carol
      keys: ["ssh-ed25519 AAAA... carol@laptop", "ssh-ed25519 AAAA... carol@desktop"]
    - name: dave
      keys: ["ssh-ed25519 AAAA... dave@laptop"]
tasks:
  - name: Each person's keys are authorised
    ansible.posix.authorized_key:
      user: "{{ item.0.name }}"
      key: "{{ item.1 }}"
    loop: "{{ people | subelements('keys') }}"
    loop_control:
      label: "{{ item.0.name }}"
```

### `loop_control`

| Option | Effect |
|---|---|
| `label: "{{ item.name }}"` | What to print in `(item=...)`. Without it, the whole dict is printed (noisy, and may **leak secrets**) |
| `index_var: idx` | A variable with the current index (0, 1, 2…) |
| `loop_var: user` | Rename `item` (needed when an included task file has its own loop, topic 34, so the inner `item` doesn't clash) |
| `pause: 2` | Wait 2 seconds between items (rate limits, gentle restarts) |
| `extended: true` | Adds `ansible_loop` with `index` (1-based), `first`, `last`, `length`, `revindex`, `previtem`, `nextitem` |
| `break_when: [cond]` | Stop the loop early when a condition becomes true (newer ansible-core) |

### `register` with a loop

After a loop, the registered variable has a **`results`** list: one full result per item, each with its own `item`, `changed`, `rc`, `stdout`...

```yaml
- name: Check each port
  ansible.builtin.wait_for:
    host: "{{ ansible_host }}"
    port: "{{ item }}"
    timeout: 2
  loop: [22, 80, 443]
  register: ports
  ignore_errors: true
  delegate_to: localhost

- name: Report closed ports
  ansible.builtin.debug:
    msg: "port {{ item.item }} is closed"
  loop: "{{ ports.results }}"
  when: item is failed
  loop_control:
    label: "{{ item.item }}"
```

- `ports.results[0].item` is `22` (the original item).
- The top-level `ports.changed` is true if **any** item changed; `ports.failed` if any failed.

### `when` with a loop

`when` is evaluated **per item**, and can use `item`:

```yaml
- name: Only users who need sudo
  ansible.builtin.copy:
    dest: "/etc/sudoers.d/{{ item.name }}"
    content: "{{ item.name }} ALL=(ALL) NOPASSWD: ALL\n"
    mode: "0440"
    validate: /usr/sbin/visudo -cf %s
  loop: "{{ app_users }}"
  when: item.sudo | default(false)
```

To skip the **whole** loop, the condition must not depend on `item`; it's still checked per item (each shows as skipped).

### Retry loops: `until`, `retries`, `delay`

Run a task again and again **until** a condition is true (or give up):

```yaml
- name: App health endpoint is up
  ansible.builtin.uri:
    url: "http://{{ ansible_host }}:8080/health"
    status_code: 200
  register: health
  until: health.status == 200
  retries: 20          # at most 20 attempts...
  delay: 3             # ...3 seconds apart
```

- Output shows `FAILED - RETRYING: [web1]: App health endpoint is up (19 retries left).`
- After the last retry, the task fails normally.
- `until` works on any task (`command`, `uri`, `wait_for`...), with or without `loop`.
- Default when you set only `until`: 3 retries, 5 s delay.

### ⚠️ `with_*`: the older loop keywords

Before `loop` (ansible 2.5), loops were `with_<lookup>`:

| Old | Modern equivalent |
|---|---|
| `with_items: "{{ list }}"` | `loop: "{{ list \| flatten(levels=1) }}"` (with_items flattens one level!) |
| `with_list` | `loop` |
| `with_dict: "{{ d }}"` | `loop: "{{ d \| dict2items }}"` |
| `with_subelements` | `loop: "{{ x \| subelements('y') }}"` |
| `with_nested` | `loop: "{{ a \| product(b) \| list }}"` |
| `with_sequence: start=1 end=3` | `loop: "{{ range(1, 4) \| list }}"` |
| `with_fileglob: "files/*.conf"` | `loop: "{{ query('ansible.builtin.fileglob', 'files/*.conf') }}"` |
| `with_together` | `loop: "{{ a \| zip(b) \| list }}"` |

`with_*` isn't removed and you'll see it in a lot of existing code, but **`loop` is the recommended form** for new code. The `with_X` keyword simply calls the lookup plugin `X`; the modern form calls it explicitly with `query`.

### `lookup` vs `query`

Both call a **lookup plugin** (code that fetches data on the control node: files, env vars, globs, secrets; topic 32):

| | `lookup('x', ...)` | `query('x', ...)` (alias `q`) |
|---|---|---|
| Returns | A comma-joined **string** by default (or a list with `wantlist=True`) | Always a **list** |
| Use in `loop` | Needs `wantlist=True` | ✅ Directly |

### When not to loop

| Slow / noisy | Better |
|---|---|
| `package` in a loop over 20 names (20 transactions) | `name: "{{ packages }}"` (one transaction) |
| `copy` in a loop over 300 small files | `synchronize` or an archive + `unarchive` |
| Loop of `lineinfile` editing the same file 30 times | One `template` of the whole file (topic 29) |
| Loop calling an API 1,000 times | A module that accepts a list, or batching |

---

## Why it exists

Without loops you'd copy the same task once per user, package, directory or port. Loops keep **data** (lists in variables) separate from **logic** (one task), so adding a user is a one-line change in `group_vars`, not a new task.

---

## Analogy

**A hotel housekeeping cart.** One routine ("make the bed, clean the bathroom") applied **room by room** down a list (`loop`). The room number is `item`. The note on each door (`when: item...`) can say "Do not disturb" (skip). The supervisor's sheet (`register`) gets one line per room (`results`). For the lift that's out of order, the cleaner presses the call button every 30 seconds until it comes (`until`/`retries`/`delay`).

---

## How it works inside

```
 task with loop: [a, b, c], when, register    (host web1)
   │
   ├─ 1. render the loop expression → a list (on the control node)
   ├─ 2. for each element (in order):
   │       set item (or loop_var) = element
   │       evaluate when with item → false? record "skipped" for this item
   │       render args with item → run the module (one module run per item, topic 13)
   │       until? re-run up to `retries` times with `delay` between attempts
   │       record the item's result
   ├─ 3. combine: task changed = any item changed; failed = any item failed
   └─ 4. register → {changed, failed, results: [ {item: a, ...}, {item: b, ...}, ... ]}
```

Each item is a **separate module execution** (SSH round trips, topic 13). That's why passing a list to a module that accepts lists is so much faster than looping.

---

## Code / commands

### The lab playbook

`~/ansible-lab/loops.yml`:

```yaml
---
- name: Loops in practice
  hosts: web
  become: true
  vars:
    loop_users:
      - { name: carol, uid: 3001, groups: devs, sudo: true }
      - { name: dave,  uid: 3002, shell: /bin/sh }
      - { name: erin,  uid: 3003 }
    app_dirs: [/opt/loopdemo, /opt/loopdemo/logs, /opt/loopdemo/tmp]
    app_limits:
      nofile: 65535
      nproc: 4096

  tasks:
    - name: Directories exist
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        mode: "0755"
      loop: "{{ app_dirs }}"

    - name: Users exist
      ansible.builtin.user:
        name: "{{ item.name }}"
        uid: "{{ item.uid }}"
        shell: "{{ item.shell | default('/bin/bash') }}"
        groups: "{{ item.groups | default(omit) }}"
        append: true
      loop: "{{ loop_users }}"
      loop_control:
        label: "{{ item.name }}"

    - name: sudo rules only for users with sudo=true
      ansible.builtin.copy:
        dest: "/etc/sudoers.d/{{ item.name }}"
        content: "{{ item.name }} ALL=(ALL) NOPASSWD: ALL\n"
        mode: "0440"
        validate: /usr/sbin/visudo -cf %s
      loop: "{{ loop_users }}"
      loop_control:
        label: "{{ item.name }}"
      when: item.sudo | default(false)

    - name: Limits from a dictionary
      ansible.builtin.lineinfile:
        path: /etc/security/limits.d/90-loopdemo.conf
        regexp: '^\*\s+-\s+{{ item.key }}\s'
        line: "*    -    {{ item.key }}    {{ item.value }}"
        create: true
        mode: "0644"
      loop: "{{ app_limits | dict2items }}"

    - name: Numbered files with index and extended info
      ansible.builtin.copy:
        dest: "/opt/loopdemo/tmp/file{{ idx }}.txt"
        content: "item {{ ansible_loop.index }} of {{ ansible_loop.length }}{{ ' (last)' if ansible_loop.last else '' }}\n"
        mode: "0644"
      loop: [a, b, c]
      loop_control:
        index_var: idx
        extended: true

    - name: Read each user's shell
      ansible.builtin.command: "getent passwd {{ item.name }}"
      loop: "{{ loop_users }}"
      loop_control:
        label: "{{ item.name }}"
      register: getent_out
      changed_when: false

    - name: Summarise the registered loop
      ansible.builtin.debug:
        msg: "{{ item.item.name }} → {{ item.stdout.split(':')[-1] }}"
      loop: "{{ getent_out.results }}"
      loop_control:
        label: "{{ item.item.name }}"

    - name: nginx answers on port 80 (retry loop)
      ansible.builtin.uri:
        url: "http://{{ ansible_host }}/"
      register: page
      until: page.status == 200
      retries: 5
      delay: 2
      delegate_to: localhost
      become: false
```

New bits:

- `extended: true` → `ansible_loop.index` (1-based), `.length`, `.last`.
- `item.stdout.split(':')[-1]`: a Python string method: split the `/etc/passwd` line on `:` and take the last field (the shell).
- The `uri` task runs **on the control node** (`delegate_to: localhost`, topic 46) and checks each web server's IP; `become: false` so it doesn't try sudo on your machine (topic 19's trap).

### Run it

```bash
ansible-playbook loops.yml -l web1
```
```
TASK [Directories exist] ***************************************************************
changed: [web1] => (item=/opt/loopdemo)
changed: [web1] => (item=/opt/loopdemo/logs)
changed: [web1] => (item=/opt/loopdemo/tmp)

TASK [Users exist] *********************************************************************
changed: [web1] => (item=carol)
changed: [web1] => (item=dave)
changed: [web1] => (item=erin)

TASK [sudo rules only for users with sudo=true] ****************************************
changed: [web1] => (item=carol)
skipping: [web1] => (item=dave)
skipping: [web1] => (item=erin)

TASK [Limits from a dictionary] ********************************************************
changed: [web1] => (item={'key': 'nofile', 'value': 65535})
changed: [web1] => (item={'key': 'nproc', 'value': 4096})

TASK [Numbered files with index and extended info] *************************************
changed: [web1] => (item=a)
changed: [web1] => (item=b)
changed: [web1] => (item=c)

TASK [Read each user's shell] **********************************************************
ok: [web1] => (item=carol)
ok: [web1] => (item=dave)
ok: [web1] => (item=erin)

TASK [Summarise the registered loop] ***************************************************
ok: [web1] => (item=carol) => { "msg": "carol → /bin/bash" }
ok: [web1] => (item=dave) => { "msg": "dave → /bin/sh" }
ok: [web1] => (item=erin) => { "msg": "erin → /bin/bash" }

TASK [nginx answers on port 80 (retry loop)] *******************************************
ok: [web1 -> localhost]
```

- `(item=carol)`: thanks to `label`. Without it you'd see the whole dict.
- `skipping: ... (item=dave)`: `when` per item.
- `[web1 -> localhost]`: task for web1, executed on localhost (delegation).

### Watch a retry loop retry

Stop nginx on web2, then run only that task's play for web2, and start nginx from another terminal while it retries:

```bash
ssh ansible@172.30.0.12 'sudo systemctl stop nginx'
ansible-playbook loops.yml -l web2 --start-at-task "nginx answers on port 80 (retry loop)"
```
```
FAILED - RETRYING: [web2 -> localhost]: nginx answers on port 80 (retry loop) (5 retries left).
FAILED - RETRYING: [web2 -> localhost]: nginx answers on port 80 (retry loop) (4 retries left).
ok: [web2 -> localhost]
```

(In the other terminal, during the retries: `ssh ansible@172.30.0.12 'sudo systemctl start nginx'`.)

### Legacy `with_items` you'll see in old code

```yaml
- name: Old style (still works)
  ansible.builtin.debug:
    msg: "{{ item }}"
  with_items:
    - [1, 2]
    - 3
```

Prints `1`, `2`, `3`: `with_items` flattened the inner list. `loop:` would print `[1, 2]` and `3`.

---

## Hands-on lab

1. Run `loops.yml`, then run it again: `changed=0` (the read-only `getent` task has `changed_when: false`).
2. Add a fourth user to `loop_users` with `sudo: true`. Run. Only the new user's tasks change.
3. Remove `loop_control: label` from the users task and look at the noisier output.
4. Use `subelements` to give carol two SSH keys (any two public keys you generate with `ssh-keygen -f /tmp/k1 -N ""` and `/tmp/k2`). Read them with `lookup('file', '/tmp/k1.pub')`.
5. Do the retry exercise with nginx stopped on web2.
6. Rewrite this old task with `loop`:

   ```yaml
   - ansible.builtin.debug: { msg: "{{ item.key }}={{ item.value }}" }
     with_dict: { a: 1, b: 2 }
   ```

7. Time a loop vs a list: install `tree`, `curl`, `git`, `rsync` with a `loop` over `package`, then remove them and install with `name: [tree, curl, git, rsync]`. Compare with `profile_tasks`.
8. Clean up: remove carol/dave/erin (`state: absent`, `remove: true`), their sudoers files and `/opt/loopdemo`, using loops.

**Done when:** you reach for `loop` + `dict2items`/`subelements` naturally, and pass lists to modules when they accept them.

---

## Where you'll use it

- **Users, keys, sudo rules** from a list in `group_vars` (onboarding = add a line).
- **Packages, directories, firewall ports, sysctl settings** from lists/dicts.
- **Waiting for things**: services to answer, DNS to resolve, a cluster to become healthy (`until`).
- **Generating per-item files**: one vhost config per site, one systemd unit per worker.
- **Fleet reports**: loop over `groups['all']` with `run_once` to summarise `hostvars`.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Invalid data passed to 'loop', it requires a list, got this instead: {...}` | Looping over a dict | `\| dict2items` |
| `... got this instead: range(1, 4)` | 2.19+: `range` returns a range object | `\| list` |
| `'item' is undefined` in an included file's loop | Inner and outer loops both use `item` | `loop_control: loop_var: outer_item` |
| Output shows huge dicts / secrets per item | No `label` | `loop_control: label: "{{ item.name }}"` (+ `no_log` for secrets, topic 40) |
| `'dict object' has no attribute 'stdout'` after a loop | Registered loop results are in `.results` | `result.results[0].stdout` or loop over `result.results` |
| A loop item missing a key fails | Not every dict has it | `item.key \| default(omit)` / `default(...)` |
| `with_items` → `loop` change broke the task | `with_items` flattened nested lists | `loop: "{{ x \| flatten(levels=1) }}"` |
| `until` never succeeds | Condition can't become true (wrong key, wrong status) | `debug: var=` the registered result once; check `status`, `rc` |
| Very slow package installs | Looping `package` per item | Pass the list to `name:` |
| Loop over `lookup('fileglob', ...)` gives one weird item | `lookup` returned a comma-joined string | Use `query(...)` |

---

## Quick recap

- `loop:` + `item` repeats a task per list element; per host, in order. Lists of dicts are the norm; dicts need **`dict2items`**.
- Nesting: `product`, `subelements`, `flatten`; numbers: `range(...) | list`.
- **`loop_control`**: `label` (readable + safer output), `index_var`, `loop_var` (nested includes), `pause`, `extended` (`ansible_loop`).
- After a loop, `register` → **`results`** list (each has `item`). `when` is evaluated **per item**.
- **Retry**: `until` + `retries` + `delay`. ⚠️ `with_*` is legacy (prefer `loop` + `query`). Don't loop when a module accepts a list.

### Interview questions

1. **Q: How do you loop over a dictionary?**
   A: Convert it with `dict2items` (`loop: "{{ mydict | dict2items }}"`) and use `item.key` / `item.value`.
2. **Q: What does a registered variable look like after a loop?**
   A: It has a `results` list with one result per item (each including the original `item`), plus top-level `changed`/`failed` that are true if any item changed/failed.
3. **Q: How do you wait for a service to come up?**
   A: A retry loop: e.g. `uri` (or `wait_for`) with `register`, `until: result.status == 200`, `retries` and `delay`.

### Scenario question (answer out loud first, then compare)

> *"Onboarding a new engineer currently means editing four playbooks (user, SSH keys, sudo, VPN group). How would you redesign this with loops so onboarding is a one-line change?"*

**Model answer:**
1. One source of truth in `group_vars/all/people.yml`: a list of dicts, e.g. `{name, uid, keys: [...], sudo: bool, groups: [...], state: present}`.
2. One role/play with looped tasks:
   - `user` over the list (`state: "{{ item.state | default('present') }}"`, `groups`, `append: true`);
   - `authorized_key` over `people | subelements('keys')`, or with `exclusive: true` and the keys joined per user;
   - sudoers drop-in per user `when: item.sudo | default(false)`, with `validate`;
   - `loop_control: label: "{{ item.name }}"` everywhere.
3. Offboarding = set `state: absent` (and later remove the entry). The same loop removes the user, keys and sudo file.
4. Review via PR on that one file; CI runs check mode; apply in batches.

**Likely follow-ups:**
- *How do you remove sudo files for people who lost sudo?* A second looped task with `state: absent` `when: not (item.sudo | default(false))`.
- *Isn't central identity better?* Yes for large orgs (LDAP/IdM/SSO + SSH certificates); this pattern is for smaller fleets or break-glass local accounts.
