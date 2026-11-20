# 15 · What Happens Inside ansible-playbook

> **Level:** ★★ intermediate
>
> **You will learn:** the complete life of an `ansible-playbook` run, step by step: read config → load inventory → parse the YAML into plays, blocks and tasks → pick hosts → run the **strategy** (linear by default) with **forks** → **gather facts** → for each task: evaluate `when` per host, render arguments with that host's variables **on the control node**, ship and run the module (topic 13), collect the JSON, update host state → run **handlers** → print the **recap** → exit code. You'll see each stage in `-vv` output, time every task with the **`profile_tasks`** callback, and understand *where* each kind of error comes from.
>
> **You need:** [14 · Your first playbook](14-first-playbook.md), [13 · Inside one module run](13-inside-a-module-run.md).
>
> **Part 2 · Playbooks** · Next: [16 · Modules, FQCNs and ansible-doc](16-modules-fqcn-and-ansible-doc.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** `ansible-playbook` does a lot **before** it touches any server: it reads config, builds the inventory, and parses and checks the whole playbook. Then, play by play and **task by task**, it runs each task on all selected hosts in parallel, waits for all of them, and moves on.

**Picture it.** A **tour guide with a group**: the guide plans the whole day first (parse), then at each stop (task) waits until **everyone** (every host) has finished before walking to the next stop. Anyone who gets lost (fails) is left behind; the rest continue.

**The one thing to know.**

```
read ansible.cfg → load inventory → parse playbook (syntax errors stop here)
   └─ for each PLAY:
        select hosts (pattern + --limit)
        gather facts (setup module)                      ← a normal task, first
        for each TASK, on all hosts in parallel (forks):
            when false?  → skipping
            render args with THIS host's variables (on the control node)
            run the module on the host → JSON → ok / changed / failed / unreachable
            failed/unreachable host → removed from the rest of this play
        run notified handlers
   └─ PLAY RECAP → exit code (0 ok, 2 failed, 4 unreachable in practice [docs: 3], 4 parse error)
```

**Try it now:**

```bash
cd ~/ansible-lab
ANSIBLE_CALLBACKS_ENABLED=ansible.posix.profile_tasks ansible-playbook web.yml
```

At the end you'll see how long each task took. Gathering Facts is usually one of the slowest.

**You've got it if you can:** explain why a YAML error stops the run before any host is contacted, but an undefined variable fails only on the task that uses it.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

The run has two big phases:

| Phase | Where | What can go wrong |
|---|---|---|
| **1. Load and parse** | Control node only, no SSH | Bad config, bad inventory, YAML syntax errors, unknown keywords, missing roles/files for static imports, unknown modules → **exit 4**, nothing changed anywhere |
| **2. Execute** | Control node + every host | Unreachable hosts, module failures, undefined variables in a task, failed conditionals → per-host failures, **exit 2 (failed) or 4 (unreachable; docs say 3)** |

Inside phase 2, three pieces of Ansible work together:

| Piece | Job |
|---|---|
| **Strategy plugin** | Decides the order: which task runs on which host when. Default **`linear`**: every host finishes task N before any host starts task N+1. Alternative: **`free`** (each host races ahead on its own, topic 45) |
| **Task queue manager + workers (forks)** | Runs up to `forks` hosts at a time, each in its own process |
| **Callback plugins** | Receive events ("task started", "host ok", "host failed") and print them. The default callback prints the output you know; others print JSON, YAML, timings, or send to Slack/logs |

---

## Why it exists

Knowing the order of events answers the questions you'll hit every week:

- "Why did nothing run? There's just an error." → Phase 1 failure: fix the file.
- "Why did web1 stop but web2 carried on?" → Per-host failure in phase 2.
- "Why is my variable empty on db1 but fine on web1?" → Variables are resolved **per host**, per task.
- "Why is the run slow?" → Fact gathering, serial waiting for the slowest host, many SSH round trips.
- "When does the handler run?" → After all tasks of the play, not right after the notify.

---

## Analogy

**A school trip.**

1. **The night before** (phase 1): the teacher reads the plan, checks the bus booking, the class list and the permission slips. A missing page in the plan → the trip is cancelled before anyone gets on the bus.
2. **Morning roll call** (host selection + gather facts): who's here, who needs what.
3. **Each stop** (task): the class does the activity together; the teacher waits until **all** pupils are done (linear strategy). A pupil who's sick (failed) goes home; the rest continue.
4. Some pupils skip an activity that doesn't apply to them (`when` false → skipped).
5. **End of day** (handlers): "everyone whose shoes got muddy, clean them now", done once at the end, not after every puddle.
6. **Report to parents** (PLAY RECAP).

---

## How it works inside

### The full trace

```
$ ansible-playbook -i inventory.ini web.yml --limit web
│
├─ 1. CLI: parse options (-i, --limit, -e, --check, -f, -v...)
├─ 2. CONFIG: find ansible.cfg (ANSIBLE_CONFIG → ./ → ~/ → /etc) and load settings
├─ 3. PLUGINS: set up search paths for modules, collections, roles
├─ 4. INVENTORY: run inventory plugins on each source → hosts, groups, group/host vars
├─ 5. VARIABLES: load extra vars (-e); prepare the variable manager
├─ 6. PLAYBOOK PARSE:
│     YAML → list of plays → each play's tasks are wrapped in "blocks"
│     static content is pulled in now: roles:, import_tasks, import_playbook (topic 34)
│     each task checked: exactly one known module/action? valid keywords?
│     ✗ any error → print it, exit 4. No host contacted.
│
└─ 7. FOR EACH PLAY:
      ├─ a. resolve hosts: pattern "web" ∩ --limit → [web1, web2]
      │     (with serial: split into batches, topic 45)
      ├─ b. start the strategy (linear) and the worker pool (forks = 10)
      ├─ c. GATHER FACTS (implicit task "Gathering Facts" = setup module)
      │     results stored per host: hostvars['web1']['ansible_facts']
      ├─ d. FOR EACH TASK:
      │     for each host still active (in parallel, ≤ forks):
      │       i.   compute this host's variables (inventory, group_vars, facts, play vars, -e ...)
      │       ii.  evaluate `when:` with them → false? report "skipping", next host
      │       iii. render the task's arguments ({{ }}) with them, on the control node
      │       iv.  action plugin → connection plugin → module on host (topic 13)
      │       v.   result JSON → ok / changed / failed / unreachable
      │       vi.  register: save result into a host variable (topic 26)
      │       vii. notify: mark handlers to run later (topic 30)
      │       viii.failed or unreachable → host removed from the rest of this play
      │     wait until every host finished this task (linear) → next task
      │     no active hosts left → "NO MORE HOSTS LEFT", play ends
      ├─ e. HANDLERS: run the notified ones, on the hosts that notified them
      └─ f. next play (failed hosts stay excluded for the rest of the run)
│
├─ 8. CALLBACK: print PLAY RECAP from the per-host statistics
└─ 9. EXIT: 0 all good · 2 some host failed · 4 some host unreachable (docs: 3) · 4 parse error · ...
```

### Three things people get wrong

1. **Templating happens on the control node.** `"{{ inventory_hostname }}"` becomes `web1` **before** the module is sent. The target never sees Jinja2 (except inside the `template` module's *output file*, which is also rendered locally).
2. **Variables are per host.** The same task gets different arguments on web1 and web2. An undefined variable fails **that task on that host**, not the whole playbook at parse time.
3. **Linear means waiting.** One slow host (a big package download) holds everyone at that task. `free` strategy or `async` (topics 45, 47) change that.

### Where each error comes from

| Error | Phase | Effect |
|---|---|---|
| `Syntax Error while loading YAML` | 1 (parse) | Nothing runs, exit 4 |
| `couldn't resolve module/action 'x'` | 1 (parse) | Nothing runs, exit 4 |
| `the role 'x' was not found` (for `roles:`/`import_role`) | 1 (parse) | Nothing runs |
| `Could not match supplied host pattern` | 7a | Warning; play has no hosts |
| `UNREACHABLE!` | 7c/7d-iv | That host drops out |
| `'my_var' is undefined` | 7d-ii/iii | That task fails on that host |
| `Conditional result (...) was of type 'str', which is not a boolean` | 7d-ii | That task fails on that host (strict conditionals since ansible-core 2.19, topic 26) |
| Module `FAILED!` | 7d-iv/v | That host drops out (unless `ignore_errors`/`rescue`) |

---

## Code / commands

### Watch the phases with -vv

```bash
ansible-playbook web.yml -vv
```
```
ansible-playbook [core 2.22.0]
  config file = /home/you/ansible-lab/ansible.cfg
  ...
Using /home/you/ansible-lab/ansible.cfg as config file
Skipping callback 'default', as we already have a stdout callback.
...
PLAYBOOK: web.yml **********************************************************************
1 plays in web.yml

PLAY [Web servers run nginx with our home page] ****************************************

TASK [Gathering Facts] *****************************************************************
task path: /home/you/ansible-lab/web.yml:2
ok: [web1]
ok: [web2]

TASK [apt cache is fresh (Debian family only)] *****************************************
task path: /home/you/ansible-lab/web.yml:8
skipping: [web1] => {"changed": false, "false_condition": "ansible_facts['os_family'] == 'Debian'", "skip_reason": "Conditional result was False"}
ok: [web2] => {"cache_update_time": 1763550104, "cache_updated": false, "changed": false}
...
```

- `PLAYBOOK: web.yml` / `1 plays in web.yml`: parse finished; it found one play.
- `task path: web.yml:8`: file and line of each task. Essential in big projects with roles.
- `skipping ... "false_condition": ...`: exactly which condition was false.
- `-vv` also prints each task's result JSON (like `-v`).

### Preview the parsed structure

```bash
ansible-playbook web.yml --list-tasks
```
```
playbook: web.yml

  play #1 (web): Web servers run nginx with our home page	TAGS: []
    tasks:
      apt cache is fresh (Debian family only)	TAGS: []
      nginx is installed	TAGS: []
      Home page is in place (RedHat family)	TAGS: []
      Home page is in place (Debian family)	TAGS: []
      nginx is running and enabled	TAGS: []
```

- This is phase 1's result: what Ansible **would** run, without contacting anything. Gathering Facts isn't listed: it's added at run time.

### Time every task

```bash
ANSIBLE_CALLBACKS_ENABLED=ansible.posix.profile_tasks,ansible.posix.timer ansible-playbook web.yml
```
```
...
PLAY RECAP *****************************************************************************
web1   : ok=4  changed=0  unreachable=0  failed=0  skipped=2  rescued=0  ignored=0
web2   : ok=5  changed=0  unreachable=0  failed=0  skipped=1  rescued=0  ignored=0

Playbook run took 0 days, 0 hours, 0 minutes, 6 seconds
Wednesday 19 November 2026  12:30:06 +0000 (0:00:00.512)       0:00:06.100 ******
===============================================================================
Gathering Facts ----------------------------------------------------------- 2.41s
nginx is installed --------------------------------------------------------- 1.62s
apt cache is fresh (Debian family only) ------------------------------------ 0.71s
nginx is running and enabled ----------------------------------------------- 0.66s
Home page is in place (Debian family) -------------------------------------- 0.39s
Home page is in place (RedHat family) -------------------------------------- 0.37s
```

- `profile_tasks`: per-task time, slowest first. Each time is for the **slowest host** of that task (linear strategy waits for it).
- `timer`: total run time.
- Callbacks are **plugins** (topic 38). Put `callbacks_enabled = ansible.posix.profile_tasks` in `ansible.cfg` to have it always.

### See "linear" waiting

Add this task to a scratch playbook `wait.yml`:

```yaml
- name: Linear strategy demo
  hosts: web
  gather_facts: false
  tasks:
    - name: web1 is slow, web2 is fast
      ansible.builtin.command: "sleep {{ 5 if inventory_hostname == 'web1' else 1 }}"
    - name: Second task
      ansible.builtin.debug:
        msg: "now on {{ inventory_hostname }}"
```

```bash
ansible-playbook wait.yml
```

- web2 finishes the first task after 1 s, but its "Second task" only starts after web1 finishes at 5 s. That's linear.
- Run again with `ANSIBLE_STRATEGY=free ansible-playbook wait.yml`: web2 prints its second task immediately.

### See per-host variables

```yaml
- name: Same task, different values
  hosts: all
  gather_facts: true
  tasks:
    - name: Each host gets its own values
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} runs {{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}"
```
```
ok: [web1] => { "msg": "web1 runs AlmaLinux 10.1" }
ok: [web2] => { "msg": "web2 runs Ubuntu 24.04" }
ok: [db1] => { "msg": "db1 runs AlmaLinux 10.1" }
ok: [lb1] => { "msg": "lb1 runs Ubuntu 24.04" }
```

One task, four different rendered messages: rendering happened per host on the control node.

---

## Hands-on lab

1. **Phase 1 failure**: add a tab to `web.yml` indentation, run it, and confirm **no** SSH happened: `ansible-playbook web.yml -vvv 2>&1 | grep -c 'SSH: EXEC'` → `0`. Fix it.

2. **Phase 2 failure on one host**: add this task, run, and see it fail on web2 only (web1 is skipped by the condition, so the undefined variable is never rendered there):

   ```yaml
   - name: Uses an undefined variable on Debian hosts only
     ansible.builtin.debug:
       msg: "{{ not_defined_anywhere }}"
     when: ansible_facts['os_family'] == 'Debian'
   ```

   Remove it afterwards.

3. Run the **linear vs free** demo above and time both with `time`.

4. Enable `profile_tasks` and `timer` permanently in `ansible.cfg`:

   ```ini
   [defaults]
   callbacks_enabled = ansible.posix.profile_tasks, ansible.posix.timer
   ```

   Run `web.yml` and note the slowest task. Then add `gather_facts: false` to a play that doesn't need facts and compare times. (Your `web.yml` *does* need facts for its `when`s. Try it on `wait.yml`.)

5. Try another output callback for one run: `ANSIBLE_STDOUT_CALLBACK=ansible.builtin.minimal ansible-playbook web.yml`, then `ANSIBLE_STDOUT_CALLBACK=ansible.builtin.oneline`. Same events, different printing. That's the callback layer.

**Done when:** you can explain, for any error message, whether it came from parsing (nothing ran) or execution (some hosts ran).

---

## Where you'll use it

- **Debugging**: "parse or run?" is the first question. `--syntax-check` and `--list-tasks` answer it without risk.
- **Speed**: `profile_tasks` in CI shows where time goes (topic 48).
- **Rolling updates**: understanding linear + batches is the base for `serial` (topic 45).
- **Design**: knowing that handlers run at the end of the play shapes how you write deploys (topic 30).
- **Interviews**: "Walk me through what happens when you run `ansible-playbook`" is very common; the trace above is the answer.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| Error before `PLAY [...]` appears | Phase 1: config, inventory, YAML, unknown module, missing role | Read the error's file/line; `--syntax-check` |
| Whole run slow, mostly at Gathering Facts | Facts collected from every host every run | `gather_facts: false` where unneeded, `gather_subset`, fact caching (topics 23, 48) |
| Everyone waits for one host | Linear strategy | Fix that host; `free` strategy; `async` (topics 45, 47) |
| A host vanished from later plays | It failed or was unreachable earlier in the run | Check the first `fatal:` for it; it stays excluded until the next run |
| Handler didn't run although a task changed | The play failed before the handler phase | `--force-handlers` or `meta: flush_handlers` (topic 30) |
| Variable correct on one host, undefined on another | Variables are per host (different groups/facts) | `ansible-inventory --host X`; debug on each host (topic 25) |
| `ERROR! the role 'nginx' was not found` | Role path wrong (parse phase) | `roles_path` in `ansible.cfg`, or install the role (topics 35, 37) |

---

## Quick recap

- Two phases: **load and parse** (control node only; errors exit 4, nothing touched) and **execute** (per host).
- Execution: per play → select hosts → **gather facts** → per task: `when` → render args **on the control node** with **this host's** variables → module on host → JSON → state. Then **handlers**, then **recap**.
- **Linear** strategy: all hosts finish a task before the next; up to **forks** hosts in parallel.
- Failed/unreachable hosts drop out of the rest of the run.
- Tools: `--syntax-check`, `--list-tasks`, `-vv` (task paths, skip reasons), `profile_tasks`/`timer` callbacks.

### Interview questions

1. **Q: Walk me through what happens when you run `ansible-playbook`.**
   A: It loads config and inventory, parses the playbook into plays and tasks (static imports resolved), then for each play selects hosts, gathers facts, and runs each task on all hosts in parallel up to the fork limit: evaluate `when`, render arguments with that host's variables, copy and run the module over SSH, read JSON, record ok/changed/failed. Handlers run at the end of the play, then the recap and exit code.
2. **Q: What is the default strategy, and what does it mean?**
   A: `linear`: each task must finish on all hosts of the batch before the next task starts. `free` lets each host proceed independently.
3. **Q: Where is Jinja2 templating done?**
   A: On the control node, per host, before the module is sent.

### Scenario question (answer out loud first, then compare)

> *"A 40-task playbook against 200 hosts takes 45 minutes. How do you find out why, and what are the usual fixes?"*

**Model answer:**
1. Measure first: enable `profile_tasks` and `timer`. See which tasks dominate.
2. Typical findings and fixes:
   - **Gathering Facts** on 200 hosts: turn it off where unused, use `gather_subset`, or cache facts (topic 48).
   - **Many SSH round trips**: enable **pipelining** and make sure ControlPersist works (topics 13, 48).
   - **Forks too low** (default 5): raise to 25–50 if the control node can handle it.
   - **One slow host** holding everyone in linear mode: find it with `-v` timing; use `free` strategy where order doesn't matter, or `async` for long tasks.
   - **Loops** installing packages one by one: pass a list to the module (topic 28).
3. Re-measure after each change, one change at a time.

**Likely follow-ups:**
- *Why not just set forks to 200?* Each fork is a process with an SSH connection; CPU and memory on the control node (and SSH limits on bastions) become the bottleneck.
- *Does `free` strategy change results?* It changes ordering across hosts; don't use it where hosts depend on each other in sequence.
