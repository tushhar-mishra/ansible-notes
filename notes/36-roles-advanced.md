# 36 · Roles II: Writing Reusable Roles

> **Level:** ★★ intermediate
>
> **You will learn:** how to design roles other people (and future you) can reuse without editing them: one job per role, **data-driven defaults**, naming conventions, splitting tasks into install/configure/service with tags, **multi-OS support** with per-OS variable files and `first_found`, **input validation with `meta/argument_specs.yml`** (types, required options, choices, automatic validation, `ansible-doc -t role`), **role dependencies** (and why explicit is better), check-mode friendliness, and a README template. You'll upgrade `nginx_web` into a production-quality role.
>
> **You need:** [35 · Roles I](35-roles-basics.md), [34 · import vs include](34-import-vs-include.md), [28 · Loops](28-loops.md), [32 · combine and lookups](32-advanced-jinja2-and-lookups.md).
>
> **Part 4 · Organizing and reusing code** · Next: [37 · Collections and Galaxy](37-collections-and-galaxy.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A reusable role is configured **only through its defaults**, works on every OS it claims to support, **validates its inputs** before changing anything, and is idempotent and check-mode friendly. Users should never need to edit it.

**Picture it.** A good kitchen appliance: a few clearly labelled knobs (defaults), works on 230 V and 110 V (multi-OS), refuses to start with the lid open (argument validation), and comes with a manual (README).

**The one thing to know.** `roles/nginx_web/meta/argument_specs.yml`:

```yaml
argument_specs:
  main:
    short_description: nginx web server with one or more sites
    options:
      nginx_web_port:
        type: int
        default: 80
        description: Port each site listens on.
      nginx_web_sites:
        type: list
        elements: dict
        required: true
        description: Sites to serve.
        options:
          name: { type: str, required: true }
          title: { type: str, default: "Welcome" }
```

Ansible now checks those types **before** the role's first task, and `ansible-doc -t role -r roles nginx_web` documents them.

**Try it now:** add the spec above and run the role with `nginx_web_port: eighty`. Read the validation error.

**You've got it if you can:** list five rules for a reusable role and explain what `argument_specs` adds.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Design rules

| # | Rule | Why |
|---|---|---|
| 1 | **One job per role** (`nginx_web`, not `webserver_and_db_and_monitoring`) | Reuse, testing, clear ownership |
| 2 | **Every tunable in `defaults/`**, prefixed with the role name | Users configure from inventory; no name clashes |
| 3 | **Data-driven inputs**: lists/dicts (`nginx_web_sites: [...]`) instead of many scalar variables | One role handles 1 or 50 sites |
| 4 | **Internals in `vars/`** with a double-underscore prefix (`__nginx_web_packages`) | Clearly "not for you to set" |
| 5 | **Multi-OS via data, not `when` everywhere**: per-OS vars files | Adding a distro = adding a file |
| 6 | **Validate inputs** (`meta/argument_specs.yml`, plus `assert` for cross-field rules) | Fail fast, before any change |
| 7 | **Idempotent and check-mode safe** (topics 20, 27) | Safe re-runs, useful plans |
| 8 | **Restarts via handlers**, with config validation | No needless outages |
| 9 | **Split tasks** by phase (`install`, `configure`, `service`) with tags | `tasks_from`, `--tags nginx_web_config` |
| 10 | **No environment-specific values** in the role (no prod hostnames, no secrets) | Same role for every environment |
| 11 | **Document** defaults and examples in `README.md` (and `argument_specs`) | Self-service |
| 12 | **Test** on every supported OS (Molecule, topic 51) | Confidence |

### Variable naming conventions

| Prefix | Meaning | Example |
|---|---|---|
| `<role>_` | Public input (defaults) | `nginx_web_port` |
| `__<role>_` | Internal (role vars, set_facts) | `__nginx_web_root` |
| `<role>_<thing>` registered | Results inside the role | `__nginx_web_conf_result` |

(Red Hat's RHEL System Roles use exactly this style.)

### Multi-OS support

```
roles/nginx_web/vars/
├── main.yml                # shared internals
├── RedHat.yml              # family
├── Debian.yml              # family
└── Ubuntu-24.yml           # a specific distro/version, if needed
```

`tasks/main.yml` starts with:

```yaml
- name: Load OS-specific variables
  ansible.builtin.include_vars: "{{ lookup('ansible.builtin.first_found', __nginx_web_vars_files) }}"
  vars:
    __nginx_web_vars_files:
      files:
        - "{{ ansible_facts['distribution'] }}-{{ ansible_facts['distribution_major_version'] }}.yml"
        - "{{ ansible_facts['distribution'] }}.yml"
        - "{{ ansible_facts['os_family'] }}.yml"
      paths:
        - "{{ role_path }}/vars"
```

- `first_found` returns the most specific file that exists.
- `role_path`: magic variable = this role's directory.
- `include_vars` has high precedence (18), so OS files can override `vars/main.yml`. Users still can't override them from inventory. If a value must be tunable **with** an OS-specific fallback, leave the public variable undefined by default and combine them where you use it: `owner: "{{ nginx_web_user | default(__nginx_web_user) }}"`. Users set `nginx_web_user`; otherwise the OS file decides. (Don't write `nginx_web_user: "{{ __nginx_web_user }}"` in `defaults/`: argument validation, below, renders it before the OS file is loaded and fails.)

### meta/argument_specs.yml

Declares the role's inputs per **entry point** (`main` = `tasks/main.yml`; others = `tasks/<name>.yml` used with `tasks_from`):

| Field | Meaning |
|---|---|
| `short_description`, `description` | Shown by `ansible-doc -t role` |
| `options.<name>.type` | `str`, `int`, `bool`, `float`, `list`, `dict`, `path`, `raw` |
| `required: true` | Must be provided (no default) |
| `default` | Documentation of the default (keep `defaults/main.yml` as the real source and in sync) |
| `choices: [a, b]` | Allowed values |
| `elements: dict` / `elements: str` | Type of list items |
| `options:` (nested) | Keys of dicts inside lists/dicts |

What happens: when the role starts, Ansible inserts a task **"Validating arguments against arg spec 'main'"**. Wrong type, missing required option, value not in `choices` → the role fails **before** its first real task. Types are also **converted** (`"8080"` from `-e` becomes `8080` when declared `int`).

Read the spec like module docs:

```bash
ansible-doc -t role -r roles nginx_web
```

### Cross-field validation with assert

`argument_specs` checks each option alone. For rules involving several options, add an assert at the start of `tasks/main.yml`:

```yaml
- name: Validate site names are unique
  ansible.builtin.assert:
    that:
      - nginx_web_sites | map(attribute='name') | unique | length == nginx_web_sites | length
    fail_msg: "nginx_web_sites contains duplicate names"
    quiet: true
```

### Role dependencies (`meta/main.yml`)

```yaml
dependencies:
  - role: motd
  - role: firewall_port
    firewall_port_number: 80
```

- Dependencies run **before** the role, automatically, wherever the role is used.
- De-duplicated like other roles (same params → once).
- **Prefer explicit composition** in playbooks (`roles: [motd, firewall_port, nginx_web]`) over hidden dependencies: readers see everything that runs, and they can reorder or drop pieces. Use dependencies only for true "can't work without it" cases.

---

## Why it exists

A role that works only for its author's project isn't reusable; it's a folder. Without these practices you get: forked copies per team, `vars/` values nobody can override, `when: ansible_distribution == ...` sprinkled everywhere, and cryptic failures deep inside the role when someone passes a string instead of a list. With them, a role behaves like a well-documented module: clear inputs, validated, predictable.

---

## Analogy

**A rental car.** Any driver can use it without being a mechanic: the controls are standard and labelled (defaults with clear names), it adapts to the country (multi-OS), it won't start without a key and a closed door (argument validation), the manual is in the glovebox (README, `ansible-doc`), and nobody has to open the engine (edit the role) to drive it.

---

## How it works inside

```
 include/import/roles: nginx_web
   │
   ├─ meta/main.yml dependencies → run those roles first
   ├─ meta/argument_specs.yml present?
   │     → implicit first task "Validating arguments against arg spec 'main'"
   │        (uses the same validator as modules: types, required, choices; converts types)
   │        ✗ → fail before any change
   ├─ tasks/main.yml
   │     ├─ include_vars first_found → OS internals
   │     ├─ assert cross-field rules
   │     ├─ import_tasks install.yml   (tags: nginx_web, nginx_web_install)
   │     ├─ import_tasks configure.yml (notify handlers)
   │     └─ import_tasks service.yml
   └─ handlers: validate config → reload
```

---

## Code / commands

### The upgraded role

```
roles/nginx_web/
├── README.md
├── defaults/main.yml
├── handlers/main.yml
├── meta/
│   ├── main.yml
│   └── argument_specs.yml
├── tasks/
│   ├── main.yml
│   ├── install.yml
│   ├── configure.yml
│   └── service.yml
├── templates/
│   └── site.conf.j2
└── vars/
    ├── main.yml
    ├── RedHat.yml
    └── Debian.yml
```

`defaults/main.yml`:

```yaml
---
nginx_web_port: 8081
nginx_web_sites:
  - name: main
    title: "Hello from {{ inventory_hostname }}"
nginx_web_extra_packages: []
nginx_web_remove_default_site: false
```

`vars/RedHat.yml`:

```yaml
---
__nginx_web_user: nginx
__nginx_web_sites_root: /srv/sites
__nginx_web_conf_dir: /etc/nginx/conf.d
__nginx_web_default_site: ""                    # RedHat's default is inside nginx.conf
```

`vars/Debian.yml`:

```yaml
---
__nginx_web_user: www-data
__nginx_web_sites_root: /srv/sites
__nginx_web_conf_dir: /etc/nginx/conf.d
__nginx_web_default_site: /etc/nginx/sites-enabled/default
```

`vars/main.yml`:

```yaml
---
__nginx_web_packages: [nginx]
```

`meta/argument_specs.yml`:

```yaml
---
argument_specs:
  main:
    short_description: nginx serving one or more static sites
    description:
      - Installs nginx, writes one server block per site, and keeps the service running.
    options:
      nginx_web_port:
        type: int
        default: 8081
        description: Port every site listens on.
      nginx_web_sites:
        type: list
        elements: dict
        required: false
        description: Sites to serve; each gets /srv/sites/<name>/index.html.
        options:
          name:
            type: str
            required: true
            description: Site name (directory and server_name prefix).
          title:
            type: str
            default: Welcome
            description: Text of the index page.
      nginx_web_extra_packages:
        type: list
        elements: str
        default: []
        description: Extra packages to install with nginx.
      nginx_web_user:
        type: str
        description: User owning site files. Defaults to the OS nginx user.
      nginx_web_remove_default_site:
        type: bool
        default: false
        description: Remove the distribution's default site (Debian family only).
```

`tasks/main.yml`:

```yaml
---
- name: Load OS-specific variables
  ansible.builtin.include_vars: "{{ lookup('ansible.builtin.first_found', __nginx_web_vars_files) }}"
  vars:
    __nginx_web_vars_files:
      files:
        - "{{ ansible_facts['distribution'] }}-{{ ansible_facts['distribution_major_version'] }}.yml"
        - "{{ ansible_facts['os_family'] }}.yml"
      paths:
        - "{{ role_path }}/vars"
  tags: [always]

- name: Site names are unique
  ansible.builtin.assert:
    that:
      - nginx_web_sites | map(attribute='name') | unique | length == nginx_web_sites | length
    fail_msg: "nginx_web_sites contains duplicate names"
    quiet: true
  tags: [always]

- name: Install
  ansible.builtin.import_tasks: install.yml
  tags: [nginx_web, nginx_web_install]

- name: Configure
  ansible.builtin.import_tasks: configure.yml
  tags: [nginx_web, nginx_web_config]

- name: Service
  ansible.builtin.import_tasks: service.yml
  tags: [nginx_web, nginx_web_service]
```

`tasks/install.yml`:

```yaml
---
- name: apt cache is fresh
  ansible.builtin.apt:
    update_cache: true
    cache_valid_time: 3600
  when: ansible_facts['os_family'] == 'Debian'

- name: Packages are installed
  ansible.builtin.package:
    name: "{{ __nginx_web_packages + nginx_web_extra_packages }}"
    state: present
```

`tasks/configure.yml`:

```yaml
---
- name: Site directories exist
  ansible.builtin.file:
    path: "{{ __nginx_web_sites_root }}/{{ site.name }}"
    state: directory
    owner: "{{ nginx_web_user | default(__nginx_web_user) }}"
    mode: "0755"
  loop: "{{ nginx_web_sites }}"
  loop_control: { loop_var: site, label: "{{ site.name }}" }

- name: Index pages are in place
  ansible.builtin.copy:
    dest: "{{ __nginx_web_sites_root }}/{{ site.name }}/index.html"
    content: "{{ site.title | default('Welcome') }}\n"
    owner: "{{ nginx_web_user | default(__nginx_web_user) }}"
    mode: "0644"
  loop: "{{ nginx_web_sites }}"
  loop_control: { loop_var: site, label: "{{ site.name }}" }

- name: Server blocks are rendered
  ansible.builtin.template:
    src: site.conf.j2
    dest: "{{ __nginx_web_conf_dir }}/nginx_web.conf"
    mode: "0644"
  notify: nginx_web config changed

- name: Distribution default site is removed (if asked)
  ansible.builtin.file:
    path: "{{ __nginx_web_default_site }}"
    state: absent
  when:
    - nginx_web_remove_default_site | bool
    - __nginx_web_default_site | length > 0
  notify: nginx_web config changed
```

`tasks/service.yml`:

```yaml
---
- name: nginx is running and enabled
  ansible.builtin.systemd_service:
    name: nginx
    state: started
    enabled: true
```

`templates/site.conf.j2`:

```jinja
{{ ansible_managed | comment }}
{% for site in nginx_web_sites %}
server {
    listen {{ nginx_web_port }}{{ ' default_server' if loop.first else '' }};
    server_name {{ site.name }}.{{ inventory_hostname }};
    root {{ __nginx_web_sites_root }}/{{ site.name }};
    location /health { return 200 "ok\n"; }
}
{% endfor %}
```

`handlers/main.yml`:

```yaml
---
- name: Check nginx config
  ansible.builtin.command: nginx -t
  changed_when: false
  listen: nginx_web config changed

- name: Reload nginx
  ansible.builtin.systemd_service:
    name: nginx
    state: reloaded
  listen: nginx_web config changed
```

`README.md` (template):

```markdown
# nginx_web

Installs nginx and serves one or more static sites on one port.

## Supported platforms
AlmaLinux/RHEL 9–10, Ubuntu 24.04.

## Variables (see defaults/main.yml and meta/argument_specs.yml)
| Variable | Default | Description |
|---|---|---|
| nginx_web_port | 8081 | Port for every site |
| nginx_web_sites | one site "main" | List of {name, title} |
| nginx_web_extra_packages | [] | Extra packages |
| nginx_web_remove_default_site | false | Remove the distro default site (Debian) |

## Example
    - hosts: web
      roles:
        - role: nginx_web
          nginx_web_sites:
            - { name: shop, title: "Shop" }
            - { name: blog, title: "Blog" }
```

### Run it, and watch validation

`web-role.yml`:

```yaml
---
- name: Web servers via the nginx_web role
  hosts: web
  become: true
  roles:
    - role: nginx_web
      nginx_web_sites:
        - { name: shop, title: "Shop on {{ inventory_hostname }}" }
        - { name: blog, title: "Blog" }
```

```bash
ansible-playbook web-role.yml
```
```
TASK [nginx_web : Validating arguments against arg spec 'main' - nginx serving one or more static sites] ***
ok: [web1]
ok: [web2]

TASK [nginx_web : Load OS-specific variables] ******************************************
ok: [web1]
ok: [web2]
...
RUNNING HANDLER [nginx_web : Check nginx config] ***************************************
ok: [web1]
ok: [web2]

RUNNING HANDLER [nginx_web : Reload nginx] *********************************************
changed: [web1]
changed: [web2]
```

```bash
curl -s -H 'Host: blog.web1' 172.30.0.11:8081/      # Blog
curl -s 172.30.0.11:8081/                           # Shop on web1 (default_server = first site)
```

Now pass bad input:

```bash
ansible-playbook web-role.yml -e nginx_web_port=eighty
```
```
TASK [nginx_web : Validating arguments against arg spec 'main' - nginx serving one or more static sites] ***
fatal: [web1]: FAILED! => {"argument_errors": ["argument 'nginx_web_port' is of type <class 'str'> and we were unable to convert to int: ..."], "changed": false, "msg": "Validation of arguments failed:\nargument 'nginx_web_port' is of type <class 'str'> and we were unable to convert to int: ...", "validate_args_context": {"argument_spec_name": "main", "name": "nginx_web", "path": "/home/you/ansible-lab/roles/nginx_web", "type": "role"}}
```

Failed **before** any change, with a clear message.

```bash
ansible-playbook web-role.yml -e '{"nginx_web_sites": [{"title": "no name"}]}'
```
```
"msg": "Validation of arguments failed:\nmissing required arguments: name found in nginx_web_sites"
```

### Document the role

```bash
ansible-doc -t role -r roles nginx_web
```
```
> ROLE: *nginx_web* (/home/you/ansible-lab/roles/nginx_web)

ENTRY POINT: *main* - nginx serving one or more static sites

        Installs nginx, writes one server block per site, and keeps the service running.

OPTIONS (red indicates it is required):

   nginx_web_extra_packages  Extra packages to install with nginx.
        default: []
        elements: str
        type: list
   nginx_web_port  Port every site listens on.
        default: 8081
        type: int
...
```

### Run one phase

```bash
ansible-playbook web-role.yml --tags nginx_web_config
```

Runs the OS vars and assert (`always`), then only `configure.yml` (+ handlers).

---

## Hands-on lab

1. Restructure `roles/nginx_web` as above. Run `web-role.yml` twice: `changed=0` the second time.
2. Trigger each validation error: wrong type, missing `name`, duplicate site names (`assert`).
3. `ansible-doc -t role -r roles nginx_web` and compare with your README.
4. Add `vars/Ubuntu-24.yml` that changes only `__nginx_web_sites_root: /var/www/sites` and confirm web2 uses it (first_found picks the most specific file). Revert.
5. Set `nginx_web_remove_default_site: true` for web2 in `host_vars/web2.yml` and run. Confirm `/etc/nginx/sites-enabled/default` is gone and nginx still works.
6. Run `--tags nginx_web_config` after editing a site title: only configure tasks + handlers run.
7. Add a dependency on the `motd` role in `meta/main.yml`, run, then remove it again and list `motd` explicitly in the playbook instead. Which do you find clearer?

**Done when:** a teammate could use your role from its README and `ansible-doc` alone.

---

## Where you'll use it

- **Shared roles** in a team or company (published in an internal collection, topic 37).
- **Platform teams**: golden roles (baseline, logging, monitoring, nginx, postgres) with strict input specs.
- **Reviewing community roles**: these rules tell you quickly whether a Galaxy role is production-grade.
- **AWX/AAP surveys**: argument specs map naturally to user inputs.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Validation of arguments failed: argument 'x' is of type ...` | Wrong input type | Pass the right type (JSON for `-e`) |
| `missing required arguments: x` | Required option not given | Provide it, or give it a default |
| `value of x must be one of: a, b` | Not in `choices` | Use an allowed value |
| `Unable to find ... in expected paths` (first_found) | No matching OS file | Add a fallback (`default.yml`) or the OS file; check `paths` |
| User can't override an OS value | It's in `vars/` | Expose it as a default that falls back to the internal var |
| Role runs a dependency you didn't expect | Hidden `meta/main.yml` dependencies | Make composition explicit in playbooks |
| `--tags nginx_web_config` fails on undefined internals | OS vars task filtered out by tags | Tag it `always` (as above) |
| Spec and defaults disagree | `default:` in argument_specs isn't automatically applied from defaults/main.yml or vice versa | Keep them in sync; lint (topic 50) |
| Validation fails with `'__x' is undefined` | A default references an internal variable that is loaded later (OS vars) | Keep the public variable undefined; use `default(__x)` where it is used |

---

## Quick recap

- Reusable role = one job, **prefixed defaults** for every tunable, data-driven inputs, internals with `__` in `vars/`, no environment specifics, idempotent, check-mode safe, handlers with validation, README.
- **Multi-OS**: `vars/<Distribution>-<major>.yml` → `<os_family>.yml` via `include_vars` + `first_found`; tunable-with-fallback pattern.
- **`meta/argument_specs.yml`**: types, required, choices, nested options; validated automatically before the role runs; documented by `ansible-doc -t role`.
- `assert` for cross-field rules. Split tasks into install/configure/service with tags.
- Dependencies exist (`meta/main.yml`), but **explicit composition** in playbooks is clearer.

### Interview questions

1. **Q: How do you make a role work on both RHEL and Ubuntu?**
   A: Put OS-specific values in `vars/<OS>.yml` files and load the most specific one with `include_vars` + `first_found` (distribution-version, then family), keeping tasks generic.
2. **Q: What is `meta/argument_specs.yml`?**
   A: A declaration of a role's inputs (types, required, choices, nested options). Ansible validates them automatically before the role's tasks run, and `ansible-doc -t role` uses it as documentation.
3. **Q: Role dependencies vs listing roles in the playbook?**
   A: Dependencies run automatically before a role wherever it's used; explicit listing keeps the execution visible and controllable. Prefer explicit composition except for hard requirements.

### Scenario question (answer out loud first, then compare)

> *"Another team wants to use your `postgres` role, but they need a different data directory, extra extensions, and Ubuntu support (you only support RHEL). They've already forked it. How do you bring them back to one role?"*

**Model answer:**
1. Find out exactly what they changed in the fork; each change is either a missing **input** or missing **OS support**.
2. Add inputs as prefixed defaults: `postgres_data_dir`, `postgres_extensions: []` (data-driven), declared in `argument_specs` with types.
3. Add Ubuntu support with `vars/Debian.yml` (package names, service name, paths) loaded via `first_found`; keep tasks generic; add Ubuntu to `meta/main.yml` platforms.
4. Keep behaviour identical for existing users: defaults equal to the old hard-coded values; prove with `--check --diff` on existing hosts (no diff).
5. Add Molecule scenarios for RHEL and Ubuntu, publish a new version (in a collection, with a changelog), and help them switch from the fork using their values in `group_vars`.

**Likely follow-ups:**
- *What if a change is truly team-specific?* An extension point: a variable like `postgres_extra_conf` (list of lines or a dict rendered into an include file), or a `tasks_from` hook, rather than a fork.
- *How do you avoid breaking consumers when you change defaults later?* Semantic versioning, changelog, and pinning in consumers' `requirements.yml` (topic 37).
