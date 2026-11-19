# 27 · Idempotency in Depth

> **Level:** ★★ intermediate
>
> **You will learn:** exactly what **idempotency** means (and the three properties a good task must have), **why it matters** in production (safe re-runs, drift repair, meaningful `changed`, check mode, handlers, CI), every common way to **break** it (`command`, `shell`, `raw`, `script`, `restarted`, `latest`, timestamps, random salts, downloads without checksums, bad regexes), and the full **toolbox to fix it**: use a real module, **`creates`/`removes`**, **`changed_when`**, **`failed_when`**, check-then-act with `register` + `when`, idempotent commands, handlers, pinned versions. You'll take a deliberately broken playbook from `changed=9` to `changed=0` and learn how to **test** idempotency automatically.
>
> **You need:** [26 · register and when](26-register-and-when.md), [17](17-modules-packages-services-users.md) and [18](18-modules-files.md) (the modules), [20 · check mode](20-check-diff-limit.md), [01 · CM basics](01-what-is-configuration-management.md) (the first definition).
>
> **Part 3 · Variables, facts and logic** · Next: [28 · Loops](28-loops.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A task is **idempotent** if running it **again** on a server that's already correct **changes nothing and says so** (`ok`, not `changed`). A whole playbook is idempotent if the **second run reports `changed=0`**.

**Picture it.** A light switch labelled "**ON**" (idempotent: press it 10 times, the light is on) vs a switch labelled "**TOGGLE**" (not idempotent: the result depends on how many times you pressed it).

**The one thing to know.** The fixes, in order of preference:

```yaml
# 1. Use a module instead of a command          (best)
- ansible.builtin.user: { name: svc_app, system: true }

# 2. Tell command/shell when NOT to run
- ansible.builtin.command: /opt/app/bin/init-db
  args:
    creates: /var/lib/app/.initialized         # skip if this file exists

# 3. Tell Ansible what "changed" and "failed" really mean
- ansible.builtin.command: cat /etc/os-release
  register: osr
  changed_when: false                          # read-only: never "changed"

- ansible.builtin.command: grep -q '^PermitRootLogin no' /etc/ssh/sshd_config
  register: prl
  changed_when: false
  failed_when: prl.rc not in [0, 1]            # 1 = "not found" is an answer, not an error
```

**Try it now:** run any playbook from earlier topics twice. If the second run shows `changed` > 0, you've found a task to fix with this topic.

**You've got it if you can:** explain why `command` always reports `changed`, and fix it three different ways.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### The definition

In maths, an operation `f` is idempotent if `f(f(x)) = f(x)`: doing it twice is the same as doing it once. "Set the volume to 5" is idempotent. "Turn the volume up by 1" isn't.

For Ansible, a good task has **three properties**:

| # | Property | Means | Example of breaking it |
|---|---|---|---|
| 1 | **Converges** | Whatever the starting state, after the task the system is in the desired state | `echo x >> file` gives a different file depending on how many times it ran |
| 2 | **Stable** | If the system is already in the desired state, the task **does nothing** | `systemctl restart nginx` restarts even when nothing changed |
| 3 | **Honest** | It reports `changed` **only** when it actually changed something | `command: cat /etc/hosts` reports `changed` but changed nothing |

A task can be safe in effect but dishonest in reporting. `command: mkdir -p /opt/x` is stable and converges (mkdir -p is harmless), but reports `changed` every run. That still matters (see "Why it matters"). Real idempotency in Ansible means **all three**.

### Where idempotency comes from

Modules are idempotent because they **check before they act** (topic 01's loop):

```
 desired state (your task)  vs  actual state (module reads the system)
            same → do nothing → changed: false
       different → smallest change to fix it → changed: true
```

`command`, `shell`, `raw` and `script` **can't** check: they don't know what your command does. So they always run it and always report `changed: true`. It's **your job** to add the check.

### Everything that breaks idempotency

| Cause | Why | Fix |
|---|---|---|
| `command` / `shell` / `raw` / `script` | Always run, always `changed` | Module; `creates`/`removes`; `changed_when` |
| Commands that fail the 2nd time (`mkdir`, `useradd`, `createdb`) | Not safe to repeat | Module, or `creates`, or an idempotent command (`mkdir -p`) |
| Commands that accumulate (`echo >> file`, `INSERT`, `iptables -A`) | Each run adds more | `lineinfile`/`blockinfile`/`copy`; proper modules (`firewalld`, DB modules) |
| `state: restarted` / `reloaded` | Always acts | `state: started` + a **handler** for restarts (topic 30) |
| `state: latest` | Changes whenever the repo updates | `present` + pinned versions; patch on purpose |
| Templates with timestamps (`{{ now() }}`, `ansible_date_time`) | Content differs every run | Don't put run-time values in files |
| `password_hash` without a fixed salt | New hash each run | `update_password: on_create` (topic 17) |
| Random/generated values (`random`, `uuid`) in tasks | Different every run | Generate once, store it (vault, `lookup('password', ...)` with a file) |
| `get_url` without `checksum`, or with `force: true` | Downloads or rewrites needlessly | `checksum: sha256:...` |
| `file: state=touch` | Updates timestamps | `access_time: preserve`, `modification_time: preserve` |
| `lineinfile` with a regexp that doesn't match the new line | Appends every run (or after drift) | Regexp that matches old **and** new (topic 18) |
| Two tasks managing the same thing differently | They undo each other each run ("flapping") | One owner per file/setting |
| `git` with `version: main` | Changes when upstream moves | Pin a tag/commit for deploys (or accept it as intended) |
| `uri` with `POST`/`PUT` | Each call creates/updates | Check first (`GET`), `status_code`, `changed_when` |

### The toolbox

**1. Use a module.** The best fix by far:

| Instead of | Use |
|---|---|
| `command: useradd x` | `ansible.builtin.user` |
| `command: mkdir /x` | `ansible.builtin.file: state=directory` |
| `shell: echo "a=b" >> /etc/x.conf` | `ansible.builtin.lineinfile` / `copy` / `template` |
| `command: systemctl enable --now x` | `ansible.builtin.systemd_service` |
| `shell: firewall-cmd --add-port=80/tcp --permanent` | `ansible.posix.firewalld` |
| `command: setsebool -P x on` | `ansible.posix.seboolean` |
| `shell: curl -o /tmp/x URL` | `ansible.builtin.get_url` + `checksum` |
| `command: tar xzf x.tgz -C /opt` | `ansible.builtin.unarchive` + `creates` |
| `command: git clone ...` | `ansible.builtin.git` |
| `shell: crontab -l \| ... \| crontab -` | `ansible.builtin.cron` |
| `command: sysctl -w x=y` | `ansible.posix.sysctl` |
| `shell: mysql -e "CREATE DATABASE x"` | `community.mysql.mysql_db` |
| `command: docker run ...` | `community.docker.docker_container` |
| `command: kubectl apply -f x.yml` | `kubernetes.core.k8s` |

Search first: `ansible-doc -l | grep -i <thing>` (topic 16).

**2. `creates` and `removes`** (arguments of `command` and `shell`, under `args:` or directly in the module's options):

| Argument | Task runs only if... | Use for |
|---|---|---|
| `creates: /path` | `/path` does **not** exist | One-time setup that leaves a file behind (init scripts, installers, `kubeadm init`) |
| `removes: /path` | `/path` **does** exist | Cleanup commands ("only if there's something to clean") |

They also make `command`/`shell` **work in check mode**: Ansible can predict "would run" or "would skip" (topic 20). When the command doesn't naturally create a file, create a **marker file** yourself as part of the command.

**3. `changed_when`**: override what counts as "changed". It takes a condition (like `when`), evaluated **after** the task runs, usually on the registered result:

| Situation | `changed_when` |
|---|---|
| Read-only command (cat, grep, ls, status) | `false` |
| Command prints something when it changes | `"'Created' in result.stdout"` |
| Command prints something when nothing to do | `"'already' not in result.stdout"` |
| Exit code signals change | `result.rc == 2` |
| Several tasks' combined effect | `a is changed or b is changed` |

**4. `failed_when`**: override what counts as "failed":

| Situation | `failed_when` |
|---|---|
| `grep` (rc 1 = no match, rc 2 = error) | `result.rc not in [0, 1]` |
| Tool exits non-zero for "already exists" | `result.rc != 0 and 'already exists' not in result.stderr` |
| Output contains an error even with rc 0 | `"'ERROR' in result.stdout"` |
| Never fail (read-only probe) | `false` |

Multiple conditions in a list are **AND**ed: `failed_when: [result.rc != 0, "'exists' not in result.stderr"]`.

**5. Check, then act** with `register` + `when` (topic 26):

```yaml
- name: Is the schema already at version 42?
  ansible.builtin.command: /opt/app/bin/schema-version
  register: schema
  changed_when: false
  check_mode: false               # safe to run even in --check

- name: Migrate the schema
  ansible.builtin.command: /opt/app/bin/migrate --to 42
  when: schema.stdout | int < 42
```

**6. Write idempotent commands**: `mkdir -p`, `ln -sfn`, `CREATE TABLE IF NOT EXISTS`, `useradd` guarded by `getent passwd x ||`, tools with `--if-not-exists` or `--dry-run` + compare.

**7. Handlers for restarts** (topic 30): `notify: restart nginx` from the config task; the restart happens **only** if the config changed.

**8. Pin versions** instead of `latest` (topic 17).

### Not everything must be idempotent

Some playbooks exist **to act**: "restart the app now", "run the backup now", "rotate keys now". That's fine. Make the intent obvious:

- keep them in **separate** playbooks (`restart-app.yml`, `rotate-keys.yml`), not in the baseline,
- or tag them `never` so they only run when asked (`--tags restart`, topic 33),
- name them as actions ("Restart app"), not states.

The rule is: your **configuration** playbooks must converge with `changed=0`; **operational** playbooks are explicit actions.

---

## Why it exists

Idempotency is what makes configuration management safe at all. Without it:

| Without idempotency | With idempotency |
|---|---|
| A playbook that failed halfway can't be safely re-run (steps 1–5 would run again and break) | Fix the problem, re-run the whole playbook; finished steps report `ok` |
| You can't run automation on a schedule (each run causes changes, restarts, duplicates) | Run every hour to repair drift; healthy servers see `changed=0` |
| `changed` means nothing (it's always 20) | `changed=1` means "something drifted" — a **signal** you can alert on |
| `--check` shows 20 "would change" items every time; nobody reads it | `--check` output = the real list of differences |
| Handlers fire every run (needless restarts, dropped connections) | Restarts only after real config changes |
| Rolling deploys/patching are risky | Re-running a batch is harmless |
| Audits can't tell real changes from noise | Change logs contain only real changes |
| CI can't verify roles | CI runs twice and fails if run 2 changed anything |

---

## Analogy

**A thermostat vs a heater button.**

- A **thermostat** set to 21°C: if the room is 21°C, nothing happens. If it's 18°C, it heats until 21°C. Set it again to 21°C: still nothing. **Idempotent**, converges, honest ("heating" light only when heating).
- A **"heat for 10 minutes" button**: each press heats more. Press it 5 times and the room is a sauna. **Not idempotent.**
- `command` is the button. Modules are thermostats. `creates:` is a note on the button saying "don't press if the room is already warm". `changed_when: false` is covering the "heating" light for a button that only *reads* the temperature.

---

## How it works inside

Where each tool acts in a task's life:

```
 task: command: X   (with creates, when, changed_when, failed_when, register)
   │
   ├─ 1. when:           evaluated on the control node        → false? skip (no SSH)
   ├─ 2. creates/removes: checked ON THE TARGET by the module  → path says skip? return "skipped"/ok,
   │                                                             command NOT run
   ├─ 3. module runs X   → rc, stdout, stderr
   ├─ 4. failed_when:     evaluated on the control node with the result → overrides "failed"
   ├─ 5. changed_when:    evaluated on the control node with the result → overrides "changed"
   ├─ 6. register:        stores the (overridden) result
   └─ 7. notify:          if (overridden) changed → queue handlers
```

So `changed_when` doesn't stop a command from running; it only **corrects the report** (and therefore whether handlers fire). To stop it running, use `creates`/`removes` or `when`.

---

## Code / commands

### The broken playbook

`~/ansible-lab/idem-bad.yml`:

```yaml
---
- name: Idempotency, broken on purpose
  hosts: web
  become: true
  tasks:
    - name: 1. App env file
      ansible.builtin.shell: echo "export APP_ENV=prod" >> /etc/profile.d/app.sh

    - name: 2. Service account
      ansible.builtin.command: useradd --system svc_app
      ignore_errors: true

    - name: 3. Data directory
      ansible.builtin.command: mkdir /opt/data
      ignore_errors: true

    - name: 4. Initialise the app (one-time)
      ansible.builtin.shell: |
        mkdir -p /var/lib/app
        echo "initialised at $(date)" >> /var/log/app-init.log

    - name: 5. Read the OS release
      ansible.builtin.command: cat /etc/os-release
      register: osr

    - name: 6. Is root login disabled?
      ansible.builtin.command: grep -q '^PermitRootLogin no' /etc/ssh/sshd_config
      register: prl
      ignore_errors: true

    - name: 7. Restart nginx
      ansible.builtin.systemd_service:
        name: nginx
        state: restarted

    - name: 8. Build stamp file
      ansible.builtin.copy:
        content: "built {{ now() }}\n"
        dest: /opt/data/BUILD
        mode: "0644"

    - name: 9. Release archive is on the host
      ansible.builtin.copy:
        src: release-1.0.tar.gz
        dest: /tmp/release-1.0.tar.gz
        mode: "0644"

    - name: 10. Unpack the release
      ansible.builtin.command: tar xzf /tmp/release-1.0.tar.gz -C /opt/data
```

(Task 9 is fine already: `copy` is a module. It's there so task 10 has something to unpack.)

```bash
ansible-playbook idem-bad.yml; ansible-playbook idem-bad.yml
```

Second run:

```
TASK [2. Service account] **************************************************************
fatal: [web1]: FAILED! => {"changed": true, "cmd": ["useradd", "--system", "svc_app"], "msg": "non-zero return code", "rc": 9, "stderr": "useradd: user 'svc_app' already exists", ...}
...ignoring
TASK [3. Data directory] ***************************************************************
fatal: [web1]: FAILED! => {"changed": true, "cmd": ["mkdir", "/opt/data"], "rc": 1, "stderr": "mkdir: cannot create directory '/opt/data': File exists", ...}
...ignoring
...
PLAY RECAP *****************************************************************************
web1   : ok=11  changed=9  unreachable=0  failed=0  skipped=0  rescued=0  ignored=3
web2   : ok=11  changed=9  unreachable=0  failed=0  skipped=0  rescued=0  ignored=3
```

- `changed=9` on a run where **nothing needed to change**: every task except the `copy` in task 9. (Ignored failures still count as `ok` and, if the result says `changed: true`, as `changed`.)
- `ignored=3`: errors hidden with `ignore_errors`, a classic smell. (`ignore_errors` hides real failures too.)
- On the servers: `/etc/profile.d/app.sh` has 2 identical lines, `/var/log/app-init.log` has 2 entries, nginx restarted twice.

### The fixed playbook

`~/ansible-lab/idem-good.yml`:

```yaml
---
- name: Idempotency, fixed
  hosts: web
  become: true
  tasks:
    - name: 1. App env file has exactly this content
      ansible.builtin.copy:
        dest: /etc/profile.d/app.sh
        content: "export APP_ENV=prod\n"
        mode: "0644"

    - name: 2. Service account exists
      ansible.builtin.user:
        name: svc_app
        system: true
        create_home: false

    - name: 3. Data directory exists
      ansible.builtin.file:
        path: /opt/data
        state: directory
        mode: "0755"

    - name: 4. App is initialised (only once)
      ansible.builtin.shell: |
        mkdir -p /var/lib/app
        echo "initialised at $(date)" >> /var/log/app-init.log
        touch /var/lib/app/.initialized
      args:
        creates: /var/lib/app/.initialized

    - name: 5. Read the OS release (read-only)
      ansible.builtin.command: cat /etc/os-release
      register: osr
      changed_when: false
      check_mode: false

    - name: 6. Is root login disabled? (probe, never fails on "not found")
      ansible.builtin.command: grep -q '^PermitRootLogin no' /etc/ssh/sshd_config
      register: prl
      changed_when: false
      failed_when: prl.rc not in [0, 1]
      check_mode: false

    - name: 7. nginx is running
      ansible.builtin.systemd_service:
        name: nginx
        state: started

    - name: 8. Build stamp file (no run-time values in content)
      ansible.builtin.copy:
        content: "release 1.0\n"
        dest: /opt/data/BUILD
        mode: "0644"

    - name: 9. Release archive is on the host
      ansible.builtin.copy:
        src: release-1.0.tar.gz
        dest: /tmp/release-1.0.tar.gz
        mode: "0644"

    - name: 10. Release is unpacked (once)
      ansible.builtin.unarchive:
        src: /tmp/release-1.0.tar.gz
        dest: /opt/data
        remote_src: true
        creates: /opt/data/release-1.0/VERSION
```

What each fix does:

| Task | Fix | Toolbox # |
|---|---|---|
| 1 | `copy` with the full content: the file *is* exactly this | 1 (module) |
| 2 | `user` module | 1 |
| 3 | `file` module | 1 |
| 4 | Marker file + `creates` | 2 |
| 5 | `changed_when: false` (+ `check_mode: false` so it also runs in `--check`) | 3 |
| 6 | `changed_when: false` + `failed_when: rc not in [0, 1]`; no `ignore_errors` | 3, 4 |
| 7 | `started` instead of `restarted` (restart on change via handlers, topic 30) | 7 |
| 8 | No timestamp in content | — |
| 10 | `unarchive` + `creates` instead of `tar` | 1, 2 |

Clean up the damage of the broken run first, then run the good one twice:

```bash
ansible web -b -m ansible.builtin.file -a "path=/etc/profile.d/app.sh state=absent"
ansible-playbook idem-good.yml
ansible-playbook idem-good.yml
```

Second run:

```
PLAY RECAP *****************************************************************************
web1   : ok=11  changed=0  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
web2   : ok=11  changed=0  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

Task 4 on the second run shows how `creates` reports:

```
TASK [4. App is initialised (only once)] ***********************************************
ok: [web1]
```

With `-v`: `"msg": "Did not run command since '/var/lib/app/.initialized' exists"`.

### `changed_when` from output

Some tools tell you whether they did something. Example: a hypothetical `appctl add-user` prints `created` or `exists`:

```yaml
- name: App admin user exists in the app's own database
  ansible.builtin.command: /opt/app/bin/appctl add-user admin
  register: appuser
  changed_when: "'created' in appuser.stdout"
  failed_when:
    - appuser.rc != 0
    - "'exists' not in appuser.stdout"
```

- Changed only when it really created the user.
- Fails only if rc ≠ 0 **and** the output isn't "exists" (list = AND).

### Test idempotency automatically

A tiny script for CI or your own use:

```bash
#!/usr/bin/env bash
# idem-test.sh PLAYBOOK [ansible-playbook args...]
set -euo pipefail
pb="$1"; shift
ansible-playbook "$pb" "$@"                                   # run 1: converge
out=$(ansible-playbook "$pb" "$@" | tee /dev/stderr)          # run 2: must not change
if grep -Eq 'changed=[1-9]' <<<"$out"; then
  echo "❌ NOT IDEMPOTENT: second run reported changes" >&2; exit 1
fi
echo "✅ idempotent"
```

```bash
chmod +x idem-test.sh
./idem-test.sh idem-good.yml          # ✅ idempotent
./idem-test.sh idem-bad.yml           # ❌ NOT IDEMPOTENT
```

Molecule (topic 51) has this built in as its **idempotence** step. A third check: after applying, `--check --diff` should show **no** changes.

### Finding the offenders in a big playbook

```bash
ansible-playbook site.yml | grep -B1 '^changed:'          # which tasks changed on run 2
ANSIBLE_CALLBACKS_ENABLED=ansible.posix.profile_tasks ansible-playbook site.yml   # slow + changed tasks
ansible-lint site.yml                                     # flags no-changed-when, command-instead-of-module...
```

`ansible-lint` (topic 50) has rules exactly for this: `no-changed-when` (command/shell without `changed_when`), `command-instead-of-module`, `command-instead-of-shell`, `package-latest`, `no-handler`.

---

## Hands-on lab

1. Write `idem-bad.yml`, run it twice, and record the second recap. Look at the damage on web1: `cat /etc/profile.d/app.sh`, `cat /var/log/app-init.log`.
2. Write `idem-good.yml`, clean up, run twice: second run `changed=0`.
3. Run `idem-good.yml --check` on a **fresh** lab (`nodes/reset.sh`, then re-run topic 14's `web.yml` so nginx exists). Notice tasks 5–6 still run (`check_mode: false`) and task 4 predicts correctly thanks to `creates`.
4. Create `idem-test.sh` and use it on every playbook you've written so far (`web.yml`, `base.yml`, `files.yml`, `web-vars.yml`). Fix anything that isn't idempotent.
5. Write a `changed_when` for this command and verify both cases:

   ```yaml
   - ansible.builtin.shell: grep -q '^vm.swappiness' /etc/sysctl.d/99-lab.conf || { echo 'vm.swappiness = 10' >> /etc/sysctl.d/99-lab.conf; echo ADDED; }
     register: swp
   ```

   (Answer: `changed_when: "'ADDED' in swp.stdout"`.) Then replace it with the right module (`lineinfile`, or `ansible.posix.sysctl` on a VM).
6. Install ansible-lint (`uv pip install ansible-lint`) and run `ansible-lint idem-bad.yml`. Match each warning to a row of the "breaks" table.

**Done when:** every configuration playbook you own passes `idem-test.sh`.

---

## Where you'll use it

- **Every role and playbook review**: "does the second run report `changed=0`?" is the first quality gate.
- **CI**: Molecule's idempotence step, or the two-run script, blocks merges that break it (topics 51, 53).
- **Scheduled drift runs** (topic 55): only possible with idempotent playbooks; `changed > 0` then becomes an alert.
- **Recovering from failures**: re-run the full playbook with confidence.
- **Wrapping legacy scripts**: installers and vendor tools become safe with `creates`, marker files, `changed_when`, `failed_when`.
- **Interviews**: "What is idempotency and how do you keep `shell` tasks idempotent?" is near-guaranteed.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| Second run shows `changed` on a `command`/`shell` task | No change detection | Module, `creates`/`removes`, or `changed_when` |
| `ansible-lint`: `no-changed-when` | Same | Same |
| Second run fails with "already exists" | Command not safe to repeat | Module or `creates`, not `ignore_errors` |
| Service restarts every run | `state: restarted` in a normal task | `started` + handler (topic 30) |
| A file "changes" every run | Timestamps/random values in content, or two tasks fighting over it | Remove run-time values; one owner per file |
| `creates` task never runs, even on new servers | The marker path already exists for another reason | Use a unique marker path |
| `creates` task runs every time | The command doesn't actually create that path (typo, different path) | Make the command `touch` the marker |
| `changed_when` correct but the command still does damage on re-run | `changed_when` only fixes the **report** | Stop it running: `creates`/`removes`/`when` |
| `failed_when` hides real errors | Condition too broad (`failed_when: false` on a real action) | Only use broad overrides on read-only probes |
| `ignored=N` in every recap | `ignore_errors` used as an idempotency workaround | Replace with proper checks; `ignore_errors` hides real failures |
| Check mode skips your commands | `command`/`shell` without `creates` | `creates`/`removes`, or `check_mode: false` for read-only |

---

## Quick recap

- **Idempotent task** = converges, stable (no action when correct), honest (`changed` only when it changed). **Idempotent playbook** = the second run says `changed=0`.
- It matters for: safe re-runs, scheduled drift repair, a meaningful `changed` signal, useful check mode, handlers, rolling changes, audits, CI.
- `command`/`shell`/`raw`/`script` can't check anything: they always run and always say `changed`.
- Fix in order: **module** → **`creates`/`removes`** (+ marker files) → **`changed_when`/`failed_when`** → check-then-act with `register` + `when` → idempotent commands → **handlers** for restarts → **pinned versions**.
- Test it: run twice (`idem-test.sh`), Molecule idempotence, `--check --diff` after apply, `ansible-lint`.

### Interview questions

1. **Q: What is idempotency in Ansible?**
   A: Running a task or playbook multiple times gives the same end state as running it once, and when the system is already correct, nothing changes and Ansible reports `ok`. The second run of a correct playbook reports `changed=0`.
2. **Q: Why do `command` and `shell` break idempotency, and how do you fix it?**
   A: Ansible can't know what the command does, so it always runs it and reports changed. Fix by using a module; or `creates`/`removes` so it only runs when needed; or `changed_when`/`failed_when` to report correctly; or check first with a registered probe and `when`.
3. **Q: What's the difference between `changed_when` and `creates`?**
   A: `creates` decides **whether the command runs** (skips if the path exists). `changed_when` only decides **how the result is reported** after it ran.

### Scenario question (answer out loud first, then compare)

> *"You inherit a role with 40 tasks, half of them `shell`. It reports `changed=23` on every run, restarts the app every time, and nobody dares to run it on a schedule. How do you make it idempotent without breaking production?"*

**Model answer:**
1. **Measure**: run it twice against a staging (or Molecule) copy; list the tasks that change on run 2. `ansible-lint` gives a first list (`no-changed-when`, `command-instead-of-module`).
2. **Replace with modules** where one exists (users, files, packages, services, firewall, cron, sysctl, downloads).
3. **Guard the rest**: `creates`/`removes` with marker files for one-time steps; `changed_when`/`failed_when` based on output or exit codes; read-only probes with `changed_when: false` and `check_mode: false`.
4. **Fix restarts**: config tasks `notify` a handler; the service task uses `state: started`.
5. **Remove `ignore_errors`** used as a workaround; replace with explicit `failed_when`.
6. **Prove it**: add a Molecule scenario (or the two-run check) to CI; it must reach `changed=0`. Then run `--check --diff` against production, apply to a canary, then roll out in batches.
7. Only then schedule it, with `changed > 0` reported as drift.

**Likely follow-ups:**
- *What if a vendor installer has no way to detect "already installed"?* Check for something the installer leaves behind (a binary, a package, a version file) with `creates:` or a `stat`/version probe + `when`.
- *Is `changed_when: false` on everything a fix?* No: that hides real changes (handlers won't fire, drift is invisible). Only for read-only tasks.
