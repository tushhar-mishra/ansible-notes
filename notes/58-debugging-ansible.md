# 58 · Debugging Ansible

> **Level:** ★★ intermediate
>
> **You will learn:** a repeatable **debugging method** (reproduce, narrow, read, inspect, isolate, fix, prevent), and every tool for it: reading error messages by **phase** (parse vs run), **verbosity** levels, the `debug` module and `type_debug`, inspecting what Ansible **thinks** (`ansible-inventory --host`, `ansible-config dump`, `--list-*`), narrowing runs (`--limit`, `--tags`, `--start-at-task`, `--step`), the interactive **task debugger** (`debugger: on_failed`: print variables, change arguments, redo), **`ansible-console`**, readable output (`callback_result_format = yaml`), **`log_path`**, `ANSIBLE_DISPLAY_TRACEBACK`, `ANSIBLE_KEEP_REMOTE_FILES`, and `ANSIBLE_DEBUG`.
>
> **You need:** [57 · Project 2](57-project-baseline-and-web-stack.md) (something real to debug), [13 · Inside a module run](13-inside-a-module-run.md), [15 · Inside ansible-playbook](15-inside-ansible-playbook.md), [25 · Precedence](25-variable-precedence.md).
>
> **Part 12 · Production operations** · Next: [59 · Error catalogue](59-error-catalogue.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Most Ansible problems fall into four buckets: **connection** (can't reach/log in/escalate), **parsing** (YAML/syntax/missing module), **data** (wrong or undefined variable), **module** (the task ran and the module said no). Identify the bucket from the error, then use the matching tool.

**Picture it.** A doctor: symptoms (error message) → which system (bucket) → the right test (tool) → treatment (fix) → prevention (test/lint).

**The one thing to know.**

| Bucket | First tool |
|---|---|
| Connection (`UNREACHABLE`, `Permission denied`, `privilege escalation`) | `ansible HOST -m ping -vvvv`, then plain `ssh` |
| Parse (error before `PLAY [...]`) | `--syntax-check`, `ansible-lint`, read the file:line |
| Data (`is undefined`, wrong value) | `ansible-inventory --host HOST`, `debug: var=...`, `type_debug` |
| Module (`FAILED!` with a `msg`) | Read `msg`/`stderr`, `-vvv`, `ansible-doc`, the **task debugger** |

```yaml
- name: Something that fails sometimes
  ansible.builtin.template: { src: app.conf.j2, dest: /etc/app.conf }
  debugger: on_failed          # drops you into an interactive prompt on failure
```

**Try it now:** add `debugger: on_failed` to a task that uses an undefined variable, run it, and type `p task_vars['inventory_hostname']` at the prompt.

**You've got it if you can:** classify any error into one of the four buckets within seconds.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### The method

```
 1. REPRODUCE  : same command, same inventory, same versions (ansible --version, collection list)
 2. NARROW     : -l one host, --tags / --start-at-task / a tiny playbook; does it still fail?
 3. READ       : which phase? which host? which task (task path)? the exact msg/stderr
 4. INSPECT    : what Ansible thinks (inventory, variables, config) vs what the host has
 5. ISOLATE    : reproduce with an ad hoc command / playground / debugger
 6. FIX        : smallest change; re-run; second run idempotent
 7. PREVENT    : lint rule, assert, Molecule test, better error message
```

### Reading the error: where did it fail?

| You see | Phase / bucket | Typical meaning |
|---|---|---|
| `ERROR! ...` before any `PLAY [...]` | **Parse** | YAML, keywords, missing role/module/file in a static import |
| `[WARNING]: Could not match supplied host pattern` | Inventory | Pattern/inventory mismatch |
| `fatal: [h]: UNREACHABLE!` | **Connection** | SSH/DNS/network/auth (topic 05) |
| `msg: "Missing sudo password"` / `privilege escalation` | Connection (become) | sudo rules (topic 19) |
| `msg: "... is undefined"` / `template error` | **Data** | Variables, Jinja2 (topics 21–25) |
| `Conditional result ... must have a boolean result` | Data | 2.19 conditionals (topic 26) |
| `fatal: [h]: FAILED! => {"msg": ..., "rc": N, "stderr": ...}` | **Module** | The module or the command it ran failed |
| `MODULE FAILURE` + `module_stdout`/`module_stderr` | Module runtime | Python missing/broken, stdout pollution (topic 13) |
| `An exception occurred during task execution` | Module/plugin bug or environment | Use `ANSIBLE_DISPLAY_TRACEBACK` / `-vvv` |

Also note **which hosts** failed: one host (host-specific: facts, vars, state) vs all hosts (code/data problem).

### Verbosity

| Flag | Adds |
|---|---|
| `-v` | Full task results (JSON) |
| `-vv` | Task file paths and line numbers, role/handler loading |
| `-vvv` | Connection details: each SSH command and its rc/stdout/stderr; which module file |
| `-vvvv` | SSH's own debug (keys offered, auth methods), connection plugin internals |

Use `-vvv` on **one host** (`-l host`) to keep output readable.

### Seeing data

| Question | Tool |
|---|---|
| What variables does the inventory give host X? | `ansible-inventory -i INV --host X` |
| What's the final value of a variable at this task? | `ansible.builtin.debug: var=name` right before it |
| What type is it? | `{{ x \| type_debug }}` |
| What are all of this host's variables right now? | `debug: var=hostvars[inventory_hostname]` (big!) |
| Which config file and settings are active? | `ansible --version`, `ansible-config dump --only-changed` |
| Which hosts/tasks/tags would run? | `--list-hosts`, `--list-tasks`, `--list-tags` |
| What facts does the host have? | `ansible X -m ansible.builtin.setup -a 'filter=...'` |
| Where is a variable defined? | `grep -rn 'name:' inventories roles playbooks` + topic 25's ladder |

### Readable output

```ini
# ansible.cfg
[defaults]
callback_result_format = yaml        # default callback prints results as YAML instead of one-line JSON
show_task_path_on_failure = True     # print file:line of the failed task
display_skipped_hosts = True
log_path = ./ansible.log             # also write everything to a log file (protect it: may contain data)
```

`callback_result_format = yaml` makes multi-line `stdout`/`stderr` readable, which helps a lot with long command errors.

### The task debugger

A built-in interactive prompt that opens when a task fails (or always):

| `debugger:` value | Opens |
|---|---|
| `on_failed` | When the task fails |
| `on_unreachable` | When the host is unreachable |
| `on_skipped` | When the task is skipped |
| `always` / `never` | Always / never |

Set it on a task, block, role or play (or `ANSIBLE_ENABLE_TASK_DEBUGGER=True` for every failed task).

Debugger commands:

| Command | Does |
|---|---|
| `p task` | Show the task |
| `p task.args` | The task's arguments (as rendered) |
| `p task_vars['x']` | A variable's value for this host |
| `p result._result` | The failed result |
| `task.args['src'] = 'fixed.j2'` | Change an argument |
| `task_vars['x'] = 'value'` | Change a variable (then `update_task`) |
| `u` / `update_task` | Re-render the task with changed vars |
| `r` / `redo` | Run the task again |
| `c` / `continue` | Continue the play |
| `q` / `quit` | Abort |

### ansible-console

An interactive shell for ad hoc commands against your inventory:

```bash
ansible-console -i inventories/lab
```
```
ansible@all (5)[f:5]$ cd web
ansible@web (2)[f:5]$ ping
ansible@web (2)[f:5]$ become true
ansible@web (2)[f:5]# shell systemctl is-active nginx
ansible@web (2)[f:5]# ansible.builtin.setup filter=ansible_distribution*
ansible@web (2)[f:5]# exit
```

Fast for "check something on these hosts" during an investigation.

### Deeper tools

| Tool | Use |
|---|---|
| `ANSIBLE_DISPLAY_TRACEBACK=always` (2.19+) | Show Python tracebacks for errors (when a plugin/module crashes) |
| `ANSIBLE_KEEP_REMOTE_FILES=1` + `explode`/`execute` | Debug a module on the target (topic 13) |
| `ANSIBLE_DEBUG=1` | Extremely verbose internal logging (last resort) |
| `meta: end_play` / `meta: end_host` | Stop early after a debug task while iterating |
| `assert` with `fail_msg` | Turn assumptions into clear checks while debugging, then keep them |
| `--step` | Confirm each task interactively |

---

## Why it exists

Automation fails in more places than a script: on the control node (parsing, templating), in the connection (SSH, sudo), on the target (modules, OS state), and in the data (precedence, facts). Without a method you jump between guesses; with one, you go straight to the layer that's broken. Knowing the tools also turns hour-long incidents into minutes.

---

## Analogy

**Car diagnostics.** The warning light (error) tells you the system: engine (module), electrics (connection), fuel (data), or the car won't even start (parse). The mechanic plugs in the diagnostic computer (`-vvv`, debugger), reads live values (`debug`, `p task_vars`), tries a part change on the spot (`task.args[...]`, `redo`), and finally adds a check to the next service (lint, assert, test) so it doesn't happen again.

---

## How it works inside

```
 run with debugger: on_failed
   task fails on web1
     → strategy pauses web1's execution, opens the debugger prompt
        task, task_vars, result available as Python objects
        you modify args/vars → update_task re-templates → redo re-executes the module
        continue → the play goes on (with the new result) ; quit → abort
```

The debugger runs on the control node, in the strategy layer (topic 15), so it works for any module and connection.

---

## Code / commands

### A broken playbook to debug

`playbooks/debug-demo.yml`:

```yaml
---
- name: Debug demo
  hosts: web
  become: true
  vars:
    app_cfg_dir: /etc/shop
  tasks:
    - name: Template uses a misspelled variable
      ansible.builtin.copy:
        dest: "{{ app_cfg_dir }}/debug-demo.conf"
        content: "port={{ app_prot }}\n"
        mode: "0644"
      debugger: on_failed

    - name: A command that fails on one distro only
      ansible.builtin.command: rpm -q nginx
      changed_when: false
```

### Session 1: the debugger

```bash
cd ~/infra
ansible-playbook -i inventories/lab playbooks/debug-demo.yml -l web1
```
```
TASK [Template uses a misspelled variable] *********************************************
fatal: [web1]: FAILED! => {"msg": "The task includes an option with an undefined variable. 'app_prot' is undefined..."}
[web1] TASK: Template uses a misspelled variable (debug)> p task_vars.get('app_port')
8081
[web1] TASK: Template uses a misspelled variable (debug)> task_vars['app_prot'] = 8081
[web1] TASK: Template uses a misspelled variable (debug)> u
[web1] TASK: Template uses a misspelled variable (debug)> r
changed: [web1]
[web1] TASK: ... (debug)> c
```

- `p task_vars.get('app_port')`: the correct variable exists (from your inventory), confirming the typo.
- Setting the misspelled variable, `u` (update) and `r` (redo) let the run continue so you can see later failures in the same run. Then fix the **code**.

(Your value may differ or be undefined depending on the variables you set in earlier topics.)

### Session 2: one host fails, the other doesn't

```bash
ansible-playbook -i inventories/lab playbooks/debug-demo.yml -l web2 --start-at-task "A command that fails on one distro only" -v
```
```
fatal: [web2]: FAILED! => {"changed": false, "cmd": ["rpm", "-q", "nginx"], "msg": "[Errno 2] No such file or directory: b'rpm'", "rc": 2, ...}
```

- `[Errno 2] No such file or directory: b'rpm'`: the **command** doesn't exist on web2 (Ubuntu). The `command` module couldn't even start it (rc 2). Fix: `ansible.builtin.package_facts` + a check, or per-family logic.

### Session 3: connection problems, layer by layer

```bash
ansible -i inventories/lab web1 -m ansible.builtin.ping -vvvv 2>&1 | grep -E 'ESTABLISH|EXEC ssh|debug1: (Offering|Server accepts|Authentication)|UNREACHABLE|BECOME' | head
ssh -v ansible@172.30.0.11 true                   # same thing without Ansible
ansible -i inventories/lab web1 -m ansible.builtin.ping -b -vvv 2>&1 | grep -E 'sudo|BECOME'
```

Work down the layers: network (`nc -vz 172.30.0.11 22`) → SSH auth (`ssh -v`) → Python (`ansible -m raw -a 'python3 -V'`) → become (`-b` with `-vvv`).

### Session 4: "why does this variable have this value?"

```bash
ansible-inventory -i inventories/lab --host web1 | grep -A2 nginx_web
ansible -i inventories/lab web1 -m ansible.builtin.debug -a "var=nginx_web_port"
grep -rn 'nginx_web_port' inventories playbooks roles | grep -v molecule
```

Then walk topic 25's ladder from the hits.

### Readable YAML output

```bash
ANSIBLE_CALLBACK_RESULT_FORMAT=yaml ansible -i inventories/lab web1 -m ansible.builtin.command -a "systemctl status nginx --no-pager"
```
```
web1 | CHANGED | rc=0 >>
● nginx.service - The nginx HTTP and reverse proxy server
     Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
     Active: active (running) since ...
```

With `-v` on a playbook, failed task results show `stderr: |-` blocks with real line breaks instead of `\n`.

### Tracebacks for crashes

```bash
ANSIBLE_DISPLAY_TRACEBACK=always ansible-playbook -i inventories/lab playbooks/broken-plugin.yml
```

Shows where in Python a plugin/module raised an exception: useful when reporting a bug or debugging your own module (topic 38).

---

## Hands-on lab

1. Run `debug-demo.yml` and use the debugger to inspect and temporarily fix the undefined variable; then fix the code.
2. Make the second task portable (e.g. `ansible.builtin.package_facts` + `assert` that nginx is in `ansible_facts.packages`).
3. Break SSH on purpose in three ways (wrong user in `-u`... remember variables win, so use `-e ansible_user=nobody`; wrong key via `-e ansible_ssh_private_key_file=/tmp/none`; stop sshd on web2) and identify each from `-vvvv` output.
4. Set `callback_result_format = yaml`, `show_task_path_on_failure = True` and `log_path` in `ansible.cfg`; run a failing task and read the output and the log.
5. Use `ansible-console` to check nginx status and disk usage on all web hosts.
6. Turn one debugging assumption into a permanent `assert` with a clear `fail_msg`.

**Done when:** you can go from any error to its root cause with the method, without guessing.

---

## Where you'll use it

- **Every failed pipeline run** and every on-call page caused by automation.
- **Upgrades** (ansible-core, collections): new errors from stricter templating or removed features.
- **Onboarding**: teaching new team members to read Ansible errors.
- **Interviews**: "A playbook fails on 3 of 100 hosts; how do you debug it?"

---

## When it breaks

(Debugging the debugging.)

| Symptom | Cause | Fix |
|---|---|---|
| Debugger prompt never appears | Not on the failing task's level, or the failure isn't a task failure (parse error) | `debugger: on_failed` on the play; parse errors need `--syntax-check` |
| Debugger hangs in CI | No TTY | Never enable the debugger in CI |
| `-vvvv` output is overwhelming | Too many hosts | `-l one_host`, grep for `EXEC`, `BECOME`, `debug1:` |
| `debug: var=x` prints `VARIABLE IS NOT DEFINED!` | Not defined for that host at that point | Check scope (task/role vars), host groups |
| Secret appears in `log_path` or `-vvv` output | Logging everything | `no_log`, protect/rotate logs, avoid `-vvv` with secrets |
| Output differs between laptop and CI | Different callbacks/config | `ansible-config dump --only-changed` in both |

---

## Quick recap

- Method: **reproduce → narrow → read → inspect → isolate → fix → prevent**.
- Buckets: **parse** (before PLAY), **connection** (UNREACHABLE/become), **data** (undefined/wrong values), **module** (FAILED with msg/rc/stderr).
- Tools: `-v`…`-vvvv` (on one host), `debug` + `type_debug`, `ansible-inventory --host`, `ansible-config dump`, `--list-*`, `--start-at-task`, `--step`, **`debugger: on_failed`** (`p`, `task.args`, `task_vars`, `u`, `r`, `c`), **`ansible-console`**, `callback_result_format = yaml`, `log_path`, `ANSIBLE_DISPLAY_TRACEBACK`, `ANSIBLE_KEEP_REMOTE_FILES`.

### Interview questions

1. **Q: How do you debug a failing Ansible task?**
   A: Read the error to classify it (parse, connection, data, module), narrow to one host and the task (`-l`, `--start-at-task`), increase verbosity (`-vvv`), inspect variables (`debug`, `ansible-inventory --host`) and the target state, use the task debugger to inspect and retry, then fix and add a check or test.
2. **Q: What do the verbosity levels show?**
   A: `-v` results, `-vv` task paths, `-vvv` connection commands and module files, `-vvvv` SSH-level debug and plugin internals.
3. **Q: What is the task debugger?**
   A: An interactive prompt (enabled with `debugger: on_failed` or the env var) that lets you inspect task arguments, variables and results, modify them, and redo or continue the task.

### Scenario question (answer out loud first, then compare)

> *"A playbook that worked yesterday now fails on 3 of 100 hosts with `MODULE FAILURE` and nothing else useful. Walk through your investigation."*

**Model answer:**
1. Compare the 3 hosts with the 97: same OS/version? `ansible <3 hosts> -m setup -a filter=ansible_distribution*`, `ansible_python_version`. Something changed on them yesterday?
2. Run the failing task on one of them with `-vvv -l host`: read `module_stdout`/`module_stderr`. Typical causes: Python missing/upgraded/broken, extra output on stdout from shell startup files, disk full (temp dir), `noexec` /tmp, SELinux denials.
3. Check the host directly: `ansible host -m raw -a 'python3 -V; df -h ~; mount | grep noexec'`.
4. If needed, `ANSIBLE_KEEP_REMOTE_FILES=1`, log in, run the AnsiballZ file by hand to see the real exception.
5. Fix the root cause on those hosts (with Ansible if possible: e.g. `raw` bootstrap), re-run, and add a precondition check (Python version, free disk) to the playbook or monitoring.

**Likely follow-ups:**
- *What if it's not reproducible?* Check for timing/load (timeouts), flapping networks, and add `log_path` + retries temporarily to capture evidence.
- *What if all 100 failed?* Then it's code/data/toolchain: diff the commits since yesterday and the ansible-core/collection versions in CI.
