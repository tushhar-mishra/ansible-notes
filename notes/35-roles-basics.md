# 35 · Roles I: Structure and Use

> **Level:** ★ foundation
>
> **You will learn:** what a **role** is (a standard folder that packages tasks, handlers, templates, files, defaults and variables for one job), the directory layout and what each folder is for, creating one with **`ansible-galaxy role init`**, **`defaults/` vs `vars/`**, where Ansible **finds** roles (`roles_path`, `./roles`, collections), the three ways to use a role (**`roles:`**, **`import_role`**, **`include_role`**) and how they differ, passing **parameters**, play ordering (`pre_tasks` → `roles` → `tasks` → `post_tasks`), role de-duplication, and how role files and templates are found. You'll turn your nginx playbook into a role.
>
> **You need:** [34 · import vs include](34-import-vs-include.md), [25 · Variable precedence](25-variable-precedence.md) (defaults = level 2, role vars = level 15), [30 · Handlers](30-handlers.md), [29 · Templates](29-templates.md).
>
> **Part 4 · Organizing and reusing code** · Next: [36 · Roles II: writing reusable roles](36-roles-advanced.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **role** is a reusable unit for **one job** ("nginx web server", "baseline hardening", "PostgreSQL"), in a **standard folder layout**. Ansible knows the layout, so you just say "apply role `nginx_web` to these hosts" and it finds the tasks, handlers, templates and defaults by itself.

**Picture it.** A flat-pack furniture box: instructions (tasks), screws (files), a template for cutting (templates), "standard size unless you choose otherwise" (defaults), and a customer service number (handlers) for when something changes. Every box from that maker is organised the same way.

**The one thing to know.**

```
roles/nginx_web/
├── defaults/main.yml     ← tunable values (lowest precedence)
├── vars/main.yml         ← internal constants (high precedence)
├── tasks/main.yml        ← what the role does (entry point)
├── handlers/main.yml     ← restarts/reloads
├── templates/            ← .j2 files for the template module
├── files/                ← static files for copy
└── meta/main.yml         ← metadata, dependencies
```

```yaml
- name: Web servers
  hosts: web
  become: true
  roles:
    - role: nginx_web
      nginx_web_port: 80          # a role parameter (overrides the default)
```

**Try it now:** `ansible-galaxy role init roles/nginx_web` and look at what it created.

**You've got it if you can:** say what goes in `defaults/` vs `vars/`, and the three ways to apply a role.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### The layout

| Path | Used by | Contains |
|---|---|---|
| `tasks/main.yml` | **Entry point** | The role's task list (can `import_tasks` other files in `tasks/`) |
| `handlers/main.yml` | Handlers | The role's handlers |
| `defaults/main.yml` | Variables (precedence **2**, lowest) | Every value a user may want to change, with sensible defaults |
| `vars/main.yml` | Variables (precedence **15**) | Constants the role needs but users shouldn't normally change (OS package names, internal paths) |
| `templates/` | `template` module | `.j2` files, found automatically: `src: nginx.conf.j2` |
| `files/` | `copy`, `script`, `unarchive` | Static files, found automatically: `src: logo.png` |
| `meta/main.yml` | Metadata | `galaxy_info` (author, license, platforms), **`dependencies`** (other roles to run first) |
| `meta/argument_specs.yml` | Input validation | Declared parameters and types (topic 36) |
| `library/`, `module_utils/`, `filter_plugins/`... | Plugins | Custom modules/filters shipped with the role (topic 38) |
| `tests/` | Testing | Old-style test playbook; Molecule uses `molecule/` instead (topic 51) |
| `README.md` | Humans | What it does, variables, example |

Only the folders you use need to exist. A role with just `tasks/main.yml` is valid.

Each `main.yml` can be a directory too (`defaults/main/` with several files), and you can load alternative files with `tasks_from`, `vars_from`, `defaults_from`, `handlers_from`.

### defaults vs vars

| | `defaults/main.yml` | `vars/main.yml` |
|---|---|---|
| Precedence | **2** (almost everything overrides it) | **15** (beats inventory and play vars) |
| Purpose | **Inputs**: "you may change this" | **Internals**: "this role needs this" |
| Examples | `nginx_web_port: 80`, `nginx_web_workers: auto`, `nginx_web_sites: []` | `_nginx_packages: {RedHat: [nginx], Debian: [nginx]}` |
| Overridable from `group_vars`? | ✅ Yes | ❌ No (only role params, `set_fact`, `-e`) |

**Rule:** if in doubt, put it in `defaults/`. Users of your role (including future you) should be able to change any behaviour from the inventory.

**Name every role variable with the role's prefix** (`nginx_web_port`, not `port`): all role variables share one namespace with everything else in the play.

### Where Ansible looks for roles

When you write `role: nginx_web`, Ansible searches, in order:

1. `roles/` next to the **playbook**,
2. the directories in **`roles_path`** (ansible.cfg; default `~/.ansible/roles:/usr/share/ansible/roles:/etc/ansible/roles`),
3. the playbook's own directory.

For roles inside a **collection**, use the FQCN: `role: redhat.rhel_system_roles.timesync` (topic 37).

You can also give a path: `role: ../shared-roles/nginx_web`.

### Three ways to use a role

| | `roles:` (play keyword) | `ansible.builtin.import_role` | `ansible.builtin.include_role` |
|---|---|---|---|
| Static/dynamic | Static | Static | **Dynamic** |
| Where | Play level | In `tasks:` | In `tasks:` |
| When it runs | **Before** the play's `tasks:` (after `pre_tasks`) | Where you put it | Where you put it, per host |
| `loop` | ❌ | ❌ | ✅ |
| `when` | Applied to every task of the role | Applied to every task | Evaluated once |
| Tags/keywords | Inherited by all role tasks | Inherited | Only on the include (use `apply:`) |
| Role's handlers & vars | Available to the play | Available to the play | Available after it runs; `public: true` to expose vars like the others |

```yaml
# 1. roles: keyword
- hosts: web
  roles:
    - common
    - role: nginx_web
      nginx_web_port: 8080
      tags: nginx

# 2. import_role (static, inside tasks)
  tasks:
    - name: Apply nginx
      ansible.builtin.import_role:
        name: nginx_web
      vars:
        nginx_web_port: 8080

# 3. include_role (dynamic: conditions evaluated once, loops possible)
  tasks:
    - name: Apply nginx only where wanted
      ansible.builtin.include_role:
        name: nginx_web
        tasks_from: configure          # run tasks/configure.yml instead of main.yml
      when: nginx_wanted | bool
```

### Play order with roles

```yaml
- hosts: web
  pre_tasks:     # 1. first (e.g. take the host out of the load balancer)
    ...
                 #    → handlers notified so far run
  roles:         # 2. then roles, in order
    ...
  tasks:         # 3. then tasks
    ...
                 #    → handlers notified by roles/tasks run
  post_tasks:    # 4. last (e.g. put the host back, smoke test)
    ...
                 #    → handlers notified here run
```

### Role de-duplication

In one play, a role listed twice with the **same** parameters runs **once**. With **different** parameters, it runs again. Set `allow_duplicates: true` in the role's `meta/main.yml` to always run it. (`include_role` in a loop always runs it each time.)

### Role variable visibility

By default, a role's `defaults` and `vars` are **visible to the rest of the play** after the role is loaded (they're "public"). That's convenient but can surprise you: a later task may pick up `some_var` from a role you forgot about. Controls:

