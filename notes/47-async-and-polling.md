# 47 · Async Tasks and Long Jobs

> **Level:** ★★ intermediate
>
> **You will learn:** how to run tasks that take longer than an SSH session should stay open, with **`async`** (maximum run time) and **`poll`** (how often to check), **fire-and-forget** (`poll: 0`) plus **`async_status`** to collect results later (with the 2.19+ `is finished` test), running long jobs **in parallel** across loop items, cleaning up job files, and the related "waiting" tools: **`ansible.builtin.reboot`**, **`wait_for`** (ports, files, text in files, drained connections) and **`wait_for_connection`**.
>
> **You need:** [46 · Delegation](46-delegation.md), [28 · Loops and until](28-loops.md), [26 · register](26-register-and-when.md).
>
> **Part 7 · Execution control and speed** · Next: [48 · Performance tuning](48-performance-tuning.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Normally Ansible keeps the SSH session open until a module finishes. For a 30-minute job (a big upgrade, a backup, a DB import), that's fragile: a network blip or SSH timeout kills it. With **`async: N`**, Ansible starts the module **in the background** on the host and checks on it every **`poll`** seconds (or not at all with `poll: 0`, collecting the result later).

**Picture it.** Instead of standing at the dry cleaner's counter for an hour, you hand in the suit and get a **ticket** (job ID). You can phone every 10 minutes ("is it ready?") or come back tomorrow with the ticket.

**The one thing to know.**

```yaml
- name: Full system upgrade (up to 1 hour, check every 30 s)
  ansible.builtin.dnf:
    name: "*"
    state: latest
  async: 3600
  poll: 30

- name: Start a long backup and move on
  ansible.builtin.command: /usr/local/bin/backup.sh
  async: 7200
  poll: 0
  register: backup_job

- name: Later... wait for the backup to finish
  ansible.builtin.async_status:
    jid: "{{ backup_job.ansible_job_id }}"
  register: backup_result
  until: backup_result is finished
  retries: 240
  delay: 30
```

**Try it now:** run the lab's `sleep 20` with `async: 60, poll: 5` and watch it check in.

**You've got it if you can:** explain `async`, `poll: 0` and how to wait for a fire-and-forget job.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### async and poll

| Keyword | Meaning |
|---|---|
| `async: N` | Run the module in the **background** on the host; kill it if it runs longer than **N seconds** |
| `poll: S` (S > 0) | Ansible **waits**, checking every S seconds, until the job finishes (or N is reached). The play continues only after it's done |
| `poll: 0` | **Fire and forget**: start the job and continue immediately. The registered result contains `ansible_job_id` |

`async` + `poll > 0` looks like a normal task (it still blocks the play), but **the SSH connection isn't held open** during the job: Ansible reconnects briefly for each check. That's the point: no SSH timeout, no broken pipe.

### async_status

Checks a background job by its ID:

```yaml
- ansible.builtin.async_status:
    jid: "{{ job.ansible_job_id }}"
  register: result
  until: result is finished       # 2.19+: use the 'finished' test, not result.finished == 1
  retries: 60
  delay: 10
```

| Test | True when |
|---|---|
| `result is started` | The job has started |
| `result is finished` | The job has finished (successfully or not) |

When finished, `result` contains the module's normal result (`rc`, `stdout`, `changed`...), and the task fails if the job failed.

Job status files live in `~/.ansible_async/` on the target (of the become user). Clean up when you don't wait for results: `async_status` with `mode: cleanup`.

### Parallel long jobs (loops)

A loop normally runs items **one after another**. To run them **at the same time** on a host, start each with `poll: 0`, then wait for all:

```yaml
- name: Start all imports in parallel
  ansible.builtin.command: "/opt/app/bin/import {{ item }}"
  loop: [customers, orders, invoices]
  async: 1800
  poll: 0
  register: imports

- name: Wait for all imports
  ansible.builtin.async_status:
    jid: "{{ item.ansible_job_id }}"
  loop: "{{ imports.results }}"
  loop_control:
    label: "{{ item.item }}"
  register: import_results
  until: import_results is finished
  retries: 180
  delay: 10
```

### Limits

- Not every action supports async: modules that are mostly **action plugins** on the control node (`copy`, `template`, `debug`, `include_*`...) don't run asynchronously.
- With `poll: 0`, failures appear **only when you check** (or never, if you don't). Always check results for anything important.
- Handlers notified by a `poll: 0` task are notified based on the **start** result, not the outcome: avoid notifying from fire-and-forget tasks.
- `async` must be larger than the job's real duration, or Ansible kills it.

### Waiting tools

| Module | Waits for | Example |
|---|---|---|
| `ansible.builtin.reboot` | Reboots the host and waits until it's back (SSH + a test command) | `reboot_timeout: 600`, `test_command: uptime`, `pre_reboot_delay`, `msg` |
| `ansible.builtin.wait_for` | A **port** to open/close, a **file** to exist/disappear, a **regex** in a file, connections to **drain** | `port: 8080`, `state: started`; `path: /var/run/app.ready`; `search_regex: "Started"`; `state: drained` |
| `ansible.builtin.wait_for_connection` | Ansible to be able to connect and run Python again | After creating/rebooting VMs (topic 43) |
| `until` + `retries` + `delay` | Any task's result to meet a condition | `uri` health checks (topic 28) |

`wait_for` runs **on the host** by default (checks its own ports/files); delegate it to localhost to check a port **from** the control node.

---

## Why it exists

SSH sessions aren't designed to stay open for an hour: firewalls drop idle connections, laptops sleep, CI runners hit timeouts. Long operations (OS upgrades, big package installs, database restores, compiles, backups) need to survive that. And some workflows need **parallelism inside a host** (several imports at once) or **waiting for external events** (a port opening after a restart, a host coming back after reboot). `async`, `async_status`, `reboot` and `wait_for` cover these cases cleanly.

---

## Analogy

**Ordering at a busy restaurant.** Normal tasks: you wait at the counter until your food is ready. `async` with `poll`: you get a buzzer and sit down; every few minutes you glance at it. `poll: 0`: you order, go shopping, and come back later with your receipt number (`async_status`). Ordering for three friends at once with three receipts = parallel async loop items. `reboot` = the restaurant closes for cleaning and you wait outside until the doors reopen (`wait_for_connection`).

---

## How it works inside

```
 task with async: 600, poll: 10
   │
   ├─ 1. ship the module + a small wrapper (async_wrapper) to the host (topic 13)
   ├─ 2. the wrapper forks into the background, writes ~/.ansible_async/<jid> (status file),
   │      starts the module, and returns immediately: {"started": 1, "ansible_job_id": "j123.456"}
   ├─ 3. SSH session closes
   ├─ 4. every 10 s: Ansible runs async_status on the host → reads the status file
   │        still running → wait
   │        finished      → full module result → ok/changed/failed as usual
   │        > 600 s       → wrapper kills the module → failed ("async task did not complete within the requested time")
   └─ poll: 0 → steps 1–3 only; you call async_status yourself later
```

---

## Code / commands

### Async with polling

`playbooks/async-demo.yml`:

```yaml
---
- name: Async demo
  hosts: web
  gather_facts: false
  tasks:
    - name: A 20-second job, checked every 5 s
      ansible.builtin.command: sleep 20
      async: 60
      poll: 5
      changed_when: false
```

```bash
cd ~/infra
ansible-playbook -i inventories/lab playbooks/async-demo.yml -v
```
```
TASK [A 20-second job, checked every 5 s] **********************************************
ASYNC POLL on web1: jid=j512633876104.2311 started=1 finished=0
ASYNC POLL on web2: jid=j190451223358.2290 started=1 finished=0
ASYNC POLL on web1: jid=j512633876104.2311 started=1 finished=0
...
ASYNC OK on web1: jid=j512633876104.2311
ok: [web1] => {"ansible_job_id": "j512633876104.2311", "changed": false, "cmd": ["sleep", "20"], "finished": 1, "rc": 0, ...}
```

- `ASYNC POLL`: a status check; `ASYNC OK`: finished.
- Both hosts ran their job in parallel, as usual.

### Too short an async

```yaml
    - name: async shorter than the job
      ansible.builtin.command: sleep 20
      async: 5
      poll: 2
```
```
fatal: [web1]: FAILED! => {"changed": false, "msg": "async task did not complete within the requested time - 5s"}
```

### Fire and forget, collect later

```yaml
- name: Background job, collect later
  hosts: web
  gather_facts: false
  tasks:
    - name: Start a 15-second "backup"
      ansible.builtin.shell: sleep 15 && echo "backup done at $(date +%T)"
      async: 120
      poll: 0
      register: backup_job

    - name: Do other work meanwhile
      ansible.builtin.debug:
        msg: "started job {{ backup_job.ansible_job_id }}, doing other things"

    - name: Wait for the backup to finish
      ansible.builtin.async_status:
        jid: "{{ backup_job.ansible_job_id }}"
      register: backup_result
      until: backup_result is finished
      retries: 30
      delay: 2

    - name: Show its output
      ansible.builtin.debug:
        msg: "{{ backup_result.stdout }}"
```
```
TASK [Start a 15-second "backup"] ******************************************************
changed: [web1]
changed: [web2]

TASK [Do other work meanwhile] *********************************************************
ok: [web1] => { "msg": "started job j77..., doing other things" }
...
TASK [Wait for the backup to finish] ***************************************************
FAILED - RETRYING: [web1]: Wait for the backup to finish (30 retries left).
...
changed: [web1]

TASK [Show its output] *****************************************************************
ok: [web1] => { "msg": "backup done at 12:31:44" }
```

- `changed` on the start task = "a job was started".
- The `FAILED - RETRYING` lines are the `until` loop waiting, not real failures.

### Wait for ports and files

```yaml
- name: Restart the app and wait until it listens
  hosts: web
  become: true
  tasks:
    - name: Restart nginx
      ansible.builtin.systemd_service: { name: nginx, state: restarted }

    - name: Port 80 is open again (checked on the host)
      ansible.builtin.wait_for:
        port: 80
        state: started
        timeout: 30

    - name: Port 80 is reachable from the control node
      ansible.builtin.wait_for:
        host: "{{ hostvars[inventory_hostname]['ansible_host'] }}"
        port: 80
        timeout: 30
      delegate_to: localhost
      become: false
```

(This play restarts nginx on purpose, as an operational action: topic 27's "explicit actions" rule.)

Wait for connections to drain before stopping a service:

```yaml
- ansible.builtin.wait_for:
    port: 80
    state: drained              # wait until no active connections on port 80
    timeout: 120
```

### Reboot and wait

```yaml
- name: Reboot if needed
  ansible.builtin.reboot:
    msg: "Rebooting for kernel update (Ansible)"
    reboot_timeout: 600
    test_command: systemctl is-system-running --wait || true
  when: reboot_needed | default(false) | bool
```

- `reboot` issues the reboot, waits for the host to go down, then waits for SSH and the `test_command`.
- In the Docker lab, a "reboot" stops the container's systemd; Compose's `restart: unless-stopped` starts it again, so it mostly works, but timings differ from real servers. Use the VM from topic 09 for realistic reboot tests.

---

## Hands-on lab

1. Run `async-demo.yml` with `-v` and identify `ASYNC POLL` / `ASYNC OK`.
2. Make `async` shorter than the job and read the error.
3. Run the fire-and-forget play. On web1 during the job: `ls ~ansible/.ansible_async/` and `cat` the file.
4. Run three `sleep 10` jobs in parallel with the async loop pattern; time it (≈10 s, not 30 s).
5. Use `wait_for` with `search_regex` to wait until a file contains "READY" while a background task writes it after 5 s (`shell: sleep 5; echo READY > /tmp/ready.txt` with `async`/`poll: 0`).
6. On the VM (or a container, knowing the caveat), use `ansible.builtin.reboot` and time how long it waits.

**Done when:** no task of yours depends on an SSH session staying open for more than a few minutes.

---

## Where you'll use it

- **OS upgrades/patching**: long `dnf`/`apt` upgrades with `async`/`poll`, then `reboot`.
- **Database operations**: restores, imports, migrations that take a long time; parallel imports.
- **Backups and batch jobs** triggered by Ansible, collected later.
- **Rolling restarts**: `wait_for` port/drained around service restarts.
- **Provisioning**: `wait_for_connection` after VM creation or reboot.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `async task did not complete within the requested time` | `async` too short | Raise `async` above the real duration |
| `Shared connection to x closed` / broken pipe on long tasks | Long blocking task without async | `async` + `poll` |
| Fire-and-forget job failed silently | Never checked with `async_status` | Always collect results for important jobs |
| `Conditional result ... was derived from value of type 'int'` with `until: job.finished` | 2.19+ wants booleans | `until: job is finished` |
| `async_status`: `could not find job` | Different become user (status file in another home), or files cleaned up | Same `become`/`become_user` as the starting task |
| `copy`/`template` with async does nothing special | Action plugins don't run asynchronously | Use async on real remote modules only |
| `reboot` times out | Host takes longer, or SSH/firewall not up after boot, or `test_command` fails | Raise `reboot_timeout`; check `test_command`; console logs |
| `wait_for` passes but the app isn't ready | Port opens before the app is healthy | Health-check URL with `uri` + `until` instead |

---

## Quick recap

- **`async: N`** = run in the background with a maximum duration; **`poll: S`** = check every S seconds (play waits); **`poll: 0`** = fire and forget.
- Collect later with **`async_status`** + `until: result is finished` (2.19+ boolean tests); `mode: cleanup` to remove job files.
- Parallel jobs on one host: loop with `poll: 0`, then loop `async_status` over `results`.
- Waiting: **`reboot`**, **`wait_for`** (port/file/regex/drained), **`wait_for_connection`**, `until` health checks.
- Action plugins (`copy`, `template`...) don't support async; always check fire-and-forget results.

### Interview questions

1. **Q: How do you run a task that takes an hour without SSH timeouts?**
   A: Use `async` (maximum duration) with `poll` (check interval). The module runs in the background on the host, and Ansible reconnects to check its status instead of holding the session open.
2. **Q: What does `poll: 0` do and how do you get the result?**
   A: It starts the job and continues immediately. Register the task, then use `async_status` with the `ansible_job_id`, looping `until: result is finished`.
3. **Q: How do you reboot a server and wait for it with Ansible?**
   A: `ansible.builtin.reboot`, which reboots, waits for the host to go down and come back, and runs a test command; tune `reboot_timeout` and `test_command`.

### Scenario question (answer out loud first, then compare)

> *"A nightly playbook upgrades 50 servers; some upgrades take 40 minutes and the CI job keeps failing with broken SSH connections and a 60-minute job timeout. How do you redesign it?"*

**Model answer:**
1. Wrap the upgrade in `async: 3600` with `poll: 60`, so SSH isn't held open; set SSH keepalives (`ServerAliveInterval`) as an extra safety.
2. Use `serial` batches large enough to finish within the CI window (e.g. 25% at a time with forks ≥ batch size), or move the job to AWX/AAP, which isn't tied to a CI job timeout (topic 56).
3. Reduce the work: pre-download packages (`dnf --downloadonly` async earlier), use a local mirror, and reboot only when needed (topic 26 scenario).
4. After the upgrade: `reboot` if needed, `wait_for_connection`, health check, re-enable in the LB.
5. Report results per host; failed hosts drained and listed, rollout stopped with `max_fail_percentage`.

**Likely follow-ups:**
- *Why not just `poll: 0` for all and finish the CI job?* You'd lose visibility of failures unless a later job collects results; fine only with a follow-up `async_status` stage.
- *What about the CI runner dying mid-run?* The background jobs continue on the hosts; a re-run is safe because the playbook is idempotent.
