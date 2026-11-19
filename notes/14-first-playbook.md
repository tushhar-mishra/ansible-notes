# 14 · Your First Playbook

> **Level:** ★ foundation
>
> **You will learn:** what a **playbook**, a **play** and a **task** are in YAML, the main play keywords (`name`, `hosts`, `become`, `gather_facts`, `tasks`), how to write tasks with **FQCN** modules, how to run a playbook with `ansible-playbook`, how to read **every line** of the output (PLAY, TASK, `ok`, `changed`, `skipping`, `fatal`, and the **PLAY RECAP**: `ok`, `changed`, `unreachable`, `failed`, `skipped`, `rescued`, `ignored`), what happens when a task fails on one host, and why the **second run must say `changed=0`**. You'll deploy nginx with a custom page to both web servers (AlmaLinux and Ubuntu) and open it with `curl`.
>
> **You need:** [12 · Ad hoc commands](12-ad-hoc-commands.md), [04 · YAML](04-yaml-for-ansible.md), [06 · Linux recap II](06-linux-recap-admin-tasks.md).
>
> **Part 2 · Playbooks** · Next: [15 · What happens inside ansible-playbook](15-inside-ansible-playbook.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **playbook** is a YAML file that lists **plays**. Each play says **which hosts** and **which tasks**, in order. You run it with `ansible-playbook`. It's ad hoc commands, written down, reviewed and repeatable.

**Picture it.** A playbook is a **recipe book**; a play is **one recipe** ("for the web servers"); tasks are the **numbered steps**.

**The one thing to know.** `~/ansible-lab/web.yml`:

```yaml
---
- name: Web servers run nginx
  hosts: web
  become: true
  tasks:
    - name: nginx is installed
      ansible.builtin.package:
        name: nginx
        state: present

    - name: nginx is running and enabled
      ansible.builtin.systemd_service:
        name: nginx
        state: started
        enabled: true
```

```bash
cd ~/ansible-lab
ansible-playbook web.yml          # first run: changed=2 per host
ansible-playbook web.yml          # second run: changed=0  ← idempotent
curl -s 172.30.0.11 | head -3     # nginx answers
```

**Try it now:** run it twice and compare the two `PLAY RECAP` blocks.

**You've got it if you can:** read a PLAY RECAP line and say what each of the 7 counters means.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### The three levels

```
playbook (file)          = a LIST of plays
└── play                 = a MAP: hosts + settings + tasks
    └── tasks            = a LIST of tasks
        └── task         = a MAP: name + ONE module + task keywords
```

(Topic 04 showed the same shape as JSON.)

### Play keywords you need now

| Keyword | Meaning | Example |
|---|---|---|
| `name` | Description printed as `PLAY [...]` | `Web servers run nginx` |
| `hosts` | **Pattern** of hosts (topic 12 patterns work here) | `web`, `all`, `app:!db` |
| `become` | Use privilege escalation (sudo) for every task in the play | `true` |
| `gather_facts` | Collect facts first (OS, IPs, ...)? Default `true` | `false` to save time when unneeded |
| `tasks` | The list of tasks | |
| `vars` | Variables for this play (topic 21) | `http_port: 8080` |
| `handlers` | Tasks that run only when notified (topic 30) | |
| `roles` | Roles to apply (topic 35) | |

### Task keywords you need now

| Keyword | Meaning |
|---|---|
| `name` | Printed as `TASK [...]`. **Always** write one, phrased as the desired state |
| `<module FQCN>:` | Exactly **one** module per task, with its arguments as a map |
| `become` | Override the play's `become` for this task |
| `when` | Run only if a condition is true (topic 26). If false → **skipped** |

### The output words

During the run:

| Line | Meaning |
|---|---|
| `PLAY [name] ***` | A play starts |
| `TASK [Gathering Facts] ***` | The automatic first task (the `setup` module) |
| `TASK [name] ***` | A task starts; one result line per host follows |
| `ok: [web1]` | Ran fine, **nothing changed** (green) |
| `changed: [web1]` | Ran fine and **changed** something (yellow) |
| `skipping: [web1]` | `when` was false: not run on this host (cyan) |
| `fatal: [web1]: FAILED! => {...}` | Task **failed** on this host (red). The host is **removed from the rest of the play** |
| `fatal: [web1]: UNREACHABLE! => {...}` | Couldn't connect. Host removed from the rest of the play |
| `...ignoring` | Failed, but `ignore_errors: true` (topic 31) let the play continue |
| `PLAY RECAP ***` | Totals per host |

### The PLAY RECAP, field by field

```
web1   : ok=4  changed=2  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0
```

| Counter | Counts | Notes |
|---|---|---|
| `ok` | Tasks that **succeeded** on this host, **including** the changed ones and fact gathering | Not "unchanged"! `ok` ≥ `changed` |
| `changed` | Tasks that **changed** something | Second run of good code: **0** |
| `unreachable` | Connection failures | Host stops for this play |
| `failed` | Task failures | Host stops for this play (unless handled) |
| `skipped` | Tasks skipped by `when` (or other skip reasons) | Normal for multi-distro code |
| `rescued` | Failures caught by a `rescue:` block (topic 31) | |
| `ignored` | Failures ignored by `ignore_errors: true` (topic 31) | |

---

## Why it exists

Ad hoc commands are great for one-off jobs, but they:

- live in your **shell history**, not in Git,
- run **one module** at a time,
- can't easily **combine** steps with logic, variables and templates,
- can't be **reviewed** before they run.

A playbook turns the steps into **code**: versioned, reviewed in a Pull Request, run the same way by anyone or any pipeline, and safe to re-run because each task is idempotent.

---

## Analogy

**A pilot's pre-flight checklist.** The checklist (playbook) has sections (plays) for different aircraft types (host groups). Each line (task) is phrased as a state: "flaps: set", "fuel: full". The pilot checks each line; if it's already right, they tick it (`ok`); if not, they fix it (`changed`); if a line doesn't apply to this aircraft, they skip it (`skipped`); if something is broken, that aircraft doesn't fly (`failed`, removed from the play) while the others continue. At the end, a summary sheet per aircraft (PLAY RECAP).

---

## How it works inside

A short version (topic 15 traces it in full):

```
ansible-playbook web.yml
 1. read ansible.cfg, inventory, playbook (YAML → plays → tasks)
 2. PLAY "Web servers run nginx": hosts: web → [web1, web2]
 3. TASK Gathering Facts          → setup module on web1 + web2 (in parallel)
 4. TASK nginx is installed        → package module on web1 + web2 (in parallel)
        wait until BOTH finish  ←── "linear" strategy: task by task, all hosts in step
 5. TASK nginx is running          → systemd_service module on web1 + web2
 6. PLAY RECAP
```

Key behaviour:

- **Tasks run in order.** Task 2 starts only when task 1 has finished on **all** hosts of the play.
- **Hosts run in parallel** within a task (up to `forks`).
- A host that **fails** stops there; the other hosts **continue** with the next tasks.
- If **every** host has failed, Ansible prints `NO MORE HOSTS LEFT` and ends the play.

---

## Code / commands

### The lab playbook

`~/ansible-lab/web.yml`:

```yaml
---
- name: Web servers run nginx with our home page
  hosts: web
  become: true

  tasks:
    - name: apt cache is fresh (Debian family only)
      ansible.builtin.apt:
        update_cache: true
        cache_valid_time: 3600
      when: ansible_facts['os_family'] == 'Debian'

    - name: nginx is installed
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Home page is in place (RedHat family)
      ansible.builtin.copy:
        dest: /usr/share/nginx/html/index.html
        content: "Hello from {{ inventory_hostname }}\n"
        mode: "0644"
      when: ansible_facts['os_family'] == 'RedHat'

    - name: Home page is in place (Debian family)
      ansible.builtin.copy:
        dest: /var/www/html/index.html
        content: "Hello from {{ inventory_hostname }}\n"
        mode: "0644"
      when: ansible_facts['os_family'] == 'Debian'

    - name: nginx is running and enabled
      ansible.builtin.systemd_service:
        name: nginx
        state: started
        enabled: true
```

Line by line:

- `---`: optional YAML document start.
- `- name: ...`: the **first play** (a list item).
- `hosts: web`: web1 (AlmaLinux) and web2 (Ubuntu).
- `become: true`: every task runs through sudo.
- `when: ansible_facts['os_family'] == 'Debian'`: a **condition**. `ansible_facts` holds what "Gathering Facts" found; `os_family` is `RedHat` or `Debian`. The task runs only where it's true. Full story: topics 23 and 26.
- `ansible.builtin.apt` with `update_cache: true` + `cache_valid_time: 3600`: like `apt-get update`, but only if the cache is older than an hour. Idempotent-ish (it reports `changed` only when it actually refreshed).
- `"Hello from {{ inventory_hostname }}\n"`: `{{ }}` inserts a variable. `inventory_hostname` is the host's name in the inventory (`web1`). Quoted because it contains `{{` (topic 04). Variables: topic 21.
- The two home-page tasks write to the right directory per distribution: `/usr/share/nginx/html` on AlmaLinux, `/var/www/html` on Ubuntu.

### First run

```bash
ansible-playbook web.yml
```
```
PLAY [Web servers run nginx with our home page] ****************************************

TASK [Gathering Facts] *****************************************************************
ok: [web2]
ok: [web1]

TASK [apt cache is fresh (Debian family only)] *****************************************
skipping: [web1]
changed: [web2]

TASK [nginx is installed] **************************************************************
changed: [web2]
changed: [web1]

TASK [Home page is in place (RedHat family)] *******************************************
skipping: [web2]
changed: [web1]

TASK [Home page is in place (Debian family)] *******************************************
skipping: [web1]
changed: [web2]

TASK [nginx is running and enabled] ****************************************************
ok: [web2]
changed: [web1]

PLAY RECAP *****************************************************************************
web1                       : ok=4    changed=3    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
web2                       : ok=5    changed=3    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
```

Reading it:

- `ok: [web2]` before `ok: [web1]` under Gathering Facts: hosts finish in any order (parallel).
- `skipping: [web1]` on the apt task: web1 is AlmaLinux, so `when` is false.
- `changed: [web2]` on the apt task: the cache was refreshed.
- **web1's recap**: `ok=4` = Gathering Facts + nginx installed + RedHat home page + service. `changed=3` = the last three. `skipped=2` = the apt task and the Debian home page.
- **web2's recap**: `ok=5` (facts, apt, nginx, Debian page, service), `changed=3` (apt, nginx, page), `skipped=1`.
- Why is the service task `ok` on web2 but `changed` on web1? **Debian/Ubuntu packages start and enable their service when installed**; RHEL-family packages don't. So on web2 nginx was already running and enabled. One more reason to always write the service task explicitly: you get the same end state on both families.

### Second run: the idempotency check

```bash
ansible-playbook web.yml
```
```
...
TASK [nginx is installed] **************************************************************
ok: [web1]
ok: [web2]
...
PLAY RECAP *****************************************************************************
web1                       : ok=4    changed=0    unreachable=0    failed=0    skipped=2    rescued=0    ignored=0
web2                       : ok=5    changed=0    unreachable=0    failed=0    skipped=1    rescued=0    ignored=0
```

- `changed=0` on both hosts: everything was already correct, **nothing was touched**. This is the most important line you'll read in Ansible. Topic 27 is entirely about keeping it at 0.
- (If the apt cache is older than `cache_valid_time` on a later run, that one task may show `changed` again. That's expected behaviour of that option.)

Check the result:

```bash
curl -s 172.30.0.11
curl -s 172.30.0.12
```
```
Hello from web1
Hello from web2
```

### A failing task

Change `name: nginx` to `name: nginxx` and run again:

```
TASK [nginx is installed] **************************************************************
fatal: [web1]: FAILED! => {"changed": false, "failures": ["No package nginxx available."], "msg": "Failed to install some of the specified packages", "rc": 1, "results": []}
fatal: [web2]: FAILED! => {"changed": false, "msg": "No package matching 'nginxx' is available"}

NO MORE HOSTS LEFT *********************************************************************

PLAY RECAP *****************************************************************************
web1                       : ok=1    changed=0    unreachable=0    failed=1    skipped=1    rescued=0    ignored=0
web2                       : ok=2    changed=0    unreachable=0    failed=1    skipped=0    rescued=0    ignored=0
```

- `fatal: [web1]: FAILED!`: the task failed; the JSON shows the module's message (`dnf` and `apt` word it differently).
- `NO MORE HOSTS LEFT`: every host has failed, so the play stops. Later tasks never ran.
- Exit code: `echo $?` → `2` (a host failed), same codes as topic 12.

Fix the typo before continuing.

### Several plays in one playbook

```yaml
---
- name: Everyone has the base tools
  hosts: all
  become: true
  tasks:
    - name: Base tools are installed
      ansible.builtin.package:
        name: [tree, curl]
        state: present

- name: Web servers run nginx
  hosts: web
  become: true
  tasks:
    - name: nginx is installed
      ansible.builtin.package:
        name: nginx
        state: present
```

- Plays run **one after another**: all hosts finish play 1 before play 2 starts.
- `name: [tree, curl]`: the `package` module accepts a **list** and installs both in one transaction (faster than a loop, topic 28).

### Useful options (more in topic 20)

```bash
ansible-playbook web.yml --syntax-check      # parse only: catches YAML/structure errors
ansible-playbook web.yml --list-tasks        # what would run
ansible-playbook web.yml --list-hosts        # on which hosts
ansible-playbook web.yml -l web1             # limit to web1
ansible-playbook web.yml -v                  # show each task's result JSON
```

---

## Hands-on lab

1. Write `web.yml` above in `~/ansible-lab`. Run `--syntax-check` first.
2. Run it twice. Write down both recaps. Confirm `changed=0` the second time.
3. `curl` both web servers.
4. **Drift test**: change the page by hand, then let Ansible fix it.

   ```bash
   ssh ansible@172.30.0.11 'echo HACKED | sudo tee /usr/share/nginx/html/index.html'
   curl -s 172.30.0.11                       # HACKED
   ansible-playbook web.yml                  # changed=1 on web1 (only the home page task)
   curl -s 172.30.0.11                       # Hello from web1
   ```

5. **Stop the service by hand**: `ssh ansible@172.30.0.12 'sudo systemctl stop nginx'`, re-run, see exactly one `changed` on web2 (the service task).
6. **Failure**: introduce the `nginxx` typo, run, read the `fatal` lines and the recap, check `echo $?`, fix it.
7. **Partial failure**: put the typo only in a task with `when: inventory_hostname == 'web2'`. Run. web2 fails, but web1 **continues** to the end. Read the recap. Fix it.
8. Add a second play for `all` that installs `tree` and `curl`. Run with `-l db1` and see only db1 in the base play and **no hosts matched** for the web play.

**Done when:** a second run of your playbook always shows `changed=0`, and you can explain every number in the recap.

---

## Where you'll use it

- **Everything** from here on is a playbook: baselines, deploys, patching, rolling restarts.
- **CI/CD**: pipelines run `ansible-playbook` (topics 53–54); the exit code (0 / 2 / 3...) fails or passes the job.
- **Drift repair**: re-running a playbook puts manual changes back.
- **Code review**: a PR shows exactly which tasks change; reviewers read task names like a checklist.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `ERROR! 'hosts' is not a valid attribute for a Task` (or similar) | Indentation: a play keyword ended up inside `tasks` (or vice versa) | Check YAML levels (topic 04) |
| `ERROR! no module/action detected in task` | Task has no module, or the module line is mis-indented | Exactly one module key per task |
| `ERROR! conflicting action statements: copy, file` | Two modules in one task (often a missing `- `) | Split into two tasks |
| `ERROR! couldn't resolve module/action 'nginx.install'` | Not a real module name | Look it up with `ansible-doc` (topic 16) |
| `Syntax Error while loading YAML` | Tabs, bad indentation, unquoted `{{` | `--syntax-check`; topic 04 |
| `fatal: ... No package matching 'x' is available` | Wrong package name for that distro, or stale apt cache | Check names per family; `update_cache: true` |
| `fatal: ... This command has to be run under the root user` | Missing `become: true` | Add it to the play or task |
| `skipping` where you expected the task to run | `when` condition false on that host | Print the fact with `debug` (topic 21) |
| `[WARNING]: Could not match supplied host pattern, ignoring: webs` + `skipping: no hosts matched` | `hosts:` pattern typo | `--list-hosts` |
| Second run still shows `changed` | A non-idempotent task (`command`, `shell`, `state: restarted`, `latest`) | Topic 27 |

---

## Quick recap

- **Playbook** = list of **plays**; play = `hosts` + settings (`become`, `gather_facts`) + `tasks`; task = `name` + **one module**.
- Run: `ansible-playbook file.yml`. Tasks run **in order**, hosts **in parallel**; a failed host drops out, others continue.
- Output: `ok` (green), `changed` (yellow), `skipping` (cyan), `fatal` (red).
- PLAY RECAP: `ok` includes changed; `changed` should be **0 on the second run**; also `unreachable`, `failed`, `skipped`, `rescued`, `ignored`.
- Name tasks as desired states; use FQCNs; quote `{{ }}` values.

### Interview questions

1. **Q: What's the difference between a play and a playbook?**
   A: A playbook is a YAML file containing a list of plays. A play maps a set of hosts to a list of tasks with settings like `become`.
2. **Q: What does `ok=5 changed=2` mean in the recap?**
   A: Five tasks succeeded on that host, two of which made changes (so three found everything already correct). `ok` includes changed tasks.
3. **Q: What happens if a task fails on one of ten hosts?**
   A: That host stops executing the rest of the play; the other nine continue. The recap shows `failed=1` for it and the exit code is 2.

### Scenario question (answer out loud first, then compare)

> *"Your playbook reports `changed=3` on every single run, even when nobody touched the servers. Your manager asks whether that's a problem. Is it, and what would you do?"*

**Model answer:**
1. Yes. A correct playbook should report `changed=0` when nothing drifted. Constant changes mean some tasks aren't idempotent.
2. Why it matters: you can't tell real drift from noise, check mode is useless, handlers (e.g. service restarts) may fire every run and cause outages, and audit logs fill with false changes.
3. Find them: run with `-v` or `--diff` and look at which tasks say `changed`. Typical culprits: `command`/`shell` tasks, `state: restarted`, `state: latest`, templates with timestamps, `get_url` without checksums.
4. Fix each: use a proper module, or add `creates`/`changed_when` to commands, use `started` + handlers instead of `restarted` (topics 27, 30).
5. Add an idempotency check to CI: run twice, fail if the second run reports any changes (Molecule does this, topic 51).

**Likely follow-ups:**
- *Can `changed` ever be legitimately non-zero every run?* Rarely: e.g. a task that must always run (a deployment trigger). Mark it clearly, or use `changed_when: false` if it only reads.
- *How do you see what changed?* `--diff` shows file differences; `-v` shows each module's result.