- `public: false` / `public: true` on `import_role`/`include_role`,
- `private_role_vars = True` in `ansible.cfg` (`DEFAULT_PRIVATE_ROLE_VARS`) to keep role vars private,
- and, again, **prefix** every role variable.

---

## Why it exists

Playbooks grow into hundreds of tasks mixing concerns: users, nginx, PostgreSQL, monitoring. Roles give you:

- **Reuse**: one `nginx_web` role used by five projects and every environment.
- **Standard layout**: anyone can open any role and know where things are.
- **Encapsulation**: the role's templates, files, handlers and defaults travel together.
- **Sharing**: publish on Galaxy or in a collection (topic 37); thousands of community roles exist.
- **Testing**: Molecule tests a role in isolation (topic 51).

Playbooks then become short: "these hosts get these roles with these parameters".

---

## Analogy

**Standard shipping containers.** Before containers, every ship was loaded differently. With standard containers, any port knows how to lift, stack and unload them. A role is a standard container for automation: same corners, same doors (folders), whatever is inside. The playbook is the ship's manifest: "container `common`, container `nginx_web` with these settings".

---

## How it works inside

```
 play: roles: [ {role: nginx_web, nginx_web_port: 8080} ]
   │
 PARSE TIME (static)
   ├─ find roles/nginx_web (search path)
   ├─ load meta/main.yml → dependencies? load those roles first (topic 36)
   ├─ load defaults/main.yml   (precedence 2)
   ├─ load vars/main.yml       (precedence 15)
   ├─ load tasks/main.yml (+ its imports) → insert into the play before tasks:
   │     each task remembers it belongs to role nginx_web
   ├─ load handlers/main.yml → add to the play's handlers
   └─ role params (nginx_web_port: 8080) attached (precedence 20)
 RUN TIME
   └─ for each role task: search path for src files is
        roles/nginx_web/templates (or files) → then the playbook dir
```

