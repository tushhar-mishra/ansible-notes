# 13 · What Happens Inside: One Module Run

> **Level:** ★★ intermediate
>
> **You will learn:** exactly what Ansible does to run **one module on one host**, step by step: connect, find the home directory, create a **remote temp directory**, build the **AnsiballZ** payload (module + helper code + arguments in one Python file), copy it over **SFTP**, run it with the target's Python, read the **JSON** result, delete the temp directory. You'll see every step live with **`-vvv`**, keep and unpack the payload with **`ANSIBLE_KEEP_REMOTE_FILES`**, learn what **action plugins** are (why `debug` never touches the host), how **forks** run hosts in parallel, and the common keys of a module's **return JSON**.
>
> **You need:** [12 · Ad hoc commands](12-ad-hoc-commands.md), [07 · Python on the managed node](07-python-on-managed-nodes.md), [05 · SSH recap](05-linux-recap-ssh.md).
>
> **Part 1 · Setup and first contact** · Next: [14 · Your first playbook](14-first-playbook.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** For every task on every host, Ansible: **(1)** makes a temp folder on the host, **(2)** copies a self-contained Python file (the module, zipped with its helpers and your arguments), **(3)** runs it with the host's Python, **(4)** reads the JSON it prints, **(5)** deletes the folder. Nothing stays behind.

**Picture it.** A visiting technician brings a sealed toolbox, opens it in your hallway, does one job, writes a short report, packs up and leaves. Next job, a new toolbox.

**The one thing to know.** Watch it happen:

```bash
cd ~/ansible-lab
ansible web1 -m ansible.builtin.ping -vvv 2>&1 | grep -E 'EXEC|PUT|Using module'
```

```
<172.30.0.11> SSH: EXEC ssh ... '/bin/sh -c '"'"'echo ~ansible && sleep 0'"'"''          ← 1. find home dir
<172.30.0.11> SSH: EXEC ssh ... '... mkdir ... ansible-tmp-1763550000.1-4242-9876 ...'   ← 2. make temp dir
Using module file .../ansible/modules/ping.py                                              ← 3. build payload
<172.30.0.11> PUT /home/you/.ansible/tmp/.../tmpk2j1 TO .../AnsiballZ_ping.py             ← 4. copy it
<172.30.0.11> SSH: EXEC sftp -b - ...
<172.30.0.11> SSH: EXEC ssh ... 'chmod u+x ... AnsiballZ_ping.py ...'                    ← 5. make it executable
<172.30.0.11> SSH: EXEC ssh ... '/usr/bin/python3 .../AnsiballZ_ping.py ...'              ← 6. RUN it
<172.30.0.11> SSH: EXEC ssh ... 'rm -f -r .../ansible-tmp-... ...'                        ← 7. clean up
```

**Try it now:** run that command and count the steps.

**You've got it if you can:** explain the 7 steps from memory and say what "AnsiballZ" is.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### The words in this topic

| Word | Meaning |
|---|---|
| **Module** | The Python program that does the work on the target (`ping.py`, `dnf.py`, `user.py`) |
| **module_utils** | Shared helper code modules import (argument parsing, running commands, JSON output) |
| **AnsiballZ** | Ansible's packaging format: **one** Python file that contains the module + the module_utils it needs + your arguments, as an embedded zip. The name is a joke on "Ansible ball, zipped" |
| **Remote temp dir** | `~/.ansible/tmp/ansible-tmp-<time>-<pid>-<random>/` on the target, created per task |
| **Action plugin** | Code that runs **on the control node** first, for a task. Most modules use a simple default one that just ships the module. Some (`copy`, `template`, `fetch`, `debug`, `set_fact`, `assert`) do real work locally |
| **Connection plugin** | How Ansible reaches the host (topic 11): here, `ssh` |
| **Fork** | A separate worker process on the control node, handling one host at a time |
| **Return JSON** | The module's report: `changed`, `failed`, `msg`, and module-specific data |

### The full sequence for one task on one host

```
 CONTROL NODE                                                    TARGET (web1)
 ────────────                                                    ─────────────
 0. (first contact only) interpreter discovery  ── SSH ───────▶  shell snippet: which python?
 1. action plugin starts the task
 2. find remote home dir                         ── SSH ───────▶  echo ~ansible
 3. make a private temp dir                      ── SSH ───────▶  umask 77; mkdir ~/.ansible/tmp/ansible-tmp-XYZ
 4. build AnsiballZ_ping.py locally:
      ping.py + module_utils + {"data": "pong"} → zip → base64 → wrapper script
 5. copy it                                      ── SFTP ──────▶  ~/.ansible/tmp/ansible-tmp-XYZ/AnsiballZ_ping.py
 6. make it executable                           ── SSH ───────▶  chmod u+x ...
 7. run it                                       ── SSH ───────▶  /usr/bin/python3 AnsiballZ_ping.py
                                                                   ├ wrapper unpacks the zip (in memory/temp)
                                                                   ├ module checks/changes state
                                                                   └ prints ONE JSON object to stdout
                        ◀─────────── stdout: {"ping": "pong", "changed": false, "invocation": {...}}
 8. delete the temp dir                          ── SSH ───────▶  rm -f -r ~/.ansible/tmp/ansible-tmp-XYZ
 9. parse JSON, decide ok/changed/failed, print the result line
```

That's **about 6 SSH round trips per task per host**. ControlPersist (topic 05) makes each round trip cheap after the first, because they reuse one open connection. **Pipelining** (topic 48) cuts it to **one** round trip: the payload is piped into `python3` over stdin, with no temp file.

### Why a self-contained file?

The target has **no Ansible installed**. So everything the module needs (its own code and the helper code) must travel with it. AnsiballZ makes the module a single runnable file. Only the Python standard library (and any system bindings, topic 07) must already be on the target.

### Action plugins: work done on the control node

Every task first runs an **action plugin** on the control node. For most modules it's the generic `normal` action: "build the AnsiballZ, ship it, run it". Some modules have their own:

| Module | What its action plugin does on the control node |
|---|---|
| `ansible.builtin.copy` | Reads your local file, computes its checksum, transfers it, **then** runs the remote `copy` module to put it in place |
| `ansible.builtin.template` | Renders the Jinja2 template **locally** (with your variables), then uses `copy` logic |
| `ansible.builtin.fetch` | Pulls a remote file back to the control node |
| `ansible.builtin.debug` | Prints a message. **Never contacts the host** |
| `ansible.builtin.set_fact`, `assert`, `fail` | Pure control-node logic. **No SSH** |
| `ansible.builtin.include_tasks`, `import_tasks` | Load more tasks (topic 34) |

So `debug` on 500 hosts is fast and works even if a host's Python is broken. It runs entirely locally.

### Forks: hosts in parallel

```
 ansible-playbook (main process)
 ├── strategy: "run task T on hosts web1, web2, db1, lb1"
 ├── fork worker #1 → web1  (steps 1–9 above)
 ├── fork worker #2 → web2
 ├── fork worker #3 → db1
 └── fork worker #4 → lb1
        results go back to the main process through a queue, printed as they arrive
```

- At most `forks` workers at once (default 5; lab 10). With 100 hosts and forks=10, hosts run in waves of 10.
- Workers are separate processes, so one slow host doesn't block others within the same task.

### The return JSON

Every module prints one JSON object. Common keys:

| Key | Meaning |
|---|---|
| `changed` | `true` if the module changed something |
| `failed` | `true` if it failed (absent = false) |
| `msg` | Human message, especially on failure |
| `rc`, `stdout`, `stderr`, `stdout_lines` | For modules that run commands |
| `invocation.module_args` | The arguments the module actually received (after defaults). Gold for debugging |
| `diff` | Before/after data for `--diff` (topic 20) |
| `warnings`, `deprecations` | Printed by Ansible as `[WARNING]` / `[DEPRECATION WARNING]` |
| `ansible_facts` | Facts to add to the host's variables (e.g. `discovered_interpreter_python`) |
| module-specific | `results` (dnf), `uid` (user), `checksum` (copy), `status` (systemd_service)… listed in each module's docs |

You'll capture this JSON with `register` and make decisions on it (topic 26).

---

## Why it exists

This design is what makes Ansible **agentless** and **safe**:

- **Nothing installed**: each task brings its own code, so targets never need an Ansible version.
- **Nothing left behind**: temp dirs are deleted, so there's no state on targets that could go stale.
- **Structured results**: JSON instead of free text, so Ansible can reliably tell `ok` from `changed` from `failed`.
- **Private temp files**: `umask 77` means only the login user can read the payload (it may contain arguments such as passwords).

The cost is overhead per task. That's why topic 48 (pipelining, ControlPersist, fewer facts) matters on big fleets.

---

## Analogy

**A specialist technician on a house call.**

1. Knock and find the hallway (home dir).
2. Lay down a private mat only they can touch (temp dir with `umask 77`).
3. Bring in a **sealed kit** with the tool, the spare parts and the job sheet inside (AnsiballZ).
4. Open it and do the job with the household's own electricity (target's Python).
5. Fill in a standard report form: "changed / not changed / failed" (JSON).
6. Roll up the mat and leave nothing behind (rm temp dir).

