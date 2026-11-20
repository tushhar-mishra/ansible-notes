# 21 · Variables Basics

> **Level:** ★ foundation
>
> **You will learn:** what variables are for, the naming rules (and the names you must **not** use), the basic places to define them (**play `vars`**, **`vars_files`**, **`vars_prompt`**, **`-e` extra vars**), how to use them with `{{ }}` and when you must quote, the **types** (string, number, boolean, list, dictionary) and how to reach inside lists and dictionaries, the **`debug`** module (`msg` vs `var`), why `-e port=8080` gives you a **string**, and what "`'x' is undefined`" means. Variables in the inventory come in topic 24; the full precedence order in topic 25.
>
> **You need:** [20 · Safe runs](20-check-diff-limit.md), [04 · YAML](04-yaml-for-ansible.md) (types and quoting), [14 · First playbook](14-first-playbook.md).
>
> **Part 3 · Variables, facts and logic** · Next: [22 · Jinja2 basics](22-jinja2-basics.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **variable** is a named value. Instead of writing `8080` in ten tasks, you write `app_port: 8080` once and use `{{ app_port }}` everywhere. Change it in one place; the same playbook works for dev, staging and prod with different values.

**Picture it.** A form letter: "Dear `{{ name }}`, your order `{{ order_id }}` ships on `{{ date }}`." One template, many letters.

**The one thing to know.**

```yaml
- name: Variables demo
  hosts: web
  vars:
    app_name: shop
    app_port: 8080
    app_packages: [git, tree]
    app_owner:
      name: alice
      uid: 2001
  tasks:
    - name: Show them
      ansible.builtin.debug:
        msg: "{{ app_name }} listens on {{ app_port }}, owner {{ app_owner.name }} ({{ app_owner['uid'] }}), first package {{ app_packages[0] }}"
```

```bash
ansible-playbook vars.yml                       # app_port = 8080
ansible-playbook vars.yml -e app_port=9090      # -e beats everything → 9090 (as a STRING)
```

**Try it now:** run both and compare the `msg`.

**You've got it if you can:** define a list and a dictionary variable, print one element of each, and override a value from the command line.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Naming rules

| Rule | ✅ Good | ❌ Bad |
|---|---|---|
| Letters, digits, underscores only | `app_port`, `db_name2` | `app-port` (dash = minus), `app.port` (dot = attribute access), `app port` |
| Must start with a letter or `_` | `_internal`, `nginx_workers` | `2nd_port` |
| Not a Python keyword | | `class`, `for`, `if`, `in`, `not`, `async`, `lambda`... |
| Not a playbook keyword (confusing, sometimes breaks) | | `name`, `hosts`, `environment`, `tags`, `vars`, `when` |
| Don't start with `ansible_` | | `ansible_port` (that's a **connection** variable!), `ansible_mything` |
| Avoid names of Ansible's magic variables | | `hostvars`, `groups`, `inventory_hostname`, `item`, `omit` |

**Convention:** prefix variables with their purpose or role: `nginx_port`, `app_version`, `db_backup_hour`. In roles this prevents two roles from fighting over `port` (topic 36).

### Types

YAML decides the type when you write the value (topic 04):

```yaml
vars:
  app_name: shop                 # string
  app_port: 8080                 # integer
  app_ratio: 0.75                # float
  app_debug: false               # boolean
  app_nothing: null              # None (also ~ or empty)
  app_packages:                  # list
    - git
    - tree
  app_owner:                     # dictionary (map)
    name: alice
    uid: 2001
  app_users:                     # list of dictionaries (very common)
    - name: alice
      shell: /bin/bash
    - name: bob
      shell: /bin/zsh
```

### Using variables

| You write | Gets |
|---|---|
| `"{{ app_name }}"` | `shop` |
| `"{{ app_packages[0] }}"` | `git` (lists start at index 0) |
| `"{{ app_packages[-1] }}"` | `tree` (last item) |
| `"{{ app_owner.name }}"` | `alice` (dot notation) |
| `"{{ app_owner['name'] }}"` | `alice` (bracket notation) |
| `"{{ app_users[1].shell }}"` | `/bin/zsh` |

**Dot or bracket?** Both work, but **brackets are safer**:

- Keys with dashes or starting with digits only work in brackets: `config['max-conn']`.
- Keys that clash with Python dict methods break with dots: `mydict.items`, `mydict.keys`, `mydict.values`, `mydict.update` return the **method**, not your key. `mydict['items']` is always your key.
- Keys stored in another variable need brackets: `app_owner[field_name]`.

### Quoting rule (from topic 04)

If the YAML value **starts** with `{{`, quote the whole value:

