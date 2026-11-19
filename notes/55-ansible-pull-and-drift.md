# 55 · ansible-pull and Drift Detection

> **Level:** ★★ intermediate
>
> **You will learn:** two ways to keep servers converged **after** the first deploy: **pull mode** with **`ansible-pull`** (each host clones a Git repo and applies a playbook to itself on a timer: options, how it limits itself to its own host, scheduling with a systemd timer, and its trade-offs: Ansible and secrets on every host, no central view), and **drift detection** from a central place (scheduled `--check --diff` runs, the **JSON callback**, a script that reports which hosts drifted, report-only vs auto-correct policies).
>
> **You need:** [54 · Jenkins](54-ansible-in-jenkins.md) / [53 · GitHub Actions](53-ansible-in-github-actions.md), [01 · drift](01-what-is-configuration-management.md), [20 · check and diff](20-check-diff-limit.md), [27 · idempotency](27-idempotency-in-depth.md), [Linux 039 · systemd timers](../../linux-mastery-rhel/notes/039-job-scheduling-cron-and-timers.md).
>
> **Part 9 · CI/CD** · Next: [56 · AWX and Ansible Automation Platform](56-awx-and-aap.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Ansible is push-based (topic 01): nothing happens between runs. Two ways to fix that:

- **Pull mode**: install Ansible on each server; a timer runs **`ansible-pull`**, which clones your Git repo and applies a playbook **to itself**. Like Puppet/Chef agents, but with Ansible playbooks.
- **Drift detection**: keep push mode, and run **`--check --diff`** on a schedule from CI/AWX. If anything would change, someone changed a server by hand: **report** (or fix) it.

**Picture it.** Pull mode = every shop manager checks the head-office manual every 30 minutes and fixes their own shop. Drift detection = an inspector visits every night and writes a report of shops that don't match the manual.

**The one thing to know.**

```bash
# on each host (pull)
ansible-pull -U https://git.example.com/infra.git -C main -i inventories/pull/hosts.yml playbooks/local.yml --only-if-changed

# central drift report (push, check only)
ANSIBLE_STDOUT_CALLBACK=ansible.posix.json ansible-playbook -i inventories/prod playbooks/site.yml --check --diff > drift.json
jq -r '.stats | to_entries[] | select(.value.changed > 0) | .key' drift.json      # hosts that drifted
```

**Try it now:** run the drift command against the lab after changing a file by hand on web1.

**You've got it if you can:** list two advantages and two disadvantages of `ansible-pull`, and explain why idempotency is required for drift detection.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### ansible-pull

`ansible-pull` runs **on the managed host**:

1. clone (or update) a Git repo into a local directory,
2. run `ansible-playbook` on a playbook from it, against **localhost** (connection local),
3. optionally only if the repo changed.

| Option | Meaning |
|---|---|
| `-U URL` | Git repository URL (https or ssh) |
| `-C BRANCH` | Branch/tag/commit to check out |
| `-d DIR` | Where to clone (default `~/.ansible/pull/<hostname>`) |
| `-i INVENTORY` | Inventory (inside the repo, or `localhost,`) |
| `PLAYBOOK` | Playbook path inside the repo (default: `<hostname>.yml`, then `local.yml`) |
| `-o` / `--only-if-changed` | Run only if the checkout changed since last time |
| `--accept-host-key` | Accept the Git server's SSH host key (first contact) |
| `--purge` | Delete the checkout after running |
| `-s N` / `--sleep N` | Random delay up to N seconds before starting (spreads load on the Git server) |
| `--vault-id` / `--vault-password-file` | Vault access (the secret must exist **on the host**) |
| `--check`, `--diff`, `-t TAGS` | As usual |

**How a host picks its configuration.** `ansible-pull` automatically limits the run to the host's own names (`localhost`, its hostname, its FQDN, `127.0.0.1`). So you can keep a normal inventory with groups in the repo: the host matches its own entry, gets its groups' variables, and plays like `hosts: web` apply to it if it's in `web`. The connection is local (no SSH).

### Pull vs push

| | Push (`ansible-playbook` from CI/AWX) | Pull (`ansible-pull` on each host) |
|---|---|---|
| Who starts | Central | Each host on a timer |
| Ansible installed on targets | No | **Yes** |
| Network | Central → all hosts (SSH) | Each host → Git server (HTTPS/SSH) |
| Secrets | Only on the central runner | **On every host** (vault password, Git credentials) ⚠️ |
| Scale | Bounded by forks/runner capacity | Scales naturally (each host does its own work) |
| Autoscaling / short-lived hosts | Must discover them (dynamic inventory) | Self-configure at boot (cloud-init starts the first pull) |
| Orchestration across hosts (drain LB, order) | ✅ Easy | ❌ Each host acts alone |
| Central results / visibility | ✅ One log/recap | ❌ Need log shipping or a callback to a central system |
| Drift repair | On the next push | Continuous (every timer run) |

Common use: pull mode for **large, uniform, autoscaled** fleets (baseline hardening, agents), push for **orchestrated** changes (deploys, rolling restarts).

### Drift detection

**Drift** = actual state ≠ code (topic 01). With idempotent playbooks, a `--check --diff` run reports `changed > 0` **only** where reality differs from the code. That's a drift report.

| Approach | How | Notes |
|---|---|---|
| **Report only** | Scheduled `--check --diff`; summarise hosts/tasks with changes; alert/ticket | ✅ Safe default; humans decide |
| **Auto-correct** | Scheduled real run (no `--check`) | Repairs drift automatically; can also "repair" an emergency hotfix someone made deliberately |
| **Hybrid** | Auto-correct low-risk roles (baseline, agents), report-only for apps/data | Common in practice |

Requirements for useful drift reports:

1. **Idempotent playbooks** (topic 27): otherwise every run shows "changes" and the report is noise.
2. **Check-mode friendly tasks**: commands guarded with `creates`, read-only probes with `check_mode: false`, dependent tasks handled (topic 20).
3. **Machine-readable output**: the JSON callback.

### The JSON callback

`ANSIBLE_STDOUT_CALLBACK=ansible.posix.json` replaces the normal output with one JSON document:

```json
{
  "plays": [ { "play": {...}, "tasks": [ { "task": {"name": "..."}, "hosts": { "web1": {"changed": true, "diff": [...] } } } ] } ],
  "stats": {
    "web1": {"changed": 1, "failures": 0, "ok": 14, "skipped": 2, "unreachable": 0, "ignored": 0, "rescued": 0},
    "web2": {"changed": 0, ...}
  }
}
```

`stats` gives per-host counters; `plays[].tasks[].hosts` gives per-task details (including diffs).

---

## Why it exists

Push mode leaves gaps: between runs, people make manual changes, packages auto-update, files get edited during incidents. Without a regular check, drift accumulates until the next deploy surprises you (topic 01's snowflakes). Pull mode closes the gap by **continuous convergence**; drift detection closes it by **continuous visibility**. Both turn "we think the servers match the code" into something you can show an auditor.

---

## Analogy

**Keeping a hotel chain consistent.**

- **Pull mode**: each hotel has its own housekeeping team that re-reads the head-office standards every evening and fixes its rooms. Great for 1,000 hotels; but every hotel needs a copy of the master key (secrets) and head office only knows what happened if each hotel reports back.
- **Drift detection**: a mystery-shopper visits every hotel at night and writes "room 12: wrong pillows" without touching anything. The manager decides whether to fix it or update the standard.

---

## How it works inside

```
 PULL (on each host, every 30 min via systemd timer)
   ansible-pull -U repo -C main -o -i inventories/pull/hosts.yml playbooks/local.yml
     ├─ git fetch/checkout main → /var/lib/ansible-pull
     ├─ changed since last run? (-o) no → exit
     └─ ansible-playbook -c local --limit localhost,web7,web7.example.com,127.0.0.1 playbooks/local.yml
            → logs to the journal (systemd) → ship to central logging

 DRIFT (central, nightly)
   ansible-playbook -i inventories/prod site.yml --check --diff   (JSON callback)
     └─ jq: hosts with stats.changed > 0, tasks that would change → report/alert
```

---

## Code / commands

### Lab 1: ansible-pull on web2

**1. Serve the repo from WSL2.** Commit your `~/infra` repo (topic 39), then serve it read-only with `git daemon` (no credentials needed for the lab):

```bash
cd ~/infra && git add -A && git commit -qm "pull lab" || true
git daemon --base-path="$HOME" --export-all --reuseaddr --listen=0.0.0.0 --port=9418 "$HOME/infra" &
```

From the containers, the WSL2 host is the bridge gateway `172.30.0.1`, so the URL is `git://172.30.0.1/infra`.

**2. A pull inventory and playbook in the repo.**

`inventories/pull/hosts.yml`:

```yaml
all:
  children:
    web:
      hosts:
        web1: {}
        web2: {}
```

`playbooks/local.yml`:

```yaml
---
- name: Self-managed baseline (ansible-pull)
  hosts: web
  connection: local
  become: true
  tasks:
    - name: motd says the host is self-managed
      ansible.builtin.copy:
        dest: /etc/motd
        content: "{{ inventory_hostname }} is managed by ansible-pull\n"
        mode: "0644"

    - name: Base tools present
      ansible.builtin.package:
        name: [tree, curl]
        state: present
```

Commit both.

**3. Install Ansible and Git on web2, run a pull by hand.**

```bash
ssh ansible@172.30.0.12 'sudo apt-get install -y -qq ansible-core git'
ssh ansible@172.30.0.12 'sudo ansible-pull -U git://172.30.0.1/infra -C main -d /var/lib/ansible-pull -i inventories/pull/hosts.yml playbooks/local.yml'
```
```
Starting Ansible Pull at 2026-11-19 13:10:02
/usr/bin/ansible-pull -U git://172.30.0.1/infra -C main -d /var/lib/ansible-pull -i inventories/pull/hosts.yml playbooks/local.yml
localhost | CHANGED => {
    "after": "3f2c1d9...",
    "before": null,
    "changed": true
}

PLAY [Self-managed baseline (ansible-pull)] ********************************************
TASK [Gathering Facts] *****************************************************************
ok: [web2]
TASK [motd says the host is self-managed] **********************************************
changed: [web2]
...
PLAY RECAP *****************************************************************************
web2   : ok=3  changed=1  unreachable=0  failed=0  ...
```

- The first block is the `git` module checking out the repo (`before: null` = first clone).
- The play ran only for **web2** (its own hostname), although the inventory also lists web1.
- Ubuntu's packaged `ansible-core` (2.16) is enough for a simple local playbook. In production, pin the version on hosts (pip/venv or packages) and keep playbooks compatible.

**4. Schedule it with a systemd timer** (deployed with Ansible from your control node):

`playbooks/pull-setup.yml`:

```yaml
---
- name: Set up ansible-pull on web2
  hosts: web2
  become: true
  vars:
    pull_repo: git://172.30.0.1/infra
  tasks:
    - name: Ansible and Git are installed
      ansible.builtin.package:
        name: [ansible-core, git]
        state: present

    - name: ansible-pull service
      ansible.builtin.copy:
        dest: /etc/systemd/system/ansible-pull.service
        mode: "0644"
        content: |
          [Unit]
          Description=ansible-pull self-configuration
          After=network-online.target
          [Service]
          Type=oneshot
          ExecStart=/usr/bin/ansible-pull -U {{ pull_repo }} -C main -d /var/lib/ansible-pull -o -i inventories/pull/hosts.yml playbooks/local.yml
      notify: Reload systemd

    - name: ansible-pull timer (every 10 minutes, randomised)
      ansible.builtin.copy:
        dest: /etc/systemd/system/ansible-pull.timer
        mode: "0644"
        content: |
          [Unit]
          Description=Run ansible-pull periodically
          [Timer]
          OnBootSec=2min
          OnUnitActiveSec=10min
          RandomizedDelaySec=60
          [Install]
          WantedBy=timers.target
      notify: Reload systemd

    - name: Flush handlers before enabling
      ansible.builtin.meta: flush_handlers

    - name: Timer is enabled and started
      ansible.builtin.systemd_service:
        name: ansible-pull.timer
        state: started
        enabled: true

  handlers:
    - name: Reload systemd
      ansible.builtin.systemd_service:
        daemon_reload: true
```

```bash
cd ~/infra
ansible-playbook -i inventories/lab playbooks/pull-setup.yml
ssh ansible@172.30.0.12 'systemctl list-timers ansible-pull.timer; sudo journalctl -u ansible-pull -n 20 --no-pager'
```

Change the motd text in `local.yml`, commit, wait for the timer (or `sudo systemctl start ansible-pull.service`), and watch web2 update itself.

### Lab 2: central drift report

**1. Create drift by hand:**

```bash
ssh ansible@172.30.0.11 'echo "edited by hand" | sudo tee -a /srv/sites/main/index.html'    # or any file a role manages
```

**2. Run a check with the JSON callback:**

```bash
cd ~/infra
ANSIBLE_STDOUT_CALLBACK=ansible.posix.json \
  ansible-playbook -i inventories/lab playbooks/site.yml --check --diff > /tmp/drift.json
```

**3. Summarise with `jq`:**

`scripts/drift-report.sh`:

```bash
#!/usr/bin/env bash
# Usage: drift-report.sh drift.json   → prints drifted hosts and tasks; exit 1 if any drift
set -euo pipefail
f="$1"
echo "Hosts with drift:"
jq -r '.stats | to_entries[] | select(.value.changed > 0) | "  \(.key): \(.value.changed) change(s)"' "$f"
echo "Tasks that would change:"
jq -r '.plays[].tasks[] | .task.name as $t | .hosts | to_entries[] | select(.value.changed == true) | "  \(.key): \($t)"' "$f"
jq -e '[.stats[] | .changed] | add == 0' "$f" >/dev/null || exit 1
```

```bash
chmod +x scripts/drift-report.sh
scripts/drift-report.sh /tmp/drift.json; echo "exit=$?"
```
```
Hosts with drift:
  web1: 1 change(s)
Tasks that would change:
  web1: nginx_web : Index pages are in place
exit=1
```

**4. Schedule it**: add a `drift` job to the GitHub Actions workflow (topic 53) triggered by `schedule`, running the same two commands and failing (red) when drift exists, with the report in `$GITHUB_STEP_SUMMARY`; or a nightly Jenkins job (topic 54) or an AWX schedule (topic 56).

**5. Decide**: re-run the real playbook to repair, or update the code if the manual change was right.

---

## Hands-on lab

1. Do Lab 1: run `ansible-pull` by hand on web2, then install the timer and watch it apply a committed change.
2. Use `-o` (`--only-if-changed`) and confirm the next timer run does nothing when the repo didn't change.
3. Think through secrets: what would web2 need to decrypt a vaulted variable in pull mode? (The vault password **on the host**: one more secret to distribute and protect, e.g. via cloud-init from a secret manager.)
4. Do Lab 2: create drift on web1, run the report, repair it with a normal run, and confirm the next report is clean (`exit=0`).
5. Make a task non-idempotent on purpose (e.g. a `command` without `changed_when`) and see the drift report become useless noise. Fix it.
6. Write the `drift` job for your GitHub Actions workflow.
7. Clean up: `ssh ansible@172.30.0.12 'sudo systemctl disable --now ansible-pull.timer'`; stop `git daemon` (`kill %1` or `pkill git-daemon`).

**Done when:** you can show which hosts drifted last night and why, and you know when pull mode would be the better design.

---

## Where you'll use it

- **Autoscaling fleets**: user data installs Ansible and runs the first `ansible-pull`; a timer keeps converging.
- **Edge/remote sites** with unreliable inbound connectivity: hosts pull when they can.
- **Compliance**: nightly drift reports as evidence that servers match the approved configuration.
- **Change management**: drift alerts reveal undocumented manual changes after incidents.
- **AWX/AAP**: scheduled check-mode job templates are the platform's drift reports (topic 56).

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `ansible-pull: command not found` on hosts | Ansible not installed on targets (pull mode needs it) | Install a pinned version (package or venv) via cloud-init/bootstrap |
| `fatal: could not read Username for 'https://...'` | Private repo, no credentials on the host | Deploy key (read-only) or token, distributed securely |
| `Host key verification failed` for the Git server | SSH Git URL, server key unknown | `--accept-host-key` (TOFU) or pre-seed known_hosts |
| Play doesn't run on the host | Host's name not in the pull inventory / not in the play's group | Match hostname/FQDN; check `ansible-pull` output for the limit used |
| All hosts hit the Git server at once | Timers aligned | `RandomizedDelaySec` / `--sleep` |
| Vaulted vars fail on hosts | No vault password on the host | Distribute it (secret manager at boot) or avoid vault in pull playbooks |
| No central visibility | Pull results only in each host's journal | Ship logs; or a callback plugin to a central service |
| Drift report shows changes on every host every night | Non-idempotent or check-mode-unfriendly tasks | Topics 20 and 27 |
| JSON output mixed with warnings breaks `jq` | Warnings printed to stdout | Redirect stderr separately; fix warnings |

---

## Quick recap

- **`ansible-pull`**: each host clones a Git repo and applies a playbook to itself (connection local, limited to its own names), usually on a **systemd timer** with `-o` and random delay.
- Pull = scale and self-configuration (autoscaling, edge); push = orchestration and central control. Pull puts **Ansible and secrets on every host**.
- **Drift detection**: scheduled `--check --diff` with the **`ansible.posix.json`** callback; `stats.changed > 0` = drift; report-only by default, auto-correct for low-risk roles.
- Both depend on **idempotent, check-mode-friendly** playbooks.

### Interview questions

1. **Q: What is `ansible-pull` and when would you use it?**
   A: A command that runs on the managed host, clones a Git repo and applies a playbook locally. Useful for large or autoscaled fleets and hosts that can't be reached inbound; trade-offs are installing Ansible and distributing secrets to every host and losing central orchestration/visibility.
2. **Q: How do you detect configuration drift with Ansible?**
   A: Run the idempotent playbooks in `--check --diff` mode on a schedule and report hosts where tasks would change (e.g. using the JSON callback's `stats`).
3. **Q: Why must playbooks be idempotent for drift detection to work?**
   A: Otherwise check mode reports changes on every run regardless of real drift, so the report is noise.

### Scenario question (answer out loud first, then compare)

> *"Auditors want proof, every month, that all 800 production servers match the approved baseline, and a record of every deviation and how it was handled. Design it."*

**Model answer:**
1. **Baseline as code**: an idempotent, check-mode-safe `baseline` role in Git; changes only via reviewed PRs (the approval record).
2. **Nightly drift job** (AWX schedule or CI): `--check --diff` against prod with the JSON callback; store the JSON per run (artifact storage) as evidence.
3. **Reporting**: a script/dashboard turns `stats` and task diffs into a report: hosts compliant / drifted, which settings, since when. Drift opens a ticket automatically.
4. **Handling**: each ticket ends in either a repair run (logged job ID) or a code change (PR link). Low-risk baseline items can auto-correct in a second scheduled real run, also logged.
5. **Monthly evidence**: the stored nightly results + tickets + PR history. Unreachable hosts are listed and followed up (no result = not compliant).

**Likely follow-ups:**
- *What about servers that are legitimately different?* Model the exception in code (host_vars/group) through a PR, so it's approved and visible, instead of tolerating drift.
- *Why not just auto-correct everything?* An emergency hotfix may be deliberately in place; report-first lets humans decide and keeps change control honest.
