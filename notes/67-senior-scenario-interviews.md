# 67 · Senior Scenario Interviews

> **Level:** ★★★ advanced
>
> **You will learn:** how to answer **open design and troubleshooting questions** at senior DevOps/SRE/platform level: a framework for structuring answers, and twelve full scenarios (platform design for a large organisation, Puppet → Ansible migration, patching a database cluster, cross-account secrets and access, a slow and flaky run, multi-environment promotion, images vs runtime configuration, compliance at scale, Kubernetes cluster lifecycle, an automation-caused outage, a shared role library, and mixed Windows/Linux patching), each with what the interviewer is looking for, a model answer, likely follow-ups and red flags to avoid.
>
> **You need:** [66 · Interview question bank](66-interview-question-bank.md) and the whole course.
>
> **Part 12 · Production operations** · Next: [68 · Cheat sheet](68-cheat-sheet.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Senior questions have no single right answer. Interviewers judge **how you think**: do you clarify requirements, consider safety and failure, choose trade-offs consciously, and plan for people and operations, not just YAML?

**Picture it.** An architect asked to design a house doesn't start drawing walls; they ask how many people live there, the budget, the site, and what must never fail.

**The one thing to know.** A five-step answer frame:

```
 1. CLARIFY    : scale, environments, constraints (compliance, network, team skills), what "done" means
 2. DESIGN     : the components and the flow (repo, inventory, credentials, execution, CI/AWX)
 3. SAFETY     : idempotency, testing, batching, rollback, secrets, access, audit
 4. OPERATE    : monitoring, drift, upgrades, scaling, ownership
 5. TRADE-OFFS : what you chose not to do and why; how you'd evolve it
```

**Try it now:** answer scenario 1 out loud with the frame, in 5 minutes, before reading the model answer.

**You've got it if you can:** use the frame naturally and mention failure modes without being asked.

> 📖 **That's the core.** Everything below is the **scenario set**.

---

## What it is

### Scenario 1: Design an Ansible platform for a large organisation

> *"50 engineers in 6 teams manage 3,000 servers: on-prem RHEL, AWS (Amazon Linux/Ubuntu), some Windows. Today everyone runs Ansible from laptops with their own copies of roles. Design the target platform."*

**Looking for:** separation of content/config/credentials, central execution with RBAC, testing, supply chain, scale.

**Model answer:**
1. **Clarify**: compliance requirements, network zones (DMZ, on-prem, AWS accounts), change windows, existing tools (CI, secret manager, identity).
2. **Content**: shared roles in **versioned internal collections** (baseline, monitoring, web, db) with Molecule tests and lint gates; team repos for playbooks and inventories; CODEOWNERS (topics 37, 39, 50, 51).
3. **Inventory**: dynamic for AWS (tags), CMDB/NetBox or static per environment on-prem; per-environment `group_vars` (topic 42).
4. **Execution**: **AAP** (or AWX if unsupported is acceptable) with organisations per team, RBAC, credentials (with a secret-manager credential plugin), **execution environments** per team, **automation mesh** execution nodes in each network zone, schedules for patching and drift (topics 52, 56, 61).
5. **CI**: PR → lint, Molecule, check-mode plan; merge → AAP job via API with approvals for prod (topic 53).
6. **Security**: dedicated automation identities per environment, short-lived SSH certificates, no laptop access to prod, audit to SIEM (topic 60).
7. **Operate**: drift reports, run-time metrics, upgrade cadence, platform as code (`ansible.controller`).
8. **Trade-offs**: AAP cost vs supported platform; central team owns the platform and baseline, teams own their roles; migration in waves.

**Follow-ups:** *How do you migrate teams?* Start with the baseline collection and one pilot team; provide templates; deprecate laptop runs with a date. *What if AAP isn't approved?* CI-only (self-hosted runners per zone) with environments and approvals, plus a scheduler for drift.

**Red flags:** "Just put everything in one big repo and run it from Jenkins" without RBAC/credentials thinking; ignoring network zones; no testing story.

### Scenario 2: Migrate from Puppet to Ansible

> *"We have 1,200 servers managed by Puppet (agents every 30 minutes). Leadership wants Ansible. How?"*

**Looking for:** not a big-bang; parity proof; drift/continuous enforcement replacement.

**Model answer:**
1. Inventory what Puppet manages (modules, Hiera data) and classify: baseline, apps, exceptions.
2. Rebuild as Ansible roles **with parity tests**: run Ansible in `--check --diff` against Puppet-managed servers; the goal is **no diff** (Puppet already enforces the desired state).
3. Migrate per role and per server group: disable that Puppet class (or the agent) for the group, enable Ansible runs.
4. Replace Puppet's continuous enforcement: scheduled drift checks (report) and/or `ansible-pull` timers for the baseline (topic 55).
5. Data: Hiera → `group_vars`/`host_vars`; secrets (eyaml) → vault/secret manager.
6. Decommission Puppet masters at the end.

**Follow-ups:** *What's the hardest part?* Implicit behaviour (Puppet's ordering/resources) and losing continuous enforcement. *Rollback?* Re-enable the Puppet class for that group.

