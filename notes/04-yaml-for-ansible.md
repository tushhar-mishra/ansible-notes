# 04 · YAML for Ansible

> **Level:** ★ foundation
>
> **You will learn:** the YAML (YAML Ain't Markup Language) you need for Ansible: **maps**, **lists**, **scalars**, indentation rules, comments, multi-line strings (`|` and `>`), quoting, the **boolean traps** (`yes`, `no`, `on`, `NO` for Norway), why a value that starts with `{{` must be quoted, anchors, and how to read the **shape** of a playbook as nested lists and maps. You'll convert YAML to JSON to see exactly what Ansible sees.
>
> **You need:** [03 · What is Ansible?](03-what-is-ansible.md). If you've read [GitHub Actions topic 03](../../GitHub-Actions-Basics-To-Production/notes/03-yaml-for-github-actions.md) or [Kubernetes topic 13](../../CKA-Certification-Course-2025/notes/13-yaml-and-manifest-anatomy.md), this is a fast recap plus the Ansible-specific traps.
>
> **Part 0 · Foundations** · Next: [05 · Linux recap I: SSH](05-linux-recap-ssh.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Almost everything you write in Ansible is YAML: inventories, playbooks, variables, roles. YAML is just a way to write **data** as text. It has only three shapes:

```yaml
name: nginx              # MAP (key: value)
ports:                   # LIST (items start with "- ")
  - 80
  - 443
enabled: true            # SCALAR (a single value: string, number, boolean)
```

**Picture it.** A **map** is a form with labelled boxes. A **list** is a numbered shopping list. You can put lists inside boxes, and forms inside list items, as deep as you like. **Indentation** shows what's inside what.

**The one thing to know.** Three rules prevent 90% of YAML errors in Ansible:

1. **Spaces only, never tabs.** Use 2 spaces per level.
2. **Quote a value that starts with `{{`**: `msg: "{{ greeting }}"`.
3. **Write booleans as `true`/`false`**, not `yes`/`no`.

**Try it now:**

```bash
sudo apt install -y python3-yaml
printf 'country: NO\nport: 08\nok: yes\n' | python3 -c 'import yaml,sys; print(yaml.safe_load(sys.stdin))'
```

You'll see `{'country': False, 'port': '08', 'ok': True}`. Norway became `False`, and `08` became a **string** (not a number). That's why you quote.

**You've got it if you can:** look at any playbook and say "this is a list of maps" for each level.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

**YAML** is a text format for data, designed to be easy for humans to read. It's a superset of JSON (JavaScript Object Notation): every JSON file is valid YAML. Ansible reads YAML with a Python library (PyYAML) and turns it into Python data: **dicts** (maps), **lists** and **scalars**.

### The three shapes

| Shape | YAML | Python sees | Also called |
|---|---|---|---|
| **Map** | `key: value` lines at the same indent | `dict` | Dictionary, hash, mapping |
| **List** | lines starting with `- ` at the same indent | `list` | Sequence, array |
| **Scalar** | a single value | `str`, `int`, `float`, `bool`, `None` | |

**Map:**

```yaml
user:
  name: deploy
  shell: /bin/bash
  uid: 1500
```

`user` is a key whose value is another map with three keys.

**List:**

```yaml
packages:
  - nginx
  - git
  - chrony
```

**List of maps** (the most common shape in Ansible):

```yaml
users:
  - name: alice
    groups: wheel
  - name: bob
    groups: developers
```

Read it as: `users` is a list with 2 items; each item is a map with `name` and `groups`. The `- ` starts a new item, and the keys under it line up with the first key **after** the dash.

**Flow style** (JSON-like, on one line). Valid, useful for short things:

```yaml
ports: [80, 443]
user: {name: deploy, shell: /bin/bash}
```

### Scalars and their types

YAML guesses the type of an unquoted value:

| You write | Type Ansible gets | Notes |
|---|---|---|
| `8080` | integer | |
| `3.14` | float | |
| `"8080"` or `'8080'` | string | Quoting forces string |
| `true` / `false` | boolean | ✅ Use these |
| `yes` / `no` / `on` / `off` / `True` / `NO` | boolean | ⚠️ YAML 1.1 rules. Works, but ansible-lint warns. Avoid |
| `~` or `null` or nothing | None (null) | `key:` with no value = null |
| `0755` | integer 493 (YAML 1.1 reads a leading 0 as **octal**) | Works for `mode:` by luck. A YAML 1.2 tool reads it as decimal 755. **Always quote** modes |
| `644` | integer 644 (decimal!) | ⚠️ As a file mode this is octal 1204: a broken mode. Quote it: `"0644"` |
| `08` | string `'08'` | Not valid octal, so not a number |
| `1e3` | string in YAML 1.1 | Surprising. Quote numbers you mean as text |
| `2024-01-15` | may become a **date object**, not a string | Quote dates: `"2024-01-15"` |

**The famous traps:**

```yaml
country: NO        # → False   (the "Norway problem")
answer: yes        # → True
version: 1.10      # → float 1.1 (the trailing 0 is lost!)
mode: 644          # → decimal 644 = octal 1204: wrong permissions!
```

Fix: **quote them**.

```yaml
country: "NO"
answer: "yes"
version: "1.10"
mode: "0644"
```

### Quoting rules

| Quote style | Escapes like `\n` work? | Use when |
|---|---|---|
| No quotes | No | Plain words and numbers |
| `'single'` | No (to write `'` inside, double it: `'it''s'`) | Literal text, regexes, Windows paths |
| `"double"` | Yes (`\n`, `\t`, `\"`, `\\`) | Text with special characters, or any value starting with `{{` |

**You must quote** a value when it:

- starts with `{`, `[`, `*`, `&`, `!`, `|`, `>`, `%`, `@`, or a backtick,
- contains `: ` (colon + space) or ` #` (space + hash),
- should stay a string but looks like a number/boolean/null,
- **starts with `{{`** (Ansible's variable syntax, topic 21).

Why the last one? `{{ ... }}` is how you insert a variable's value (Jinja2 templating, topic 22). But `{` at the start of a value means "a flow-style map" to YAML, so:

```yaml
msg: {{ greeting }}           # ❌ YAML error: YAML thinks it's a map
msg: "{{ greeting }}"         # ✅
msg: Hello {{ greeting }}     # ✅ OK unquoted: doesn't START with {
```

Ansible even tells you: `We could be wrong, but this one looks like it might be an issue with missing quotes`.

### Multi-line strings

| Style | Keeps line breaks? | Use for |
|---|---|---|
| `\|` (literal block) | **Yes**, exactly | Config file content, scripts, certificates |
| `>` (folded block) | **No**, lines joined with spaces (blank line = newline) | Long sentences, long commands |

```yaml
motd: |
  Welcome to web1.
  Unauthorized access is prohibited.

long_cmd: >
  /usr/bin/backup
  --target /srv/backup
  --compress
```

- `motd` = `"Welcome to web1.\nUnauthorized access is prohibited.\n"`
- `long_cmd` = `"/usr/bin/backup --target /srv/backup --compress\n"`

Add `-` to remove the final newline (`|-`, `>-`). Useful when the trailing `\n` would break something, like a password.

### Comments, documents, anchors

```yaml
---                         # optional "start of document" marker (Ansible files often begin with it)
# a comment: everything after " #" is ignored
defaults: &web_defaults     # & = define an ANCHOR named web_defaults
  port: 80
  workers: 4

site_a:
  <<: *web_defaults         # * = use the anchor; << = merge its keys here
  name: a.example.com
```

- Anchors (`&`) and aliases (`*`) let you reuse a block in the **same file**. You'll see them sometimes; in Ansible, variables and roles are usually the better way to reuse.

### Indentation rules

1. **Spaces only.** A tab anywhere in indentation = error. Set your editor to insert spaces.
2. Items at the same level must have **exactly** the same indentation.
3. The amount (2 or 4 spaces) is up to you, but be consistent. Ansible's convention is **2**.
4. List dashes can be indented under their key or not. Both are valid:

   ```yaml
   packages:
     - nginx        # indented: common in Ansible
   packages:
   - nginx          # not indented: also valid (Kubernetes style)
   ```

---

## Why it exists

Ansible needed a format that:

- **ops people can read** without learning a programming language,
- maps cleanly to **data** (lists of tasks, maps of arguments),
- supports **comments** (JSON doesn't),
- works in **Git diffs** (one setting per line).

YAML fits. The price is that YAML's guessing (types, indentation) causes subtle bugs, which is why linters like `yamllint` and `ansible-lint` exist (topic 50).

---

## Analogy

**A filing cabinet.**

- A **map** is a folder with **labelled tabs**: "name", "shell", "uid". You find things by label.
- A **list** is a **stack of numbered sheets**: first, second, third. You find things by position.
- **Indentation** is which folder a sheet is *inside*. Move a sheet two spaces left and it falls out of its folder into the parent one, and now it means something different.
- **Quoting** is writing "TEXT" on a sheet so the clerk doesn't "helpfully" convert `NO` into "False".

---

## How it works inside

What happens to your YAML when Ansible reads a playbook:

```
 site.yml (text)
      │
      ▼  1. PyYAML parser (Ansible uses its own safe loader on top)
 Python data: list → dict → list → dict ...
      │       (types decided here: "yes" → True, 644 → decimal 644, "0644" → "0644")
      ▼  2. Ansible checks the structure
 "Is this a list of plays? Does each play have 'hosts'? Is each task a map
  with exactly one module name?"   → error if not
      │
      ▼  3. Later, when a task runs: Jinja2 renders "{{ ... }}" strings
 "{{ http_port }}" → 8080
```

Two stages matter for debugging:

- **Stage 1 errors** = YAML syntax: bad indentation, tabs, unquoted `{{`. Message: `Syntax Error while loading YAML`.
- **Stage 3 errors** = templating: a variable doesn't exist. Message: `'x' is undefined`. YAML was fine.

### The shape of a playbook

Every playbook has the same skeleton. Learn to "see" it:

```yaml
---
- name: Configure web servers      # ← LIST item 1 = a PLAY (a map)
  hosts: web                       #    key in the play map
  become: true                     #    key in the play map
  vars:                            #    key whose value is a MAP
    http_port: 8080
  tasks:                           #    key whose value is a LIST of tasks
    - name: nginx is installed     #    ← task 1 (a map)
      ansible.builtin.package:     #       module name = key, its value = MAP of arguments
        name: nginx
        state: present
    - name: nginx is running       #    ← task 2 (a map)
      ansible.builtin.systemd_service:
        name: nginx
        state: started

- name: Configure db servers       # ← LIST item 2 = another PLAY
  hosts: db
  tasks: []
```

The same thing as JSON shows the nesting without any guessing:

```json
[
  {
    "name": "Configure web servers",
    "hosts": "web",
    "become": true,
    "vars": { "http_port": 8080 },
    "tasks": [
      { "name": "nginx is installed",
        "ansible.builtin.package": { "name": "nginx", "state": "present" } },
      { "name": "nginx is running",
        "ansible.builtin.systemd_service": { "name": "nginx", "state": "started" } }
    ]
  },
  { "name": "Configure db servers", "hosts": "db", "tasks": [] }
]
```

**Playbook = list of plays. Play = map. `tasks` = list. Task = map with one module key.**

### ⚠️ The old `key=value` style

You will see this in old playbooks and blog posts:

```yaml
- name: install nginx
  dnf: name=nginx state=present           # ⚠️ old "k=v" string style
```

It still works, but it's discouraged: no types (everything is a string), hard to read when long, and linters flag it. Always write arguments as a YAML map:

```yaml
- name: nginx is installed
  ansible.builtin.dnf:
    name: nginx
    state: present
```

(The `k=v` style is still normal for **ad hoc** commands on the command line, topic 12, because you can't easily type YAML there.)

---

## Code / commands

A tiny helper to see what Ansible will see. Put this in your `~/.bashrc` in WSL2 Ubuntu:

```bash
y2j() { python3 -c 'import yaml,json,sys; print(json.dumps(yaml.safe_load(open(sys.argv[1])), indent=2, default=str))' "$1"; }
```

- `yaml.safe_load`: parse YAML into Python data (the same library family Ansible uses).
- `json.dumps(..., indent=2)`: print it as pretty JSON, so you see exact types and nesting.

**Example 1: catch a type trap.**

```yaml
# traps.yml
mode_bad: 644
mode_lucky: 0644
mode_good: "0644"
version_bad: 1.10
answer: yes
```

```bash
y2j traps.yml
```
```json
{
  "mode_bad": 644,
  "mode_lucky": 420,
  "mode_good": "0644",
  "version_bad": 1.1,
  "answer": true
}
```

- `"mode_bad": 644`: a plain **decimal** number. A file module turns a number straight into permission bits, and decimal 644 is octal `1204` (sticky bit, owner write only). The file gets nonsense permissions.
- `"mode_lucky": 420`: the leading `0` made YAML read it as **octal** 644 = decimal 420, which happens to be the right bits. It works only because of an old YAML 1.1 rule; linters and other YAML tools may disagree.
- `"mode_good": "0644"`: a string. Ansible always reads it as octal. **Quote modes.**
- `"version_bad": 1.1`: your "1.10" became 1.1.
- `"answer": true`: `yes` became a boolean.

**Example 2: an indentation mistake.**

```yaml
# bad-indent.yml
- name: demo
  hosts: all
  tasks:
    - name: say hi
      ansible.builtin.debug:
      msg: hello            # ← should be indented under debug
```

```bash
y2j bad-indent.yml
```
```json
[
  {
    "name": "demo",
    "hosts": "all",
    "tasks": [
      {
        "name": "say hi",
        "ansible.builtin.debug": null,
        "msg": "hello"
      }
    ]
  }
]
```

- The YAML is *valid*, but `msg` is now a key of the **task**, not an argument of `debug` (which got `null`). Ansible stops with `'msg' is not a valid attribute for a Task`.
- Lesson: valid YAML ≠ correct playbook. Read the JSON.

**Example 3: unquoted `{{`.**

```yaml
# braces.yml
greeting: {{ name }}
```
```bash
y2j braces.yml
```
```
yaml.constructor.ConstructorError: ... found unhashable key
```

- YAML read `{{ name }}` as "a map inside a map" and choked. Fix: `greeting: "{{ name }}"`.

---

## Hands-on lab

1. Install PyYAML and add the `y2j` helper above to `~/.bashrc`, then `source ~/.bashrc`.

2. Save this as `lab04.yml` and **predict** the JSON before running `y2j lab04.yml`:

   ```yaml
   ---
   - name: Lab play
     hosts: web
     vars:
       country: NO
       port: "8080"
       workers: 4
       banner: |
         Line one
         Line two
       cmd: >
         echo one
         echo two
     tasks:
       - name: first
         ansible.builtin.debug:
           msg: "Port is {{ port }}"
   ```

   Check: `country` → `false`; `port` → string `"8080"`; `workers` → number `4`; `banner` → `"Line one\nLine two\n"`; `cmd` → `"echo one echo two\n"` (folded!).

3. Break it on purpose, one at a time, and read each error:
   - Replace the 2 spaces before `hosts:` with a **tab** → `found character '\t' that cannot start any token`.
   - Remove the quotes around `"Port is {{ port }}"` → still works! (It doesn't *start* with `{`.) Now change it to `msg: {{ port }}` → error.
   - Indent `hosts:` by 3 spaces instead of 2 → `mapping values are not allowed here` or similar.

4. Rewrite `country: NO` so it stays the string `NO`. Confirm with `y2j`.

5. Bonus: install `yamllint` (`sudo apt install -y yamllint`) and run `yamllint lab04.yml`. You'll see warnings like `truthy value should be one of [false, true]` if you used `yes`. Topic 50 sets it up properly.

---

## Where you'll use it

- **Every Ansible file**: playbooks, inventories (YAML form), `group_vars/`, roles, `requirements.yml`, Molecule configs.
- **Code review**: spotting an indentation slip that moves an argument out of a module.
- **Debugging**: "why is my mode wrong?" → it wasn't quoted.
- The same YAML skills carry over to **Kubernetes manifests**, **GitHub Actions workflows**, **Docker Compose** and **Helm values**.

---

## When it breaks

| Error message (shortened) | Cause | Fix |
|---|---|---|
| `Syntax Error while loading YAML. found character '\t'` | A **tab** in indentation | Replace with spaces; set editor to "insert spaces" |
| `mapping values are not allowed in this context` | Wrong indentation, or an unquoted `: ` inside a value | Align keys; quote values with colons |
| `found unhashable key` / `We could be wrong, but this one looks like it might be an issue with missing quotes` | Value starts with `{{` unquoted | `"{{ var }}"` |
| `did not find expected key` / `expected <block end>` | A line indented differently from its siblings, often a list item | Line up items exactly |
| `'msg' is not a valid attribute for a Task` (any argument name) | A module argument is indented at task level instead of under the module | Indent it under the module |
| `conflicting action statements: X, Y` | Two module names in one task (often two tasks merged by a missing `- `) | One module per task; check the dashes |
| A file ends up with mode `1204` or other strange permissions | `mode: 644` unquoted (read as decimal) | `mode: "0644"` |
| `NO` / `off` / `yes` turned into booleans | YAML 1.1 boolean rules | Quote them, use `true`/`false` |
| Version `1.10` becomes `1.1` | Float conversion | Quote it |
| Copy-pasted from a web page, weird errors | Non-breaking spaces or "smart quotes" | Retype the line; `cat -A file` shows hidden characters ([Linux 009](../../linux-mastery-rhel/notes/009-viewing-files.md)) |

---

## Quick recap

- YAML has three shapes: **map** (`key: value`), **list** (`- item`), **scalar**. A playbook is a **list of plays**; a play is a **map**; `tasks` is a **list of maps**.
- **Spaces, never tabs**; 2 spaces per level; siblings line up exactly.
- **Quote** values starting with `{{`, file modes (`"0644"`), versions (`"1.10"`) and anything that looks like a boolean (`"NO"`).
- Use `true`/`false`. `|` keeps newlines, `>` folds lines.
- Valid YAML is not a correct playbook: convert to JSON to see the real structure.

### Interview questions

1. **Q: Why must `"{{ var }}"` be quoted when it starts a value?**
   A: A `{` at the start of a YAML value begins a flow-style map, so the parser fails before Ansible ever sees the Jinja2 expression. Quoting makes it a string that Ansible then templates.
2. **Q: What's the difference between `|` and `>` in YAML?**
   A: `|` keeps line breaks exactly (good for file content); `>` folds lines into one line separated by spaces (good for long text or commands).
3. **Q: Why quote `mode: "0644"`?**
   A: Unquoted, it's a YAML number. `644` is decimal (= octal 1204, wrong permissions), and `0644` is right only because of a YAML 1.1 octal rule. As a string, Ansible always reads it as an octal mode.

### Scenario question (answer out loud first, then compare)

> *"A teammate's playbook runs without errors, but the `copy` task always deploys the file with the default owner, ignoring `owner: nginx`. The YAML passes `yamllint`. What do you check?"*

**Model answer:**
1. Valid YAML can still have the wrong **structure**. I'd suspect `owner:` is indented at the task level instead of under the module.
2. Convert the file to JSON (or read carefully) to see whether `owner` sits inside the `ansible.builtin.copy` map.
3. If it's at the task level, ansible-core normally errors on unknown keywords. If it doesn't, check whether it accidentally landed inside another map, like `vars:`, where it's just an unused variable.
4. Fix the indentation, then run with `--check --diff` (topic 20) to confirm the owner change shows up.
5. Prevent it: `ansible-lint` in CI (topic 50) catches most structure mistakes.

**Likely follow-ups:**
- *How would you see what Ansible actually received for a task?* Run with `-vvv`; the module invocation and its arguments are shown.
- *Why does ansible-lint complain about `yes`?* Truthy values are ambiguous across YAML versions; `true`/`false` is unambiguous.
