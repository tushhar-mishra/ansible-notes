# 20 · Safe Runs: check, diff, limit and Friends

> **Level:** ★ foundation
>
> **You will learn:** how to run a playbook against real servers **without fear**: **check mode** (`--check`, a dry run where modules *predict* changes), **diff mode** (`--diff`, showing exactly what would change inside files), how they combine, which modules support them and what happens to those that don't (`command`/`shell` are skipped), the `check_mode` and `diff` task keywords and the `ansible_check_mode` variable, why later tasks can fail in check mode, and the run-control options **`--limit`**, **`--syntax-check`**, **`--list-hosts`**, **`--list-tasks`**, **`--start-at-task`**, **`--step`**. It ends with a step-by-step **safe production run procedure**.
>
> **You need:** [19 · become](19-become.md), [18 · Essential modules II](18-modules-files.md), [16 · ansible-doc](16-modules-fqcn-and-ansible-doc.md) (the ATTRIBUTES section).
>
> **Part 2 · Playbooks** · Next: [21 · Variables basics](21-variables-basics.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Before changing production, ask Ansible **"what would you change?"**. `--check` runs every task in **prediction mode** (no changes made). `--diff` shows the **line-by-line difference** for files. Together they're Ansible's `terraform plan`.

**Picture it.** A **dress rehearsal**: actors walk through the play and say "here I would pick up the sword", but nobody gets hurt.

**The one thing to know.**

```bash
ansible-playbook site.yml --check --diff               # predict + show file changes, change nothing
ansible-playbook site.yml --check --diff --limit web1  # same, one host
ansible-playbook site.yml --limit web1                 # apply on one host (canary)
ansible-playbook site.yml                              # then everyone
```

**Try it now:** edit `files/app.conf` (change `listen = 8080` to `9090`), then run `ansible-playbook files.yml --check --diff`. You'll see the diff and `changed`, but the servers are untouched (`ssh` in and check).

**You've got it if you can:** explain why a `command` task is skipped in check mode and why a service task can fail in check mode when its package isn't installed yet.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Check mode (`--check`, `-C`)

Ansible passes `_ansible_check_mode: true` to every module (topic 13's internal arguments). Each module then:

| Module's check-mode support (`ansible-doc` ATTRIBUTES) | What happens in `--check` |
|---|---|
| **full** (`file`, `copy`, `template`, `user`, `package`, `lineinfile`…) | Checks the current state and **reports what it would do** (`changed` / `ok`), changes nothing |
| **partial** (`command`, `shell` with `creates`/`removes`; some others) | Predicts only in some situations; otherwise skipped |
| **none** (`command`/`shell` without `creates`/`removes`, `raw`, `script`, many API modules) | **Skipped**: `skipping: [web1]` (with a note that check mode isn't supported) |

Check mode output looks like a normal run: same task lines, same recap. The difference is that nothing was actually changed.

### Diff mode (`--diff`, `-D`)

Modules that support diff (`copy`, `template`, `lineinfile`, `blockinfile`, `file` for some changes, `ini_file`, ...) add a **before/after** to their result, and Ansible prints it as a unified diff (like `diff -u`, [Linux 018](../../linux-mastery-rhel/notes/018-text-filters-and-sed.md)).

- `--diff` alone: **makes** the change and shows what changed (useful in logs).
- `--check --diff`: shows what **would** change. ✅ The pre-flight you'll use most.
- ⚠️ Diffs print file **contents**. For files with secrets, set `diff: false` on the task (and `no_log: true`, topic 40).

### The task keywords

| Keyword | Value | Effect |
|---|---|---|
| `check_mode: true` | on a task | **Always** run this task in check mode, even in a normal run ("never really change this") |
| `check_mode: false` | on a task | **Really run** this task even during `--check` (for read-only commands whose output later tasks need) |
| `diff: false` / `diff: true` | on a task | Turn diff output off/on for this task |

And the **variable** `ansible_check_mode`: `true` during `--check`. Use it in conditions (topic 26):

```yaml
- name: Restart only in real runs
  ansible.builtin.command: /usr/local/bin/app-reload
  when: not ansible_check_mode
```

### Why later tasks can fail in check mode

Check mode **predicts** each task on its own, but nothing really happens. So a later task can't see an earlier task's predicted change:

```
TASK [nginx is installed]                 changed: [web3]    ← predicted, NOT installed
TASK [nginx is running and enabled]       fatal: [web3]: FAILED! => {"msg": "Could not find the requested service nginx: host"}
```

On a brand-new server, check mode will often fail at "service" or "template into a directory that a previous task would have created". Ways to handle it:

| Fix | Example |
|---|---|
| Accept it on fresh hosts: check mode is most useful on servers **already** converged | Use check runs on existing fleets |
| Skip the dependent task in check mode | `when: not ansible_check_mode` |
| Don't fail it in check mode | `ignore_errors: "{{ ansible_check_mode }}"` |
| Let a read-only task run for real | `check_mode: false` on a `command` that only reads (with `changed_when: false`) |

### Run-control options

| Option | What it does | Use it for |
|---|---|---|
| `--syntax-check` | Parse only (phase 1, topic 15) | CI on every commit |
| `--list-hosts` | Show which hosts each play targets | "Is this really staging?" |
| `--list-tasks` | Show the tasks that would run | Review; with `--tags` (topic 33) |
| `-l` / `--limit PATTERN` | Restrict to hosts matching a pattern | Canary runs, single-host fixes |
| `--limit @file` | Limit to hosts listed in a file | Re-run only failed hosts (from a saved list) |
| `--start-at-task "name"` | Skip tasks before the named one | Resume a long playbook after fixing a failure |
| `--step` | Ask before each task (`y`/`n`/`c`) | Careful manual walk-through |
| `-f N` | Forks | Go slower (`-f 1`) during risky changes |
| `-e var=value` | Extra variable | Override for one run (highest precedence, topic 25) |

⚠️ `--start-at-task` skips earlier tasks **including** ones that set variables or register results; later tasks may then fail. Use it only when you understand the dependencies.

---

## Why it exists

Ansible changes real servers immediately. Without a preview:

- A wrong variable changes 300 config files at once.
- A regex in `lineinfile` matches the wrong line and you only find out when sshd restarts.
- Someone's manual "temporary" change gets overwritten and you don't know what it was.

Check + diff gives you a **plan** you can read and review (and paste into a change ticket or PR comment) before anything happens. `--limit` lets you apply to **one** server first and watch it. These two habits prevent most automation outages.

---

## Analogy

**Renovating with a contractor.**

- `--check` = the contractor walks through the house and tells you: "I'd repaint this room, replace that tap, leave the kitchen".
- `--diff` = they show you **paint swatches**: old colour vs new colour, room by room.
- Some jobs can't be previewed ("I'd run this mysterious machine": `command`), so they say "can't tell you, I'll skip it in the walk-through".
- The walk-through can't account for work not yet done ("I'd paint the new wall"… but the wall isn't built yet): that's why later tasks fail in check mode on new servers.
- `--limit` = "do the guest bathroom first and let me look before you do the rest".

---

## How it works inside

```
 ansible-playbook files.yml --check --diff
   │
   ├─ every task gets internal args: _ansible_check_mode=true, _ansible_diff=true
   │
   ├─ copy module (check_mode: full, diff: full)
   │     compare local checksum vs remote → different
   │     read old content, compute new content → build {"diff": {"before": "...", "after": "..."}}
   │     check mode → DON'T write → return changed=true + diff
   │
   ├─ command module without creates/removes (check_mode: none)
   │     → action plugin returns skipped ("check mode not supported for this module")
   │
   ├─ task with check_mode: false
   │     → internal arg forced to false → really runs
   │
   └─ callback prints the diff before the "changed:" line
```

Modules must implement check mode **themselves**; that's why support varies, and why custom or community modules sometimes behave badly in check mode. `ansible-doc` tells you.

---

## Code / commands

### Check + diff on a file change

Edit `files/app.conf`: `listen = 8080` → `listen = 9090`. Then:

```bash
ansible-playbook files.yml --check --diff -l web1
```
```
TASK [App config is in place] **********************************************************
--- before: /opt/app/app.conf
+++ after: /home/you/ansible-lab/files/app.conf
@@ -1,4 +1,4 @@
 # app.conf: managed by Ansible
-listen = 8080
+listen = 9090
 workers = 4
 log_level = info

changed: [web1]
...
PLAY RECAP *****************************************************************************
web1   : ok=8  changed=1  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

- `--- before:` / `+++ after:`: remote file now vs what it would become.
- `-listen = 8080` / `+listen = 9090`: removed and added lines.
- `changed: [web1]`: **would** change. Prove nothing happened:

```bash
ssh ansible@172.30.0.11 'grep listen /opt/app/app.conf'     # still 8080
```

Now apply for real on the canary only, then everyone:

```bash
ansible-playbook files.yml --diff -l web1
ansible-playbook files.yml --diff
```

### Diff for lineinfile and blockinfile

```
TASK [sshd allows only 3 auth tries] ***************************************************
--- before: /etc/ssh/sshd_config (content)
+++ after: /etc/ssh/sshd_config (content)
@@ -38,7 +38,7 @@
 #LoginGraceTime 2m
 #PermitRootLogin prohibit-password
 #StrictModes yes
-MaxAuthTries 3
+MaxAuthTries 4
 #MaxSessions 10
```

### Commands in check mode

```yaml
- name: Show uptime
  ansible.builtin.command: uptime
```
```
TASK [Show uptime] *********************************************************************
skipping: [web1]
```

With `-v` you'd see `"msg": "Command would have run if not in check mode"`. Make a **read-only** command run anyway:

```yaml
- name: Read the current app version (safe in check mode)
  ansible.builtin.command: cat /opt/app/releases/release-1.0/VERSION
  register: app_version
  changed_when: false
  check_mode: false
```

### The `ansible_check_mode` variable

```yaml
- name: Tell people this is a dry run
  ansible.builtin.debug:
    msg: "DRY RUN: nothing will change"
  when: ansible_check_mode
```

### Limit tricks

```bash
ansible-playbook site.yml -l web1                  # one host
ansible-playbook site.yml -l 'web:&ubuntu'         # Ubuntu web servers
ansible-playbook site.yml -l 'all:!db'             # all except db
printf 'web2\nlb1\n' > /tmp/retry.txt
ansible-playbook site.yml -l @/tmp/retry.txt       # hosts from a file
```

### Step through a playbook

```bash
ansible-playbook files.yml --step -l web1
```
```
Perform task: TASK: App directories exist (N)o/(y)es/(c)ontinue: y
...
Perform task: TASK: sshd allows only 3 auth tries (N)o/(y)es/(c)ontinue: n
```

- `y`: run this task. `n`: skip it. `c`: run all remaining tasks without asking.

### Resume at a task

```bash
ansible-playbook files.yml --start-at-task "Release archive is unpacked"
```

### The safe production run procedure

```
 1. ansible-playbook site.yml --syntax-check
 2. ansible-playbook site.yml -i inventories/prod --list-hosts         # right hosts?
 3. ansible-playbook site.yml -i inventories/prod --check --diff       # read every diff
      → paste the summary into the change ticket / PR
 4. ansible-playbook site.yml -i inventories/prod --diff -l canary1    # one host for real
      → check the service: curl, logs, monitoring
 5. ansible-playbook site.yml -i inventories/prod --diff               # the rest
      (big fleets: serial batches + max_fail_percentage, topic 45)
 6. ansible-playbook site.yml -i inventories/prod                      # run once more: expect changed=0
```

Step 6 is the idempotency proof: if a second run still changes things, something's not idempotent (topic 27).

---

## Hands-on lab

1. Change `listen` in `files/app.conf`, run `files.yml --check --diff`, confirm with `ssh` that nothing changed. Apply on web1 only, then on all.
2. Change the `MaxAuthTries` value, run `--check --diff`, read the diff.
3. Add a `command: uptime` task and run with `--check`: it's skipped. Add `check_mode: false` + `changed_when: false` and run again: it runs.
4. **Check mode on a fresh host**: on a clean lab (`nodes/reset.sh`), run `web.yml --check` from topic 14. Watch "nginx is installed" say `changed` and "nginx is running" **fail** (service doesn't exist). Add `ignore_errors: "{{ ansible_check_mode }}"` to the service task and run `--check` again.
5. Use `--step` and skip one task.
6. Write the 6-step safe procedure as a shell script `safe-run.sh` that stops between steps and asks you to press Enter (`read -p`), and use it on `files.yml`.

**Done when:** you never run a changed playbook without first reading its `--check --diff` output.

---

## Where you'll use it

- **Every production change**: `--check --diff` output pasted into the change request; reviewers approve the *plan*.
- **CI on pull requests** (topic 53): a check-mode run against staging (or prod, read-only credentials permitting) posts the diff as a PR comment.
- **Drift detection**: a scheduled `--check --diff` run shows what has drifted since the last apply, without fixing it (topic 55).
- **Canaries**: `--limit` to one host or one group before the rest.
- **Incident recovery**: `--start-at-task` and `--limit @file` to finish a run that failed halfway.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `skipping` for `command`/`shell` in `--check` | No check-mode support | `creates`/`removes`, or `check_mode: false` for read-only commands (topic 27) |
| Service/template task fails only in `--check` | Earlier task's change was only predicted (package/dir not really there) | `when: not ansible_check_mode` or `ignore_errors: "{{ ansible_check_mode }}"` on the dependent task |
| `--check` says `changed` but the real run says `ok` | Module's prediction is approximate (some modules always predict change) | Know your modules; read the diff, not only the counter |
| `--diff` shows nothing for a task | Module doesn't support diff mode (e.g. `user`, `package`) | Check ATTRIBUTES; use `-v` to see the result |
| Secrets visible in diff output / CI logs | Diff prints file contents | `diff: false` + `no_log: true` on those tasks |
| `--start-at-task` run fails with undefined variables | Skipped tasks set facts or registered results | Run from the start, or make tasks independent |
| `--limit` matched nothing: `skipping: no hosts matched` | Pattern typo, wrong inventory | `--list-hosts` first |
| A `--check` run changed something | A task had `check_mode: false` and wasn't read-only, or a custom module ignored check mode | Audit `check_mode: false` tasks; test custom modules in check mode |

---

## Quick recap

- `--check` = **predict** (no changes); `--diff` = **show** file differences; `--check --diff` = the plan.
- Modules support check mode fully, partially or not at all (`ansible-doc` ATTRIBUTES). Unsupported tasks (`command`, `shell`, `raw`) are **skipped**.
- Task keywords: `check_mode: false` (really run read-only tasks), `check_mode: true` (never change), `diff: false` (hide secrets). Variable: `ansible_check_mode`.
- Check mode can't see predicted changes from earlier tasks, so dependent tasks can fail on fresh hosts.
- Safe procedure: syntax-check → list-hosts → check+diff → canary with `--limit` → all → re-run expecting `changed=0`.

### Interview questions

1. **Q: What's the difference between `--check` and `--diff`?**
   A: `--check` runs a dry run where modules report what they would change without changing anything. `--diff` shows before/after differences for supported modules; it can be used with or without `--check`.
2. **Q: Why are `command` and `shell` tasks skipped in check mode?**
   A: Ansible can't know what an arbitrary command would do, so it can't predict safely. With `creates`/`removes` it can predict; or set `check_mode: false` for read-only commands.
3. **Q: How do you apply a change to one server first?**
   A: `--limit` with that host or a canary group, verify, then run without the limit (or with `serial` batches).

### Scenario question (answer out loud first, then compare)

> *"Your change board requires a 'plan' for every Ansible change, like Terraform's plan. How do you produce one, and what are its limits you'd warn them about?"*

**Model answer:**
1. Produce it with `ansible-playbook site.yml -i inventories/prod --check --diff`, ideally in CI from the PR branch, saving the output as an artifact or PR comment. Summarise per host from the recap (`changed=N`).
2. Warn about limits:
   - modules without check-mode support (commands, scripts, some API modules) are **skipped**, so the plan is incomplete for those tasks;
   - tasks depending on earlier predicted changes may fail or be wrong in check mode (mostly on new servers);
   - some modules only approximate their prediction;
   - Ansible has no state file, so the plan reflects the servers **at that moment**. A drift between plan and apply changes the outcome.
3. Make it better: keep `command`/`shell` rare and guarded (`creates`, `changed_when`), mark read-only commands `check_mode: false`, apply to a canary first, and keep `--diff` on for the real run so the log shows what really happened.

**Likely follow-ups:**
- *How do you avoid leaking secrets in the plan?* `diff: false` and `no_log: true` on secret-bearing tasks; 2.22 masks vault values and `no_log` values as `$REDACTED$`.
- *Can you block an apply if the plan shows unexpected changes?* Yes, in CI: parse the JSON callback output (`ANSIBLE_STDOUT_CALLBACK=json`) and require approval when `changed > 0` for protected hosts (topic 53).