The office (control node) sends several technicians at once (forks). Some jobs need no visit at all: printing the schedule (`debug`) happens in the office.

---

## How it works inside

### The AnsiballZ file, opened up

```
AnsiballZ_ping.py
├── #!/usr/bin/python3  (wrapper script, small)
├── ZIPDATA = """UEsDBBQAAAAIA... (base64)"""      ← the payload
│     └── zip contains:
│           ansible/__init__.py
│           ansible/module_utils/basic.py          (argument parsing, exit_json, fail_json)
│           ansible/module_utils/common/...        (whatever ping imports)
│           ansible/modules/ping.py                (the module itself)
│           args (JSON): {"ANSIBLE_MODULE_ARGS": {"data": "pong", "_ansible_check_mode": false, ...}}
└── main(): decode → import from the zip → run module → module prints JSON → exit
```

The wrapper also supports two debugging sub-commands:

- `python3 AnsiballZ_ping.py explode` → unpack everything into a `debug_dir/` next to it.
- `python3 AnsiballZ_ping.py execute` → run the unpacked copy (so you can edit `debug_dir/.../ping.py`, add prints, and run it again).

### What the module itself does (`ping`, simplified)

```python
from ansible.module_utils.basic import AnsibleModule

def main():
    module = AnsibleModule(
        argument_spec=dict(data=dict(type='str', default='pong')),
        supports_check_mode=True,
    )
    if module.params['data'] == 'crash':
        raise Exception("boom")
    module.exit_json(ping=module.params['data'])     # prints {"ping": "pong", "changed": false, ...}

if __name__ == '__main__':
    main()
```

