# 68 · Cheat Sheet

> **Level:** ★ foundation
>
> **You will learn:** nothing new: this is the **one-page reference** for everything in the course: commands and flags, patterns, playbook/task/role skeletons, keywords, the most-used modules, Jinja2 filters and tests, variable precedence, idempotency fixes, safe-run procedure, config, project layout, and the top errors with fixes. Each block points to its topic.
>
> **You need:** the course (this page summarises it).
>
> **Part 12 · Production operations** · This is the last topic · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Keep this page open while you work. If a line isn't clear, follow its topic number.

**The one thing to know.** The safe production run:

```bash
ansible-playbook -i inventories/prod playbooks/site.yml --syntax-check
ansible-playbook -i inventories/prod playbooks/site.yml --list-hosts
ansible-playbook -i inventories/prod playbooks/site.yml --check --diff
ansible-playbook -i inventories/prod playbooks/site.yml --diff -l canary
ansible-playbook -i inventories/prod playbooks/site.yml --diff
ansible-playbook -i inventories/prod playbooks/site.yml          # expect changed=0
```

> 📖 Everything below is the **reference**.

---

## What it is

### Commands (topics 03, 08, 12)

| Command | Use |
|---|---|
| `ansible PATTERN -m MOD -a "k=v" [-b] [-o]` | Ad hoc |
| `ansible-playbook PB [-i INV] [options]` | Run a playbook |
| `ansible-inventory -i INV --graph [--vars] / --list / --host H` | Inspect inventory |
| `ansible-doc FQCN` / `-s` / `-l` / `-t TYPE` | Docs (modules, plugins, keywords) |
| `ansible-config dump --only-changed` / `list` / `init --disabled` | Config |
| `ansible-galaxy collection install -r requirements.yml -p ./collections` | Collections |
| `ansible-galaxy collection list` / `verify` / `init` / `build` / `publish` | Collections |
| `ansible-galaxy role init roles/x` / `role install` | Roles |
| `ansible-vault create/edit/view/encrypt/decrypt/rekey FILE` | Vault files |
| `ansible-vault encrypt_string --name VAR 'value'` | Vault one value |
| `ansible-console -i INV` | Interactive ad hoc shell |
| `ansible-pull -U REPO -C BRANCH -o PLAYBOOK` | Pull mode |
| `ansible-lint` / `yamllint .` | Lint |
| `molecule test` / `converge` / `verify` / `login -h H` / `destroy` | Role tests |
| `ansible-builder build -t ee:1.0` | Build an EE |
| `ansible-navigator run PB -i INV --eei ee:1.0 -m stdout` | Run in an EE |

### ansible-playbook flags (topics 12, 20, 33, 45)

| Flag | Meaning |
|---|---|
| `-i INV` | Inventory (file, dir, `host1,host2,`) |
| `-l PATTERN` / `-l @file` | Limit hosts |
| `-e var=v` / `-e '{"v": 1}'` / `-e @file.yml` | Extra vars (strings / typed / file) |
| `-C` `--check` / `-D` `--diff` | Dry run / show changes |
| `-t TAGS` / `--skip-tags TAGS` | Select tasks |
| `--list-hosts` / `--list-tasks` / `--list-tags` | Preview |
| `--syntax-check` | Parse only |
| `--start-at-task "NAME"` / `--step` | Partial / interactive |
| `-b` / `-K` / `--become-user U` | Become / ask password / target user |
| `-u USER` / `--private-key F` / `-c CONN` | Connection (variables win over `-u`) |
| `-f N` | Forks |
| `-v` … `-vvvv` | Verbosity |
| `--vault-id label@source` / `-J` | Vault password |
| `--force-handlers` / `--flush-cache` | Handlers despite failures / refresh fact cache |

### Host patterns (topic 12)

| Pattern | Means |
|---|---|
| `all`, `*` | Everything |
| `web:db` | Union |
| `web:&prod` | Intersection |
| `web:!web3` | Exclusion (quote it!) |
| `web*`, `~web[0-9]+` | Wildcard / regex |
| `web[0]`, `web[-1]`, `web[0:2]` | Index / slice |

