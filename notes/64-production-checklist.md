# 64 · Production Checklist

> **Level:** ★★ intermediate
>
> **You will learn:** one checklist covering everything to verify **before** Ansible touches production, and to keep true **while** you operate it: repository and code quality, inventory and variables, secrets, identity and access, toolchain, testing, the change process (before, during and after a run), operations (drift, monitoring, upgrades, recovery), and documentation. Each item links to the topic that explains it. You'll turn the "before a run" part into an automated **preflight** script and score `~/infra` against the list.
>
> **You need:** [63 · Upgrades](63-upgrades-and-changes.md) and, really, the whole course; this page is the summary you'll actually reuse.
>
> **Part 12 · Production operations** · Next: [65 · Production scenarios](65-production-scenarios.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Production incidents caused by automation almost always come from a short list of mistakes: wrong environment, untested change, non-idempotent task, missing validation, leaked secret, everything changed at once, no way back. A checklist makes those impossible to forget.

**Picture it.** A pilot's pre-flight checklist: experienced pilots still use it every time, because skipping one line is how accidents happen.

**The one thing to know.** The ten lines that prevent most incidents:

| ☐ | Check |
|---|---|
| ☐ | Right inventory/environment (`--list-hosts`, environment guard) |
| ☐ | Change reviewed in a PR; lint + Molecule green |
| ☐ | `--check --diff` read, and the diff is what you expect |
| ☐ | Idempotent (second run `changed=0`) |
| ☐ | Configs validated before replace (`validate:`), restarts via handlers |
| ☐ | Batches (`serial`) + stop on failure (`max_fail_percentage`) |
| ☐ | Health checks after changes; rollback path known |
| ☐ | Secrets vaulted/managed; `no_log`; nothing secret in logs |
| ☐ | Pinned toolchain (core, collections, EE tag) |
| ☐ | Run from CI/AWX with approvals, not a laptop |

**Try it now:** go through the ten lines for the last playbook you ran in this course.

**You've got it if you can:** explain why each of the ten lines exists, from memory.

> 📖 **That's the core.** Everything below is the **full checklist**.

---

## What it is

### 1. Repository and code

| ☐ | Item | Topic |
|---|---|---|
| ☐ | Standard layout: `inventories/<env>/`, `playbooks/`, `roles/` (or collections), committed `ansible.cfg`, README | 39 |
| ☐ | FQCN everywhere; no deprecated syntax (`include:`, short names, `with_*` where avoidable, injected facts) | 16, 63 |
| ☐ | Every task named as a desired state | 14 |
| ☐ | Modules instead of `command`/`shell`; any remaining commands have `creates`/`removes`/`changed_when`/`failed_when` | 27 |
| ☐ | No `ignore_errors` used to hide problems; precise `failed_when` or `rescue` | 31 |
| ☐ | Restarts/reloads only via handlers; validate-then-restart for critical services | 30 |
| ☐ | `validate:` on sudoers, sshd, nginx, and any config that can lock you out or break a service | 18, 29 |
| ☐ | Explicit, quoted `mode` on every file/directory | 04, 18 |
| ☐ | No run-time values (dates, random) in templates; sorted data for stable output | 27, 29 |
| ☐ | Roles: one job, prefixed defaults, OS vars via `first_found`, `argument_specs`, README | 35, 36 |
| ☐ | `ansible-lint` at `production` profile passes; `yamllint` passes | 50 |
| ☐ | pre-commit hooks (lint, secret scanning) | 50, 60 |

### 2. Inventory and variables

| ☐ | Item | Topic |
|---|---|---|
| ☐ | One inventory directory per environment; no default inventory pointing at prod | 10, 39 |
| ☐ | Values in `group_vars`/`host_vars` (not inline INI); no overlapping variables across sibling groups | 24 |
| ☐ | Dynamic inventory for cloud (filters by env tag, `keyed_groups`, `strict: true`) | 42 |
| ☐ | `env_name` variable per inventory + an environment guard (`assert`) in production playbooks | 31, 60 |
| ☐ | Variable precedence understood for critical values (no surprise role `vars/` overrides) | 25 |
| ☐ | Facts accessed via `ansible_facts['...']` | 23 |
| ☐ | Required inputs validated (`argument_specs`, `assert`, `mandatory`) | 22, 31, 36 |

### 3. Secrets

| ☐ | Item | Topic |
|---|---|---|
| ☐ | No plaintext secrets in Git (scanner in CI/pre-commit) | 40, 60 |
| ☐ | Vault IDs per environment, or a secret manager with per-identity access | 40, 41 |
| ☐ | Prod vault password / secret-manager access only in CI/AWX | 40, 53, 56 |
| ☐ | `no_log: true` on secret-handling tasks; `register_secret` for looked-up values; `diff: false` on secret files | 40, 41 |
| ☐ | Secrets never on command lines (use module options, `environment:`, `0600` files) | 41 |
| ☐ | No `-vvv` in CI for production runs; logs protected and retained appropriately | 58, 60 |
| ☐ | Rotation procedure documented (and tested) | 40, 60 |

### 4. Identity and access

| ☐ | Item | Topic |
|---|---|---|
| ☐ | Dedicated automation user; key-only; ideally short-lived SSH certificates | 19, 60 |
| ☐ | Automation SSH restricted to runners/execution nodes/bastions (`from=`, firewalls) | 60 |
| ☐ | Separate credentials per environment | 60 |
| ☐ | Host key checking on; trusted `known_hosts` or host certificates | 05, 60 |
| ☐ | Protected main branch, CODEOWNERS, required reviews | 53, 60 |
| ☐ | Production applies only via CI environments with approval or AWX RBAC | 53, 54, 56 |
| ☐ | Humans use personal accounts (SSO/MFA), not the automation identity | 60 |

### 5. Toolchain

| ☐ | Item | Topic |
|---|---|---|
| ☐ | ansible-core pinned (`requirements.txt`); within its support window | 08, 63 |
| ☐ | Collections pinned (`requirements.yml`), installed project-locally or in the EE | 37 |
| ☐ | Execution environment built from those pins, tagged, scanned, used by CI/AWX | 52 |
| ☐ | Target OS Python versions supported by this ansible-core | 07, 63 |
| ☐ | Same toolchain on laptops (navigator) and pipelines | 52 |

### 6. Testing

| ☐ | Item | Topic |
|---|---|---|
| ☐ | Molecule scenarios for every role, on every supported OS, including idempotence | 51 |
| ☐ | `--syntax-check` and lint on every PR | 50, 53 |
| ☐ | Check-mode plan against staging on every PR | 20, 53 |
| ☐ | Real run on staging before production | 53 |
| ☐ | End-to-end verification plays (service answers, dependencies reachable) | 57 |

### 7. The change: before the run

| ☐ | Item | Topic |
|---|---|---|
| ☐ | `--list-hosts` shows exactly the intended hosts | 10, 20 |
| ☐ | `--check --diff` reviewed; unexpected diffs explained | 20 |
| ☐ | Change ticket/PR links the plan; approver ≠ author | 53, 60 |
| ☐ | Maintenance window / stakeholders informed if user-visible | 65 |
| ☐ | Rollback plan written (previous release, revert commit, restore backup) | 49 |
| ☐ | Backups/snapshots for stateful changes (DB, disks) | 44, 65 |

### 8. The change: during the run

| ☐ | Item | Topic |
|---|---|---|
| ☐ | `serial` batches (canary first) and `max_fail_percentage` | 45 |
| ☐ | LB drain/enable around service-affecting changes | 46, 49 |
| ☐ | Health checks inside each batch; `block`/`rescue` rollback | 31, 49 |
| ☐ | `force_handlers` for plays where lost restarts would be dangerous | 30 |
| ☐ | Concurrency control (one apply per environment) | 53, 54 |
| ☐ | Long tasks with `async`; time limits on the job | 47 |

### 9. The change: after the run

| ☐ | Item | Topic |
|---|---|---|
| ☐ | Recap reviewed: no unexpected `changed`, no `failed`/`unreachable` left unexplained | 14 |
| ☐ | Second run (or check run) shows `changed=0` | 27 |
| ☐ | Monitoring/smoke tests green | 49 |
| ☐ | Unreachable/failed hosts tracked and re-run | 61 |
| ☐ | Logs/artifacts stored (plan, results) | 54, 55 |

### 10. Operations

| ☐ | Item | Topic |
|---|---|---|
| ☐ | Scheduled drift detection with alerts | 55 |
| ☐ | Run times measured (`profile_tasks`) and acceptable at current scale | 48, 61 |
| ☐ | Upgrade cadence for ansible-core/collections/EEs | 63 |
| ☐ | Access reviews and key/cert rotation scheduled | 60 |
| ☐ | Runbooks for common failures (error catalogue) | 59 |
| ☐ | Disaster recovery: can you rebuild an environment from Git + Terraform + Ansible? | 43, 57 |

### 11. Documentation

| ☐ | Item |
|---|---|
| ☐ | README: prerequisites, setup (`make deps`), how to run per environment, where secrets live, how to add a host/role |
| ☐ | Role READMEs and `argument_specs` |
| ☐ | Ownership (CODEOWNERS) and escalation contacts |
| ☐ | Exceptions (host_vars overrides, `# noqa`) explained in comments/PRs |

---

## Why it exists

Under time pressure, people skip steps. Checklists capture what experienced engineers learned from incidents and make it routine for everyone. They also make reviews faster ("which items does this change touch?") and audits easier.

---

## Analogy

**Surgical safety checklist.** Before the first cut: right patient, right side, instruments counted, allergies known. It takes two minutes and prevents the rare but catastrophic mistakes, even for the best surgeons.

---

## How it works inside

Most "before the run" items can be **automated** and enforced by the pipeline, so humans focus on judgement (reading the diff, approving):

```
 PR ──▶ lint, syntax, Molecule, secret scan, check-mode plan  (automated gates)
     ──▶ human review of the plan (judgement)
 merge ──▶ preflight script (env guard, list-hosts, pinned versions) ──▶ approval ──▶ apply in batches
```

---

## Code / commands

### An automated preflight

`~/infra/scripts/preflight.sh`:

```bash
#!/usr/bin/env bash
# Usage: scripts/preflight.sh <inventory-dir> <playbook> [expected-host-count]
set -euo pipefail
inv="$1"; pb="$2"; expected="${3:-}"
fail() { echo "❌ $*" >&2; exit 1; }
ok()   { echo "✅ $*"; }

# 1. Toolchain is the pinned one
want=$(grep -E '^ansible-core==' requirements.txt | cut -d= -f3)
have=$(ansible --version | head -1 | sed -E 's/.*core ([0-9.]+).*/\1/')
[[ "$have" == "$want"* ]] || fail "ansible-core $have, expected $want"
ok "ansible-core $have"

# 2. Lint and syntax
ansible-lint -q "$pb" >/dev/null || fail "ansible-lint failed"
ansible-playbook -i "$inv" "$pb" --syntax-check >/dev/null || fail "syntax check failed"
ok "lint + syntax"

# 3. Targets
hosts=$(ansible-playbook -i "$inv" "$pb" --list-hosts | awk '/^ {6}[^ ]/ {print $1}' | sort -u | wc -l)   # unique hosts across all plays
[[ -z "$expected" || "$hosts" -eq "$expected" ]] || fail "targets $hosts hosts, expected $expected"
ok "targets $hosts hosts"

# 4. No plaintext vault files
for f in $(git ls-files "$inv" | grep -E 'vault.*\.ya?ml$' || true); do
  head -1 "$f" | grep -q '^\$ANSIBLE_VAULT' || fail "unencrypted vault file: $f"
done
ok "vault files encrypted"

# 5. Host key checking not disabled
! ansible-config dump | grep -q 'HOST_KEY_CHECKING(.*) = False' || fail "host key checking disabled"
ok "host key checking on"

echo "Preflight passed. Next: --check --diff, review, then apply in batches."
```

```bash
chmod +x scripts/preflight.sh
scripts/preflight.sh inventories/lab playbooks/site.yml 5
```
```
✅ ansible-core 2.22.0
✅ lint + syntax
✅ targets 5 hosts
✅ vault files encrypted
✅ host key checking on
Preflight passed. Next: --check --diff, review, then apply in batches.
```

Add a `make preflight` target and run it in CI before every apply job.

---

## Hands-on lab

1. Score `~/infra` against every section; list the unchecked items.
2. Create `scripts/preflight.sh`, run it, and make it fail on purpose (wrong host count, a decrypted vault file, `ANSIBLE_HOST_KEY_CHECKING=False`).
3. Add it as a step in your GitHub Actions apply jobs (topic 53) or Jenkins pipeline (topic 54).
4. Fix the three most important unchecked items from step 1.

**Done when:** your preflight runs automatically before every apply, and the remaining unchecked items have owners and dates.

---

## Where you'll use it

- **Every production change** (the "before/during/after" sections).
- **Quarterly reviews** of the automation platform (sections 1–6, 10–11).
- **Audits** and **onboarding** (shows what "good" looks like).

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| Checklist ignored under pressure | Manual and long | Automate gates in CI; keep the human part short |
| Checklist becomes stale | New lessons not added | Add an item after every incident's post-mortem |
| Everything "checked" but incidents still happen | Box-ticking without understanding | Pair reviews; scenario drills (topic 65) |

---

## Quick recap

- Ten lines prevent most incidents: right environment, reviewed + tested, read the diff, idempotent, validated configs + handlers, batches + stop on failure, health checks + rollback, secrets safe, pinned toolchain, run from CI/AWX with approval.
- Full checklist: code, inventory/variables, secrets, identity/access, toolchain, testing, before/during/after the run, operations, documentation.
- Automate what you can (preflight, CI gates); keep humans for judgement (reading plans, approving).

### Interview questions

1. **Q: What do you check before running a playbook against production?**
   A: The target list (`--list-hosts`), that the change was reviewed and tested (lint, Molecule, staging), the `--check --diff` plan, that it runs from the approved pipeline with pinned tooling, batching and stop-on-failure settings, health checks and a rollback plan, and that secrets are protected.
2. **Q: How do you make sure checks aren't skipped?**
   A: Automate them as CI gates (lint, tests, preflight, plan) and require approvals in protected environments, so the only path to production enforces them.
3. **Q: What should happen after a production run?**
   A: Review the recap, verify a second run/check shows no changes, confirm health/monitoring, follow up unreachable/failed hosts, and store the logs/artifacts.

### Scenario question (answer out loud first, then compare)

> *"A junior engineer asks: 'Why do we need all this? I tested the playbook on my laptop and it worked.' How do you explain it?"*

**Model answer:**
1. "Worked on my laptop" proves it runs in **one** environment with **your** toolchain against **some** hosts. Production differs in inventory, data, scale, OS versions and network.
2. Each checklist item maps to a real failure mode: wrong environment (env guard, list-hosts), non-idempotent tasks causing restarts every run, configs that lock out SSH (validate), all servers changing at once (serial), leaked secrets (no_log), version drift (pins/EE).
3. Most items are **automated** and cost nothing once set up; the human part is reading the diff and approving.
4. Offer to pair on the next change through the pipeline so they see the gates catch things.

**Likely follow-ups:**
- *Which single item would you never skip?* Reading the `--check --diff` against the real target inventory.
- *How do you keep the checklist relevant?* Update it after each post-mortem.
