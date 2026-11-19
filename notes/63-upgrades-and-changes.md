# 63 · Upgrades and What's Changed

> **Level:** ★★ intermediate
>
> **You will learn:** how ansible-core is released and supported (two releases a year, ~18 months of support, Python requirements per version), a **safe upgrade process** (porting guides, a branch, lint, Molecule, check mode, canary, new EE, rollback), the **important changes in ansible-core 2.19–2.22** (the 2.19 templating overhaul, 2.20's Python and fact-variable changes, 2.21's failure-inference deprecation, 2.22's secret masking), and a **table of deprecated patterns** with their modern replacements. You'll take an old-style playbook through an upgrade from 2.18 to 2.22.
>
> **You need:** [62 · Project 3](62-project-terraform-ansible-aws.md), [08 · pinning](08-installing-ansible.md), [07 · target Python](07-python-on-managed-nodes.md), [22](22-jinja2-basics.md)/[26](26-register-and-when.md) (2.19 rules), [52 · EEs](52-execution-environments.md), [51 · Molecule](51-testing-with-molecule.md).
>
> **Part 12 · Production operations** · Next: [64 · Production checklist](64-production-checklist.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** ansible-core ships a new version every ~6 months, and each is supported for ~18 months. New versions remove deprecated features and sometimes change behaviour (2.19 changed templating a lot). Upgrade **deliberately**: on a branch, with the porting guide open, linted, tested, check-mode-run against staging, then rolled out via a new pinned version or EE tag.

**Picture it.** Upgrading the engine of a plane: read the manufacturer's change notes, test on the ground, fly an empty test flight (staging/canary), then put passengers on it, and keep the old engine available for a while.

**The one thing to know.**

```
 1. read the porting guides (each version between yours and the target)
 2. branch: bump requirements.txt / EE (ansible-core, collections, ansible-lint)
 3. ansible-lint + --syntax-check on everything
 4. Molecule for every role
 5. --check --diff against staging, then a real staging run
 6. canary in prod, then everyone; keep the old EE/venv for rollback
```

**Try it now:** `ansible --version | head -1` on every machine/CI image you have. Are they all the same and still supported?

**You've got it if you can:** name two breaking changes from 2.19 and the fix for each.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Release and support model

- A new ansible-core **minor** release about every 6 months (around May and November).
- The latest release plus the two previous ones are maintained: roughly **18 months** of bug and security fixes per version.
- Patch releases (2.22.1, 2.22.2…) contain fixes only.
- Each release declares supported **control node** and **target** Python versions.

| ansible-core | GA | End of life | Control node Python | Target Python | Community `ansible` package |
|---|---|---|---|---|---|
| 2.19 | Jul 2025 | Nov 2026 | 3.11–3.13 | 3.8–3.13 | 12 |
| 2.20 | Nov 2025 | May 2027 | 3.12–3.14 | 3.9–3.14 | 13 |
| 2.21 | May 2026 | Nov 2027 | 3.12–3.14 | 3.9–3.14 (PowerShell 5.1–7) | 14 |
| **2.22** | **Nov 2026** | **May 2028** | **3.13–3.15** | **3.9–3.15** (PowerShell 5.1–7) | **15** |

(From the official release-and-maintenance page, checked November 2026. Always confirm the current table before planning.)

### The safe upgrade process

| Step | What | Why |
|---|---|---|
| 1 | Read **every porting guide** between current and target (2.18 → 2.22 = four guides), plus collection changelogs | Know what breaks before it breaks |
| 2 | Check **Python** on control nodes and **targets** | e.g. 2.22 needs 3.13+ locally; targets older than 3.9 stop working |
| 3 | Branch: bump `ansible-core`, `ansible-lint`, collections; rebuild the **EE** with a new tag | Isolated change, easy rollback |
| 4 | `ansible-lint` (new rules flag deprecated syntax) + `--syntax-check` on all playbooks | Cheap, catches most issues |
| 5 | **Molecule** for every role on every supported OS | Real execution |
| 6 | Watch for `[DEPRECATION WARNING]` in output; fix them now | They become errors in the next versions |
| 7 | `--check --diff` against staging, then a real staging run; compare recaps with the old version (`changed=0` expected) | Behaviour unchanged? |
| 8 | Production: canary group first, then everyone; old EE/venv kept for rollback | Controlled risk |
| 9 | Update docs/CI images/AWX EE assignments | No stragglers on the old version |

### What changed: ansible-core 2.19 (templating overhaul, "data tagging")

The biggest behavioural change in years:

| Change | Old behaviour | New behaviour | Fix |
|---|---|---|---|
| **Conditionals must be booleans** | `when: my_string` was "truthy" | Error: *Conditionals must have a boolean result* | `\| bool`, `\| length > 0`, comparisons (topic 26). Temporary: `ALLOW_BROKEN_CONDITIONALS` |
| **No multi-pass / nested templating** | `"{{ foo_{{ env }} }}"`, `when: "{{ x }} == 1"` worked | Syntax error | `lookup('ansible.builtin.vars', 'foo_' ~ env)`, plain expressions in `when` (topic 22) |
| **Untrusted templates not rendered** | Templates inside module results/fetched data were rendered | Treated as plain text (safer) | Don't rely on rendering data from targets |
| **Native types always** | Jinja2 "native" was optional | Always native; a lone `"{{ x }}"` keeps its type | Check places that expected strings |
| Loop items that resolve to `omit` | Kept | Dropped | `\| default(omit)` behaviour changed accordingly |
| `range()` | List-like | A range object | `\| list` |
| Undefined handling | Some tests/filters silently passed | Undefined values surface as errors | Define/guard variables |
| Become timeout | Task failure | **Unreachable** | `ignore_unreachable` instead of `ignore_errors` if you must continue |
| `bool` filter on unrecognised values | Coerced | Deprecated; becomes an **error in 2.23** | Pass real booleans/recognised strings |
| SSH password prompts | `sshpass` | `SSH_ASKPASS` mechanism by default | Set `password_mechanism = sshpass` to revert (prefer keys) |
| Error output | | `ANSIBLE_DISPLAY_TRACEBACK` added | Use for debugging (topic 58) |

### ansible-core 2.20

| Change | Fix |
|---|---|
| Control node needs Python **3.12+**; targets need **3.9+** (3.8 dropped) | Upgrade Python / OS (topics 07, 08) |
| **`INJECT_FACTS_AS_VARS` deprecated** (default flips to `False` in 2.24): `ansible_distribution`-style variables go away | Use `ansible_facts['distribution']` (topic 23) |
| Removed: `smart` transport, `vaultid` for vault filters, Galaxy **v2** API, `dnf install_repoquery`, some paramiko settings, `yum_repository keepcache` | Update configs/servers/tasks |
| `include_vars`: `extensions` and `ignore_files` must be lists | Use lists |
| The internal `vars` cache (e.g. `vars['x']`) deprecated (removal 2.24) | `lookup('ansible.builtin.vars', ...)` / `varnames` |
| `replace` module works on text with unicode regex | Check byte-oriented regexes |

### ansible-core 2.21

| Change | Fix |
|---|---|
| **Failure inference from non-zero `rc` deprecated** for modules that don't set `failed` (warnings from 2.22; removal planned) | Custom modules: always `fail_json` explicitly (topic 38) |
| `password_hash` uses libcrypt/libxcrypt instead of passlib (hashes differ for the same input, still valid) | Expect one-time `changed` on users with fixed-salt hashes; use `update_password: on_create` |
| PowerShell 7 supported on Windows targets | Optional |

### ansible-core 2.22

| Change | Impact |
|---|---|
| **Secret masking**: decrypted vault values, `no_log` option values and prompted passwords are replaced with **`$REDACTED$`** everywhere (output, logs, callbacks) | Safer logs; `debug` can't show secrets any more |
| New filters `register_secret`, `mask_secrets`; module_utils `ansible.module_utils.secrets` | Mark lookup values as secrets (topic 41) |
| Plugin options can be declared `secret: true` (SSH password/private key, become passwords...) | Automatic masking |
| `no_log` values are no longer replaced by `VALUE_SPECIFIED_IN_NO_LOG_PARAMETER` | Update code that compared against that placeholder |
| Very short secrets can mask parts of unrelated words | Avoid trivial secret values (good practice anyway) |
| Control node Python **3.13+**; targets 3.9–3.15 | WSL2 Ubuntu 24.04 needs uv/another Python (topic 08) |
| Deprecations: `heuristic_log_sanitize`, `remove_values`, `sanitize_keys` (removal 2.25) | Custom modules/callbacks |

### Deprecated and removed patterns → modern replacements

| ⚠️ Old | ✅ Modern | Topic |
|---|---|---|
| `include:` (removed 2.16) | `import_tasks` / `include_tasks` / `import_playbook` | 34 |
| `sudo:` / `sudo_user:` (removed long ago) | `become:` / `become_user:` | 19 |
| Short module names (`copy:`) | FQCN (`ansible.builtin.copy:`) | 16 |
| `k=v` strings in playbooks | YAML arguments | 04 |
| `with_items`, `with_dict`, ... | `loop` (+ filters, `query`) | 28 |
| `ansible_distribution` etc. (injected facts) | `ansible_facts['distribution']` | 23 |
| `yum` (redirected to dnf since 2.17) | `ansible.builtin.dnf` | 17 |
| `apt_key` | `deb822_repository` with `signed_by` | 17 |
| `systemd` (alias) | `systemd_service` | 17 |
| `local_action:` | `delegate_to: localhost` | 46 |
| Nested `{{ }}` / `when: "{{ x }}"` | Single expressions; `lookup('ansible.builtin.vars', ...)` | 22 |
| Non-boolean `when` | Explicit booleans | 26 |
| `hash_behaviour = merge` | `combine` filter | 25 |
| `ec2.py` and other inventory scripts | Inventory plugins (`aws_ec2`...) | 42 |
| `community.general.docker_*`, `community.general.json_query` short names | `community.docker.*`, FQCN filters | 16, 32 |
| `ansible.utils.ipaddr` as short `ipaddr` | FQCN + `netaddr` installed | 32 |
| `paramiko` connection for normal Linux hosts | OpenSSH (`ssh`) | 11 |
| `ansible_ssh_pass` / `ansible_sudo_pass` | `ansible_password` (avoid) / `ansible_become_password` (vaulted) | 11, 19 |
| "Ansible Tower" | Automation Controller / AAP | 56 |
| "Ansible 2.9" thinking (all modules in one package) | ansible-core + collections | 03, 37 |

### Collections upgrades

- Collections follow **semantic versioning**: a **major** bump may remove modules/options.
- Read their **changelogs** (`changelogs/changelog.yaml` in the collection, or the project page).
- Watch for `requires_ansible` in `meta/runtime.yml`: new collection versions may require newer ansible-core.
- Upgrade collections in the same controlled way (branch, lint, Molecule, staging).

---

## Why it exists

Staying on an old ansible-core means no security fixes, no support for new OS versions, and a bigger jump later. Upgrading carelessly means broken production runs (2.19's conditional change broke a lot of real playbooks). A routine, tested upgrade process every 6–12 months keeps the jumps small and safe.

---

## Analogy

**Keeping your car serviced vs restoring a 20-year-old wreck.** Small, regular services (one minor version at a time, with the manual's notes) are cheap and predictable. Skipping ten years of services means a long, risky restoration where everything breaks at once.

---

## How it works inside

```
 old toolchain (EE infra-ee:1.4, core 2.18)          new toolchain (EE infra-ee:2.0, core 2.22)
        │                                                    │
        │  production keeps using 1.4 ◀──── rollback ──────┤
        │                                                    ├─ lint / syntax-check
        │                                                    ├─ Molecule (all OSes)
        │                                                    ├─ check + real run on staging
        │                                                    ├─ canary in prod
        └──────────────────────── switch job templates / CI image to 2.0
```

Because the toolchain is pinned (requirements files, EE tags), "upgrade" and "rollback" are just changing a version number.

---

## Code / commands

### Lab: upgrade an old-style playbook from 2.18 to 2.22

**1. An old venv** (2.18 runs on Python 3.12, which Ubuntu 24.04 has):

```bash
uv venv --python 3.12 ~/.venvs/ansible-218
uv pip install --python ~/.venvs/ansible-218 "ansible-core==2.18.*"
~/.venvs/ansible-218/bin/ansible --version | head -1
```

**2. An old-style playbook** `~/infra/playbooks/legacy.yml`:

```yaml
- hosts: web
  vars:
    port_dev: 8080
    env: dev
    feature_flag: "yes"
    names: [a, b]
  tasks:
  - name: nested template (old trick)
    debug: msg="{{ port_{{ env }} }}"
    ignore_errors: yes
  - name: truthy string conditional
    debug: msg="feature on"
    when: feature_flag
  - name: injected fact variable
    debug: msg="{{ ansible_distribution }}"
  - name: old loop
    debug: msg="{{ item }}"
    with_items: "{{ names }}"
  - name: range without list
    debug: msg="{{ item }}"
    loop: "{{ range(2) }}"
```

**3. Run with 2.18:**

```bash
cd ~/infra
~/.venvs/ansible-218/bin/ansible-playbook -i inventories/lab playbooks/legacy.yml -l web1
```

Mostly works (with deprecation warnings; the nested template may already error or warn depending on the exact version).

**4. Run with 2.22:**

```bash
ansible-playbook -i inventories/lab playbooks/legacy.yml -l web1
```
```
TASK [nested template (old trick)] ******************************************************
fatal: [web1]: FAILED! => {"msg": "Syntax error in template: unexpected '{' ..."}
...ignoring
TASK [truthy string conditional] ********************************************************
fatal: [web1]: FAILED! => {"msg": "Conditional result (True) was derived from value of type 'str' ... Conditionals must have a boolean result."}
```

**5. Lint it with the new ansible-lint**:

```bash
ansible-lint playbooks/legacy.yml
```

It lists FQCN, `yaml[truthy]`, `no-free-form`, `deprecated-...`, and more.

**6. The upgraded version** `playbooks/legacy-fixed.yml`:

```yaml
---
- name: Legacy playbook, upgraded
  hosts: web
  vars:
    port_dev: 8080
    env: dev
    feature_flag: true
    names: [a, b]
  tasks:
    - name: Build the variable name in one expression
      ansible.builtin.debug:
        msg: "{{ lookup('ansible.builtin.vars', 'port_' ~ env) }}"

    - name: Real boolean conditional
      ansible.builtin.debug:
        msg: feature on
      when: feature_flag | bool

    - name: Facts through ansible_facts
      ansible.builtin.debug:
        msg: "{{ ansible_facts['distribution'] }}"

    - name: Modern loop
      ansible.builtin.debug:
        msg: "{{ item }}"
      loop: "{{ names }}"

    - name: Range as a list
      ansible.builtin.debug:
        msg: "{{ item }}"
      loop: "{{ range(2) | list }}"
```

```bash
ansible-playbook -i inventories/lab playbooks/legacy-fixed.yml -l web1     # clean
ansible-lint playbooks/legacy-fixed.yml                                     # passes
```

### Find old patterns across a repo

```bash
grep -rnE '^\s*-?\s*include:' --include='*.yml' .                    # removed include
grep -rnE 'with_(items|dict|nested|subelements|fileglob|sequence)' --include='*.yml' .
grep -rnE 'ansible_(distribution|os_family|default_ipv4|hostname|memtotal_mb|architecture)\b' --include='*.yml' --include='*.j2' .
grep -rnE '\{\{[^}]*\{\{' --include='*.yml' --include='*.j2' .        # nested templates
grep -rnE '^\s*when:\s*"?\{\{' --include='*.yml' .                    # templated when
grep -rnE '^\s*(sudo|sudo_user):' --include='*.yml' .
```

### Watch deprecation warnings in CI

Make CI fail (or at least report) when new deprecation warnings appear:

```bash
ansible-playbook -i inventories/lab playbooks/site.yml --check 2>&1 | tee run.log
grep -c 'DEPRECATION WARNING' run.log
```

---

## Hands-on lab

1. Do the 2.18 → 2.22 lab: run the legacy playbook on both, read every error, fix it into `legacy-fixed.yml`.
2. Run the repo-wide grep commands on `~/infra` (and any real repo you have access to). Count findings per pattern.
3. Build `infra-ee:2.0` with a newer collection set (topic 52) and run `site.yml` with both EE tags; compare the recaps.
4. Write an **upgrade checklist** for your team from the process table (who reads the porting guide, which tests must pass, how to roll back).
5. Check every CI image/runner/AWX EE you use: which ansible-core is it, and is it still supported?

**Done when:** you can plan and execute an ansible-core upgrade with a test path and a rollback.

---

## Where you'll use it

- **Twice a year** (or at least yearly) in every team using Ansible.
- **Inheriting legacy code**: the deprecated-pattern table is your refactoring plan.
- **AAP upgrades**: each AAP release ships newer ansible-core in its EEs; the same testing applies.
- **Interviews**: "What changed in recent Ansible versions?" (2.19 templating, FQCN/collections, `ansible_facts`, secret masking.)

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| Many `Conditionals must have a boolean result` | 2.19 rules | Explicit booleans |
| `template error ... unexpected '{'` | Nested templating | Single expressions, `vars` lookup |
| `ansible_distribution is undefined` (2.24+) | Injected facts disabled | `ansible_facts['...']` |
| `ansible-core requires Python >=3.13` when installing | Old control-node Python | uv/new image (topic 08) |
| Targets fail with `SyntaxError` after the upgrade | Target Python too old | OS upgrade or separate toolchain |
| `password` task `changed` once after 2.21 | New hashing backend | Expected once; `on_create` |
| Galaxy installs fail | v2 API removed (2.20) | v3-capable server |
| Code comparing against `VALUE_SPECIFIED_IN_NO_LOG_PARAMETER` breaks | 2.22 change | Remove that comparison |
| Different behaviour in CI vs laptop after upgrading one of them | Unpinned toolchain | Pin and upgrade everywhere together |

---

## Quick recap

- ansible-core: ~2 releases/year, ~18 months support; check control/target Python per version (2.22: 3.13+ control, 3.9–3.15 targets).
- Process: porting guides → branch + pins + new EE → lint/syntax → Molecule → staging check/run → canary → rollout; keep rollback.
- 2.19: boolean conditionals, no nested templating, untrusted templates, native types, `range | list`, become timeout = unreachable.
- 2.20: Python 3.12+/targets 3.9+, `ansible_facts[...]` (injected facts deprecated), Galaxy v3 only, `vars` cache deprecated.
- 2.21: failure inference from `rc` deprecated, libcrypt hashing. 2.22: **secret masking** (`$REDACTED$`, `register_secret`).
- Replace deprecated patterns with their modern equivalents (table above).

### Interview questions

1. **Q: How do you upgrade Ansible safely in a team?**
   A: Read the porting guides for every version jump, bump pinned versions (and the EE) on a branch, run ansible-lint, syntax checks and Molecule, do check-mode and real runs on staging, canary in production, then roll out while keeping the previous toolchain for rollback.
2. **Q: What was the biggest change in ansible-core 2.19?**
   A: The templating overhaul (data tagging): conditionals must be booleans, nested/multi-pass templating removed, untrusted data isn't templated, native types are always used.
3. **Q: Why use `ansible_facts['distribution']` instead of `ansible_distribution`?**
   A: Injected top-level fact variables are deprecated (2.20) and will be disabled by default in 2.24; `ansible_facts` is the stable interface and avoids name clashes.

### Scenario question (answer out loud first, then compare)

> *"Your company runs ansible-core 2.15 everywhere (out of support), 300 roles, RHEL 8/9 and Ubuntu 20.04/22.04 targets. Plan the move to 2.22."*

**Model answer:**
1. **Inventory the blockers**: targets with Python < 3.9 (RHEL 8's 3.6, Ubuntu 20.04's 3.8) can't be managed by 2.22. Plan OS upgrades; meanwhile keep a **legacy toolchain** (2.16 EE) for those hosts only.
2. **Control side**: new EE with Python 3.13+, ansible-core 2.22, current collections.
3. **Code**: run ansible-lint (production profile) with the new version and the grep checks; fix FQCN, `include:`, nested templates, non-boolean conditionals, injected facts, `with_*` where needed, `yum`/`apt_key` modules. Prioritise most-used roles.
4. **Tests**: Molecule scenarios per OS for the top roles; check mode on staging for every playbook; compare recaps old vs new.
5. **Rollout**: by team/inventory, canary first; AWX job templates switched to the new EE one by one; the legacy EE stays only for not-yet-upgraded OSes.
6. **Cadence**: afterwards, upgrade every release (or every other) so the gap never gets this big again.

**Likely follow-ups:**
- *Why not go straight to 2.22 for RHEL 8?* Target Python 3.6 isn't supported; modules won't run.
- *How long would you keep the legacy toolchain?* Until the last old-OS host is upgraded, with a deadline and tracking.