### Playbook skeleton (topics 14, 30, 31, 45)

```yaml
---
- name: Describe the play
  hosts: web
  become: true
  gather_facts: true
  serial: [1, "25%", "100%"]
  max_fail_percentage: 0
  vars:
    app_port: 8080
  vars_files: [vars/common.yml]
  pre_tasks:
    - name: Preconditions
      ansible.builtin.assert:
        that: ansible_facts['os_family'] in ['RedHat', 'Debian']
  roles:
    - role: nginx_web
      nginx_web_port: "{{ app_port }}"
  tasks:
    - name: Something is in the desired state
      ansible.builtin.template:
        src: app.conf.j2
        dest: /etc/app.conf
        mode: "0644"
        validate: /usr/bin/app --check %s
      notify: Restart app
  post_tasks:
    - name: Health check
      ansible.builtin.uri: { url: "http://{{ ansible_host }}:{{ app_port }}/health" }
      delegate_to: localhost
      become: false
  handlers:
    - name: Restart app
      ansible.builtin.systemd_service: { name: app, state: restarted }
```

### Task keywords (topics 14, 26–34, 45–47)

| Keyword | Use |
|---|---|
| `name` | Desired-state description |
| `when` | Condition (boolean!) |
| `loop` / `loop_control` (`label`, `loop_var`, `index_var`, `extended`) | Repeat |
| `register` | Save result |
| `changed_when` / `failed_when` | Redefine changed/failed |
| `until` / `retries` / `delay` | Retry |
| `notify` / `listen` | Handlers |
| `become` / `become_user` | Privilege |
| `delegate_to` / `delegate_facts` / `run_once` | Where/once |
| `tags` | Selection |
| `ignore_errors` / `ignore_unreachable` | Continue (use sparingly) |
| `check_mode` / `diff` | Force check / hide diff |
| `no_log` | Hide output (secrets) |
| `async` / `poll` | Background |
| `throttle` | Per-task concurrency |
| `environment` | Env vars for the task |
| `vars` | Task-scoped variables |
| `args: { creates/removes }` | Idempotent commands |
| `block` / `rescue` / `always` | Groups / try-catch |
| `debugger: on_failed` | Interactive debug |

### Most-used modules (topics 16–18, 44)

| Area | Modules |
|---|---|
| Packages | `ansible.builtin.package`, `.dnf`, `.apt` (`update_cache`), `.yum_repository`, `.deb822_repository`, `.package_facts` |
| Services | `ansible.builtin.systemd_service` (`state`, `enabled`, `daemon_reload`), `.service_facts` |
| Users | `ansible.builtin.user` (`append: true`), `.group`, `ansible.posix.authorized_key` |
| Files | `ansible.builtin.file`, `.copy`, `.template`, `.lineinfile`, `.blockinfile`, `.replace`, `.stat`, `.fetch`, `.get_url` (`checksum`), `.unarchive` (`creates`), `ansible.posix.synchronize` |
| Commands | `ansible.builtin.command`, `.shell`, `.raw`, `.script` |
| Logic | `ansible.builtin.debug`, `.assert`, `.fail`, `.set_fact`, `.include_tasks`, `.import_tasks`, `.include_role`, `.import_role`, `.include_vars`, `.meta`, `.group_by`, `.add_host` |
| Waiting | `ansible.builtin.wait_for`, `.wait_for_connection`, `.reboot`, `.uri`, `.async_status`, `.pause` |
| Security | `ansible.posix.firewalld`, `community.general.ufw`, `ansible.posix.selinux`, `.seboolean`, `community.general.sefcontext` |
| System | `ansible.builtin.cron`, `.hostname`, `ansible.posix.sysctl`, `.mount`, `community.general.timezone` |
| Cloud/containers | `amazon.aws.*`, `community.docker.docker_container`, `kubernetes.core.k8s`, `.helm` |
| Windows | `ansible.windows.win_*` |

