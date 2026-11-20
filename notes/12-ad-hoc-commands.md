# 12 · Ad Hoc Commands

> **Level:** ★ foundation
>
> **You will learn:** the `ansible` command's shape (`ansible PATTERN -m MODULE -a ARGS`), **host patterns** (groups, union `:`, intersection `:&`, exclusion `:!`, wildcards, indexes) and `--limit`, the difference between **`command`**, **`shell`** and **`raw`**, running real admin jobs on many hosts at once (packages, files, services, users, facts), reading every kind of output (`SUCCESS`, `CHANGED`, `FAILED`, `UNREACHABLE`), the colours, `-b` for sudo, `-o` one-line output, `--check`, and the command's **exit codes**.
>
> **You need:** [11 · Connection settings and ansible.cfg](11-connection-settings-and-ansible-cfg.md) (lab `ansible.cfg` in place), [06 · Linux recap II](06-linux-recap-admin-tasks.md) (the modules).
>
> **Part 1 · Setup and first contact** · Next: [13 · What happens inside: one module run](13-inside-a-module-run.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** An **ad hoc command** runs **one module** against **many hosts**, right now, from the command line. No playbook. Perfect for quick checks and one-off fixes.

**Picture it.** A playbook is a **planned recipe**. An ad hoc command is **shouting one instruction across the kitchen**: "Everyone, check your oven temperature!"

**The one thing to know.**

```
ansible  web          -m ansible.builtin.package  -a "name=tree state=present"  -b
         └ pattern    └ module                     └ module arguments (k=v)        └ become (sudo)
```

```bash
cd ~/ansible-lab
ansible all -m ansible.builtin.command -a "uptime"               # read-only check
ansible web -m ansible.builtin.package -a "name=tree state=present" -b   # install
ansible web -m ansible.builtin.package -a "name=tree state=present" -b   # again → green, changed: false
```

**Try it now:** run the three lines. The first install prints yellow **CHANGED**; the second prints green **SUCCESS** with `"changed": false`.

**You've got it if you can:** target "all web servers except web2" and explain why `command` always reports `CHANGED`.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### The command's shape

```
ansible <PATTERN> [-m MODULE] [-a "ARGUMENTS"] [OPTIONS]
```

| Part | Meaning | Default |
|---|---|---|
| `PATTERN` | Which hosts (from the inventory) | required |
| `-m MODULE` | Which module to run (use the FQCN) | `ansible.builtin.command` |
| `-a "ARGS"` | Module arguments, as `key=value` pairs (or a free-form command for `command`/`shell`/`raw`) | none |
| `-b` / `--become` | Run with sudo (topic 19) | off |
| `-K` | Ask for the sudo password | off |
| `-f N` | Forks: hosts in parallel | from `ansible.cfg` (lab: 10) |
| `-o` / `--one-line` | One output line per host | off |
| `-C` / `--check` | Dry run: report what *would* change (topic 20) | off |
| `-D` / `--diff` | Show file differences (topic 20) | off |
| `-l` / `--limit` | Narrow the pattern further | none |
| `-e` | Extra variable | none |
| `-v` … `-vvvv` | More detail (topic 13) | off |
| `--list-hosts` | Show which hosts match; run nothing | |

### Host patterns

| Pattern | Selects | Lab example → result |
|---|---|---|
| `all` or `*` | Every host | web1 web2 db1 lb1 |
| `web` | A group | web1 web2 |
| `web1` | One host | web1 |
| `web:db` | **Union**: in web **or** db | web1 web2 db1 |
| `app:!db` | **Exclusion**: in app but **not** db | web1 web2 |
| `web:&alma` | **Intersection**: in web **and** alma | web1 |
| `web*` | Wildcard on names | web1 web2 |
| `~(web\|db)[0-9]` | **Regex** (starts with `~`) | web1 web2 db1 |
| `web[0]` | First host of a group | web1 |
| `web[-1]` | Last host | web2 |
| `172.30.0.*` | Wildcard on inventory names (only if hosts are listed by IP) | |

Rules:

- **Quote patterns** that contain `!`, `&`, `*` or `~`, so your shell doesn't interpret them: `'app:!db'`.
- Ansible evaluates **unions first, then intersections, then exclusions**, whatever order you write them in.
- `--limit` applies an **extra** pattern on top: `ansible all -l web` = web hosts only. In playbooks (`hosts: all`), `--limit` is how you narrow a run without editing the file.

### command vs shell vs raw

| Module | Runs through a shell? | Pipes `\|`, redirects `>`, `&&`, `$VAR`, `*` | Needs Python | Use for |
|---|---|---|---|---|
| `ansible.builtin.command` (default) | **No**: runs the program directly | ❌ Not interpreted (passed as literal text) | Yes | Simple commands. Safer: no shell injection |
| `ansible.builtin.shell` | Yes: `/bin/sh -c "..."` | ✅ | Yes | When you really need shell features |
| `ansible.builtin.raw` | Whatever the SSH login shell is | ✅ | **No** | Bootstrapping (topic 07) |

Example of the difference:

```bash
ansible web1 -a "echo $HOME > /tmp/x"              # command: prints the literal text, no file created
ansible web1 -m ansible.builtin.shell -a 'echo $HOME > /tmp/x'   # shell: creates /tmp/x
```

(In the first line, your **local** shell expands `$HOME` before Ansible even runs, because of the double quotes. Use single quotes to send `$HOME` to the target.)

**All three always report `CHANGED`** when the command succeeds. They can't know whether the command changed anything. That's the idempotency problem topic 27 solves. Prefer a real module (`package`, `file`, `copy`, `user`…) whenever one exists.

### Output formats

Two styles, depending on the module:

**JSON style** (most modules):

```
web1 | CHANGED => {
    "changed": true,
    ...
}
```

**Command style** (`command`, `shell`, `raw`):

```
web1 | CHANGED | rc=0 >>
<stdout of the command>
```

The **status word** after the host name:

| Status | Colour | Meaning |
|---|---|---|
| `SUCCESS` | Green | Module ran, **nothing changed** (`changed: false`) |
| `CHANGED` | Yellow | Module ran and **changed** something |
| `FAILED!` | Red | Module ran but failed (bad arguments, package not found, non-zero exit code...) |
| `UNREACHABLE!` | Red | Ansible couldn't **connect** (SSH, DNS, timeout, auth). The module never ran |

(In playbooks the same idea shows as `ok`, `changed`, `failed`, `unreachable`, plus `skipped`, topic 14.)

### Exit codes of the `ansible` command

| Code | Meaning |
|---|---|
| `0` | Everything succeeded (changed or not) |
| `1` | Generic error |
| `2` | One or more hosts **failed** |
| `4` | One or more hosts **unreachable** (in practice; see the warning below) |
| `4` | Parser error (bad YAML, bad options in a playbook) |
| `5` | Bad or incomplete command-line options |
| `99` | You interrupted it (Ctrl+C) |
| `250` | Unexpected error (a bug or crash) |

⚠️ **Corrected:** older docs list `3` for "unreachable", but ansible-core's task queue manager returns `RUN_UNREACHABLE_HOSTS = 4`, so real runs with an unreachable host commonly exit **4**, the same number as a parser error. (A run with both failed and unreachable hosts can also produce other values.) Confirm on your version: stop one lab container, run `ansible all -m ping; echo $?`, and paste the real number here.

Scripts and CI use these: `ansible all -m ping || alert`. **Treat any non-zero as failure**, and if you need to know *why*, parse the PLAY RECAP or use the JSON callback (`ANSIBLE_STDOUT_CALLBACK=json`) instead of branching on 3 vs 4.

---

## Why it exists

Not everything deserves a playbook:

- "Is NTP (Network Time Protocol) running on all 200 servers?"
- "Which servers still have OpenSSL 3.0.13?"
- "Restart the stuck agent on the 3 db servers."
- "Copy this hotfix file to the web tier, now."

Before Ansible, these were `for h in $(cat hosts); do ssh $h ...; done` loops: serial (slow), no sudo handling, no structured output, no idea which ones failed. Ad hoc commands do it **in parallel**, with **sudo**, **colour-coded results**, and **proper exit codes**, using the same inventory and modules as your playbooks.

The rule of thumb: if you'll do it **again**, or it **changes** something important, write it into a playbook (reviewed, in Git). Ad hoc is for **looking** and **emergencies**.

---

## Analogy

A **manager with a walkie-talkie** on a factory floor with 200 workers (hosts), organised into teams (groups). "Packing team, report your stock levels!" (a read-only ad hoc command). "Everyone except the night shift, put on your helmets!" (`all:!night`). Each worker radios back: "done" (green), "fixed it" (yellow), "can't, machine broken" (red FAILED), or silence (UNREACHABLE).

---

## How it works inside

```
 ansible 'web:!web2' -m ansible.builtin.package -a "name=tree state=present" -b
   │
   ├─ 1. load ansible.cfg + inventory
   ├─ 2. resolve pattern 'web:!web2'           → [web1]
   ├─ 3. parse -a "k=v k=v"                    → {name: tree, state: present}
   ├─ 4. build ONE task in memory: {package: {...}, become: true}
   │     (ad hoc = a tiny one-task play, gather_facts off)
   ├─ 5. run it on up to -f hosts in parallel   (topic 13 traces this part)
   ├─ 6. print each host's result as it arrives
   └─ 7. exit with 0 / 2 / 3 ...
```

Ad hoc commands use the **same engine** as playbooks. Everything you learn here (patterns, modules, become, check mode) carries over directly.

---

## Code / commands

All run from `~/ansible-lab` (the `ansible.cfg` there supplies inventory, user and key).

### Check things (read-only)

```bash
ansible all -a "uptime" -o
```
```
db1 | CHANGED | rc=0 | (stdout)  12:01:44 up 15 min,  0 users,  load average: 0.08, 0.05, 0.02
web1 | CHANGED | rc=0 | (stdout)  12:01:44 up 15 min,  0 users,  load average: 0.08, 0.05, 0.02
web2 | CHANGED | rc=0 | (stdout)  12:01:44 up 15 min,  0 users,  load average: 0.08, 0.05, 0.02
lb1 | CHANGED | rc=0 | (stdout)  12:01:44 up 15 min,  0 users,  load average: 0.08, 0.05, 0.02
```

- No `-m` → the default `command` module.
- `-o`: one line per host. `rc=0` = the command's exit code.
- `CHANGED` even though `uptime` changes nothing: `command` can't know.
- All four show the same load: containers share the WSL2 kernel.

```bash
ansible all -m ansible.builtin.setup -a 'filter=ansible_distribution*' -o
```
```
web1 | SUCCESS => {"ansible_facts": {"ansible_distribution": "AlmaLinux", "ansible_distribution_file_parsed": true, "ansible_distribution_file_path": "/etc/redhat-release", "ansible_distribution_file_variety": "RedHat", "ansible_distribution_major_version": "10", "ansible_distribution_release": "Heliotrope Lion", "ansible_distribution_version": "10.1"}, "changed": false}
web2 | SUCCESS => {"ansible_facts": {"ansible_distribution": "Ubuntu", ..., "ansible_distribution_version": "24.04"}, "changed": false}
...
```

- A real module (`setup`) correctly reports `SUCCESS` / `changed: false`.

### Install a package (with sudo)

```bash
ansible web -m ansible.builtin.package -a "name=tree state=present" -b
```
```
web1 | CHANGED => {
    "ansible_facts": {
        "pkg_mgr": "dnf"
    },
    "changed": true,
    "msg": "",
    "rc": 0,
    "results": [
        "Installed: tree-2.1.1-8.el10.x86_64"
    ]
}
web2 | CHANGED => {
    "ansible_facts": {
        "pkg_mgr": "apt"
    },
    "cache_update_time": 1763550104,
    "cache_updated": false,
    "changed": true,
    "stderr": "",
    "stdout": "Reading package lists...\n...Setting up tree (2.1.1-2ubuntu3) ...\n",
    ...
}
```

- `pkg_mgr`: the `package` module detected `dnf` on web1 and `apt` on web2 and called the right one.
- `results` / `stdout`: what the package manager reported.
- Run it again: both show **`SUCCESS`** and `"changed": false`. Idempotent.

Without `-b`:

```
web1 | FAILED! => {
    "changed": false,
    "msg": "This command has to be run under the root user.",
    "results": []
}
```

- `FAILED!`, not `UNREACHABLE!`: the connection worked; the module refused because it lacked root rights.

### Files and directories

```bash
ansible web -m ansible.builtin.file -a "path=/opt/demo state=directory owner=ansible mode=0755" -b
ansible web -m ansible.builtin.copy -a "content='hello from ansible\n' dest=/opt/demo/hello.txt mode=0644" -b
ansible web -a "cat /opt/demo/hello.txt"
```

- `content=`: write this text instead of copying a local file (`src=`).
- On the command line `mode=0644` is fine: `k=v` arguments are strings, not YAML numbers.

### Services

```bash
ansible db -m ansible.builtin.systemd_service -a "name=sshd state=started enabled=true" -b
```
```
db1 | SUCCESS => {
    "changed": false,
    "enabled": true,
    "name": "sshd",
    "state": "started",
    "status": { "ActiveState": "active", ... }
}
```

- Already running and enabled → `SUCCESS`, `changed: false`.

### Users

```bash
ansible all -m ansible.builtin.user -a "name=alice shell=/bin/bash groups=users append=true" -b
ansible all -a "id alice" -o
```

### Gather information across the fleet

```bash
ansible all -m ansible.builtin.shell -a 'free -m | awk "/Mem:/ {print \$2\" MB\"}"' -o
ansible all -m ansible.builtin.command -a "rpm -q openssl" -l alma -o
```

- `shell` is needed for the pipe. Note the quoting: single quotes for your local shell, `\$2` so awk gets `$2`.
- `-l alma`: limit the `all` pattern to the `alma` group.

### Patterns in practice

```bash
ansible 'app:!db' --list-hosts                   # web1 web2
ansible 'web:&ubuntu' --list-hosts               # web2
ansible '~(web|lb)[0-9]' --list-hosts            # web1 web2 lb1
ansible web --list-hosts -l 'web[0]'             # web1
```

### UNREACHABLE: what it looks like

```bash
docker stop ansible-lab-db1-1
ansible db -m ansible.builtin.ping
```
```
db1 | UNREACHABLE! => {
    "changed": false,
    "msg": "Failed to connect to the host via ssh: ssh: connect to host 172.30.0.21 port 22: No route to host",
    "unreachable": true
}
```
```bash
echo $?        # usually 4 = at least one host was unreachable (older docs say 3; check yours)
docker start ansible-lab-db1-1
```

- `msg`: the actual SSH error. Read it; it usually says exactly what's wrong (topic 05's table).