Task names in output are prefixed with the role: `TASK [nginx_web : nginx is installed]`.

---

## Code / commands

### Create the skeleton

```bash
cd ~/ansible-lab
ansible-galaxy role init roles/nginx_web
tree roles/nginx_web
```
```
- Role roles/nginx_web was created successfully
roles/nginx_web
├── README.md
├── defaults
│   └── main.yml
├── files
├── handlers
│   └── main.yml
├── meta
│   └── main.yml
├── tasks
│   └── main.yml
├── templates
├── tests
│   ├── inventory
│   └── test.yml
└── vars
    └── main.yml
```

### Fill it in (from topics 14, 24, 29, 30)

`roles/nginx_web/defaults/main.yml`:

```yaml
---
nginx_web_port: 80
nginx_web_page_text: "Hello from {{ inventory_hostname }}"
nginx_web_extra_packages: []
```

`roles/nginx_web/vars/main.yml`:

```yaml
---
_nginx_web_root:
  RedHat: /usr/share/nginx/html
  Debian: /var/www/html
_nginx_web_user:
  RedHat: nginx
  Debian: www-data
```

- Leading underscore: a common convention for "internal, don't set this". Topic 36 tightens it to a **double** underscore plus the role name (`__nginx_web_root`), which is what ansible-lint's `var-naming` rule and most style guides expect; use that form in real roles.

`roles/nginx_web/tasks/main.yml`:

```yaml
---
- name: apt cache is fresh (Debian family)
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
  when: ansible_facts['os_family'] == 'Debian'

- name: nginx is installed
  ansible.builtin.package:
    name: "{{ ['nginx'] + nginx_web_extra_packages }}"
    state: present

- name: Site config is in place
  ansible.builtin.template:
    src: site.conf.j2
    dest: /etc/nginx/conf.d/site.conf
    mode: "0644"
  notify: Reload nginx

- name: Home page is in place
  ansible.builtin.copy:
    dest: "{{ _nginx_web_root[ansible_facts['os_family']] }}/index.html"
    content: "{{ nginx_web_page_text }}\n"
    mode: "0644"

- name: nginx is running and enabled
  ansible.builtin.systemd_service:
    name: nginx
    state: started
    enabled: true
```

`roles/nginx_web/templates/site.conf.j2`:

```jinja
{{ ansible_managed | comment }}
server {
    listen {{ nginx_web_port }};
    server_name {{ inventory_hostname }};
    root {{ _nginx_web_root[ansible_facts['os_family']] }};
    location /health { return 200 "ok\n"; }
}
```

