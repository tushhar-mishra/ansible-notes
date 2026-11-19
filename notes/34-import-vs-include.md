# 34 · import vs include

> **Level:** ★★ intermediate
>
> **You will learn:** how to split playbooks into reusable files with **`import_tasks`** / **`include_tasks`** and **`import_playbook`**, the core difference between **static** (import: resolved when the playbook is parsed) and **dynamic** (include: resolved while running, per host), and how that changes the behaviour of `when`, `tags`, `loop`, `--list-tasks`, `--start-at-task`, handlers and variable file names. Plus `include_vars`, ⚠️ the removed bare `include:`, and a decision table.
>
> **You need:** [33 · Tags](33-tags.md), [15 · Inside ansible-playbook](15-inside-ansible-playbook.md) (parse vs execute), [28 · Loops](28-loops.md), [26 · when](26-register-and-when.md).
>
> **Part 4 · Organizing and reusing code** · Next: [35 · Roles I](35-roles-basics.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Two ways to pull tasks from another file:

- **`import_tasks`** = **copy-paste at parse time**. The tasks become part of the playbook before anything runs. Keywords you put on the import (`when`, `tags`, `become`) are copied onto **every** task inside.
- **`include_tasks`** = **a task that loads the file while running**, per host. It can **loop**, and its file name can use **facts**. Keywords apply to the include **itself**.

**Picture it.** Import = photocopying a chapter into your binder before the meeting. Include = a sticky note saying "at this point, fetch chapter X", where X is decided on the day.

**The one thing to know.**

```yaml
tasks:
  - name: Common setup (static)
    ansible.builtin.import_tasks: tasks/common.yml

  - name: OS-specific setup (dynamic: file name uses a fact)
    ansible.builtin.include_tasks: "tasks/{{ ansible_facts['os_family'] | lower }}.yml"

  - name: One vhost per site (dynamic: loops are only allowed on include)
    ansible.builtin.include_tasks: tasks/vhost.yml
    loop: "{{ sites }}"
    loop_control:
      loop_var: site
```

Rule of thumb: **import by default; include when you need a loop, a runtime file name, or to load tasks conditionally as one unit.**

**Try it now:** run the lab playbook with `--list-tasks`: imported tasks are listed, included ones aren't.

**You've got it if you can:** explain why `when:` on an import can skip *some* of the imported tasks but not others.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### The keywords

| Keyword | Static/dynamic | Pulls in | Where |
|---|---|---|---|
| `ansible.builtin.import_tasks` | **Static** | A task list file | In `tasks:` / handlers |
| `ansible.builtin.include_tasks` | **Dynamic** | A task list file | In `tasks:` / handlers |
| `ansible.builtin.import_playbook` | **Static** (only option) | A whole playbook | At the top level, between plays |
| `ansible.builtin.import_role` / `include_role` | Static / dynamic | A role (topic 35) | In `tasks:` |
| `roles:` (play keyword) | Static | Roles | Play level |
| `ansible.builtin.include_vars` | Dynamic | A variables file | In `tasks:` |
| ⚠️ `include:` (bare) | — | — | **Removed** in ansible-core 2.16. Replace with `import_tasks`/`include_tasks`/`import_playbook` |

A task file is just a YAML **list of tasks**, without `hosts:` or `tasks:`:

```yaml
# tasks/common.yml
- name: Base tools are installed
  ansible.builtin.package:
    name: [tree, curl]
    state: present

- name: motd is set
  ansible.builtin.template:
    src: motd.j2
    dest: /etc/motd
    mode: "0644"
```

### Static vs dynamic: the full comparison

| Behaviour | `import_*` (static) | `include_*` (dynamic) |
|---|---|---|
| When is the file read? | **Parse time**, before any host is contacted | **Run time**, when the include task is reached, **per host** |
| File name can use... | Only variables known at parse time (extra vars, play `vars`/`vars_files`) | **Any** variable, including facts and registered results |
| `loop` | ❌ Not allowed | ✅ Allowed |
| `when` on it | **Copied to every task inside**, evaluated **per task** | Evaluated **once**, for the include itself; if false, nothing inside runs |
| `tags` on it | Inherited by every task inside | Only on the include itself; pass inside with `apply: { tags: [...] }` |
| `become`, `vars`, etc. on it | Inherited by every task | `vars:` passed to tasks (as include params); other keywords via `apply:` |
| `--list-tasks` / `--list-tags` | Shows the inner tasks | Shows only the include |
| `--start-at-task` | Can start at an inner task | Can't start inside |
| Missing file | **Parse error**: nothing runs | Error only when (and on the hosts where) the include runs |
| Notify a handler defined inside | ✅ By the inner handler's name | ❌ Notify the **include's** name instead (it runs all handlers in the file) |
| Output | Inner tasks shown normally | An extra line: `included: /path/file.yml for web1, web2` |
| Cost | Free at run time | A small overhead per include, per host |

### The `when` difference, in one example

```yaml
- name: Install the app if it isn't installed (STATIC)
  ansible.builtin.import_tasks: tasks/install-app.yml
  when: not app_installed.stat.exists
```

`tasks/install-app.yml`:

```yaml
- name: Unpack app
  ansible.builtin.unarchive: { src: app.tgz, dest: /opt }          # creates /opt/app/bin/app
- name: Re-check
  ansible.builtin.stat: { path: /opt/app/bin/app }
  register: app_installed
- name: Initialise app
  ansible.builtin.command: /opt/app/bin/app --init
```

With **import**, `when` is copied to each task and evaluated each time. After "Re-check" sets `app_installed.stat.exists = true`, "Initialise app" sees `when: not true` → **skipped**! With **include**, the condition is checked once, before the file loads, and all three tasks run.

Rule: if tasks inside change the variables your condition uses, **use include**. (A `block` with `when` behaves like import here: the condition is checked for each task in the block.)

### `include_vars`

Load variables at run time, often by OS:

```yaml
- name: OS-specific variables
  ansible.builtin.include_vars: "{{ item }}"
  with_first_found:
    - "vars/{{ ansible_facts['distribution'] }}-{{ ansible_facts['distribution_major_version'] }}.yml"
    - "vars/{{ ansible_facts['os_family'] }}.yml"
    - vars/default.yml
```

- Tries each file in order and loads the **first one that exists** (`first_found` lookup).
- `include_vars` has high precedence (level 18, topic 25).
- Modern form: `ansible.builtin.include_vars: "{{ lookup('ansible.builtin.first_found', params) }}"` with `params: { files: [...], paths: [vars] }`. `with_first_found` is still common and fine.

### `import_playbook`: composing site.yml

```yaml
# site.yml
- ansible.builtin.import_playbook: base.yml
- ansible.builtin.import_playbook: web-vars.yml
- ansible.builtin.import_playbook: lb.yml
```

- Only static; there is no `include_playbook`.
- You can put `when`/`tags` on it (inherited by every play/task), and `vars` (play vars for its plays).
- Run all with `ansible-playbook site.yml`, or one part with `ansible-playbook lb.yml`.

---

## Why it exists

Long playbooks are hard to read and impossible to reuse. Splitting them lets you:

- **reuse** the same task file in several playbooks,
- **compose** a `site.yml` out of focused playbooks,
- pick **OS-specific** files at run time,
- **loop** over a group of tasks (one vhost, one database, one user's whole setup per item).

Static vs dynamic exists because both are needed: static is predictable (visible to `--list-tasks`, linting, `--start-at-task`); dynamic is flexible (loops, runtime names).

---

## Analogy

**Assembling a binder for a training course.**

- **Import** = the night before, you photocopy chapter 3 and put the pages into everyone's binder. Every page gets the same "Day 1" header stamped on it (inherited keywords). The table of contents lists them (`--list-tasks`).
- **Include** = a page that says "now open the chapter for **your** country" (file name from a fact), or "repeat this exercise for each of your three projects" (loop). The pages aren't in the binder until that moment, so the table of contents can't list them.

---

## How it works inside

```
 PARSE TIME (before any SSH)                        RUN TIME (per host)
 ───────────────────────────                        ───────────────────
 import_playbook b.yml → plays of b.yml inlined
 import_tasks x.yml    → tasks of x.yml inlined,
                         keywords of the import
                         merged into each task
                                                    include_tasks y.yml reached for web1:
                                                      evaluate when/loop for web1
                                                      render the file name with web1's vars
                                                      read y.yml, create its tasks
                                                      insert them into web1's queue now
                                                      ("included: y.yml for web1, web2")
```

Hosts that render the same file name share one inclusion; hosts with a different name (e.g. `redhat.yml` vs `debian.yml`) each get their own. With the linear strategy, hosts still progress task by task; included tasks are interleaved for the hosts that included them.

---

## Code / commands

### The lab layout

```
~/ansible-lab/
├── split.yml
├── tasks/
│   ├── common.yml
│   ├── redhat.yml
│   ├── debian.yml
│   └── vhost.yml
└── vars/
    ├── RedHat.yml
    └── Debian.yml
```

`tasks/common.yml`: as above (tree + curl, motd).

`tasks/redhat.yml`:

```yaml
- name: RedHat-family extras
  ansible.builtin.package: { name: "{{ os_extra_packages }}", state: present }
- name: RedHat says hello
  ansible.builtin.debug: { msg: "configured as RedHat family" }
```

`tasks/debian.yml`:

```yaml
- name: Debian-family extras
  ansible.builtin.package: { name: "{{ os_extra_packages }}", state: present }
- name: Debian says hello
  ansible.builtin.debug: { msg: "configured as Debian family" }
```

`vars/RedHat.yml`: `os_extra_packages: [bash-completion]` · `vars/Debian.yml`: `os_extra_packages: [bash-completion]`

`tasks/vhost.yml`:

```yaml
- name: Site directory for {{ site.name }}
  ansible.builtin.file:
    path: "/srv/sites/{{ site.name }}"
    state: directory
    mode: "0755"
- name: Index page for {{ site.name }}
  ansible.builtin.copy:
    dest: "/srv/sites/{{ site.name }}/index.html"
    content: "{{ site.title }}\n"
    mode: "0644"
```

`split.yml`:

```yaml
---
- name: Split into files
  hosts: web
  become: true
  vars:
    sites:
      - { name: shop, title: "Shop" }
      - { name: blog, title: "Blog" }

  tasks:
    - name: OS variables
      ansible.builtin.include_vars: "vars/{{ ansible_facts['os_family'] }}.yml"

    - name: Common (static)
      ansible.builtin.import_tasks: tasks/common.yml
      tags: common

    - name: OS-specific (dynamic)
      ansible.builtin.include_tasks: "tasks/{{ ansible_facts['os_family'] | lower }}.yml"
      args:
        apply:
          tags: os
      tags: os

    - name: One set of tasks per site (dynamic loop)
      ansible.builtin.include_tasks: tasks/vhost.yml
      loop: "{{ sites }}"
      loop_control:
        loop_var: site
        label: "{{ site.name }}"
```

- `apply: { tags: os }` + `tags: os` on the include: the include runs **and** passes the tag to its tasks, so `--tags os` works.
- `loop_var: site` (topic 28): the included tasks use `site` instead of `item`, so their own loops could still use `item`.
- Task names inside `vhost.yml` use `{{ site.name }}`; names are templated per host/item in the output.

### See the static/dynamic difference

```bash
ansible-playbook split.yml --list-tasks
```
```
  play #1 (web): Split into files	TAGS: []
    tasks:
      OS variables	TAGS: []
      Base tools are installed	TAGS: [common]
      motd is set	TAGS: [common]
      OS-specific (dynamic)	TAGS: [os]
      One set of tasks per site (dynamic loop)	TAGS: []
```

- Imported tasks appear by name (with inherited `common` tag).
- Includes appear only as themselves.

```bash
ansible-playbook split.yml
```
```
TASK [OS-specific (dynamic)] ***********************************************************
included: /home/you/ansible-lab/tasks/redhat.yml for web1
included: /home/you/ansible-lab/tasks/debian.yml for web2

TASK [RedHat-family extras] ************************************************************
changed: [web1]

TASK [Debian-family extras] ************************************************************
changed: [web2]
...
TASK [One set of tasks per site (dynamic loop)] ****************************************
included: /home/you/ansible-lab/tasks/vhost.yml for web1, web2 => (item=shop)
included: /home/you/ansible-lab/tasks/vhost.yml for web1, web2 => (item=blog)

TASK [Site directory for shop] *********************************************************
changed: [web1]
changed: [web2]
...
```

- `included: ... for web1`: web1 loaded `redhat.yml`, web2 loaded `debian.yml`.
- The loop included `vhost.yml` once per site.

### Missing file: static vs dynamic

Rename `tasks/common.yml` and run: the **whole run** fails at parse time (`Could not find or access ... common.yml`), no host contacted. Rename `tasks/debian.yml` instead: web1 runs fine, only web2 fails at the include.

### site.yml

```yaml
# ~/ansible-lab/site.yml
- ansible.builtin.import_playbook: base.yml
- ansible.builtin.import_playbook: web-vars.yml
- ansible.builtin.import_playbook: lb.yml
```

```bash
ansible-playbook site.yml --list-hosts
ansible-playbook site.yml
```

---

## Hands-on lab

1. Build the layout and run `split.yml`. Run `--list-tasks` and compare with the real run output.
2. Run `--tags common` and `--tags os`. Then remove `apply:` from the OS include and run `--tags os` again: the include runs but its tasks **don't** (tags not passed inside).
3. Reproduce the `when` difference: write the install-app example with `import_tasks`, see the third task skipped, switch to `include_tasks`, see it run.
4. Use `with_first_found` for `include_vars` with a `vars/AlmaLinux-10.yml` that overrides `os_extra_packages` only for web1.
5. Create `site.yml` from your playbooks and run it end to end, then a second time: `changed=0`.
6. Find old syntax: if you ever see `- include: something.yml`, rewrite it with the right modern keyword.

**Done when:** you can say for any reuse need whether import or include fits, and why.

---

## Where you'll use it

- **`site.yml`** composed of focused playbooks (`import_playbook`).
- **Roles internally**: `tasks/main.yml` importing `install.yml`, `configure.yml`, `service.yml` (topic 35).
- **OS-specific task files and vars** (`include_tasks`/`include_vars` + facts).
- **Looping over complex units**: vhosts, databases, tenants, users with many steps.
- **Conditional features**: `include_tasks: tls.yml` `when: tls_enabled`, loaded only where needed.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `'include' is not a valid attribute` / removed-feature error | Old bare `include:` | `import_tasks` / `include_tasks` / `import_playbook` |
| `You cannot use loops on 'import_tasks' statements` | Loop on a static import | `include_tasks` |
| `'ansible_facts' is undefined` in an import's file name | Facts don't exist at parse time | `include_tasks` |
| `--tags x` skips everything inside an include | Tags don't propagate into dynamic includes | `apply: { tags: [x] }` + tag the include |
| Some imported tasks unexpectedly skipped | `when` copied to each task, and inner tasks changed the condition | `include_tasks` |
| `--start-at-task` "task not found" | The task is inside an include | Use import, or start at the include |
| Handler in an included file "not found" | Notifying inner handler names of a dynamically included handler file | Notify the include's name, or import the handlers file |
| Included tasks use the wrong `item` | Outer loop and inner loop both use `item` | `loop_control: loop_var: <name>` |
| Whole run aborts on a missing optional file | Static import | `include_tasks` + `when`, or check with `first_found` |

---

## Quick recap

- **Import** (`import_tasks`, `import_playbook`, `import_role`, `roles:`) = **static**, parse time: inherited keywords on every task, visible to `--list-tasks`, no loops, no runtime file names.
- **Include** (`include_tasks`, `include_role`, `include_vars`) = **dynamic**, run time, per host: loops, fact-based file names, keywords on the include only (`apply:` to pass them in).
- `when` on an import is evaluated **per inner task**; on an include, **once**.
- ⚠️ Bare `include:` was removed (2.16).
- Default to import; use include for loops, runtime names and "load-as-a-unit" conditions.

### Interview questions

1. **Q: What's the difference between `import_tasks` and `include_tasks`?**
   A: Import is static: the file is read at parse time and inlined, keywords are inherited by every task, no loops. Include is dynamic: it's a task that loads the file at run time per host, supports loops and runtime variables in the file name, and its keywords apply only to the include unless passed with `apply`.
2. **Q: How do you run OS-specific tasks?**
   A: `include_tasks: "{{ ansible_facts['os_family'] | lower }}.yml"` (dynamic, since facts exist only at run time), or `include_vars` with `first_found` for OS-specific variables.
3. **Q: How do you build a site playbook from smaller playbooks?**
   A: `import_playbook` statements in `site.yml`.

### Scenario question (answer out loud first, then compare)

> *"A role uses `import_tasks: upgrade.yml` with `when: needs_upgrade`. Inside, the first task upgrades the package and the last task re-reads the version into `needs_upgrade`'s source variable. In production, the final 'restart and verify' task never runs. Why, and what's the fix?"*

**Model answer:**
1. `import_tasks` copies `when: needs_upgrade` onto **every** imported task and evaluates it **per task**. Once the inner task updates the version, `needs_upgrade` becomes false, so the remaining tasks (restart and verify) are skipped.
2. Fix options:
   - switch to `include_tasks` so the condition is evaluated once, before the file loads;
   - or compute the decision into a separate, stable variable before the import (`set_fact: do_upgrade: "{{ needs_upgrade }}"`) and use `when: do_upgrade`;
   - restart via a handler notified by the upgrade task, so it doesn't depend on the condition at all.
3. Add a test (Molecule scenario that starts from an old version) so this regression is caught.

**Likely follow-ups:**
- *Why not always use include?* You lose `--list-tasks` visibility, `--start-at-task`, linting of inner tasks via imports, and tags/keywords inheritance; there's also a small runtime cost.
- *Does `block` + `when` behave like import or include?* Like import: the condition is applied to each task in the block.