---

## Hands-on lab

1. Fleet facts in one line each:

   ```bash
   ansible all -m ansible.builtin.setup -a 'filter=ansible_os_family' -o
   ```

   Which hosts are `RedHat` and which `Debian`?

2. Install `tree` on the web group (with and without `-b`), run it twice, and note SUCCESS vs CHANGED vs FAILED.

3. Prove `command` doesn't use a shell:

   ```bash
   ansible web1 -a 'echo hello > /tmp/out.txt'
   ansible web1 -a 'cat /tmp/out.txt'                           # No such file: ">" was just text
   ansible web1 -m ansible.builtin.shell -a 'echo hello > /tmp/out.txt'
   ansible web1 -a 'cat /tmp/out.txt'                           # hello
   ```

4. Patterns: write the pattern for each, check with `--list-hosts`:
   - all hosts except the load balancer,
   - Ubuntu hosts that are web servers,
   - the first web server only.

5. Make a change, then undo it:

   ```bash
   ansible all -m ansible.builtin.user -a "name=temp1 state=present" -b
   ansible all -a "getent passwd temp1" -o
   ansible all -m ansible.builtin.user -a "name=temp1 state=absent remove=true" -b
   ```

6. Exit codes:

   ```bash
   ansible all -m ansible.builtin.ping >/dev/null; echo "exit=$?"         # 0
   ansible web1 -a "false" >/dev/null; echo "exit=$?"                     # 2 (host failed)
   ansible all -m ansible.builtin.nosuchmodule; echo "exit=$?"            # an error code, with "couldn't resolve module"
   ```