### Jinja2 (topics 22, 32)

| Filters | |
|---|---|
| Defaults | `default(x)`, `default(x, true)`, `default(omit)`, `mandatory` |
| Types | `int`, `float`, `string`, `bool`, `type_debug` |
| Strings | `lower`, `upper`, `replace`, `trim`, `regex_replace`, `regex_search`, `regex_findall`, `quote`, `b64encode`, `hash('sha256')`, `password_hash('sha512')`, `basename`, `dirname` |
| Lists | `length`, `join`, `first`, `last`, `unique`, `sort(attribute=)`, `min`, `max`, `sum`, `flatten`, `union`, `difference`, `zip`, `product`, `subelements` |
| Select/map | `map(attribute=)`, `map('extract', hostvars, 'ansible_host')`, `select`, `reject`, `selectattr('x', 'equalto', v)`, `rejectattr`, `groupby` |
| Dicts | `combine(recursive=true)`, `dict2items`, `items2dict`, `dictsort` |
| Data | `to_nice_yaml`, `to_nice_json`, `from_json`, `from_yaml`, `community.general.json_query`, `ansible.utils.ipaddr` |
| Secrets (2.22) | `register_secret`, `mask_secrets` |

| Tests | `is defined`, `is undefined`, `is none`, `is string`, `is mapping`, `is match('re')`, `is search('re')`, `is version('2.0', '>=')`, `is changed`, `is failed`, `is succeeded`, `is skipped`, `is finished` |
|---|---|

| Lookups (control node) | `lookup('ansible.builtin.file', p)`, `env`, `pipe`, `template`, `password`, `vars`, `first_found`; `query('ansible.builtin.fileglob', 'x/*')` |
|---|---|

Rules (2.19+): quote values starting with `{{`; no nested `{{ }}`; `when` must be boolean; `range(n) | list`.

### Variable precedence (topic 25)

```
 LOW  role defaults
      inventory: inline group vars < group_vars/all < group_vars/<group> (child > parent) < inline host vars < host_vars
      facts (gathered / cached)
      play vars < vars_prompt < vars_files
      role vars
      block vars < task vars
      include_vars
      set_fact / register
      role & include params
 HIGH extra vars (-e)
```

### Idempotency fixes (topic 27)

| Problem | Fix |
|---|---|
| `command`/`shell` | Module → `creates`/`removes` → `changed_when`/`failed_when` → check-then-act |
| `state: restarted` | `started` + handler |
| `state: latest` | `present` + pinned versions |
| Timestamps in templates | Remove run-time values; sort data |
| `password_hash` | `update_password: on_create` |
| Downloads | `get_url` + `checksum` |
| Test it | Run twice → `changed=0`; Molecule idempotence |

### ansible.cfg starter (topics 11, 48)

```ini
[defaults]
inventory = ./inventories/dev
roles_path = ./roles
collections_path = ./collections:~/.ansible/collections
interpreter_python = auto_silent
forks = 25
gathering = smart
fact_caching = ansible.builtin.jsonfile
fact_caching_connection = ~/.ansible/facts_cache
fact_caching_timeout = 7200
callbacks_enabled = ansible.posix.profile_tasks, ansible.posix.timer
callback_result_format = yaml
host_key_checking = True
retry_files_enabled = False

[ssh_connection]
pipelining = True
ssh_args = -C -o ControlMaster=auto -o ControlPersist=600s -o ServerAliveInterval=30
```

### Project layout (topic 39)

```
infra/
├── ansible.cfg  requirements.txt  collections/requirements.yml  Makefile  README.md
├── inventories/{dev,staging,prod}/{hosts.yml|aws_ec2.yml, group_vars/, host_vars/}
├── playbooks/{site.yml, web.yml, db.yml, ops/}
└── roles/<role>/{tasks,handlers,defaults,vars,templates,files,meta,molecule}/
```

### Role skeleton (topics 35–36)

