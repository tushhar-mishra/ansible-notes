# 26 · register and when

> **Level:** ★ foundation
>
> **You will learn:** how to **capture a task's result** with `register` and what's inside it (`rc`, `stdout`, `stdout_lines`, `stderr`, `changed`, `failed`, `skipped`, module-specific keys like `stat.exists`), how to run a task **only if** a condition is true with **`when`** (facts, variables, registered results, group membership), combining conditions (`and`, `or`, `not`, lists), the result tests (`is changed`, `is failed`, `is succeeded`, `is skipped`, `is defined`), and the **ansible-core 2.19+ rule that conditionals must be real booleans** (the most common upgrade error), with fixes.
>
> **You need:** [25 · Variable precedence](25-variable-precedence.md) (registered vars are level 19), [22 · Jinja2 basics](22-jinja2-basics.md) (expressions and tests), [23 · Facts](23-facts-and-magic-variables.md).
>
> **Part 3 · Variables, facts and logic** · Next: [27 · Idempotency in depth](27-idempotency-in-depth.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** `register: name` saves a task's JSON result into a variable (per host). `when: condition` runs a task only on hosts where the condition is true; elsewhere it's **skipped**. Together they let tasks react to what earlier tasks found.

**Picture it.** A nurse takes your temperature (`register: temp`), and the doctor gives medicine only if it's high (`when: temp.value > 38`).

**The one thing to know.**

```yaml
- name: Does the app config exist?
  ansible.builtin.stat:
    path: /opt/app/app.conf
  register: app_conf

- name: Warn if it's missing
  ansible.builtin.debug:
    msg: "app.conf is missing on {{ inventory_hostname }}"
  when: not app_conf.stat.exists

- name: Only on Debian-family hosts
  ansible.builtin.debug:
    msg: "I'm an Ubuntu/Debian box"
  when: ansible_facts['os_family'] == 'Debian'
```

`when:` is already an expression: **no `{{ }}`**. And it must evaluate to `true` or `false`.

**Try it now:** run the lab playbook below and see which hosts print `skipping`.

**You've got it if you can:** register a command's output and run the next task only if the output contains a given word.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### register

`register: my_result` stores the task's result for **this host** as the variable `my_result` (precedence level 19, topic 25). It exists for the rest of the run (later plays too).

Common keys:

| Key | Present for | Meaning |
|---|---|---|
| `changed` | all | Did the task change anything |
| `failed` | all (false if absent) | Did it fail |
| `skipped` | skipped tasks | `true` if `when` was false (the variable still gets set!) |
| `msg` | many | Message |
| `rc` | command-like modules | Exit code |
| `stdout`, `stderr` | command-like modules | Output as one string |
| `stdout_lines`, `stderr_lines` | command-like modules | Output split into a **list of lines** |
| `start`, `end`, `delta` | command-like modules | Timing |
| `cmd` | command-like modules | What ran |
| `results` | tasks with `loop` | List of per-item results (topic 28) |
| module-specific | | `stat.exists`, `stat.mode` (stat); `uid`, `home` (user); `status` (systemd_service); `json` (uri)... see RETURN VALUES in `ansible-doc` |

**⚠️ A skipped task still registers.** If the task was skipped, `my_result` = `{"changed": false, "skipped": true, "skip_reason": "..."}` and keys like `stdout` don't exist. Code that uses `my_result.stdout` later must handle that (`my_result is not skipped`, or `| default('')`).

### when

`when:` takes a **Jinja2 expression without braces** (Ansible adds them). It's evaluated **per host** (and per loop item, topic 28):

| Pattern | Example |
|---|---|
| Fact comparison | `when: ansible_facts['os_family'] == 'RedHat'` |
| Version check | `when: ansible_facts['distribution_version'] is version('24.04', '>=')` |
| Variable is set | `when: app_version is defined` |
| Boolean variable | `when: app_debug` (must be a real boolean) |
| String from `-e` | `when: app_debug \| bool` |
| Registered result | `when: app_conf.stat.exists` |
| Output contains | `when: "'active' in svc.stdout"` |
| Output not empty | `when: svc.stdout \| length > 0` |
| Exit code | `when: check.rc == 0` |
| Result tests | `when: config_task is changed` |
| Group membership | `when: "'web' in group_names"` / `when: inventory_hostname in groups['web']` |
| Check mode | `when: not ansible_check_mode` |

**Combining:**

```yaml
when: ansible_facts['os_family'] == 'Debian' and app_port | int > 1024

when: >
  (ansible_facts['distribution'] == 'Ubuntu' and ansible_facts['distribution_major_version'] | int >= 22)
  or ansible_facts['distribution'] == 'Debian'

when:                                  # a LIST = all must be true (AND)
  - ansible_facts['os_family'] == 'RedHat'
  - app_conf.stat.exists
  - not ansible_check_mode
```

**Quoting `when`:** if the expression **starts** with a quote character or `{`/`[`, YAML needs the whole thing quoted: `when: "'web' in group_names"`.

### Result tests

| Test | True when |
|---|---|
| `result is succeeded` (or `success`) | Task didn't fail |
| `result is failed` | Task failed (only reachable with `ignore_errors` or in `rescue`, topic 31) |
| `result is changed` | Task changed something |
| `result is skipped` | Task was skipped |
| `result is defined` | The variable exists at all |

Prefer tests over raw keys (`result is changed` rather than `result.changed`): they handle missing keys correctly.

### ⚠️ Conditionals must be booleans (ansible-core 2.19+)

Old Ansible "guessed" the truth of any value (`"yes"`, a non-empty string, a list). Since 2.19, a `when:` that doesn't produce a **boolean** is an **error**:

```
fatal: [web1]: FAILED! => {"msg": "Conditional result (True) was derived from value of type 'str' at '...'. Conditionals must have a boolean result."}
```

| ❌ Broken (non-boolean) | ✅ Fixed |
|---|---|
| `when: my_string` | `when: my_string \| length > 0` |
| `when: result.stdout` | `when: result.stdout \| length > 0` or `when: "'ok' in result.stdout"` |
| `when: my_list` | `when: my_list \| length > 0` |
| `when: enable_feature` (value `"true"` from `-e`) | `when: enable_feature \| bool` |
| `when: inventory_hostname` | `when: inventory_hostname == 'web1'` |
| `when: "{{ x }} == 1"` (nested template) | `when: x == 1` |
| `when: result` (a dict) | `when: result is succeeded` |

While migrating a large codebase, `ALLOW_BROKEN_CONDITIONALS = True` in `ansible.cfg` temporarily turns the error into a warning. Fix the code; don't leave it on.

### Undefined variables in `when`

```yaml
when: maintenance_mode          # if maintenance_mode isn't defined anywhere → error
when: maintenance_mode | default(false)
when: maintenance_mode is defined and maintenance_mode
```

---

## Why it exists

Servers differ and situations change. You need tasks that:

- run only on certain OS families or versions,
- run only if a file/service/package exists (or doesn't),
- react to a previous command's output ("if the migration is pending, run it"),
- are switched on/off by variables ("feature flags"),
- behave differently in check mode.

`register` provides the data, `when` makes the decision. Without them, every playbook would be a fixed list of steps, like a script without `if`.

---

## Analogy

**A pilot's checklist with branches.** "Check fuel gauge" (task + `register: fuel`). "If fuel below 20%, divert" (`when: fuel.level < 20`). "If flying over water, check life vests" (`when: "'ocean' in route"`). Each aircraft (host) reads its **own** gauges and follows only the branches that apply. And the rule since 2.19: the answer to "is fuel low?" must be **yes or no**, not "fuel gauge says 'probably'".

---

## How it works inside

```
 task: debug ... when: app_conf.stat.exists      (host web1)
   │
   ├─ 1. build web1's variables (includes app_conf, registered earlier for web1)
   ├─ 2. template the condition:  "{{ app_conf.stat.exists }}"  → True
   ├─ 3. result must be a bool (2.19+) → else error
   ├─ 4. False → report "skipping", register (if any) = {skipped: true}
   │     True  → run the task normally
   └─ 5. if the task has register: store its result JSON as web1's app_conf
```

- Conditions run **on the control node**; skipping a task costs **no SSH**.
- `changed_when`/`failed_when` (topic 27) are applied **before** the result is stored, so the registered `changed`/`failed` values reflect your overrides.

---

## Code / commands

### The lab playbook

`~/ansible-lab/register.yml`:

```yaml
---
- name: register and when
  hosts: all
  become: true

  tasks:
    - name: Does the app config exist?
      ansible.builtin.stat:
        path: /opt/app/app.conf
      register: app_conf

    - name: Report app.conf presence
      ansible.builtin.debug:
        msg: "app.conf exists ({{ app_conf.stat.mode }})"
      when: app_conf.stat.exists

    - name: Kernel version (read-only command)
      ansible.builtin.command: uname -r
      register: kernel
      changed_when: false

    - name: Show the kernel, using stdout
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} runs kernel {{ kernel.stdout }}"

    - name: Collect service states
      ansible.builtin.service_facts:

    - name: Only where nginx is running
      ansible.builtin.debug:
        msg: "nginx is running here"
      when:
        - "'nginx.service' in ansible_facts['services']"
        - ansible_facts['services']['nginx.service']['state'] == 'running'

    - name: Is sshd configured with MaxAuthTries 3?
      ansible.builtin.command: grep -E '^MaxAuthTries 3$' /etc/ssh/sshd_config
      register: maxauth
      changed_when: false
      failed_when: false

    - name: Report hosts that are not hardened yet
      ansible.builtin.debug:
        msg: "MaxAuthTries not set to 3 (grep rc={{ maxauth.rc }})"
      when: maxauth.rc != 0

    - name: Only web servers on Ubuntu 24.04 or newer
      ansible.builtin.debug:
        msg: "modern Ubuntu web server"
      when:
        - "'web' in group_names"
        - ansible_facts['distribution'] == 'Ubuntu'
        - ansible_facts['distribution_version'] is version('24.04', '>=')

    - name: Feature flag from -e (string-safe)
      ansible.builtin.debug:
        msg: "feature X is ON"
      when: feature_x | default(false) | bool

    - name: React to a change
      ansible.builtin.copy:
        content: "marker\n"
        dest: /tmp/marker.txt
        mode: "0644"
      register: marker

    - name: Only when the previous task changed something
      ansible.builtin.debug:
        msg: "marker was (re)written"
      when: marker is changed
```

New bits, explained:

- `changed_when: false` on read-only commands: they never "change" anything, so don't report it (topic 27 explains fully).
- `failed_when: false` on the `grep`: grep exits 1 when nothing matches; that's information, not an error. We read `rc` instead.
- `ansible.builtin.service_facts` → `ansible_facts['services']`, a dictionary keyed by unit name (`'nginx.service'`).
- First condition in the list checks the key exists, so the second one doesn't fail on hosts without nginx (conditions in a list are evaluated in order and stop at the first false one).

### Run it

```bash
ansible-playbook register.yml
```
```
TASK [Report app.conf presence] ********************************************************
skipping: [db1]
skipping: [lb1]
ok: [web1] => { "msg": "app.conf exists (0644)" }
ok: [web2] => { "msg": "app.conf exists (0644)" }

TASK [Show the kernel, using stdout] ***************************************************
ok: [web1] => { "msg": "web1 runs kernel 6.6.87.2-microsoft-standard-WSL2" }
...
TASK [Only where nginx is running] *****************************************************
skipping: [db1]
skipping: [lb1]
ok: [web1] => { "msg": "nginx is running here" }
ok: [web2] => { "msg": "nginx is running here" }

TASK [Report hosts that are not hardened yet] ******************************************
skipping: [web1]
skipping: [web2]
ok: [db1] => { "msg": "MaxAuthTries not set to 3 (grep rc=1)" }
ok: [lb1] => { "msg": "MaxAuthTries not set to 3 (grep rc=1)" }

TASK [Only web servers on Ubuntu 24.04 or newer] ***************************************
skipping: [web1]
skipping: [db1]
skipping: [lb1]
ok: [web2] => { "msg": "modern Ubuntu web server" }

TASK [Feature flag from -e (string-safe)] **********************************************
skipping: [web1]
...
TASK [Only when the previous task changed something] ***********************************
ok: [web1] => { "msg": "marker was (re)written" }
...
```

- The kernel is the WSL2 kernel on every node: containers share it.
- Run again: the last task is **skipped** everywhere, because `/tmp/marker.txt` is already correct (`marker is changed` is false).
- `ansible-playbook register.yml -e feature_x=true`: now the feature task runs (the string `"true"` passes through `| bool`).

### Look at a registered variable

```yaml
- name: Show everything that was registered
  ansible.builtin.debug:
    var: kernel
```
```
ok: [web1] => {
    "kernel": {
        "changed": false,
        "cmd": ["uname", "-r"],
        "delta": "0:00:00.003104",
        "end": "2026-11-19 12:40:01.551203",
        "failed": false,
        "msg": "",
        "rc": 0,
        "start": "2026-11-19 12:40:01.548099",
        "stderr": "",
        "stderr_lines": [],
        "stdout": "6.6.87.2-microsoft-standard-WSL2",
        "stdout_lines": ["6.6.87.2-microsoft-standard-WSL2"]
    }
}
```

Always `debug: var=` a registered variable once to see its shape before writing conditions on it.

### The broken-conditional error, live

```yaml
- name: Broken on purpose
  ansible.builtin.debug:
    msg: "kernel output is non-empty"
  when: kernel.stdout
```
```
fatal: [web1]: FAILED! => {"msg": "Conditional result (True) was derived from value of type 'str' at '/home/you/ansible-lab/register.yml:...'. Conditionals must have a boolean result."}
```

Fix: `when: kernel.stdout | length > 0`.

### The skipped-register trap, live

```yaml
- name: Only on web
  ansible.builtin.command: cat /opt/app/app.conf
  register: conf
  changed_when: false
  when: "'web' in group_names"

- name: Use it everywhere (wrong)
  ansible.builtin.debug:
    msg: "{{ conf.stdout_lines | length }} lines"
```
```
fatal: [db1]: FAILED! => {"msg": "The task includes an option with an undefined variable. 'dict object' has no attribute 'stdout_lines'..."}
```

On db1 the first task was skipped, so `conf` = `{skipped: true, ...}`. Fix: add `when: conf is not skipped` to the second task.

---

## Hands-on lab

1. Run `register.yml` twice and explain each `skipping`.
2. Add `debug: var=app_conf` and find `stat.exists`, `stat.mode`, `stat.pw_name`.
3. Write a task pair: `command: systemctl is-active nginx` (register, `changed_when: false`, `failed_when: false`), then a debug that prints "nginx down!" only when `stdout != 'active'`. Stop nginx on web2 by hand and run again.
4. Reproduce the broken conditional (`when: kernel.stdout`), read the error, fix it two ways.
5. Reproduce the skipped-register trap and fix it with `is not skipped`.
6. Run with `-e feature_x=no` and `-e feature_x=yes`. Then remove `| bool` and run with `-e feature_x=yes`: read the error.
7. Write one `when` (list form) that is true only for AlmaLinux hosts in the `app` group with more than 1 vCPU.

**Done when:** you write conditions that always produce booleans and you check a registered variable's shape before using it.

---

## Where you'll use it

- **Multi-OS logic** where variables alone aren't enough (refresh apt cache only on Debian).
- **Guards**: "only migrate the DB if the schema version is old", "only reboot if `/var/run/reboot-required` exists" (`stat` + `when`).
- **Health checks** after changes: `uri` or `command` + `register` + `failed_when`/`until` (topics 27–28).
- **Feature flags** for gradual rollouts: `when: new_proxy_enabled | bool`.
- **Reporting**: collect `register`ed data and print or template a summary.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Conditionals must have a boolean result` | Non-boolean `when` (string, list, dict) — 2.19+ | `\| length > 0`, `\| bool`, `in`, or a real comparison |
| `error while evaluating conditional ... 'x' is undefined` | Variable not set on this host | `x is defined and x`, or `x \| default(false)` |
| `'dict object' has no attribute 'stdout'` on some hosts | The registering task was **skipped** there | `when: result is not skipped` / `\| default('')` |
| `when: "{{ var }}"` errors (or warns in old versions) | Nested template in `when` | Write `when: var` |
| YAML error on `when: 'web' in group_names` | Value starts with a quote | Quote the whole expression: `when: "'web' in group_names"` |
| Condition true but task "skipped" | `when` inherited from an enclosing block/include/role is false (topic 34) | Check parent conditions with `-vv` (`false_condition`) |
| `-e enabled=false` still enables | String `"false"` without `\| bool` (old versions treated non-empty as true) | Always `\| bool` for values from `-e` or INI inventories |
| Version check wrong (`10 < 9`) | String comparison of versions | `is version(...)` |
| Command task fails before your `when` can check `rc` | Non-zero exit = failure by default | `failed_when: false` (or a real `failed_when`, topic 27) |

---

## Quick recap

- **`register: x`** saves the task's result per host: `changed`, `failed`, `skipped`, `rc`, `stdout`, `stdout_lines`, module-specific keys. Inspect with `debug: var=x`.
- A **skipped** task still registers (`skipped: true`, no `stdout`): guard with `is not skipped`.
- **`when:`** = an expression without `{{ }}`, evaluated per host; list = AND; quote it if it starts with a quote.
- Use tests: `is defined`, `is changed`, `is failed`, `is succeeded`, `is skipped`, `is version(...)`.
- **2.19+: conditions must be booleans.** Use `| length > 0`, `| bool`, `in`, comparisons.

### Interview questions

1. **Q: How do you run a task only if a file exists on the target?**
   A: Use `ansible.builtin.stat` with `register: f`, then `when: f.stat.exists` on the next task.
2. **Q: What does a registered variable contain if its task was skipped?**
   A: Just `changed: false`, `skipped: true` and a skip reason; keys like `stdout` are missing, so later tasks must check `is not skipped`.
3. **Q: What changed about conditionals in ansible-core 2.19?**
   A: `when` expressions must evaluate to real booleans; strings, lists or other values that were previously "truthy" now cause an error. Convert explicitly (`| bool`, `| length > 0`, comparisons).

### Scenario question (answer out loud first, then compare)

> *"You need to reboot servers after patching, but only those that actually need it, and never more than one database server at a time. How do you express 'needs a reboot' in Ansible on both RHEL and Ubuntu?"*

**Model answer:**
1. Detect the need per family:
   - Ubuntu: `stat: path=/var/run/reboot-required` → `register: ubuntu_reboot`.
   - RHEL: `command: needs-restarting -r` (from `dnf-utils`/`yum-utils`), `register: rhel_reboot`, `changed_when: false`, `failed_when: rhel_reboot.rc not in [0, 1]` (rc 1 = reboot needed).
2. Combine into one fact: `set_fact: reboot_needed: "{{ (ubuntu_reboot.stat.exists | default(false)) or (rhel_reboot.rc | default(0) == 1) }}"`, with the detection tasks guarded by `when: ansible_facts['os_family'] == ...` and the expression handling skipped results via `default`.
3. Reboot with `ansible.builtin.reboot` `when: reboot_needed | bool`.
4. "One DB at a time": put the reboot in a play with `serial: 1` for the `db` group (topic 45), and health-check the database before moving on.

**Likely follow-ups:**
- *Why `default(...)` in the set_fact?* On each host one of the detection tasks was skipped, so its registered variable lacks `stat`/`rc`.
- *How do you make sure the check itself doesn't count as a change?* `changed_when: false` on read-only commands (topic 27).