**Done when:** you can predict each command's status colour before running it.

---

## Where you'll use it

- **Incident response**: "check disk space on every web server", "who's logged in?", "restart the agent on these 3 hosts".
- **Fleet questions**: versions of a package, kernel versions, uptime, failed systemd units.
- **Before a playbook run**: `ansible all -m ping` to confirm reachability; `--list-hosts` to confirm targeting.
- **Emergency fixes**: copy a hotfix file or flip a setting now, then put the change into a playbook afterwards so it isn't lost (drift, topic 01).
- **Interviews**: "How would you check X on 500 servers?" → ad hoc command with a pattern.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `UNREACHABLE! ... No route to host / Connection timed out / Connection refused` | Host down, wrong IP, firewall, sshd not running | Topic 05's SSH table; `ssh` by hand |
| `UNREACHABLE! ... Permission denied (publickey)` | Wrong user or key | Check `remote_user`/`ansible_user`, key (topic 11) |
| `FAILED! ... This command has to be run under the root user` / `Permission denied` on a file | Needs root | Add `-b` |
| `FAILED! ... Missing sudo password` | sudo requires a password | `-K`, or a NOPASSWD rule (topic 19) |
| `couldn't resolve module/action 'x'` | Typo, or the collection isn't installed | Check the FQCN; `ansible-doc -l \| grep x` (topic 16) |
| `Unsupported parameters for (ansible.builtin.file) module: owne` | Typo in an argument | `ansible-doc ansible.builtin.file` |
| `No hosts matched, nothing to do` | Pattern typo or wrong inventory | `--list-hosts`, `ansible-inventory --graph` |
| `-bash: !db: event not found` | Your shell expanded `!` | Single-quote the pattern: `'app:!db'` |
| Redirect/pipe silently ignored | Used `command` | Use `shell`, or better a real module |
| `$VAR` shows *your* value, not the target's | Your local shell expanded it inside double quotes | Use single quotes |