```yaml
dest: "{{ app_dir }}/app.conf"        # must quote: starts with {{
dest: /opt/{{ app_name }}/app.conf    # fine unquoted: starts with /
msg: "Port: {{ app_port }}"           # quoting anyway is a good habit
```

### A bare value keeps its type; a mixed string becomes a string

```yaml
port: "{{ app_port }}"          # the value is the integer 8080 (just the variable)
url: "http://x:{{ app_port }}"  # a string: "http://x:8080"
```

When a value is exactly one `{{ expression }}`, Ansible keeps the result's **native type** (a list stays a list, a number stays a number). Older versions already did this in most cases; ansible-core 2.19's new templating engine made it consistent and stricter (for example, conditionals must now evaluate to real booleans). Mixing text with `{{ }}` always gives a string.

### Where to define variables (the basics)

| Place | Scope | Example |
|---|---|---|
| **Play `vars:`** | That play | `vars: { app_port: 8080 }` |
| **`vars_files:`** | That play, loaded from YAML files | `vars_files: [vars/common.yml]` |
| **`vars_prompt:`** | That play, asked interactively at start | Passwords for manual runs |
| **Block/task `vars:`** | That block/task only | Override one value for one task |
| **`-e` / `--extra-vars`** | Whole run, **highest precedence** | `-e app_port=9090`, `-e @prod.yml` |
| Inventory, `group_vars/`, `host_vars/` | Per host / group | Topic 24 |
| Role `defaults/` and `vars/` | Per role | Topic 35 |
| Facts, `register`, `set_fact` | Per host, created at run time | Topics 23, 26 |

All of them merge into **one set of variables per host**. Which one wins when the same name appears twice is topic 25. Short version for now: **`-e` always wins**; more specific usually beats more general.

### `-e` / `--extra-vars` formats

| Format | Example | Types |
|---|---|---|
| `key=value` pairs | `-e "app_port=9090 app_debug=true"` | ⚠️ **All strings**: `"9090"`, `"true"` |
| JSON | `-e '{"app_port": 9090, "app_debug": true}'` | Real types |
| YAML/JSON file | `-e @vars/prod.yml` | Real types |

The string surprise matters in conditions: `-e app_debug=false` gives the **string** `"false"`, which is "truthy" (non-empty). Ansible 2.19+ refuses non-boolean conditions, so `when: app_debug` with a string fails (topic 26). Use JSON, or convert: `when: app_debug | bool` (filters, topic 22).

### The `debug` module

| Form | Prints |
|---|---|
| `msg: "text {{ var }}"` | A rendered message |
| `var: app_owner` | The variable's value, pretty-printed. **No `{{ }}`**: `var:` takes a name or expression |
| `verbosity: 1` | Only print with `-v` or higher (debug output that stays in the code quietly) |

### Undefined variables

Using a variable that isn't defined (for this host) fails **that task on that host**:

```
fatal: [web1]: FAILED! => {"msg": "The task includes an option with an undefined variable. 'app_portt' is undefined. ..."}
```

Usually a typo, a variable defined for another group, or a missing vars file. Topic 22 shows the `default` and `mandatory` filters for handling optional and required values.

---

## Why it exists

Without variables, playbooks are full of copy-pasted values: ports, versions, paths, user names. Changing one means editing many files and hoping you caught them all. Different environments need different values (prod has 8 workers, dev has 1), so you'd end up with three almost-identical playbooks.

Variables separate **what you do** (tasks, the same everywhere) from **the values you do it with** (different per environment, group or host). That's the foundation for roles, inventories per environment, and reuse.

---

## Analogy

**A recipe with blanks.** "Bake at `{{ temp }}`°C for `{{ minutes }}` minutes, using `{{ flour_type }}` flour." The recipe (playbook) stays the same; the bakery in Paris fills in different values from the one in Delhi (group variables). The head chef can override any value for today only by shouting it (`-e`), and nobody argues with the head chef.

---

## How it works inside

```
 before each task, for each host:
   1. collect variables from every source (inventory, group_vars, play vars, vars_files,
      facts, registered results, set_fact, -e, ...)
   2. merge them by precedence → ONE dictionary for this host        (topic 25)
   3. render the task's arguments with Jinja2 using that dictionary   (on the control node)
        "{{ app_owner.name }}" → "alice"
        undefined name → error for this host
   4. send the rendered arguments to the module
```

Variables are **lazy**: an expression is only evaluated when a task actually uses it. A variable that refers to an undefined one doesn't fail until something uses it.

---

## Code / commands

### The lab playbook

`~/ansible-lab/vars/common.yml`:

```yaml
app_name: shop
app_base_dir: /opt/shop
app_packages:
  - git
  - tree
```

