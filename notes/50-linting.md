# 50 · Linting: ansible-lint and yamllint

> **Level:** ★★ intermediate
>
> **You will learn:** how to catch mistakes **before** anything runs: **yamllint** (YAML style and syntax) and **ansible-lint** (Ansible best practices: FQCNs, task names, idempotency smells, risky permissions, deprecated syntax), ansible-lint's **profiles** (`min` → `production`), the most common rules and how to fix them, **`--fix`** auto-fixing, configuration (`.ansible-lint`, `.yamllint`), skipping rules properly (`# noqa`, ignore files), **pre-commit** hooks, the **ansible-dev-tools** bundle and the **VS Code** extension. You'll lint your `~/infra` project to the `production` profile.
>
> **You need:** [49 · Project 1](49-project-rolling-deploy.md), [27 · Idempotency](27-idempotency-in-depth.md) (the smells lint finds), [16 · FQCNs](16-modules-fqcn-and-ansible-doc.md), [04 · YAML](04-yaml-for-ansible.md).
>
> **Part 8 · Quality and testing** · Next: [51 · Testing with Molecule](51-testing-with-molecule.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **linter** reads your code without running it and points out problems: broken YAML, short module names, tasks without names, `command` tasks that will report `changed` every run, files created with unsafe permissions, deprecated syntax. Run it on every commit, locally and in CI.

**Picture it.** A spell-and-grammar checker for automation: it doesn't know if your story is good, but it catches the typos and bad habits before an editor (or production) sees them.

**The one thing to know.**

```bash
uv pip install ansible-lint yamllint
cd ~/infra
yamllint .
ansible-lint                       # uses .ansible-lint if present
ansible-lint --fix                 # auto-fix what can be fixed safely (FQCN, YAML formatting, ...)
```

```
playbooks/old.yml:5: fqcn[action-core]: Use FQCN for builtin module actions (copy).
playbooks/old.yml:9: no-changed-when: Commands should not change things if nothing needs doing.
playbooks/old.yml:12: risky-file-permissions: File permissions unset or incorrect.
```

**Try it now:** run `ansible-lint` on `idem-bad.yml` from topic 27 and match each finding to a fix you learned there.

**You've got it if you can:** read a lint finding, explain why it matters, and fix it or justify a skip.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### yamllint vs ansible-lint

| | yamllint | ansible-lint |
|---|---|---|
| Knows about | YAML only | Ansible: playbooks, roles, tasks, vars, collections, `meta/`, Molecule, requirements files |
| Finds | Syntax errors, indentation, trailing spaces, line length, truthy values, duplicate keys | Everything yamllint finds (it runs yamllint-style checks itself) **plus** Ansible best-practice rules |
| Config | `.yamllint` | `.ansible-lint` (+ `.yamllint` for its YAML rules) |

Use both, or just ansible-lint (which includes YAML checks); keep a `.yamllint` so editors and other tools agree on the style.

### ansible-lint profiles

Profiles group rules by strictness. Each includes the previous ones:

| Profile | Adds | Use for |
|---|---|---|
| `min` | Only fatal problems (syntax, unloadable content) | Legacy code, first step |
| `basic` | Style and common mistakes (YAML, naming) | |
| `moderate` | Readability and maintainability rules | |
| `safety` | Rules about risky behaviour (unsafe permissions, `latest`, ...) | |
| `shared` | Rules for content you'll share (Galaxy/collections) | Roles/collections used by others |
| `production` | Strictest: everything, including FQCN everywhere | ✅ Production repos, certified content |

Set it in `.ansible-lint` (`profile: production`) or with `--profile`.

### The rules you'll meet most

| Rule | Means | Fix |
|---|---|---|
| `fqcn[action-core]` / `fqcn[action]` | Short module name | `ansible.builtin.copy`, `community.general.ufw` |
| `name[missing]` | Task without a name | Add `name:` (a desired-state sentence) |
| `name[casing]` | Name doesn't start with a capital | Capitalise |
| `name[template]` | `{{ }}` in the middle of a name | Put templates at the end of the name |
| `no-changed-when` | `command`/`shell` without `changed_when` | Add `changed_when`, `creates`, or use a module (topic 27) |
| `command-instead-of-module` | `command: systemctl ...`, `command: git ...` | Use the module |
| `command-instead-of-shell` | `shell` used with no shell features | Use `command` |
| `risky-file-permissions` | File created without an explicit `mode` | Add `mode: "0644"` etc. |
| `risky-shell-pipe` | Pipes without `set -o pipefail` | Add it (`executable: /bin/bash`) |
| `package-latest` | `state: latest` | `present` + pinned versions (topic 17) |
| `no-handler` | `when: x is changed` to trigger work | Use `notify` + a handler (topic 30) |
| `ignore-errors` | `ignore_errors: true` | `failed_when`, or `rescue` (topic 31) |
| `yaml[truthy]` | `yes`/`no`/`on` | `true`/`false` (topic 04) |
| `yaml[line-length]`, `yaml[indentation]`, `yaml[trailing-spaces]` | Style | Fix or relax in `.yamllint` |
| `jinja[spacing]` | `{{var}}` | `{{ var }}` |
| `var-naming[no-role-prefix]` | Role variable without the role prefix | `rolename_var` (topic 36) |
| `no-free-form` | `command: echo hi` free-form or `k=v` strings | `cmd:` / YAML arguments |
| `key-order[task]` | `name` not first, `block` keys out of order | Reorder |
| `partial-become` | `become_user` without `become` | Add `become: true` |
| `run-once[task]` | `run_once` with `strategy: free` | Rethink (run_once is unreliable there) |
| `schema[...]` | File doesn't match the schema (meta, argument_specs, galaxy.yml, requirements...) | Fix structure |
| `syntax-check[...]` | `ansible-playbook --syntax-check` failed | Fix the error shown |
| `internal-error` / `load-failure` | Couldn't load a file | Fix YAML or path |

`ansible-lint -L` lists all rules; `ansible-lint -T` lists tags.

### Configuration: `.ansible-lint`

```yaml
---
profile: production
exclude_paths:
  - collections/ansible_collections/      # installed third-party content
  - .cache/
  - dist/
warn_list:                                 # report but don't fail
  - experimental
skip_list: []                              # rules to ignore entirely (keep this short!)
enable_list:
  - no-same-owner                          # opt-in rules
kinds:                                     # tell lint what non-standard files are
  - playbook: "playbooks/**/*.yml"
  - tasks: "roles/*/tasks/*.yml"
offline: true                              # don't install requirements during lint (CI decides)
```

### Skipping, the right way

| Method | Scope | Use |
|---|---|---|
| `# noqa: rule-name` at the end of a line | That task | A justified exception: **add a comment why** |
| `.ansible-lint-ignore` file (`path rule`) | Specific files | Legacy files you'll fix later (generate with `--generate-ignore`) |
| `skip_list` in `.ansible-lint` | Whole repo | Only for rules your team deliberately rejects |

```yaml
- name: Reload systemd only via this vendor script (no module exists for it)
  ansible.builtin.command: /opt/vendor/bin/reload   # noqa: no-changed-when  (vendor tool, always safe to run)
```

### `--fix`

`ansible-lint --fix` rewrites files to fix some rules automatically: FQCNs, YAML formatting (indentation, quotes, `true`/`false`), `key-order`, `name[casing]`... It also **reformats YAML** (it re-dumps files), so review the diff before committing (`git diff`). Use `--fix=fqcn,yaml` to limit it to chosen rules.

### Tooling around it

| Tool | What |
|---|---|
| **ansible-dev-tools** (`pip install ansible-dev-tools`) | One package installing the dev toolchain: ansible-lint, molecule, ansible-navigator, ansible-builder, ansible-creator, pytest-ansible, tox-ansible |
| **ansible-creator** | Scaffolds new collections/playbook projects with lint/test configs |
| **VS Code "Ansible" extension** (Red Hat) | Syntax highlighting, completion of modules/options, docs on hover, **ansible-lint on save** |
| **pre-commit** | Runs yamllint/ansible-lint before every `git commit` |
| CI | The same commands; fail the PR on findings (topics 53–54) |

---

## Why it exists

Reviews catch logic problems; machines are better at the repetitive ones. Without linting, every reviewer re-explains "use FQCN", "quote your modes", "this shell task needs `changed_when`", and some slip through to production (wrong permissions, non-idempotent tasks, deprecated syntax that breaks on the next ansible-core upgrade). A linter makes the team's standards automatic and consistent.

---

## Analogy

**A pre-flight checklist inspector.** Before the plane taxis, an inspector walks around with a checklist: tyres, flaps, fuel caps. They don't fly the plane (run the playbook), but they catch the loose fuel cap that would have caused trouble mid-flight. `--fix` is the inspector tightening the obvious screws themselves; `# noqa` is a signed note saying "known deviation, approved by X".

---

## How it works inside

```
 ansible-lint
   ├─ find files (git ls-files or the directory), classify them: playbook, tasks, vars, meta, requirements... ("kinds")
   ├─ (unless offline) install collections/roles from requirements files into a cache
   ├─ run ansible-playbook --syntax-check on playbooks (uses ansible-core itself)
   ├─ load each file as YAML (+ yamllint-style rules)
   ├─ walk tasks/plays; each rule inspects them (module name, arguments, keywords, Jinja2...)
   ├─ validate special files against JSON schemas (meta/main.yml, argument_specs, galaxy.yml...)
   └─ report findings (file:line: rule: message), filtered by profile/skip/warn/noqa
       exit code 0 = clean (or only warnings), 2 = violations
```

ansible-lint needs **ansible-core** installed (same venv) and benefits from the same collections your playbooks use, so modules can be resolved.

---

## Code / commands

### Install and run

```bash
cd ~/infra
source ~/.venvs/ansible/bin/activate
uv pip install ansible-lint yamllint
ansible-lint --version
```
```
ansible-lint 25.x.y using ansible-core:2.22.0 ansible-compat:25.x.y ruamel-yaml:0.18.x ruamel-yaml-clib:0.2.x
```

### Lint a deliberately bad playbook

`playbooks/lint-demo.yml`:

```yaml
- hosts: web
  become: yes
  tasks:
  - copy: src=app.conf dest=/opt/app/app.conf
  - name: restart nginx
    command: systemctl restart nginx
  - name: install tools
    package:
      name: tree
      state: latest
  - shell: cat /etc/os-release | grep ID
    register: osid
  - name: "show {{ osid.stdout }} id"
    debug: msg={{osid.stdout}}
```

```bash
ansible-lint playbooks/lint-demo.yml
```
```
WARNING  Listing 14 violation(s) that are fatal
playbooks/lint-demo.yml:1: name[play]: All plays should be named.
playbooks/lint-demo.yml:2: yaml[truthy]: Truthy value should be one of [false, true]
playbooks/lint-demo.yml:4: fqcn[action-core]: Use FQCN for builtin module actions (copy).
playbooks/lint-demo.yml:4: name[missing]: All tasks should be named.
playbooks/lint-demo.yml:4: no-free-form: Avoid using free-form when calling module actions. (copy)
playbooks/lint-demo.yml:4: risky-file-permissions: File permissions unset or incorrect.
playbooks/lint-demo.yml:5: command-instead-of-module: systemctl used in place of systemd module
playbooks/lint-demo.yml:5: fqcn[action-core]: Use FQCN for builtin module actions (command).
playbooks/lint-demo.yml:5: name[casing]: All names should start with an uppercase letter.
playbooks/lint-demo.yml:5: no-changed-when: Commands should not change things if nothing needs doing.
playbooks/lint-demo.yml:7: package-latest: Package installs should not use latest.
playbooks/lint-demo.yml:11: risky-shell-pipe: Shells that use pipes should set the pipefail option.
playbooks/lint-demo.yml:13: jinja[spacing]: Jinja2 spacing could be improved: msg={{osid.stdout}} -> msg={{ osid.stdout }}
playbooks/lint-demo.yml:13: name[template]: Jinja templates should only be at the end of 'name'
...
Read documentation for instructions on how to ignore specific rule violations.

                     Rule Violation Summary
 count tag                      profile    rule associated tags
     1 name[play]               basic      idiom
     1 yaml[truthy]             basic      formatting, yaml
     ...
Failed: 14 failure(s), 0 warning(s) on 1 files. Last profile that met the validation criteria was 'min'.
```

- Each line: **file:line: rule: message**.
- The summary shows which profile each rule belongs to and "Last profile that met the validation criteria": how far you are from `production`.

### Auto-fix, then fix the rest by hand

```bash
cp playbooks/lint-demo.yml /tmp/lint-demo.before.yml
ansible-lint --fix playbooks/lint-demo.yml
diff -u /tmp/lint-demo.before.yml playbooks/lint-demo.yml
```

`--fix` converts `copy:` → `ansible.builtin.copy:` with YAML arguments, `yes` → `true`, fixes spacing and casing where it can. You still fix **logic** by hand: a handler instead of `systemctl restart`, `present` instead of `latest`, a `mode`, `changed_when: false` and `pipefail` (or a better command).

The clean version:

```yaml
---
- name: Lint demo, fixed
  hosts: web
  become: true
  tasks:
    - name: App config is in place
      ansible.builtin.copy:
        src: app.conf
        dest: /opt/app/app.conf
        mode: "0644"
      notify: Restart nginx

    - name: Tools are installed
      ansible.builtin.package:
        name: tree
        state: present

    - name: Read the OS ID
      ansible.builtin.shell:
        cmd: set -o pipefail && grep '^ID=' /etc/os-release | cut -d= -f2
        executable: /bin/bash
      register: osid
      changed_when: false

    - name: Show the OS ID
      ansible.builtin.debug:
        msg: "OS ID is {{ osid.stdout }}"

  handlers:
    - name: Restart nginx
      ansible.builtin.systemd_service:
        name: nginx
        state: restarted
```

### Project config files

`~/infra/.ansible-lint`:

```yaml
---
profile: production
exclude_paths:
  - collections/ansible_collections/
  - dist/
  - terraform/
offline: true
```

`~/infra/.yamllint`:

```yaml
---
extends: default
rules:
  line-length:
    max: 160
    level: warning
  truthy:
    allowed-values: ["true", "false"]
    check-keys: false
  comments:
    min-spaces-from-content: 1
  comments-indentation: false
  braces:
    max-spaces-inside: 1
  octal-values:
    forbid-implicit-octal: true
    forbid-explicit-octal: true
ignore: |
  collections/ansible_collections/
  dist/
```

- `octal-values`: forbids unquoted modes like `0644` (topic 04's trap).
- `check-keys: false`: allows `on:` as a key (e.g. GitHub Actions files in the same repo).

### pre-commit

`~/infra/.pre-commit-config.yaml`:

```yaml
---
repos:
  - repo: https://github.com/adrienverge/yamllint
    rev: v1.37.1                      # pin a real release tag
    hooks:
      - id: yamllint
  - repo: https://github.com/ansible/ansible-lint
    rev: v25.11.0                     # pin a real release tag
    hooks:
      - id: ansible-lint
```

```bash
uv pip install pre-commit
pre-commit install                   # adds a git hook
pre-commit run --all-files           # run now on everything
```

Now `git commit` runs both linters; a failure blocks the commit (bypass with `--no-verify` only in emergencies, and CI will still catch it).

### Lint the whole project

```bash
ansible-lint
```

Fix findings until it reports `Passed: 0 failure(s), 0 warning(s)... Profile 'production' was required, and it passed.` Common real fixes in `~/infra`: missing `mode` on files, `changed_when` on read-only commands, role variable prefixes, FQCNs in old lab playbooks, names in casing.

---

## Hands-on lab

1. Install the linters, run `ansible-lint` on `lint-demo.yml`, read every finding.
2. Run `--fix`, inspect the diff, then fix the remaining findings by hand until it passes.
3. Add `.ansible-lint` and `.yamllint` to `~/infra` and lint the whole project at `profile: min`, then `basic`, `moderate`, `safety`, `production`. Fix each level before moving up.
4. Add one justified `# noqa` with a comment, and generate an ignore file for one legacy playbook: `ansible-lint --generate-ignore` (then look at `.ansible-lint-ignore`).
5. Set up pre-commit and make a commit with a deliberate short module name; watch the hook block it.
6. Install the VS Code Ansible extension (with WSL remote) and open a role: hover a module to see its docs, and watch lint findings appear on save.

**Done when:** `ansible-lint` passes at `production` on `~/infra` and runs automatically before each commit.

---

## Where you'll use it

- **Every PR** (CI job: `yamllint . && ansible-lint`), topics 53–54.
- **Upgrades**: new ansible-lint rules flag deprecated syntax before ansible-core removes it.
- **Shared content** (collections, Galaxy roles): `shared`/`production` profile; Automation Hub certification requires clean lint.
- **Onboarding**: the linter teaches team conventions automatically.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `couldn't resolve module/action` during lint | Collections not available to ansible-lint | Install `requirements.yml` in the same environment (or let lint install them, `offline: false`) |
| Lint scans installed third-party collections | Not excluded | `exclude_paths: [collections/ansible_collections/]` |
| `--fix` reformatted lots of files | It re-dumps YAML | Review diffs; run once on a branch; limit with `--fix=fqcn` |
| Different results locally and in CI | Different ansible-lint/ansible-core versions or configs | Pin versions in `requirements.txt` and pre-commit `rev` |
| `load-failure` / `internal-error` | A file that isn't Ansible YAML (e.g. Helm templates, Jinja YAML) is scanned | Exclude it or set its `kind` |
| `name[template]` everywhere | Templates in the middle of names | Move `{{ }}` to the end, or rephrase |
| Team skips rules wholesale to "get green" | `skip_list` growing | Review skips; prefer `# noqa` with reasons and fix debt over time |
| `schema[meta]` on `meta/main.yml` | Invalid platform names/versions or keys | Use valid Galaxy platform names (e.g. `EL`, `Ubuntu`) and fields |

---

## Quick recap

- **yamllint** = YAML style/syntax; **ansible-lint** = YAML + Ansible best practices + syntax check + schemas.
- Profiles `min` → `basic` → `moderate` → `safety` → `shared` → **`production`**; aim for production.
- Common rules: FQCN, names, `no-changed-when`, `command-instead-of-module`, `risky-file-permissions`, `package-latest`, `yaml[truthy]`, `jinja[spacing]`, role var prefixes.
- `--fix` for mechanical fixes (review the diff); `# noqa: rule` with a reason; `.ansible-lint-ignore` for legacy debt.
- Automate: pre-commit locally, the same commands in CI; ansible-dev-tools and the VS Code extension for the editor.

### Interview questions

1. **Q: What does ansible-lint check?**
   A: YAML formatting, syntax (via ansible-core), schemas for special files, and best-practice rules such as FQCN use, named tasks, idempotency smells (`command` without `changed_when`), risky permissions, `latest` packages and deprecated syntax, grouped into profiles.
2. **Q: How do you ignore a lint rule for one task?**
   A: Add `# noqa: rule-name` on that line, ideally with a short justification; use `.ansible-lint-ignore` for files and `skip_list` only for team-wide decisions.
3. **Q: How do you enforce linting in a team?**
   A: Commit `.ansible-lint`/`.yamllint` and pinned tool versions, run them in pre-commit hooks locally and as a required CI check on pull requests.

### Scenario question (answer out loud first, then compare)

> *"You inherit a 5-year-old Ansible repo. ansible-lint reports 2,300 violations at the production profile. Leadership wants lint in CI 'this month'. How do you get there without a giant risky refactor?"*

**Model answer:**
1. **Baseline**: pin ansible-lint and ansible-core; run at `min` and fix the **fatal** findings (syntax, load failures) first, since they're real bugs.
2. **Freeze the debt**: generate `.ansible-lint-ignore` for existing findings at the target profile, so CI fails only on **new** violations from day one.
3. **Mechanical fixes in small PRs**: `--fix=fqcn`, then `yaml`, one role at a time, each with a check-mode run against staging to confirm no behaviour change.
4. **Logic fixes by risk**: `no-changed-when`, `risky-file-permissions`, `package-latest`, `ignore-errors` in the most-used roles first, with Molecule tests (topic 51).
5. **Ratchet**: shrink the ignore file every sprint; raise the profile when a level is clean.

**Likely follow-ups:**
- *Why not just `skip_list` everything?* It hides new problems too; the ignore file only covers existing ones.
- *Risk of `--fix`?* It reformats YAML and changes module names; review diffs and test, but behaviour shouldn't change.
