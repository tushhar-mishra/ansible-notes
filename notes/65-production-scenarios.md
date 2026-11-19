# 65 · Production Scenarios

> **Level:** ★★★ advanced
>
> **You will learn:** how to handle the real incidents that happen around Ansible in production, each walked through the same way (**situation → symptoms → investigate → fix now → prevent**): the wrong environment, a half-applied run, SSH lock-out, a leaked secret, a run that suddenly got slow, lost handlers, noisy "always changed" runs, a dependency upgrade that broke CI, broken temp directories on targets, a growing list of unreachable cloud hosts, two pipelines racing, `state: latest` shipping a bad package, a hotfix overwritten by automation, and a stuck AWX job. You'll rehearse several of them in the lab.
>
> **You need:** [64 · Production checklist](64-production-checklist.md), [58 · Debugging](58-debugging-ansible.md), [59 · Error catalogue](59-error-catalogue.md).
>
> **Part 12 · Production operations** · Next: [66 · Interview question bank](66-interview-question-bank.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Incidents follow patterns. For each one, know **what you'd see**, **how you'd confirm the cause**, **what you'd do in the next 15 minutes**, and **what you'd change so it can't happen again**. The "prevent" part is what interviewers and post-mortems care about most.

**Picture it.** Fire drills: you practise the exit routes before there's a fire, so when the alarm rings you don't think, you act.

**The one thing to know.** First moves in any automation incident:

```
 1. STOP the bleeding   : cancel/stop running jobs, pause schedules, freeze merges if needed
 2. SCOPE               : which hosts changed? (job output, recap, --check --diff now, drift report)
 3. RESTORE service     : rollback (previous release, revert commit + re-run, restore backup)
 4. COMMUNICATE         : status to stakeholders
 5. ROOT CAUSE + PREVENT: post-mortem, checklist/test/lint/gate added
```

**Try it now:** read scenario 1 and answer "fix now" before reading the answer.

**You've got it if you can:** for any scenario here, name one technical control that would have prevented it.

> 📖 **That's the core.** Everything below is the **scenario library**.

---

## What it is

### Scenario 1: The playbook ran against production instead of staging

| | |
|---|---|
| **Situation** | An engineer ran `ansible-playbook site.yml` from a laptop "against staging"; prod web servers restarted at noon |
| **Symptoms** | Brief outage, unexpected config in prod, job ran from a laptop |
| **Investigate** | Shell history / terminal output; `ansible.cfg` default `inventory =`; which `-i`; sudo logs on prod hosts for the automation user's activity |
| **Fix now** | Assess what changed (`--check --diff` with the **correct** prod inputs shows what's now different from intended); re-apply the approved prod config through the pipeline; restore anything stateful |
| **Prevent** | No default inventory; one inventory dir per env; **environment guard** assert (topic 60); prod credentials only in CI/AWX (laptops physically can't reach prod); approvals |

### Scenario 2: A run failed halfway

| | |
|---|---|
| **Situation** | A config rollout failed on task 18 of 40 for 120 of 300 hosts (a repo mirror was down) |
| **Symptoms** | Mixed state: some hosts fully converged, some half, some untouched; recap shows `failed=1` on 120 |
| **Investigate** | Recap + job JSON: which hosts failed where; is the failure environmental (mirror) or code? |
| **Fix now** | Fix the cause (mirror), then **re-run the same playbook** (idempotent: finished tasks report `ok`); or retry only failed hosts (`-l @failed.txt`, topic 61). Check for lost handlers (scenario 6) |
| **Prevent** | Idempotent playbooks (so re-runs are safe); `serial` + `max_fail_percentage` (stop after a few failures instead of 120); preconditions (`assert` repo reachable); local mirrors |

### Scenario 3: The SSH hardening change locked out servers

| | |
|---|---|
| **Situation** | A baseline change added `AllowUsers alice bob` to sshd; the automation user wasn't listed; sshd restarted on 80 hosts |
| **Symptoms** | `UNREACHABLE ... Permission denied` for the automation user everywhere the change ran |
| **Investigate** | The PR diff; `validate` passed because the syntax was fine |
| **Fix now** | Out-of-band access: cloud serial console / SSM / hypervisor console / still-open ControlPersist masters; revert the drop-in (or add the user), restart sshd; then re-run the corrected role |
| **Prevent** | Role always includes the automation identity in access lists (assert); Molecule verify that logs in as the automation user after converge; **post-change connection check per batch** (`wait_for_connection` + `serial` + `max_fail_percentage: 0`); out-of-band access tested regularly (topics 18, 45, 57) |

### Scenario 4: A secret appeared in CI logs

| | |
|---|---|
| **Situation** | A debugging PR ran with `-vvv`; the DB password from a secret-manager lookup appears in the public-to-org CI log |
| **Symptoms** | Secret visible in job logs (lookup values aren't auto-masked unless registered as secrets) |
| **Fix now** | **Rotate the secret** (in the DB and the secret manager); delete/expire the log if your CI allows; check DB access logs for misuse |
| **Prevent** | `no_log` on secret tasks; `register_secret` on lookups (2.22); CI policy: no `-vvv` on jobs with secrets; secret scanning on logs; least-privilege log access (topics 40, 41, 60) |

### Scenario 5: The nightly run suddenly takes 4× longer

| | |
|---|---|
| **Situation** | From 25 minutes to 1h40 overnight, no code changes |
| **Investigate** | `profile_tasks` compare with a previous run; per-host timing (`-v`); control node CPU/RAM; network latency to targets/bastion; inventory size (did autoscaling double the fleet?); fact gathering time (hung NFS mounts?); package repo speed |
| **Typical causes** | A slow/hung mount making `setup` hang (fix: `gather_timeout`, `gather_subset`); bastion `MaxSessions` throttling; ControlPersist broken (path too long after a rename); a loop over a now-much-larger list; a slow mirror |
| **Prevent** | Track run duration as a metric with alerts; `profile_tasks` always on; timeouts (`gather_timeout`, task `timeout:`); capacity planning (topic 61) |

### Scenario 6: The config is new, but the service runs the old one

| | |
|---|---|
| **Situation** | nginx config changed during a deploy that failed later on some hosts; a week later they still serve the old config |
| **Investigate** | `nginx -T` vs the file; job history shows the host failed after the template task |
| **Fix now** | Reload/restart those hosts (ad hoc or ops playbook) |
| **Prevent** | `force_handlers: true`; `flush_handlers` right after critical changes; a verification task comparing running vs desired (topic 30) |

### Scenario 7: Every run shows `changed`, so drift alerts are ignored

| | |
|---|---|
| **Situation** | The nightly drift report says "45 hosts changed" every night; nobody reads it; real drift went unnoticed for weeks |
| **Investigate** | Run twice and list tasks that change on the second run; `ansible-lint` (`no-changed-when`, `package-latest`) |
| **Fix now** | Fix or quarantine the noisy tasks (`changed_when`, modules, no timestamps, `present`) |
| **Prevent** | Idempotence gate in CI (Molecule / two-run check); treat any non-zero `changed` in drift reports as an incident (topics 27, 55) |

### Scenario 8: A collection upgrade broke CI but "works on my laptop"

| | |
|---|---|
| **Situation** | CI installs collections with `>=` ranges; a new major release removed an option |
| **Symptoms** | `Unsupported parameters ...` only in CI |
| **Investigate** | `ansible-galaxy collection list` in CI vs laptop; the collection changelog |
| **Fix now** | Pin the last working version in `requirements.yml`; re-run |
| **Prevent** | Exact pins or bounded ranges (`<next major`); EE images built from pins; upgrade via PR with tests (topics 37, 52, 63) |

### Scenario 9: Modules fail on some hosts with temp-directory errors

| | |
|---|---|
| **Situation** | `Failed to create temporary directory` / `MODULE FAILURE` on 12 hosts |
| **Investigate** | `ansible host -m raw -a 'df -h ~ /tmp; mount | grep -E "noexec| /home"'`; recent hardening that mounted `/home` or `/tmp` `noexec`; full disks |
| **Fix now** | Free disk; set `remote_tmp` to an allowed path for those hosts (host_vars), or enable pipelining (no temp files) |
| **Prevent** | Monitoring on disk space; baseline role aware of `noexec` policies; pipelining on by default (topics 13, 48) |

### Scenario 10: More and more UNREACHABLE hosts in the cloud

| | |
|---|---|
| **Situation** | Each run reports dozens of unreachable hosts; most were terminated by autoscaling |
| **Investigate** | Inventory cache age; filters missing `instance-state-name: running`; static inventory files |
| **Fix now** | `--flush-cache`; fix filters |
| **Prevent** | Dynamic inventory with state filters and short cache; alert only on hosts unreachable across several runs (topic 42) |

### Scenario 11: Two pipelines applied at the same time

| | |
|---|---|
| **Situation** | Two merges 30 seconds apart triggered two apply jobs; the older commit finished last and overwrote the newer config |
| **Symptoms** | Config "flaps" back to an older version |
| **Fix now** | Re-run the latest `main` |
| **Prevent** | Concurrency groups / locks per environment (`cancel-in-progress: false` so applies queue); apply always from the **latest** `main` (topics 53, 54) |

### Scenario 12: `state: latest` shipped a broken package to the whole fleet

| | |
|---|---|
| **Situation** | The baseline role installs `openssl` with `state: latest`; a broken build in the mirror reached every server in one nightly run |
| **Fix now** | Downgrade/pin the previous version (`dnf: name=openssl-3.x.y state=present allow_downgrade=true`), restart affected services in batches; quarantine the mirror |
| **Prevent** | `present` + pinned versions in config roles; patching as a separate, batched, canaried playbook; staging mirrors promoted after testing (topics 17, 45) |

### Scenario 13: An emergency hotfix was overwritten by the next run

| | |
|---|---|
| **Situation** | During an incident someone raised `worker_connections` by hand; the next scheduled run reverted it; the incident came back |
| **Fix now** | Put the change into code (PR, fast-tracked review), run the pipeline |
| **Prevent** | Incident process: "every manual change gets a PR before the end of the incident"; drift report shows manual changes; consider report-only drift for that role during incidents (topic 55) |

### Scenario 14: AWX/AAP jobs stuck or failing at start

| | |
|---|---|
| **Situation** | Jobs stay "pending" or fail before the first task |
| **Investigate** | Instance/container group capacity; EE image pull errors; project sync failures (Git credentials expired); credential plugin errors (secret manager token expired); the job's "explanation" field |
| **Fix now** | Restore capacity (execution nodes/pods), fix registry/Git/secret-manager credentials, re-launch |
| **Prevent** | Monitor controller and execution-node health; alert on credential expiry; config-as-code for the platform (topic 56) |

---

## Why it exists

Knowing Ansible's features isn't the same as operating it safely. These scenarios connect the features to the failures they prevent, and they're exactly the "tell me about a time..." and "what would you do if..." questions in senior interviews.

---

## Analogy

**Flight simulator training.** Pilots practise engine failures, bird strikes and lost hydraulics in the simulator, so the real thing triggers trained responses instead of panic.

---

## How it works inside

Every scenario maps to controls you've already learned:

| Control | Prevents scenarios |
|---|---|
| Environment guard, no default prod inventory, prod creds only in CI/AWX | 1 |
| Idempotency + re-run safety | 2, 7 |
| `serial` + `max_fail_percentage` + health/connection checks | 2, 3, 12 |
| `validate` + validate-then-restart + Molecule verify | 3 |
| `no_log`, `register_secret`, no `-vvv`, rotation | 4 |
| `profile_tasks`, timeouts, metrics | 5 |
| `force_handlers`, `flush_handlers`, verification | 6 |
| Pins, EEs, controlled upgrades | 8 |
| Pipelining, disk monitoring | 9 |
| Dynamic inventory filters and caching | 10 |
| Concurrency/locks | 11 |
| `present` + pinned packages, separate patching | 12 |
| Drift reports + "PR for every manual change" | 13 |
| Platform monitoring, config as code | 14 |

---

## Code / commands

### Rehearse scenario 2 (half-applied run) in the lab

```bash
cd ~/infra
# make the package mirror "fail" on web2 only by breaking DNS for it
ssh ansible@172.30.0.12 'sudo cp /etc/resolv.conf /tmp/resolv.bak && echo "nameserver 192.0.2.1" | sudo tee /etc/resolv.conf'
ansible-playbook -i inventories/lab playbooks/site.yml        # web2 fails at a package task (if it needs to install something)
ssh ansible@172.30.0.12 'sudo cp /tmp/resolv.bak /etc/resolv.conf'
ansible-playbook -i inventories/lab playbooks/site.yml        # converges; finished tasks are ok
```

(If nothing needs installing, add a new package to `baseline_packages` first so there's real work.)

### Rehearse scenario 6 (lost handler)

Use topic 30's `handlers-demo.yml` failing-task variant; then add `force_handlers: true` and repeat.

### Rehearse scenario 11 (race)

```bash
# two "applies" with different page texts, started at the same time
( ansible-playbook -i inventories/lab playbooks/web.yml -e '{"nginx_web_sites":[{"name":"main","title":"VERSION A"}]}' >/tmp/a.log ) &
( ansible-playbook -i inventories/lab playbooks/web.yml -e '{"nginx_web_sites":[{"name":"main","title":"VERSION B"}]}' >/tmp/b.log ) &
wait; curl -s -H 'Host: main.web1' 172.30.0.11:8081/
```

Whichever finished last wins: not deterministic. Then run them through your pipeline with a concurrency group and observe they queue.

### Rehearse scenario 3 safely (lock-out) on one container

Add `AllowUsers alice` via `baseline_sshd_settings` for web2 only (`host_vars/web2.yml`), run `baseline.yml -l web2`, see the automation user lose access, then recover through `docker exec ansible-lab-web2-1 rm /etc/ssh/sshd_config.d/40-baseline.conf && docker exec ansible-lab-web2-1 systemctl restart ssh` (your "out-of-band console"). Then add a Molecule/verify check that would have caught it.

---

## Hands-on lab

1. For each scenario, write your own "fix now" and "prevent" before reading the answers; compare.
2. Rehearse scenarios 2, 3, 6 and 11 in the lab as described.
3. Pick the three scenarios most likely in your (current or target) job and turn their "prevent" columns into tickets/checklist items.
4. Write a one-page post-mortem template: timeline, impact, root cause, what went well, what didn't, action items with owners.

**Done when:** you can talk through any scenario here in two minutes, including prevention.

---

## Where you'll use it

- **On-call** and incident response.
- **Post-mortems** and improvement backlogs.
- **Senior interviews** (topic 67).

---

## When it breaks

If you face an incident not listed here: apply the five first moves (stop, scope, restore, communicate, prevent), use the debugging method (topic 58), and add the new scenario to your team's library afterwards.

---

## Quick recap

- First moves: **stop → scope → restore → communicate → root cause + prevent**.
- Most incidents trace back to: wrong environment, non-idempotent code, no batching/health checks, unvalidated configs, secrets in logs, unpinned dependencies, lost handlers, races, `latest`, manual changes.
- Every scenario has a technical prevention you've learned in this course; the post-mortem's job is to add it.

### Interview questions

1. **Q: A playbook failed halfway through a fleet. What do you do?**
   A: Find which hosts failed where (recap/job data), fix the cause, then re-run the idempotent playbook (or only the failed hosts) so finished tasks report ok and the rest converge; check for lost handlers. Prevent with batching and failure thresholds.
2. **Q: How do you prevent running against the wrong environment?**
   A: Separate inventories per environment, no default prod inventory, an environment guard assertion, prod credentials only available in a protected pipeline/AWX with approvals.
3. **Q: A manual hotfix keeps being reverted by automation. What's the fix?**
   A: Put the change into code through a PR (fast-tracked if urgent) so the desired state includes it; use drift reports to catch manual changes and a process requiring a PR for every manual change.

### Scenario question (answer out loud first, then compare)

> *"At 09:05 a scheduled AWX job pushed a new sudoers template to all 500 servers. By 09:15 the help desk reports admins can't use sudo on many servers. Walk through your response."*

**Model answer:**
1. **Stop**: disable the schedule and cancel any related jobs; freeze merges to the role.
2. **Scope**: job output/recap → which hosts changed the sudoers file; sample a few hosts (with the automation user if it still works: NOPASSWD may be in a separate file).
3. **Restore**: if `validate` passed, the file is syntactically valid but wrong (e.g. lost a group). Revert the commit and re-run the job on the affected hosts in batches; if the automation user lost sudo too, use out-of-band access (console/SSM) on a canary, restore the previous file (backup from `backup: true`), then automate the fix.
4. **Communicate**: status updates to admins and stakeholders with ETA.
5. **Prevent**: Molecule verify that checks expected sudo rules for the admin and automation users; a test that renders the template for each environment's data; schedules with `serial` and a canary group; review rules for security-critical templates; keep a separate, never-templated drop-in for the automation user.

**Likely follow-ups:**
- *Why didn't `validate` catch it?* It checks syntax, not intent; intent needs tests.
- *How do you find which servers are still broken?* An ad hoc `sudo -l -U <admin>` check across the fleet, or a check-mode run listing hosts that would change.