- `argument_spec`: the module declares its parameters, types and defaults. That's why typos give `Unsupported parameters` errors.
- `supports_check_mode`: the module promises it can run in dry-run mode (topic 20).
- `exit_json(...)`: print the JSON result and exit successfully. `fail_json(msg=...)` prints a failure.

Every module follows this pattern. Topic 38 writes one.

---

## Code / commands

### See every step with -vvv

```bash
ansible web1 -m ansible.builtin.ping -vvv
```
```
ansible [core 2.22.0]
  config file = /home/you/ansible-lab/ansible.cfg
  ...
Using /home/you/ansible-lab/ansible.cfg as config file
...
<172.30.0.11> ESTABLISH SSH CONNECTION FOR USER: ansible
<172.30.0.11> SSH: EXEC ssh -C -o ControlMaster=auto -o ControlPersist=60s ... -o 'User="ansible"' -o ConnectTimeout=10 -o 'ControlPath="/home/you/.ansible/cp/0c5e4b2a1f"' 172.30.0.11 '/bin/sh -c '"'"'echo ~ansible && sleep 0'"'"''
<172.30.0.11> (0, b'/home/ansible\n', b'')
<172.30.0.11> SSH: EXEC ssh ... 172.30.0.11 '/bin/sh -c '"'"'( umask 77 && mkdir -p "` echo /home/ansible/.ansible/tmp `"&& mkdir "` echo /home/ansible/.ansible/tmp/ansible-tmp-1763550000.123-4242-98765 `" && echo ansible-tmp-1763550000.123-4242-98765="` echo /home/ansible/.ansible/tmp/ansible-tmp-1763550000.123-4242-98765 `" ) && sleep 0'"'"''
<172.30.0.11> (0, b'ansible-tmp-1763550000.123-4242-98765=/home/ansible/.ansible/tmp/ansible-tmp-1763550000.123-4242-98765\n', b'')
Using module file /home/you/.venvs/ansible/lib/python3.13/site-packages/ansible/modules/ping.py
<172.30.0.11> PUT /home/you/.ansible/tmp/ansible-local-4242g1d0/tmpq8w3e TO /home/ansible/.ansible/tmp/ansible-tmp-1763550000.123-4242-98765/AnsiballZ_ping.py
<172.30.0.11> SSH: EXEC sftp -b - -C -o ControlMaster=auto ... '[172.30.0.11]'
<172.30.0.11> (0, b'sftp> put /home/you/.ansible/tmp/ansible-local-4242g1d0/tmpq8w3e /home/ansible/.ansible/tmp/ansible-tmp-.../AnsiballZ_ping.py\n', b'')
<172.30.0.11> SSH: EXEC ssh ... '/bin/sh -c '"'"'chmod u+x /home/ansible/.ansible/tmp/ansible-tmp-.../ /home/ansible/.ansible/tmp/ansible-tmp-.../AnsiballZ_ping.py && sleep 0'"'"''
<172.30.0.11> (0, b'', b'')
<172.30.0.11> SSH: EXEC ssh ... '/bin/sh -c '"'"'/usr/bin/python3 /home/ansible/.ansible/tmp/ansible-tmp-.../AnsiballZ_ping.py && sleep 0'"'"''
<172.30.0.11> (0, b'\n{"ping": "pong", "invocation": {"module_args": {"data": "pong"}}}\n', b'')
<172.30.0.11> SSH: EXEC ssh ... '/bin/sh -c '"'"'rm -f -r /home/ansible/.ansible/tmp/ansible-tmp-.../ > /dev/null 2>&1 && sleep 0'"'"''
<172.30.0.11> (0, b'', b'')
web1 | SUCCESS => {
    "changed": false,
    "invocation": {
        "module_args": {
            "data": "pong"
        }
    },
    "ping": "pong"
}
```