**Red flags:** turning off Puppet everywhere on day one; no parity check.

### Scenario 3: Patch a 3-node PostgreSQL cluster with no downtime

> *"Primary + 2 replicas with automatic failover. Patch OS and PostgreSQL minor version, reboot as needed, no downtime."*

**Looking for:** order, health checks, `serial: 1`, failover awareness, stop on problems.

**Model answer:**
1. Preconditions: cluster healthy, replication lag low, backups recent (assert, topic 31).
2. Play 1: replicas, `serial: 1`: drain from read pools (if any), patch, reboot, wait, health (replication streaming, lag), re-enable; `any_errors_fatal`.
3. Play 2: **controlled switchover** to a patched replica (via the cluster manager's API, delegated), verify the app connects.
4. Play 3: patch the old primary like a replica; optionally switch back.
5. Post: verify versions, replication, app health; report.

**Follow-ups:** *Why not patch the primary first?* It's the riskiest; patch replicas, then move the role. *How does Ansible know which node is primary?* Query the cluster (command + register / module) and build dynamic groups with `group_by`.

**Red flags:** `serial` without health checks; rebooting the primary without a switchover.

### Scenario 4: Secrets and access from CI to production across AWS accounts

> *"CI must configure servers in 5 AWS accounts (dev..prod). Security forbids long-lived keys."*

**Model answer:** OIDC from CI to a role per account (trust limited to repo + environment), environments with approvals for prod; secrets in each account's Secrets Manager fetched with lookups (`register_secret`, `no_log`); SSM connection (no SSH keys, no inbound) or SSH certificates from a CA; per-account dynamic inventory; CloudTrail/Session Manager audit; least-privilege policies (topics 41, 53, 60, 62).

**Follow-ups:** *Cross-account in one run?* Separate jobs per account (cleaner blast radius) or `assume_role` per inventory source/connection settings. *Rotation?* Managed rotation in Secrets Manager; the next run picks it up.

**Red flags:** "store keys as CI secrets"; disabling host key checking.

### Scenario 5: The 3-hour, flaky nightly run

> *"Nightly site run takes 3 hours and fails randomly on ~2% of hosts. Fix it."*

**Model answer:** measure with `profile_tasks`/`profile_roles`; classify failures by error (topic 59): unreachable (terminated instances → dynamic inventory filters), timeouts (bastion limits, slow mirrors), non-idempotent tasks (changed every night → noise and restarts). Speed: pipelining, ControlPersist, fact cache, lists not loops, sharding by region with execution near targets, free strategy for independent hosts. Reliability: retries for flaky network operations (`until`), timeouts, `max_fail_percentage` per shard, retry lists. Track duration and failure rate as metrics (topics 48, 61, 65).

**Red flags:** "just add more forks"; ignoring the failures.

### Scenario 6: Multi-environment promotion

> *"How do you promote the same Ansible change from dev to staging to prod safely?"*

**Model answer:** one code version (Git commit/tag) promoted through environments; inventories per environment hold differences; CI: PR plan on dev/staging, merge → dev apply, then staging apply with smoke tests, then prod with approval; artifacts (EE tag, collection versions) identical across environments; prod applies use the **same commit** that passed staging (tags or release branches); check-mode diff on prod attached to the approval (topics 20, 39, 53).

**Follow-ups:** *Environment-specific code?* Avoid; use variables/feature flags. *Hotfix path?* Same pipeline, fast-tracked review.

### Scenario 7: Immutable images vs runtime configuration

> *"Should we bake everything into images or configure at runtime with Ansible?"*

**Model answer:** depends on change frequency and scale. Bake slow-changing, heavy parts (packages, agents, hardening) into images with Packer + the same Ansible roles; keep runtime Ansible for instance-specific data, secrets, frequent config changes and operations (patching between image rebuilds, rotations). Benefits of baking: fast autoscaling, no drift; costs: rebuild pipeline, slower small fixes. Hybrid is common (topics 43, 61).

### Scenario 8: Compliance at scale

> *"Prove CIS compliance for 2,000 servers monthly, with exceptions tracked."*

**Model answer:** baseline role with per-control toggles; exceptions as reviewed `group_vars`/`host_vars` data with justification; nightly check-mode compliance runs (JSON stored) + OpenSCAP scans; drift tickets automated; monthly report from stored results; remediation runs in batches (topics 55, 57).

### Scenario 9: Kubernetes cluster lifecycle

> *"Would you use Ansible to build and upgrade Kubernetes clusters?"*

**Model answer:** for self-managed clusters on VMs: yes, commonly (kubeadm-based roles, Kubespray) for node OS prep, container runtime, kubeadm init/join, and **rolling node upgrades** (`serial: 1`, drain via `kubernetes.core` or kubectl delegated, upgrade, uncordon, health). For managed clusters (EKS/GKE/AKS): Terraform for the cluster, GitOps (Argo CD/Flux) for workloads; Ansible for the gaps (node bootstrap on self-managed node groups, one-off ops). Don't use Ansible as the continuous app deployer when GitOps fits better (topic 44).

### Scenario 10: An automation-caused outage

> *"Your Ansible change caused a 40-minute outage. Run the post-mortem."*

**Model answer:** blameless timeline (change merged, plan reviewed?, run started, first errors, detection, rollback, recovery); impact; root cause (e.g. config change applied to all hosts at once without health checks); contributing factors (no staging parity, plan not read, no batching); what went well; action items with owners: batching + health checks in that playbook, Molecule verify for the config, plan required in approvals, monitoring alert sooner (topics 45, 49, 64, 65).

**Red flags:** blaming the engineer; "be more careful" as the action item.

### Scenario 11: A shared role library for many teams

> *"Design how 6 teams share and evolve common roles without breaking each other."*

**Model answer:** internal collection(s) with semantic versioning, changelogs, CODEOWNERS (platform team) + contributions via PRs; Molecule matrix for all supported OSes; `argument_specs` as the contract; consumers pin versions in `requirements.yml` and upgrade deliberately; deprecation policy (warn for one minor, remove in the next major); EEs per team built from pins (topics 36, 37, 51, 52).

### Scenario 12: Mixed Windows/Linux patching orchestration

> *"Monthly patching for 800 Linux and 300 Windows servers, respecting app dependencies (DB before app servers, reboot windows per business unit)."*

**Model answer:** inventory groups by business unit, OS and tier; AAP workflow: pre-checks → snapshot/backups (cloud modules) → DB tier (`serial: 1`, failover-aware) → app tier (batches with LB drain) → Windows (`win_updates`, reboots, `serial`) → post-checks/report; schedules per business unit window; approvals for prod; failure thresholds stop the workflow; reporting per host (patched, rebooted, failed) via JSON/notifications (topics 44, 45, 49, 56).

---

## Why it exists

Senior roles are about designing systems that other people can operate safely. These scenarios combine many topics into the kind of discussion you'll have in architecture reviews and senior interviews.

---

## Analogy

**A chess player studying common openings and endgames.** You won't see the exact same game twice, but recognising patterns lets you play strong moves quickly and explain why.

---

## How it works inside

Most answers are built from the same building blocks; practise assembling them:

| Building block | Topics |
|---|---|
| Repo/inventory/collection structure | 37, 39, 42 |
| Credentials, identity, secrets | 40, 41, 60 |
| Testing and gates | 50, 51, 53 |
| Safe execution (serial, checks, rollback) | 31, 45, 46, 49 |
| Platform (EE, AWX/AAP, mesh) | 52, 56, 61 |
| Operations (drift, debugging, upgrades) | 55, 58, 63 |
| Ownership and process | 64, 65 |

---

## Code / commands

A snippet that often helps in interviews: dynamic group membership for "find the primary, then act":

```yaml
- name: Discover roles in the DB cluster
  hosts: db
  gather_facts: false
  tasks:
    - name: Is this node in recovery (replica)?
      ansible.builtin.command: psql -tAc "select pg_is_in_recovery()"
      become: true
      become_user: postgres
      register: recovery
      changed_when: false

    - name: Group by role
      ansible.builtin.group_by:
        key: "db_{{ 'replica' if recovery.stdout | trim == 't' else 'primary' }}"

- name: Patch replicas first, one at a time
  hosts: db_replica
  serial: 1
  any_errors_fatal: true
  tasks: [...]
```

`ansible.builtin.group_by` creates groups at run time from data, so later plays can target `db_primary`/`db_replica`.

---

## Hands-on lab

1. For each scenario, write your answer using the five-step frame before reading the model answer; time yourself (5–8 minutes).
2. Record yourself answering scenarios 1, 3 and 10; listen for missing safety/operations points.
3. Implement the `group_by` discovery play against the lab DB (with one node it'll be `db_primary`).
4. Pair with someone: one asks follow-ups from the lists, the other answers.

**Done when:** you can handle any of these in a 10-minute conversation without notes.

---

## Where you'll use it

- **Senior/staff interviews** and **architecture reviews**.
- **Writing proposals** for automation changes in your team.

---

## When it breaks

| Problem | Fix |
|---|---|
| Jumping into YAML details immediately | Start with clarifying questions and the overall design |
| Forgetting failure modes | Always cover batching, health checks, rollback and blast radius |
| Over-engineering for a small team | State the scale assumption; propose a simpler v1 and an evolution path |
| No people/process | Mention ownership, reviews, documentation, on-call |

---

## Quick recap

- Use the frame: **clarify → design → safety → operate → trade-offs**.
- Recurring building blocks: collections/inventories, credentials and identity, testing gates, safe execution, platform (EE/AAP/mesh), operations, ownership.
- Interviewers value failure thinking (blast radius, rollback, audit) and evolution plans over tool trivia.

### Interview questions

(The scenarios above.)

### Scenario question (answer out loud first, then compare)

> *"You join a team where Ansible 'mostly works' but nobody trusts it: runs are manual, slow, noisy and occasionally break things. What do you do in your first 90 days?"*

**Model answer:**
1. **Days 1–30: observe and stabilise**: inventory the repos, toolchain versions, how runs happen; pin the toolchain; add lint and `--syntax-check` to CI; stop the worst risks (laptop prod runs, plaintext secrets).
2. **Days 31–60: trust**: make the baseline idempotent (fix "always changed" tasks), add Molecule to the top roles, check-mode plans on PRs, batching and health checks in risky playbooks, run from CI with approvals.
3. **Days 61–90: value**: scheduled drift reports, performance tuning, documentation and a short training session; measure (run time, change failure rate, drift count) and share progress.
4. Throughout: small PRs, quick wins visible to the team, no big-bang rewrite.

**Likely follow-ups:**
- *How do you measure "trust"?* Second-run `changed=0` rate, failed-run rate, mean run time, number of manual changes found by drift checks.
- *What would you not change early?* Working but ugly structure: refactor after tests exist.