---

## Quick recap

- `ansible PATTERN -m MODULE -a "k=v ..." [-b] [-o] [--check]`. Default module: `command`.
- Patterns: `group`, `a:b` (or), `a:&b` (and), `a:!b` (not), `web*`, `~regex`, `web[0]`; `--limit` narrows further; `--list-hosts` previews.
- `command` (no shell) / `shell` (shell features) / `raw` (no Python): all always say `CHANGED`. Prefer real modules.
- Statuses: **SUCCESS** (green, no change), **CHANGED** (yellow), **FAILED!** (red, module error), **UNREACHABLE!** (red, connection error). Exit codes: 0 ok, 2 failed, unreachable usually 4 (docs say 3; don't branch on it, treat non-zero as failure).
- Ad hoc for looking and emergencies; anything repeatable goes into a playbook.

### Interview questions

1. **Q: What's the difference between the `command` and `shell` modules?**
   A: `command` runs a program directly without a shell, so pipes, redirects and variables aren't interpreted (safer). `shell` runs through `/bin/sh`, so they are. Both always report changed; prefer real modules.
2. **Q: How do you run something on all web servers except one?**
   A: A pattern with exclusion: `ansible 'web:!web3' ...`, or `--limit 'web:!web3'`.
3. **Q: FAILED vs UNREACHABLE?**
   A: UNREACHABLE means Ansible couldn't connect or authenticate, so the module never ran. FAILED means it connected and ran the module, but the module reported an error.

### Scenario question (answer out loud first, then compare)

> *"There's a critical OpenSSL vulnerability. Your manager wants to know within 10 minutes which of your 300 servers run an affected version, and then wants it patched safely. How do you use Ansible?"*

**Model answer:**
1. **Find out (read-only, ad hoc):** `ansible all -m ansible.builtin.package_facts` is heavy for output, so a quick version: `ansible alma -a "rpm -q openssl" -o` and `ansible ubuntu -a "dpkg-query -W openssl" -o`, saving output to a file and grepping affected versions. Unreachable hosts are listed separately to follow up.
2. **Patch (not ad hoc):** write a short playbook that updates `openssl` (`state: latest` for that package only), restarts services that link to it (or schedules reboots), and runs in **batches** (`serial`, topic 45) with a health check, stopping if failures pass a threshold.
3. Run it first with `--check` on all hosts, then on a canary group, then the rest.
4. Commit the playbook so the fix is reviewed, repeatable and auditable.

**Likely follow-ups:**
- *Why not just `ansible all -m dnf -a "name=openssl state=latest" -b`?* No batching or health checks, all 300 at once; services keep using the old library until restarted; nothing recorded in Git.
- *How would you get structured output instead of grep?* `package_facts` in a playbook, then write a report with a template (topics 23, 29).