Line by line:

- `ESTABLISH SSH CONNECTION FOR USER: ansible`: the resolved user (topic 11 precedence).
- `SSH: EXEC ssh ...`: the full OpenSSH command. Compare it with a manual `ssh` when debugging.
- `(0, b'/home/ansible\n', b'')`: each remote command's result as **(exit code, stdout, stderr)**.
- `umask 77 && mkdir ...`: temp dir readable only by `ansible`.
- `Using module file ...`: which module file was packed. Proves which collection/version was used.
- `PUT ... TO .../AnsiballZ_ping.py` + `sftp`: the transfer.
- `/usr/bin/python3 .../AnsiballZ_ping.py`: the run, with the discovered interpreter.
- `{"ping": "pong", ...}`: the module's raw JSON on stdout. Ansible adds `changed: false` because the module didn't set it.
- `rm -f -r ...`: cleanup.

Verbosity levels:

| Flag | Shows |
|---|---|
| `-v` | Task results in full (the JSON) |
| `-vv` | + task file paths, which handlers/roles are loaded |
| `-vvv` | + connection details: every SSH command and its result |
| `-vvvv` | + SSH's own debug output (`ssh -vvv`): keys offered, auth methods. For "why won't it log in?" |

### Keep and unpack the payload

```bash
ANSIBLE_KEEP_REMOTE_FILES=1 ansible web1 -m ansible.builtin.ping -vvv 2>&1 | grep 'PUT'
```
```
<172.30.0.11> PUT /home/you/.ansible/tmp/ansible-local-5150x/tmpa1b2 TO /home/ansible/.ansible/tmp/ansible-tmp-1763550100.5-5150-1111/AnsiballZ_ping.py
```

Now log in and look:

```bash
ssh ansible@172.30.0.11
cd ~/.ansible/tmp/ansible-tmp-1763550100.5-5150-1111/
ls -la
python3 AnsiballZ_ping.py                  # runs the module directly: prints the JSON
python3 AnsiballZ_ping.py explode          # unpack
find debug_dir -name '*.py' | head
cat debug_dir/args                          # the arguments that were embedded
```
```
Module expanded into:
/home/ansible/.ansible/tmp/ansible-tmp-1763550100.5-5150-1111/debug_dir
debug_dir/ansible/__init__.py
debug_dir/ansible/module_utils/basic.py
debug_dir/ansible/module_utils/common/...
debug_dir/ansible/modules/ping.py
{"ANSIBLE_MODULE_ARGS": {"data": "pong", "_ansible_check_mode": false, "_ansible_no_log": false, "_ansible_debug": false, "_ansible_diff": false, "_ansible_verbosity": 0, ...}}
```