```
defaults/main.yml          # <role>_* tunables
vars/{main,RedHat,Debian}.yml   # __<role>_* internals
meta/main.yml              # galaxy_info, dependencies
meta/argument_specs.yml    # input validation
tasks/main.yml             # include_vars first_found → import install/configure/service
handlers/main.yml          # validate → reload/restart
templates/  files/  molecule/default/
```

### Top errors (topic 59)

| Error | Fix |
|---|---|
| `UNREACHABLE ... Permission denied (publickey)` | User/key/agent; `ssh -v` |
| `Host key verification failed` | Verified `known_hosts` / `accept-new` |
| `Missing sudo password` | NOPASSWD, `-K`, vaulted `ansible_become_password` |
| `python3: not found` | `raw` bootstrap |
| `couldn't resolve module/action` | FQCN; install collection |
| `'x' is undefined` | Define/`default`; check groups |
| `Conditionals must have a boolean result` | `| bool`, `| length > 0` |
| `unexpected '{'` | No nested `{{ }}` |
| `Syntax Error while loading YAML` | Spaces, quotes, alignment |
| `failed to validate` | Fix the content (real file untouched) |
| `Attempting to decrypt but no vault secrets found` | Provide the vault password |
| `Failed to set permissions on the temporary files ... unprivileged user` | `acl` or pipelining |

### Debug toolkit (topic 58)

```bash
ansible H -m ansible.builtin.ping -vvvv                # connection
ansible-inventory --host H                             # inventory vars
ansible H -m ansible.builtin.debug -a var=X            # value
ansible-config dump --only-changed                     # config
ansible-playbook PB -l H --start-at-task "T" -vvv      # narrow
ANSIBLE_KEEP_REMOTE_FILES=1 ...                        # inspect module on target
ANSIBLE_DISPLAY_TRACEBACK=always ...                   # crashes
```

---

## Why it exists

To save you from searching through 67 topics for one flag or filter.

---

## Analogy

The laminated quick-reference card next to a machine: the full manual is on the shelf, but the card answers 90% of everyday questions.

---

## How it works inside

Every block mirrors a topic; when a block raises a question, the topic number tells you where the full explanation, lab and pitfalls are.

---

## Code / commands

See the blocks above.

---

## Hands-on lab

1. Without looking, write the playbook skeleton from memory, then compare.
2. From memory, list the precedence ladder and five idempotency fixes.
3. Print this page (or keep it in a second window) for your next real project.

**Done when:** you rarely need to open the full topics for everyday work.

---

## Where you'll use it

Daily work, interviews (last-minute revision), on-call.

---

## When it breaks

If something here doesn't match your installed version, trust `ansible-doc` and `ansible --version` (topic 16), and check the porting guide for your version (topic 63).

---

## Quick recap

- Safe run: syntax → list-hosts → check+diff → canary → all → re-run `changed=0`.
- Know: commands/flags, patterns, skeletons, keywords, top modules, filters/tests, precedence, idempotency fixes, config, layout, top errors, debug toolkit.

### Interview questions

1. **Q: Give the safe sequence for running a playbook against production.** A: Syntax check, list hosts, check+diff and review, canary, full run in batches, re-run expecting `changed=0`.
2. **Q: Name five task keywords for controlling execution.** A: `when`, `loop`, `register`, `changed_when`/`failed_when`, `delegate_to`/`run_once`, `notify`, `become`, `async`, `throttle`.
3. **Q: Where do tunable role values go, and what beats everything?** A: Role `defaults/`; extra vars (`-e`).

### Scenario question (answer out loud first, then compare)

> *"You have five minutes before an interview. What do you revise?"*

**Model answer:** The ten must-nail questions from topic 66 (idempotency, Ansible vs Terraform, agentless push, playbook execution flow, precedence, handlers, roles, vault, rolling updates, debugging), the precedence ladder and idempotency fixes from this page, and one project story (topic 57 or 62) with its design choices and failure handling.

**Likely follow-ups:**
- *Which project would you describe?* The one where you can explain trade-offs and failures, not just features.