`~/ansible-lab/vars.yml`:

```yaml
---
- name: Variables in practice
  hosts: web
  become: true
  vars_files:
    - vars/common.yml
  vars:
    app_port: 8080
    app_debug: false
    app_owner:
      name: alice
      uid: 2001
    app_users:
      - name: alice
        shell: /bin/bash
      - name: bob
        shell: /bin/bash

  tasks:
    - name: Show the scalars
      ansible.builtin.debug:
        msg: "{{ app_name }} on port {{ app_port }} (debug={{ app_debug }}) in {{ app_base_dir }}"

    - name: Show a whole dictionary
      ansible.builtin.debug:
        var: app_owner

    - name: Show one item of a list of dictionaries
      ansible.builtin.debug:
        msg: "Second user is {{ app_users[1].name }} with {{ app_users[1]['shell'] }}"

    - name: Packages from a list variable
      ansible.builtin.package:
        name: "{{ app_packages }}"
        state: present

    - name: App directory from variables
      ansible.builtin.file:
        path: "{{ app_base_dir }}"
        state: directory
        owner: "{{ app_owner.name }}"
        mode: "0755"

    - name: Config file built from variables
      ansible.builtin.copy:
        dest: "{{ app_base_dir }}/app.env"
        content: |
          APP_NAME={{ app_name }}
          APP_PORT={{ app_port }}
          APP_DEBUG={{ app_debug }}
        owner: "{{ app_owner.name }}"
        mode: "0640"

    - name: Only printed with -v
      ansible.builtin.debug:
        msg: "Detailed info for {{ inventory_hostname }}"
        verbosity: 1
```

- `vars_files: [vars/common.yml]`: load variables from a file (path relative to the playbook).
- `name: "{{ app_packages }}"`: passes the whole **list** to the module (native type kept).
- `content: |` with `{{ }}` inside: rendered per host before copying.

### Run it

```bash
ansible-playbook vars.yml
```
```
TASK [Show the scalars] ****************************************************************
ok: [web1] => {
    "msg": "shop on port 8080 (debug=False) in /opt/shop"
}
ok: [web2] => {
    "msg": "shop on port 8080 (debug=False) in /opt/shop"
}

TASK [Show a whole dictionary] *********************************************************
ok: [web1] => {
    "app_owner": {
        "name": "alice",
        "uid": 2001
    }
}
...
TASK [Show one item of a list of dictionaries] *****************************************
ok: [web1] => {
    "msg": "Second user is bob with /bin/bash"
}
...
TASK [Only printed with -v] ************************************************************
skipping: [web1]
skipping: [web2]
```