- `_ansible_check_mode`, `_ansible_diff`...: internal arguments Ansible passes to every module (check mode, diff mode, verbosity).
- ⚠️ `ANSIBLE_KEEP_REMOTE_FILES` leaves files with your **arguments** (possibly secrets) on the target. Debug only, then delete `~/.ansible/tmp`.

### Prove `debug` never touches the host

```bash
ansible web1 -m ansible.builtin.debug -a 'msg="hello"' -vvv 2>&1 | grep -c 'SSH: EXEC'
```
```
0
```

Zero SSH commands: `debug` is an action plugin that runs entirely on the control node.

### Watch the forks

In one terminal:

```bash
ansible all -m ansible.builtin.command -a "sleep 5"
```

In another, while it runs:

```bash
ps -ef --forest | grep -E 'ansible|ssh' | grep -v grep | cut -c1-150
```
```
you  6001  5000  ... /home/you/.venvs/ansible/bin/python3 /home/you/.venvs/ansible/bin/ansible all -m ...
you  6010  6001  ...  \_ /home/you/.venvs/ansible/bin/python3 /home/you/.venvs/ansible/bin/ansible all ...
you  6024  6010  ...  |   \_ ssh -C -o ControlMaster=auto ... 172.30.0.11 /bin/sh -c '/usr/bin/python3 ...
you  6011  6001  ...  \_ /home/you/.venvs/ansible/bin/python3 /home/you/.venvs/ansible/bin/ansible all ...
you  6025  6011  ...  |   \_ ssh -C -o ControlMaster=auto ... 172.30.0.12 ...
...
you  5990     1  ... ssh: /home/you/.ansible/cp/0c5e4b2a1f [mux]
```

- One parent `ansible` process, one **fork** per host, each running its own `ssh`.
- `ssh: ... [mux]`: a ControlPersist **master** connection kept alive in the background (topic 05).

---

## Hands-on lab

1. Run `ansible web1 -m ansible.builtin.ping -vvv` and label each `SSH: EXEC` line with its step number (2–8 from the sequence above).

2. Compare a module that does more work: `ansible web1 -m ansible.builtin.setup -vvv 2>&1 | grep 'Using module'` (the big `setup` module), then `ansible web1 -m ansible.builtin.copy -a "content=hi dest=/tmp/hi" -vvv 2>&1 | grep -E 'PUT|Using module'`. How many `PUT`s does `copy` do? (Two: the file content, then the module.)

3. Keep the payload (`ANSIBLE_KEEP_REMOTE_FILES=1`), log in to web1, `explode` it, read `debug_dir/args`. Edit `debug_dir/ansible/modules/ping.py` so `exit_json(ping=module.params['data'] + '!!!')`, then run `python3 AnsiballZ_ping.py execute`. You've just debugged a module **on the target**. Clean up: `rm -rf ~/.ansible/tmp/*`.

