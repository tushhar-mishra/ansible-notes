# 33 · Tags

> **Level:** ★ foundation
>
> **You will learn:** how to label tasks, blocks, plays and roles with **tags** so you can run **part** of a playbook (`--tags`, `--skip-tags`, `--list-tags`), the special tags **`always`**, **`never`**, `tagged`, `untagged`, `all`, how tags are **inherited**, the traps (skipping a task that registers a variable later tasks need, handlers, fact gathering), the `ansible_run_tags` variable, and when tags help versus when a separate playbook is better.
>
> **You need:** [32 · Advanced Jinja2](32-advanced-jinja2-and-lookups.md), [31 · Error handling](31-error-handling.md) (blocks), [30 · Handlers](30-handlers.md).
>
> **Part 4 · Organizing and reusing code** · Next: [34 · import vs include](34-import-vs-include.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **tag** is a label on a task (or block, play, role). On the command line you say "run only tasks with these labels" (`--tags`) or "everything except these" (`--skip-tags`). Great for running just the config part of a big playbook.

**Picture it.** Colour-coded sticky notes in a big cookbook: run only the "dessert" pages tonight.

**The one thing to know.**

```yaml
- name: nginx is installed
  ansible.builtin.package: { name: nginx, state: present }
  tags: [packages, nginx]

- name: nginx config
  ansible.builtin.template: { src: nginx.conf.j2, dest: /etc/nginx/nginx.conf }
  tags: [config, nginx]

- name: Wipe the cache (only when asked)
  ansible.builtin.file: { path: /var/cache/nginx, state: absent }
  tags: [never, wipe_cache]
```

```bash
ansible-playbook site.yml --list-tags            # which tags exist
ansible-playbook site.yml --tags config          # only config tasks
ansible-playbook site.yml --skip-tags packages   # everything except packages
ansible-playbook site.yml --tags wipe_cache      # the 'never' task runs only when named
```

**Try it now:** tag two tasks in `web-vars.yml` and run with `--tags` and `--list-tasks --tags ...` to preview.

**You've got it if you can:** explain what `always` and `never` do, and why `--tags` can make a later task fail with an undefined variable.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Where you can put tags

| Level | Effect |
|---|---|
| Task | That task |
| Block | Every task in the block (and its rescue/always) |
| Play | Every task in the play |
| Role (`roles: - { role: x, tags: [x] }`) | Every task of the role |
| `import_tasks` / `import_role` / `import_playbook` | Every imported task (static, topic 34) |
| `include_tasks` / `include_role` | ⚠️ Only the **include statement itself**, not the tasks inside (dynamic, topic 34). Use `apply: { tags: [...] }` to pass tags inside |

Tags **add up**: a task inherits the tags of its block, play, role and import, plus its own.

### Command-line options

| Option | Runs |
|---|---|
| (none) | All tasks except those tagged `never` (same as `--tags all`) |
| `--tags a,b` | Tasks with tag `a` **or** `b` (plus `always`) |
| `--skip-tags a` | All tasks except those tagged `a` (a task tagged both `always` and `a` is skipped; plain `always` tasks still run unless you `--skip-tags always`) |
| `--tags a --skip-tags b` | Tagged `a`, minus those also tagged `b` |
| `--list-tags` | Print all tags, run nothing |
| `--list-tasks --tags a` | Preview which tasks would run |

### Special tags

| Tag | Meaning |
|---|---|
| `always` | Runs even when you select other tags. Skipped only with `--skip-tags always` |
| `never` | **Never** runs unless one of its **other** tags is requested explicitly (`--tags wipe_cache`), or `--tags never` |
| `tagged` | `--tags tagged`: only tasks that have at least one tag |
| `untagged` | `--tags untagged`: only tasks with no tags |
| `all` | Every task (the default) |

The implicit **Gathering Facts** task is tagged `always`, so facts are collected even with `--tags`.

### `ansible_run_tags` / `ansible_skip_tags`

Magic variables holding what was passed: `when: "'deploy' in ansible_run_tags"` lets a task behave differently depending on the run.

---

## Why it exists

Big playbooks (a full server build) take minutes. During development or a quick config fix you want to run **only the relevant part** without editing the file. Tags also give you:

- **Opt-in dangerous actions** (`never` + a specific tag): wipe caches, reset databases, rotate keys.
- **Always-on safety checks** (`always`): assertions, required facts.
- **Per-component runs** in CI or on-call: `--tags nginx`, `--tags users`.

---

## Analogy

**Highlighter pens in a manual.** Yellow = "config", green = "packages". "Tonight, only do the yellow steps" (`--tags config`). Some steps are printed in a red box saying "safety check: always do this" (`always`). A sealed envelope at the back, "factory reset", is never opened unless the manager names it explicitly (`never`).

---

## How it works inside

```
 ansible-playbook site.yml --tags config
   │
   ├─ parse: each task's EFFECTIVE tags = own tags ∪ block ∪ role ∪ import ∪ play
   │
   ├─ for each task:
   │     effective tags ∩ {config} non-empty?  → run
   │     'always' in effective tags?            → run   (unless --skip-tags always)
   │     'never' in tags and not requested?     → skip
   │     otherwise                              → skip silently (NOT shown as "skipping")
   │
   └─ handlers: run if notified by a task that ran and changed
```

Tasks filtered out by tags are **not shown** at all (unlike `when`, which prints `skipping`). And because they don't run, their `register` variables **never get set**.

---

## Code / commands

### A tagged playbook

`~/ansible-lab/tags.yml`:

```yaml
---
- name: Tags demo on the web servers
  hosts: web
  become: true

  tasks:
    - name: Preconditions
      ansible.builtin.assert:
        that: ansible_facts['os_family'] in ['RedHat', 'Debian']
        quiet: true
      tags: always

    - name: Packages
      tags: packages
      block:
        - name: apt cache (Debian family)
          ansible.builtin.apt: { update_cache: true, cache_valid_time: 3600 }
          when: ansible_facts['os_family'] == 'Debian'
        - name: nginx is installed
          ansible.builtin.package: { name: nginx, state: present }

    - name: Home page
      ansible.builtin.copy:
        dest: "{{ web_root }}/index.html"
        content: "{{ web_page_text }}\n"
        mode: "0644"
      tags: [config, content]

    - name: Read nginx version
      ansible.builtin.command: nginx -v
      register: nginx_v
      changed_when: false
      tags: info

    - name: Show nginx version
      ansible.builtin.debug:
        msg: "{{ nginx_v.stderr }}"
      tags: report

    - name: Wipe nginx logs (only on request)
      ansible.builtin.shell: truncate -s0 /var/log/nginx/*.log
      tags: [never, wipe_logs]
```

- `nginx -v` prints its version to **stderr**, hence `nginx_v.stderr`.
- The block's `tags: packages` applies to both tasks inside.

### List and preview

```bash
ansible-playbook tags.yml --list-tags
```
```
playbook: tags.yml

  play #1 (web): Tags demo on the web servers	TAGS: []
      TASK TAGS: [always, config, content, info, never, packages, report, wipe_logs]
```

```bash
ansible-playbook tags.yml --list-tasks --tags config
```
```
  play #1 (web): Tags demo on the web servers	TAGS: []
    tasks:
      Preconditions	TAGS: [always]
      Home page	TAGS: [config, content]
```

`always` is included automatically.

### Run parts

```bash
ansible-playbook tags.yml --tags config
```
```
TASK [Gathering Facts] *****************************************************************
ok: [web1]
ok: [web2]
TASK [Preconditions] *******************************************************************
ok: [web1]
ok: [web2]
TASK [Home page] ***********************************************************************
ok: [web1]
ok: [web2]
PLAY RECAP *****************************************************************************
web1   : ok=3  changed=0  ...
```

Only facts (always), the assert (always) and the tagged task. No "skipping" lines for the rest: they're filtered out silently.

### The register trap

```bash
ansible-playbook tags.yml --tags report
```
```
TASK [Show nginx version] **************************************************************
fatal: [web1]: FAILED! => {"msg": "The task includes an option with an undefined variable. 'nginx_v' is undefined..."}
```

`Read nginx version` (tag `info`) didn't run, so `nginx_v` doesn't exist. Fixes: give both tasks the same tag (`tags: [info, report]` on the reader), or tag the reader `always` if it's cheap and read-only.

### never

```bash
ansible-playbook tags.yml                         # wipe task does NOT run
ansible-playbook tags.yml --tags wipe_logs        # runs (plus 'always' tasks)
```

---

## Hands-on lab

1. Create `tags.yml`. Run `--list-tags`, then `--list-tasks` with `--tags packages`, `--tags config`, `--skip-tags packages`.
2. Run with `--tags report`, read the undefined-variable error, and fix it by tagging the reader `[info, report]`.
3. Run with `--tags wipe_logs` and check `/var/log/nginx/access.log` is empty on web1.
4. Add `tags: web` at the **play** level and confirm with `--list-tasks` that every task now also has `web`.
5. Use `ansible_run_tags`: add a debug that prints "full run" when `'all' in ansible_run_tags` and "partial run" otherwise.
6. Decide: which of your existing playbooks would benefit from `never`-tagged maintenance tasks, and which should instead become separate playbooks?

**Done when:** you can run any slice of a playbook safely without breaking later tasks.

---

## Where you'll use it

- **Fast iteration**: `--tags config` while editing templates on a big role.
- **Per-component operations**: `--tags nginx,certs` in a site-wide playbook.
- **Dangerous maintenance on demand**: `never` + a specific tag.
- **Safety checks that always run**: `always` on asserts.
- **AWX/AAP job templates**: the same playbook exposed as several "jobs" with different tag selections (topic 56).

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `'x' is undefined` when running with `--tags` | The task that registers/sets `x` was filtered out | Tag it too, or `always` (if cheap and read-only) |
| Tagged tasks inside an `include_tasks` don't run | Dynamic includes don't pass tags inside | `apply: { tags: [...] }` on the include, or use `import_tasks` (topic 34) |
| `--tags x` runs nothing, no error | Typo, or tag doesn't exist | `--list-tags` |
| A `never` task ran unexpectedly | It also has another tag you requested | Give `never` tasks a **unique** second tag |
| Facts still gathered with `--tags` | Gathering Facts is tagged `always` | Expected; `gather_facts: false` if not needed, or `--skip-tags always` (careful) |
| Handler didn't run with `--tags` | The notifying task was filtered out, so nothing notified | Tag the notifying task |
| Tag soup: 40 tags nobody understands | Over-tagging | A few well-defined tags (role names, `config`, `packages`), document them |

---

## Quick recap

- Tags label tasks/blocks/plays/roles/imports; tags are **inherited** (except into dynamic `include_*` without `apply`).
- `--tags`, `--skip-tags`, `--list-tags`, `--list-tasks --tags ...` to preview.
- `always` runs with any selection; `never` runs only when its other tag is requested. Gathering Facts is `always`.
- Filtered tasks vanish silently and **don't register** their variables.
- Use a few meaningful tags; prefer separate playbooks for truly separate jobs.

### Interview questions

1. **Q: What are the `always` and `never` tags?**
   A: `always` tasks run whatever tags you select (unless you skip `always` explicitly). `never` tasks run only when you explicitly request one of their other tags (or `never` itself).
2. **Q: Why might `--tags deploy` cause an undefined variable error?**
   A: A task the deploy tasks depend on (one that registers or sets a variable) isn't tagged `deploy`, so it was filtered out and the variable was never created.
3. **Q: Do tags on `include_tasks` apply to the included tasks?**
   A: No, only to the include itself; use `apply: { tags: [...] }` or `import_tasks` for inheritance.

### Scenario question (answer out loud first, then compare)

> *"On-call engineers want a quick way to 'just restart the app and clear its cache' on production using the existing 300-task site playbook. How do you design that safely?"*

**Model answer:**
1. Don't make them run the full playbook with clever tags: one wrong tag selection could run unrelated changes.
2. Create a small **operational playbook** (`ops/restart-app.yml`): assert environment and approval, optional cache clear, `serial` restart with LB drain and health check (topics 45–46, 49).
3. If they must use the site playbook, put the actions behind `never` + specific tags (`--tags app_restart`), make sure every dependency of those tasks carries the same tag, and test with `--list-tasks --tags app_restart` in CI.
4. Expose it as an AWX job template with a survey and RBAC so on-call can run only that (topic 56).

**Likely follow-ups:**
- *How would you prevent a full run by accident?* Separate playbooks; or a precondition `assert` that fails unless `ansible_run_tags` contains an expected tag.
- *Why are separate playbooks better than tags for operations?* Smaller blast radius, clearer review, simpler permissions.