`roles/nginx_web/handlers/main.yml`:

```yaml
---
- name: Reload nginx
  ansible.builtin.systemd_service:
    name: nginx
    state: reloaded
```

`roles/nginx_web/meta/main.yml` (trimmed):

```yaml
---
galaxy_info:
  author: you
  description: nginx web server with a simple site
  license: MIT
  min_ansible_version: "2.16"
  platforms:
    - name: EL
      versions: ["9", "10"]
    - name: Ubuntu
      versions: [noble]
dependencies: []
```

⚠️ Both distros ship a **default site** on port 80 (Ubuntu: `/etc/nginx/sites-enabled/default`; AlmaLinux: a `server` block inside `nginx.conf`). The lab play below sets `nginx_web_port: 8081` so our site doesn't compete with it, and the load balancer from topic 29 keeps using port 80. A production role would manage or remove those defaults.

### Use the role

`~/ansible-lab/web-role.yml`:

```yaml
---
- name: Web servers via the nginx_web role
  hosts: web
  become: true
  roles:
    - role: nginx_web
      nginx_web_port: 8081
```

```bash
ansible-playbook web-role.yml
```
```
TASK [nginx_web : apt cache is fresh (Debian family)] **********************************
skipping: [web1]
ok: [web2]

TASK [nginx_web : nginx is installed] **************************************************
ok: [web1]
ok: [web2]

TASK [nginx_web : Site config is in place] *********************************************
changed: [web1]
changed: [web2]

TASK [nginx_web : Home page is in place] ***********************************************
changed: [web1]
changed: [web2]

TASK [nginx_web : nginx is running and enabled] ****************************************
ok: [web1]
ok: [web2]

RUNNING HANDLER [nginx_web : Reload nginx] *********************************************
changed: [web1]
changed: [web2]
```

```bash
curl -s 172.30.0.11:8081/health      # ok
curl -s 172.30.0.12:8081/            # page text
```

- `nginx_web :` prefix on each task and handler: they belong to the role.
- The home page now shows the **role default** (`Hello from web1`), unless your `group_vars`/`host_vars` set `nginx_web_page_text`. Try setting it in `host_vars/web1.yml` and re-run: inventory beats role defaults.

### Use parts of a role

```yaml
- name: Only reconfigure, don't install
  ansible.builtin.include_role:
    name: nginx_web
    tasks_from: configure          # needs roles/nginx_web/tasks/configure.yml
```

(To support that, split `tasks/main.yml` into `install.yml`, `configure.yml`, `service.yml` and have `main.yml` import all three. Topic 36 does this.)

---

## Hands-on lab

1. `ansible-galaxy role init roles/nginx_web` and fill in the files above. Run `web-role.yml` twice: second run `changed=0`.
2. Override the page text three ways and check each with curl: role parameter in the play, `host_vars/web1.yml`, `-e`. Rank them using topic 25 (params 20 > host_vars 9 > defaults 2; `-e` 22 beats all).
3. Try to override `_nginx_web_root` from `group_vars/web.yml`: it has no effect (role vars, 15). That's intended.
4. Apply the role twice in the same play with the same parameters (it runs once), then with a different `nginx_web_port` (it runs twice).
5. Use `import_role` and `include_role` versions of the play. With `include_role` + `when: false`, confirm the role's tasks don't even appear.
6. Create a second tiny role `motd` (tasks/main.yml with the template from topic 29, the template in `roles/motd/templates/`) and a play with `pre_tasks` (a debug), `roles: [motd, nginx_web]`, `tasks` and `post_tasks` (debugs). Check the execution order in the output.

**Done when:** your web configuration lives entirely in a role, and the playbook is a few lines long.

---

## Where you'll use it