4. Feel the cost of connections. Kill the master connections, then time with and without ControlPersist:

   ```bash
   rm -f ~/.ansible/cp/*
   time ansible all -m ansible.builtin.ping >/dev/null
   time ansible all -m ansible.builtin.ping >/dev/null            # faster: masters still open
   rm -f ~/.ansible/cp/*
   time ANSIBLE_SSH_ARGS="-o ControlMaster=no -o UserKnownHostsFile=$HOME/.ssh/known_hosts_lab" ansible all -m ansible.builtin.ping >/dev/null
   ```

   (On a local Docker network the difference is small. Across the internet or through a bastion it's large.)

5. Show that a module crash is captured as JSON: `ansible web1 -m ansible.builtin.ping -a data=crash`. Read `msg` and `exception` (add `-vvv` for the traceback).

**Done when:** you can draw the 9-step sequence and say which steps disappear with pipelining (temp dir, PUT, chmod and rm: the payload goes straight into `python3`'s stdin).

---

## Where you'll use it

- **Debugging** any odd failure: `-vvv` shows which file, which user, which interpreter, and the raw output.
- **Module bugs**: `ANSIBLE_KEEP_REMOTE_FILES` + `explode` + `execute` to test a fix on the spot.
- **Performance work** (topic 48): knowing it's ~6 round trips per task explains why pipelining and ControlPersist matter.
- **Security reviews**: "what does Ansible leave on servers?" Nothing, except `~/.ansible/tmp` (normally empty) and its log lines.
- **Interviews**: "Walk me through what happens when Ansible runs a task" is a favourite.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Failed to create temporary directory ... In some cases you may have been able to authenticate and did not have permissions on the target directory` | Home dir not writable, disk full, `noexec`/read-only filesystem | Free disk; set `remote_tmp` to a writable path (e.g. `/tmp/.ansible-${USER}`) in `ansible.cfg` |
| `sftp transfer mechanism failed` ... then works with `scp` | SFTP subsystem disabled on the server | Enable `Subsystem sftp` in `sshd_config`, or `transfer_method = smart` / `scp` in `[ssh_connection]` |
| `MODULE FAILURE` with `module_stdout` containing other text | Something printed to stdout before the JSON (e.g. a login banner from `.bashrc`, a `motd` script) | Keep non-interactive shells quiet: no `echo` in `.bashrc` for non-interactive sessions |
| `MODULE FAILURE ... /usr/bin/python3: not found`, `rc: 127` | No Python (topic 07) | Bootstrap with `raw` |
| `Permission denied` running `AnsiballZ_*.py` | `/home` or `~/.ansible/tmp` mounted `noexec` | Ansible runs `python3 file`, so this is rare; otherwise move `remote_tmp` |
| Leftover `ansible-tmp-*` dirs on targets | Runs interrupted, or `KEEP_REMOTE_FILES` was set | Delete `~/.ansible/tmp/*`; remove the variable |
| `Timeout (12s) waiting for privilege escalation prompt` | sudo is prompting for a password (topic 19) | NOPASSWD or `-K` |
| `debug` works but other tasks fail on a host | `debug` runs locally (proves nothing about the host) | Test with `ping` |

---

## Quick recap

- One task on one host = **find home → temp dir (`umask 77`) → build AnsiballZ → SFTP it → chmod → run with python3 → JSON → rm**. About 6 SSH round trips; ControlPersist makes them cheap; pipelining reduces them to one.
- **AnsiballZ** = module + module_utils + arguments in one self-contained Python file, so targets need no Ansible.
- **Action plugins** run on the control node first; `debug`, `set_fact`, `assert` never contact the host; `template` renders locally.
- **Forks** = parallel worker processes, one host each.
- Debug with `-vvv` (connection), `-vvvv` (SSH auth), `ANSIBLE_KEEP_REMOTE_FILES=1` + `explode`/`execute`.

### Interview questions

1. **Q: What happens when Ansible runs a module on a host?**
   A: It connects over SSH, creates a private temp directory, packs the module with its helper code and arguments into one Python file (AnsiballZ), copies it over, runs it with the target's Python, reads the JSON result from stdout, deletes the temp directory, and reports ok/changed/failed.
2. **Q: Why doesn't Ansible need an agent?**
   A: Every task ships its own code to the target and removes it afterwards; only SSH and Python are needed.
3. **Q: Does the `debug` module connect to the managed node?**
   A: No. It's an action plugin that runs entirely on the control node.

### Scenario question (answer out loud first, then compare)

> *"On a few hosts every task fails with `MODULE FAILURE`, and `-vvv` shows the module output starts with 'Welcome to ACME Corp!' followed by the JSON. Other hosts are fine. What's going on?"*

**Model answer:**
1. Ansible expects the module's stdout to be **only** JSON. Something on those hosts prints text first.
2. The likely culprit is the login user's shell startup files (`~/.bashrc`, `/etc/profile.d/*.sh`, `/etc/bash.bashrc`) echoing a banner even for **non-interactive** sessions. SSH `Banner` text goes to stderr and is fine; output from `.bashrc` lands in stdout.
3. Confirm: `ssh ansible@host 'true' | od -c | head` shows the banner on stdout.
4. Fix: guard the echo with `[[ $- == *i* ]]` (interactive only) or move it to `/etc/motd`. Push that fix with Ansible itself (`lineinfile`/template), using `raw` if normal modules can't run yet.

**Likely follow-ups:**
- *How would you see the raw module output?* `-vvv` prints each command's `(rc, stdout, stderr)`.
- *Could pipelining change this?* No: it still reads stdout from the remote shell.
