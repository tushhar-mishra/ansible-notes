# 25 · Variable Precedence

> **Level:** ★★ intermediate
>
> **You will learn:** every place a variable can be defined, the **complete precedence ladder** (22 levels, from role defaults at the bottom to `-e` at the top) as one table and one diagram, the **five rules** that cover real life, where to put a variable so it's easy (or hard) to override, the traps (role `vars` that can't be overridden, facts that beat `host_vars`, dictionaries that replace instead of merge), and a method to answer "**why does this variable have this value?**". You'll prove the ladder in the lab by setting one variable in ten places.
>
> **You need:** [24 · group_vars and host_vars](24-group-vars-and-host-vars.md), [23 · Facts](23-facts-and-magic-variables.md), [21 · Variables basics](21-variables-basics.md), [11 · ansible.cfg](11-connection-settings-and-ansible-cfg.md) (settings precedence). Roles appear in the ladder; they're introduced properly in topic 35. For now: a **role** is a reusable folder of tasks with its own `defaults/` (easy-to-override values) and `vars/` (hard-to-override values).
>
> **Part 3 · Variables, facts and logic** · Next: [26 · register and when](26-register-and-when.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** The same variable name can be set in many places. Ansible doesn't merge them; it picks **one** value per host, using a fixed ranking. **Higher wins.**

**Picture it.** A ladder. Each rung is a place a variable can live. Ansible starts at the bottom and climbs; every time it finds the variable on a higher rung, the new value replaces the old one. What's left at the top is the answer.

**The one thing to know.** The short ladder (lowest → highest):

```
 role defaults                        ← weakest: "suggested values", made to be overridden
 inventory: group_vars/all → group_vars/<group> → host_vars/<host>
 facts (gathered / cached)
 play: vars → vars_prompt → vars_files
 role vars
 block vars → task vars
 include_vars
 set_fact / register
 role & include parameters
 -e  extra vars                       ← strongest: always wins
```

**Five rules for real life:**

1. **`-e` always wins.**
2. **Role defaults always lose.** Put every tunable value there.
3. In the inventory, **more specific wins**: host > child group > parent group > `all`.
4. **Play-level** variables beat **inventory** variables.
5. **Runtime** values (`set_fact`, `register`, `include_vars`) beat almost everything defined in files.

**Try it now:** set `color: green` in `group_vars/all.yml` and `color: blue` in the play's `vars:`, then run with `-e color=red`. Print `{{ color }}` each time.

**You've got it if you can:** say which value wins between `group_vars/web.yml`, `host_vars/web1.yml`, play `vars:` and `-e`, and where you'd put a value that users of a role should override.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### The full ladder

From the official ansible-core rules, lowest (1) to highest (22). A higher number replaces a lower one for the same host.

| # | Source | Example | Notes |
|---|---|---|---|
| 1 | Command-line values such as `-u user` | `ansible-playbook -u deploy` | Not variables: connection settings (topic 11). Any variable beats them |
| 2 | **Role defaults** | `roles/nginx/defaults/main.yml` | Weakest variable source. Designed to be overridden |
| 3 | Inventory file/script **group** vars | `[web:vars]` in `inventory.ini` | Inline in the inventory |
| 4 | Inventory `group_vars/all` | `inventories/prod/group_vars/all.yml` | |
| 5 | Playbook `group_vars/all` | `<playbook dir>/group_vars/all.yml` | |
| 6 | Inventory `group_vars/*` | `inventories/prod/group_vars/web.yml` | Groups ordered by depth/priority/name (topic 24) |
| 7 | Playbook `group_vars/*` | `<playbook dir>/group_vars/web.yml` | |
| 8 | Inventory file/script **host** vars | `web1 http_port=8080` in `inventory.ini` | |
| 9 | Inventory `host_vars/*` | `inventories/prod/host_vars/web1.yml` | |
| 10 | Playbook `host_vars/*` | `<playbook dir>/host_vars/web1.yml` | |
| 11 | **Host facts** and cached `set_fact` | `ansible_facts`, facts from the cache | Beat everything in the inventory |
| 12 | **Play `vars:`** | `vars: { http_port: 8080 }` | |
| 13 | Play `vars_prompt:` | | |
| 14 | Play `vars_files:` | `vars_files: [vars/prod.yml]` | |
| 15 | **Role vars** | `roles/nginx/vars/main.yml` | ⚠️ Beat inventory **and** play vars: hard to override |
| 16 | **Block vars** | `block:` + `vars:` | Only for tasks in that block |
| 17 | **Task vars** | a task's `vars:` | Only for that task |
| 18 | `include_vars` | `ansible.builtin.include_vars: file.yml` | Loaded at run time |
| 19 | **`set_fact`** and **registered** vars | `set_fact: x=1`, `register: result` | Run time, per host |
| 20 | Role (and `include_role`) **params** | `roles: [{ role: nginx, http_port: 81 }]` | Arguments passed to a role |
| 21 | `include_*` params | `include_tasks: x.yml` + `vars:` | |
| 22 | **Extra vars** (`-e`) | `-e http_port=9090` | **Always win** |

### The ladder as a diagram

```
 STRONGEST ┌────────────────────────────────────────────────┐
           │ -e extra vars                                  │  CLI, per run
           ├────────────────────────────────────────────────┤
           │ role params · include params                   │
           │ set_fact · register                            │  RUN TIME
           │ include_vars                                   │
           ├────────────────────────────────────────────────┤
           │ task vars                                      │
           │ block vars                                     │  PLAYBOOK
           │ role vars  (roles/x/vars/)  ⚠️ hard to override │
           │ play vars_files · vars_prompt · vars           │
           ├────────────────────────────────────────────────┤
           │ facts (gathered/cached)                        │  HOST
           ├────────────────────────────────────────────────┤
           │ host_vars/  (playbook-side > inventory-side)   │
           │ inline host vars                               │
           │ group_vars/<group>  (child > parent)           │  INVENTORY
           │ group_vars/all                                 │
           │ inline group vars                              │
           ├────────────────────────────────────────────────┤
           │ role defaults (roles/x/defaults/)              │  DEFAULTS
 WEAKEST   └────────────────────────────────────────────────┘
```

### Where to put a variable (the practical guide)

| You want... | Put it in | Why |
|---|---|---|
| A **tunable default** shipped with reusable code | **Role `defaults/`** | Anyone can override it from inventory, play, or `-e` |
| Environment-specific values | Inventory `group_vars/` (per environment inventory) | Same playbook, different values per env |
| One host's exception | `host_vars/<host>.yml` | Most specific inventory level |
| Values that belong to one playbook | Play `vars:` or `vars_files:` | Local and visible |
| Constants a role **must not** have overridden by accident (internal names, OS-specific paths) | Role `vars/` | High precedence. Use sparingly |
| Values computed at run time | `set_fact` | |
| A one-off override for this run | `-e` | Wins always |

### The traps

| Trap | What happens | Fix |
|---|---|---|
| **Role `vars/` overriding your inventory** | You set `nginx_port: 8080` in `group_vars/web.yml`, but the role has `nginx_port: 80` in `vars/main.yml` → you get 80 | Roles should put tunables in `defaults/`. If it's a third-party role, pass the value as a role parameter or `-e` |
| **Facts beat `host_vars`** (level 11 > 9 and 10) | You define a variable named like a fact and it's replaced | Never name variables like facts. Use prefixes |
| **`set_fact` beats almost everything**, for the rest of the run | A `set_fact` in an early play silently overrides `group_vars` in later plays | Use distinct names for computed values (`computed_*`, `_tmp_*`) |
| **`-e` beats `set_fact`** | You `set_fact: x=1` but `-e x=0` was passed: x stays 0 | Expected. Don't pass via `-e` what tasks must compute |
| **Dictionaries replace, not merge** | `group_vars/all: app: {port: 80, workers: 2}` and `group_vars/web: app: {port: 8080}` → web hosts get `{port: 8080}` only; `workers` is **gone** | Flat variables (`app_port`, `app_workers`), or merge explicitly with `combine` (topic 32). ⚠️ `hash_behaviour = merge` in ansible.cfg is discouraged |
| **Same name in sibling groups** | Alphabetical group order decides (topic 24) | Don't overlap; or `ansible_group_priority` |
| **`-e` strings** | `-e workers=4` is a string | JSON `-e '{"workers": 4}'` (topic 21) |

---

## Why it exists

Variables come from different **people** with different **authority**:

- A role author knows sensible defaults (lowest).
- The inventory owner knows what each environment and host needs (middle).
- The playbook author decides what this particular run of automation should do (higher).
- The tasks compute things at run time (higher still).
- The person running the command has the final say for this run (highest).

The ladder encodes exactly that, so a role from Galaxy can be configured without editing it, and a human can always override from the command line in an emergency.

---

## Analogy

**Settings on a company laptop.**

1. The **manufacturer's factory settings** (role defaults): everything can be changed.
2. The **company IT policy** for all laptops, then per department, then for your machine (inventory: `all` → group → host).
3. What the laptop **detects about itself**: screen size, battery (facts).
4. The **project you're working on** has its own settings (play vars).
5. A few **locked settings** IT burned into an app (role vars): you can't easily change them.
6. Things the app **computes while running** (set_fact, register).
7. **You, typing a command-line flag** right now (`-e`): beats everything, for this run only.

---

## How it works inside

Ansible doesn't store "one variable"; for each task and host it builds a fresh dictionary:

```
 for task T, host H:
   vars = {}
   vars.update(role defaults of the role T belongs to)
   vars.update(inventory group vars for H's groups, in group order)
   vars.update(inventory/playbook host_vars for H)
   vars.update(facts for H)
   vars.update(play vars, vars_prompt, vars_files)
   vars.update(role vars of T's role)
   vars.update(block vars of T's blocks), vars.update(task vars of T)
   vars.update(include_vars results, set_fact/registered for H)
   vars.update(role/include params)
   vars.update(extra vars)
   + magic variables (hostvars, groups, inventory_hostname, ...)
   → render T's arguments with vars (Jinja2, lazy)
```

Each `update` **replaces** keys that already exist (that's why dictionaries don't merge). Scopes matter: task vars exist only while that task runs; role defaults/vars are visible while the role's tasks run (and, by default, role variables are also exposed to the rest of the play after the role, which surprises people; topic 35 covers `public` and `DEFAULT_PRIVATE_ROLE_VARS`).

---

## Code / commands

### Prove the ladder

Add `color` in ten places in the lab:

```yaml
# group_vars/all.yml          (add)   color: "4-group_vars-all"
# group_vars/app.yml          (new)   color: "6-group_vars-app (parent)"
# group_vars/web.yml          (add)   color: "6-group_vars-web (child)"
# host_vars/web1.yml          (add)   color: "9-host_vars-web1"
# vars/precedence.yml         (new)   color: "14-vars_files"
```

`~/ansible-lab/precedence.yml`:

```yaml
---
- name: Variable precedence demo
  hosts: web
  gather_facts: false
  vars:
    color: "12-play-vars"
  vars_files:
    - vars/precedence.yml

  tasks:
    - name: 1. after play vars and vars_files
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }}: {{ color }}"

    - name: 2. task vars
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }}: {{ color }}"
      vars:
        color: "17-task-vars"

    - name: 3. block vars vs task vars
      vars:
        color: "16-block-vars"
      block:
        - name: 3a. inside the block
          ansible.builtin.debug:
            msg: "{{ inventory_hostname }}: {{ color }}"
        - name: 3b. task vars inside the block
          ansible.builtin.debug:
            msg: "{{ inventory_hostname }}: {{ color }}"
          vars:
            color: "17-task-vars-in-block"

    - name: 4. set_fact
      ansible.builtin.set_fact:
        color: "19-set_fact"

    - name: 5. after set_fact
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }}: {{ color }}"
```

- `block:` groups tasks so they share settings like `vars` (topic 31 covers blocks properly).

```bash
ansible-playbook precedence.yml
```
```
TASK [1. after play vars and vars_files] ***********************************************
ok: [web1] => { "msg": "web1: 14-vars_files" }
ok: [web2] => { "msg": "web2: 14-vars_files" }

TASK [2. task vars] ********************************************************************
ok: [web1] => { "msg": "web1: 17-task-vars" }
ok: [web2] => { "msg": "web2: 17-task-vars" }

TASK [3a. inside the block] ************************************************************
ok: [web1] => { "msg": "web1: 16-block-vars" }
...
TASK [3b. task vars inside the block] **************************************************
ok: [web1] => { "msg": "web1: 17-task-vars-in-block" }
...
TASK [5. after set_fact] ***************************************************************
ok: [web1] => { "msg": "web1: 19-set_fact" }
ok: [web2] => { "msg": "web2: 19-set_fact" }
```

- Task 1: `vars_files` (14) beat play `vars` (12) and all inventory levels.
- Task 2: task vars (17) win, but **only for that task**.
- Task 5: `set_fact` (19) beats everything in files.

Now the extra vars:

```bash
ansible-playbook precedence.yml -e color=22-extra-vars
```
```
ok: [web1] => { "msg": "web1: 22-extra-vars" }      ← in EVERY task, even after set_fact
```

Now peel off the playbook levels to see the inventory ladder. Comment out `vars:` and `vars_files:` in the play, run it again, and look at task 1:

```bash
ansible-playbook precedence.yml
```
```
ok: [web1] => { "msg": "web1: 9-host_vars-web1" }
ok: [web2] => { "msg": "web2: 6-group_vars-web (child)" }
```

- web1: its `host_vars` file (9) wins.
- web2: no host file; `web` (child, depth 2) beats `app` (parent) and `all`.

### The dictionary-replacement trap

```yaml
# group_vars/all.yml
app:
  port: 80
  workers: 2
# group_vars/web.yml
app:
  port: 8080
```

```bash
ansible web1 -m ansible.builtin.debug -a "var=app"
```
```
web1 | SUCCESS => {
    "app": {
        "port": 8080
    }
}
```

`workers` disappeared: the whole `app` dictionary was replaced. Fix with flat names (`app_port`, `app_workers`) or an explicit merge:

```yaml
# group_vars/all.yml
app_defaults: { port: 80, workers: 2 }
# group_vars/web.yml
app_overrides: { port: 8080 }
# in the playbook
app: "{{ app_defaults | combine(app_overrides | default({})) }}"    # {port: 8080, workers: 2}
```

### "Why does this variable have this value?" — a method

1. **What's the final value?** `ansible <host> -m ansible.builtin.debug -a "var=<name>"` (inventory + defaults level), or a `debug` task right before the failing task (all levels).
2. **What does the inventory say?** `ansible-inventory --host <host> | grep <name>`.
3. **Where is it defined at all?** `grep -rn '<name>:' group_vars host_vars inventories roles/*/defaults roles/*/vars vars playbooks *.yml`.
4. **Walk up the ladder**: for each hit, find its rung. The highest rung that applies to this host and this task wins.
5. **Check runtime sources**: `set_fact`, `register`, `include_vars` in earlier tasks or plays; `-e` in the command line / CI job / AWX template.
6. Still confused? Temporarily pass `-e <name>=TEST`. If that doesn't change the outcome, the value isn't coming from this variable at all (wrong name, or the role uses a different one).

---

## Hands-on lab

1. Add `color` to the five files listed above, create `precedence.yml`, and run it. Match each output line to its rung number.
2. Run with `-e color=...` and confirm it wins everywhere, even after `set_fact`.
3. Remove levels one by one (play vars, vars_files, host_vars) and predict each result before running.
4. Reproduce the dictionary-replacement trap and fix it with `combine`.
5. Put a variable called `ansible_hostname` in `host_vars/web1.yml` (bad idea!), gather facts, and print it. The fact wins (level 11 > 9). Remove it. This is why you never name variables like facts.
6. Clean up: remove all `color` entries and the demo files.

**Done when:** you can look at any variable's definitions and say the winner without running anything.

---

## Where you'll use it

- **Writing roles**: tunables in `defaults/`, internal constants in `vars/` (topics 35–36).
- **Environments**: inventory `group_vars` per environment; play vars only for playbook-specific values.
- **CI**: `-e app_version=...` to deploy a specific version (wins over everything).
- **Incident response**: `-e` to force a value temporarily without editing files.
- **Debugging**: the "why this value" method above is a weekly task in big repos.
- **Interviews**: "Explain variable precedence" is one of the most common Ansible questions.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| Your `group_vars` value is ignored | Something higher sets it: role `vars/`, play vars, `set_fact`, `-e` | Follow the method above |
| A role ignores your inventory setting | The role put it in `vars/` (level 15) | Override via role params or `-e`; fix the role to use `defaults/` |
| Value correct in play 1, wrong in play 2 | A `set_fact`/`include_vars` in play 1 persists for the host | Use different names for computed values |
| Dictionary lost keys | Variables replace, not merge | Flat names or `combine` |
| `-e` override "doesn't work" | Wrong variable name, or the template uses a different variable | `-e name=TEST` experiment; grep the template |
| Different value on two hosts of the same group | One has a `host_vars` file, or is in another group that wins | `ansible-inventory --host` both |
| Behaviour changed after renaming a group | Alphabetical group order changed | Avoid overlapping variables across sibling groups |

---

## Quick recap

- One value per host per variable, chosen by a fixed ladder; **higher replaces lower**.
- Bottom: **role defaults**. Middle: inventory (`all` < group < child group < host), then **facts**, then **play** (`vars` < `vars_prompt` < `vars_files`), **role vars**, block/task vars. Top: `include_vars`, **`set_fact`/`register`**, role/include params, and **`-e`** (always wins).
- Put tunables in **role defaults**, environment values in **inventory group_vars**, exceptions in **host_vars**, constants in **role vars** (sparingly).
- Traps: role `vars/` override inventory; facts beat `host_vars`; `set_fact` persists; **dicts replace**.
- Debug: final value with `debug`, inventory value with `ansible-inventory --host`, all definitions with `grep`, then walk the ladder.

### Interview questions

1. **Q: Explain Ansible's variable precedence in simple terms.**
   A: Role defaults are weakest; then inventory variables (all → groups → host), then facts, then play vars and vars_files, then role vars, block and task vars, then runtime values (include_vars, set_fact, register), role/include params, and extra vars (`-e`), which always win.
2. **Q: Where should a role author put a value users are expected to change?**
   A: In the role's `defaults/main.yml`, the lowest precedence, so inventory, play or `-e` can override it.
3. **Q: You set a variable in `group_vars/web.yml` but the role uses a different value. Why?**
   A: Most likely the role defines it in `vars/main.yml` (which beats inventory), or something higher (play vars, set_fact, `-e`) sets it.

### Scenario question (answer out loud first, then compare)

> *"A third-party nginx role from Galaxy ignores your `nginx_worker_processes: 8` in `group_vars/web.yml`; it always uses `auto`. You don't want to fork the role. What are your options, and which do you pick?"*

**Model answer:**
1. Find where the role sets it: `grep -rn nginx_worker_processes roles/<role>/`. If it's in `vars/main.yml`, it outranks inventory (level 15 vs 6).
2. Options without forking:
   - pass it as a **role parameter** (`roles: [{ role: x, nginx_worker_processes: 8 }]` or `include_role` with `vars:`), which is level 20-21 and beats role vars;
   - `-e nginx_worker_processes=8` (works, but it's global for the run and easy to forget in CI);
   - check whether the role offers another variable meant for overriding (some roles compute the value from `nginx_worker_processes_default` or similar in `defaults/`).
3. Pick the role parameter for a per-play setting, documented in the playbook. Also open an issue/PR upstream: tunables belong in `defaults/`.
4. Verify with `--check --diff` that the rendered `nginx.conf` now has `worker_processes 8;`.

**Likely follow-ups:**
- *Why not `hash_behaviour = merge`?* It changes merging globally, makes behaviour surprising for everyone, and is discouraged; use `combine` where you need merges.
- *Does a role parameter leak to other roles?* Role params are scoped to that role's execution, which is the point.