- **Everywhere**: real Ansible codebases are mostly roles; playbooks just map groups to roles.
- **Team standards**: a `baseline` role every server gets; `nginx`, `postgres`, `monitoring_agent` roles per service.
- **Vendor/community roles**: from Galaxy or collections (e.g. RHEL System Roles: `timesync`, `selinux`, `firewall`, `storage`), configured through their defaults.
- **AWX/AAP**: job templates run playbooks that apply roles.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `the role 'nginx_web' was not found in ...` | Not in `./roles`, `roles_path` or the playbook dir | Fix the path; set `roles_path` in `ansible.cfg`; FQCN for collection roles |
| `Could not find or access 'site.conf.j2'` | Template not in the role's `templates/` | Move it there |
| Your `group_vars` value is ignored | The role sets it in `vars/` | Move it to `defaults/` in the role; or pass it as a role param |
| A variable from one role leaks into another | Role vars are public by default; no prefixes | Prefix names; `public: false`; `private_role_vars` |
| Role ran only once though listed twice | De-duplication (same params) | Different params, `allow_duplicates: true`, or `include_role` |
| Handler `Reload nginx` not found from a play task | Handler name differs or role not loaded yet (dynamic include) | Use the exact name; with `include_role`, the role's handlers exist after it runs |
| `roles:` runs before my `tasks:` setup | Play order: roles run before tasks | Put setup in `pre_tasks`, or use `import_role` in `tasks` |
| `when` on `roles:` skipped some role tasks, not all | Static: `when` copied to each task | `include_role` (evaluated once) |

---

## Quick recap

- A **role** = standard folders: `tasks/`, `handlers/`, `defaults/`, `vars/`, `templates/`, `files/`, `meta/` (+ `argument_specs`). Create with `ansible-galaxy role init`.
- **`defaults/`** = tunable inputs (precedence 2); **`vars/`** = internal constants (15). Prefix all role variables.
- Found in `./roles`, `roles_path`, or by FQCN in collections.
- Apply with **`roles:`** (static, before `tasks:`), **`import_role`** (static, in place) or **`include_role`** (dynamic: loops, conditions once, `tasks_from`).
- Order: `pre_tasks` → `roles` → `tasks` → `post_tasks`, handlers after each. Same role + same params = runs once.

### Interview questions

1. **Q: What's the difference between `defaults/main.yml` and `vars/main.yml` in a role?**
   A: Defaults have the lowest precedence and are meant to be overridden by users; role vars have high precedence (above inventory and play vars) and hold internal constants.
2. **Q: `roles:` vs `import_role` vs `include_role`?**
   A: `roles:` statically applies roles before the play's tasks; `import_role` statically inserts a role at a point in `tasks`; `include_role` loads it dynamically at run time, allowing loops and conditions evaluated once.
3. **Q: Why prefix role variables?**
   A: All variables share one namespace in a play; prefixes prevent different roles (or inventory) from colliding on names like `port`.

### Scenario question (answer out loud first, then compare)

> *"Your team has five playbooks that each configure nginx slightly differently by copy-paste. Plan the refactor into a role without breaking production."*

**Model answer:**
1. Inventory the differences across the five: ports, sites, TLS, users, modules. Each difference becomes a **default** variable (`nginx_*`), with the most common value as the default.
2. Create the role: `tasks/` split into install/configure/service, `templates/` for nginx.conf and sites, `handlers/` with validate-then-reload, `vars/` for OS-specific internals, `meta/argument_specs.yml` to validate inputs (topic 36).
3. Replace one playbook at a time with `roles: [{ role: nginx, ... }]`, moving its specific values to its inventory `group_vars`.
4. For each switch, run `--check --diff` against staging and prod: the goal is **no diff** (the role reproduces the existing config exactly), then apply.
5. Add Molecule tests for RHEL and Ubuntu (topic 51) and lint in CI, so future changes are safe.

**Likely follow-ups:**
- *What if one playbook needs something very different?* A feature flag variable or a separate task file selected by variable (`include_tasks`), not a fork of the role.
- *Where would the role live?* In the repo's `roles/` at first; later in an internal collection with versioned releases (topic 37).