- `debug=False`: a Python boolean printed inside a string. (In the `.env` file it's written as `False` too. Topic 22 shows how to print `false` or `0` instead.)
- `var: app_owner` prints the variable **name** as the key.
- `verbosity: 1` → `skipping` without `-v`.

### Override with -e

```bash
ansible-playbook vars.yml -e app_port=9090 -l web1
ansible-playbook vars.yml -e '{"app_port": 9090, "app_debug": true}' -l web1
ansible-playbook vars.yml -e @vars/override.yml -l web1
```

Check the type difference:

```yaml
- name: What type is app_port?
  ansible.builtin.debug:
    msg: "{{ app_port | type_debug }}"
```

- `-e app_port=9090` → `str`
- `-e '{"app_port": 9090}'` → `int`

(`type_debug` is a filter that prints a value's Python type. Very handy.)

### Ask at run time: vars_prompt

```yaml
- name: Prompt demo
  hosts: localhost
  gather_facts: false
  vars_prompt:
    - name: release_note
      prompt: "What does this release change?"
      private: false
    - name: admin_password
      prompt: "Admin password"
      private: true
  tasks:
    - name: Use the answers
      ansible.builtin.debug:
        msg: "Note: {{ release_note }}"
```

- `private: true`: input isn't echoed (and since 2.22 it's masked as a secret in output).
- Prompts are skipped if the variable is already set with `-e`, which is how CI avoids them.

### An undefined variable

```yaml
- name: Typo
  ansible.builtin.debug:
    msg: "{{ app_portt }}"
```
```
fatal: [web1]: FAILED! => {"msg": "The task includes an option with an undefined variable. 'app_portt' is undefined. 'app_portt' is undefined\n\nThe error appears to be in '/home/you/ansible-lab/vars.yml': line 62, column 7, but may\nbe elsewhere in the file depending on the exact syntax problem.\n..."}
```

- The message names the variable and the file/line.

---

## Hands-on lab

1. Create `vars/common.yml` and `vars.yml`, run them, and look at `/opt/shop/app.env` on web2.
2. Override `app_port` three ways (`key=value`, JSON, `@file`) and print `app_port | type_debug` each time.
3. Add a task-level `vars:` that sets `app_port: 7070` for one debug task only. Confirm other tasks still see 8080 (and that `-e` still beats the task-level value).
4. Try names you shouldn't use: add `app-port: 1` to `vars:` and use `{{ app-port }}`. Read the error (Jinja2 sees `app - port`). Remove it.
5. Use the `items` trap:

   ```yaml
   vars:
     shop: { items: 42 }
   tasks:
     - ansible.builtin.debug:
         msg: "dot: {{ shop.items }} / bracket: {{ shop['items'] }}"
   ```

   The dot version prints a Python method description; the bracket version prints `42`.
6. Misspell a variable and read the error's file/line.

**Done when:** you can predict the type and value of any variable from where and how it was set.

---

## Where you'll use it

- **Every role and playbook**: ports, versions, paths, users, feature flags.
- **Environments**: same playbook, different `group_vars` for dev/staging/prod (topics 24, 39).
- **CI pipelines**: pass the version being deployed with `-e app_version=$GIT_TAG` (topics 53–54).
- **Templates**: config files filled from variables (topic 29).

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `'x' is undefined` | Typo, not defined for this host, vars file not loaded | Check spelling; `ansible-inventory --host`; `debug: var=hostvars[inventory_hostname]` |
| `found unhashable key` / missing quotes hint | `key: {{ var }}` unquoted | `key: "{{ var }}"` |
| `'dict object' has no attribute 'nmae'` | Typo in a dictionary key | Check the key; print the dict with `debug: var=` |
| `list object has no element 3` | Index out of range | Check the list length (`| length`) |
| Value has the wrong type (`"8080"` vs `8080`) | `-e key=value` makes strings | Use JSON `-e '{...}'` or filters `| int`, `| bool` |
| `Conditional result ... was of type 'str'` | String from `-e` used in `when` | `when: my_flag | bool` |
| `msg: "{{ dict.items }}"` prints `<built-in method items...>` | Key clashes with a dict method | Brackets: `dict['items']` |
| Variable seems ignored | A higher-precedence source sets it (often `-e` or `host_vars`) | Topic 25 |
| `app-port` causes `'app' is undefined` | Dash = minus in Jinja2 | Use underscores |

---

## Quick recap

- Variables = named values; names use **letters, digits, underscores**; don't use keywords, magic names, or the `ansible_` prefix; prefix by purpose (`nginx_port`).
- Types: string, int, float, bool, null, **list**, **dict**. Access with `list[0]`, `dict.key` or (safer) `dict['key']`.
- Define in play `vars`, `vars_files`, `vars_prompt`, task/block `vars`, and `-e` (highest precedence). Inventory and role variables come next.
- Quote values that start with `{{`. A single `"{{ var }}"` keeps its native type; mixed text makes a string.
- `-e key=value` gives **strings**; use JSON or `@file` for real types. `debug: msg=` vs `debug: var=`.

### Interview questions

1. **Q: How do you override a variable for a single run?**
   A: With extra vars: `-e var=value`, `-e '{"var": 1}'` or `-e @file.yml`. Extra vars have the highest precedence.
2. **Q: Why might `-e debug=false` still enable debug?**
   A: `key=value` extra vars are strings, and `"false"` is a non-empty string. Pass JSON (`-e '{"debug": false}'`) or use `| bool`.
3. **Q: `dict.key` vs `dict['key']`?**
   A: Both access a key; brackets also work for keys with special characters, keys held in variables, and keys that clash with dict methods like `items` or `keys`.

### Scenario question (answer out loud first, then compare)

> *"Your team has three copies of the same deploy playbook (dev, staging, prod) that differ only in ports, worker counts and hostnames. They keep drifting apart. How do you restructure it?"*

**Model answer:**
1. Keep **one** playbook with the tasks, and move every value that differs into variables with clear prefixes (`app_port`, `app_workers`).
2. Put the values in per-environment inventories: `inventories/{dev,staging,prod}/group_vars/all.yml` (topics 24, 39). Choose the environment with `-i`.
3. Put sensible defaults in one place (later: role `defaults/`, topic 35) and only override what differs.
4. Use `-e` only for per-run values (like the version being deployed), not for environment config.
5. Verify with `ansible-inventory -i inventories/prod --host web1` and a `--check --diff` run per environment.

**Likely follow-ups:**
- *How do you stop someone forgetting to set a required value?* `mandatory` filter or `assert` (topics 22, 31), or role `argument_specs` (topic 36).
- *Where do secrets go?* Vaulted files in the same `group_vars` (topic 40).
