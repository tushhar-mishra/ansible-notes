# 45 · Strategies, Forks and Serial

> **Level:** ★★ intermediate
>
> **You will learn:** how Ansible decides **how many hosts** work at once (**`forks`**), **in which order** tasks run across hosts (**strategies**: `linear`, `free`, `host_pinned`), how to roll changes out in **batches** (**`serial`**: numbers, percentages, ramp-up lists) and stop a bad rollout early (**`max_fail_percentage`**), how to limit one task's concurrency (**`throttle`**), host **`order`**, **`run_once`** (and its interaction with `serial`), and the `ansible_play_batch` variable. These are the building blocks of zero-downtime rolling updates (topic 49).
>
> **You need:** [44 · Beyond Linux](44-beyond-linux-servers.md), [15 · Inside ansible-playbook](15-inside-ansible-playbook.md) (linear strategy, forks), [30 · Handlers](30-handlers.md), [31 · Error handling](31-error-handling.md).
>
> **Part 7 · Execution control and speed** · Next: [46 · Delegation and local actions](46-delegation.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Three dials control parallelism:

- **`forks`**: how many hosts Ansible talks to **at the same moment** (a control-node setting).
- **`serial`**: how many hosts go through the **whole play** before the next group starts (a rollout setting).
- **`strategy`**: whether hosts wait for each other after every task (`linear`) or run ahead (`free`).

**Picture it.** Painting 100 houses. `forks` = how many painters you have. `serial` = "do 10 houses completely, inspect them, then the next 10". `linear` = every painter finishes the same wall before anyone moves on; `free` = each painter works through their own house at their own speed.

**The one thing to know.**

```yaml
- name: Rolling update of the web tier
  hosts: web
  serial: [1, "25%", "50%"]        # 1 host first (canary), then 25%, then 50% at a time
  max_fail_percentage: 0           # any failure in a batch → stop the rollout
  tasks:
    - ...
```

**Try it now:** run any playbook with `serial: 1` and watch the tasks repeat once per host.

**You've got it if you can:** explain the difference between `forks: 10` and `serial: 10`.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### forks

| | |
|---|---|
| What | Maximum number of hosts processed **in parallel** for a task (worker processes, topic 13) |
| Default | **5** |
| Set | `forks = 25` in `ansible.cfg`, `-f 25`, `ANSIBLE_FORKS` |
| Scope | The whole run (control node resource limit) |
| Cost | Each fork = a process + an SSH connection: CPU and memory on the control node, connections through bastions |

With 100 hosts and forks 5, each task runs in 20 waves. Raising forks is the simplest speed-up for big fleets (topic 48), up to what your control node and network can handle.

### Strategies

| Strategy | Behaviour | Use when |
|---|---|---|
| **`linear`** (default) | Each task runs on all hosts (≤ forks at a time); **everyone waits** for the slowest before the next task | Most plays; when order across hosts matters |
| **`free`** | Each host runs through the tasks **as fast as it can**, independently | Independent hosts, long varied tasks, maximum throughput |
| **`host_pinned`** | Like `free`, but a fork stays with its host until it's finished (up to forks hosts at a time) | Many hosts, want each host done start to finish without interleaving |
| `debug` | Linear + the interactive debugger on failure | Debugging (topic 58) |

Set with the play keyword `strategy: free`, or `strategy = free` in `ansible.cfg`.

With `free`, output from different hosts and tasks is interleaved, and handlers still run at the end of the play (per host). `run_once` and some ordering assumptions behave differently, so use it for plays where hosts don't depend on each other.

### serial: batches

`serial` splits the play's hosts into **batches**. Each batch runs the **entire play** (all tasks **and** handlers) before the next batch starts.

| Value | Batches for 10 hosts |
|---|---|
| `serial: 1` | 10 batches of 1 |
| `serial: 3` | 3, 3, 3, 1 |
| `serial: "30%"` | 3, 3, 3, 1 (percent of all play hosts, rounded down, minimum 1) |
| `serial: [1, 3, "100%"]` | 1, then 3, then all remaining (6): a **ramp-up**; the last value repeats if needed |

Within a batch, `forks` still limits parallelism, and the strategy still applies.

```
serial: [1, 3, "100%"] on web01..web10

 batch 1: web01                → all tasks → handlers → OK?
 batch 2: web02 web03 web04    → all tasks → handlers → OK?
 batch 3: web05 ... web10      → all tasks → handlers
```

Facts are gathered per batch (when the batch starts).

### max_fail_percentage

Stop the **whole play** if too many hosts in the **current batch** fail:

| Setting | Meaning |
|---|---|
| `max_fail_percentage: 0` | Any failed host in a batch → abort (no further batches) |
| `max_fail_percentage: 25` | Abort if **more than** 25% of the batch fails |
| (not set) | Failed hosts drop out; next batches continue regardless |

It's evaluated per batch, after the batch's tasks. With `serial` not set, the "batch" is all hosts.

Related: `any_errors_fatal: true` (topic 31) aborts **immediately** after the task where any host failed.

### throttle

Limit how many hosts run **one specific task** at the same time, whatever forks/serial say:

```yaml
- name: Database migration must run on one host at a time
  ansible.builtin.command: /opt/app/bin/migrate
  throttle: 1

- name: Call a rate-limited API (max 2 at once)
  ansible.builtin.uri: { url: "https://api.example.com/register/{{ inventory_hostname }}" }
  delegate_to: localhost
  throttle: 2
```

`throttle` can be set on tasks and blocks (and plays), and can't exceed forks.

### order

Which hosts go first:

| `order:` | Order |
|---|---|
| `inventory` (default) | As in the inventory |
| `reverse_inventory` | Reversed |
| `sorted` / `reverse_sorted` | Alphabetical by host name |
| `shuffle` | Random each run |

Matters most with `serial`: decides which hosts form the first (canary) batch.

### run_once

`run_once: true` runs a task on **one** host (the first in the batch) and applies its result (registered vars, `set_fact`) to all hosts in the batch. Typical: a DB schema migration, a notification, fetching a value once.

⚠️ **With `serial`, `run_once` runs once per batch.** A "run once" migration in a 5-batch rollout runs 5 times. Options: make it idempotent (it should be anyway), move it to its own play without `serial`, or guard with `when: inventory_hostname == ansible_play_hosts_all[0]`.

### Magic variables

| Variable | Contains |
|---|---|
| `ansible_play_hosts_all` | All hosts targeted by the play |
| `ansible_play_hosts` | Hosts still active (not failed) |
| `ansible_play_batch` | Hosts in the **current batch** |

---

## Why it exists

Changing all servers at once is fast but risky: a bad config takes down the whole service. Changing one at a time is safe but slow. These settings let you choose:

- **Speed** (forks, `free`) for independent, low-risk work like fact collection or package checks.
- **Safety** (serial batches, canaries, `max_fail_percentage`) for anything that restarts services or changes behaviour.
- **Protection of shared resources** (throttle, run_once) for databases, APIs, licence servers.

Together with the load-balancer drain pattern (topic 46), they give you zero-downtime rolling updates (topic 49).

---

## Analogy

**Boarding a plane.** `forks` = how many gate agents scan boarding passes at once. `serial` = boarding **groups** (group 1 boards fully before group 2 is called). A ramp-up (`[1, 3, "100%"]`) = priority passengers first, then a small group, then everyone. `max_fail_percentage` = "if a group has problems at the door, stop boarding". `throttle` = "only one wheelchair lift, one passenger at a time". `linear` = everyone in the group must have stowed their bags before anyone sits; `free` = sit when you're ready.

---

## How it works inside

```
 play: hosts=web (10 hosts), serial=[1, 3, "100%"], forks=5, strategy=linear
   │
   ├─ compute batches: [web01] [web02..web04] [web05..web10]
   ├─ for each batch:
   │     gather facts for the batch
   │     for each task:  run on batch hosts, ≤ 5 at a time (forks); wait for all (linear)
   │                     throttle on a task → ≤ N at a time for that task
   │     handlers for the batch
   │     failed% in batch > max_fail_percentage? → abort the play (remaining batches never start)
   └─ recap
```

---

## Code / commands

Run these from `~/infra` against the lab (`-i inventories/lab`).

### Batches with serial

`playbooks/serial-demo.yml`:

```yaml
---
- name: Rolling demo
  hosts: app                       # web1, web2, db1
  gather_facts: false
  serial: [1, 2]
  tasks:
    - name: Show the batch
      ansible.builtin.debug:
        msg: "batch = {{ ansible_play_batch }}"
      run_once: true

    - name: Pretend work
      ansible.builtin.command: sleep 1
      changed_when: false
```

```bash
ansible-playbook -i inventories/lab playbooks/serial-demo.yml
```
```
PLAY [Rolling demo] ********************************************************************

TASK [Show the batch] ******************************************************************
ok: [web1] => { "msg": "batch = ['web1']" }

TASK [Pretend work] ********************************************************************
ok: [web1]

PLAY [Rolling demo] ********************************************************************

TASK [Show the batch] ******************************************************************
ok: [web2] => { "msg": "batch = ['web2', 'db1']" }

TASK [Pretend work] ********************************************************************
ok: [web2]
ok: [db1]

PLAY RECAP *****************************************************************************
db1    : ok=1  ...
web1   : ok=2  ...
web2   : ok=2  ...
```

- The `PLAY [...]` header and all tasks repeat **per batch**.
- `run_once` ran **once per batch** (web1 in batch 1, web2 in batch 2), which is why db1 has `ok=1`.

### Stop a bad rollout

Add a task that fails on the first host:

```yaml
    - name: Fails on web1 only
      ansible.builtin.command: /bin/false
      when: inventory_hostname == 'web1'
```

and `max_fail_percentage: 0` on the play:

```bash
ansible-playbook -i inventories/lab playbooks/serial-demo.yml
```
```
TASK [Fails on web1 only] **************************************************************
fatal: [web1]: FAILED! => {...}

NO MORE HOSTS LEFT *********************************************************************

PLAY RECAP *****************************************************************************
web1   : ok=2  changed=0  unreachable=0  failed=1  ...
```

web2 and db1 were **never touched**: the canary failed, the rollout stopped. Without `max_fail_percentage`, the next batch would have started.

### free vs linear

```yaml
- name: Free strategy demo
  hosts: app
  gather_facts: false
  strategy: free
  tasks:
    - ansible.builtin.command: "sleep {{ 4 if inventory_hostname == 'web1' else 1 }}"
      changed_when: false
    - ansible.builtin.debug: { msg: "done step 2" }
```

With `free`, web2 and db1 print "done step 2" while web1 is still sleeping. Switch to `linear` and they wait.

### throttle

```yaml
- name: Throttle demo
  hosts: app
  gather_facts: false
  tasks:
    - name: Only one host at a time
      ansible.builtin.shell: date +%T; sleep 2
      register: t
      throttle: 1
      changed_when: false
    - ansible.builtin.debug: { msg: "{{ t.stdout_lines[0] }}" }
```

The start times are ~2 s apart (one by one), although forks allows all three at once.

### forks

```bash
time ansible -i inventories/lab all -m ansible.builtin.command -a "sleep 2" -f 1     # ~8 s+ (one by one)
time ansible -i inventories/lab all -m ansible.builtin.command -a "sleep 2" -f 10    # ~2 s+
```

---

## Hands-on lab

1. Run `serial-demo.yml` with `serial: 1`, `serial: "50%"`, and `serial: [1, 2]`; predict the batches before each run.
2. Add the failing task; run with and without `max_fail_percentage: 0`; compare which hosts were touched.
3. Add `order: reverse_sorted` and see which host becomes the canary.
4. Run the free/linear demo and the throttle demo; note the timings.
5. Make `run_once` truly once with `serial`: put the "once" task in a **separate play before** the serial play (no `serial` there), and confirm it runs once.
6. Time `ansible all -m ping` with `-f 1` vs `-f 10`.

**Done when:** you can design a rollout: canary first, then growing batches, stop on failure, with protected shared steps.

---

## Where you'll use it

- **Rolling deploys/restarts**: `serial` + `max_fail_percentage` + LB drain (topics 46, 49).
- **Patching**: `serial: "10%"` with reboots and health checks.
- **Database clusters**: `serial: 1` or `throttle: 1`, replicas before primary (`order`, separate plays).
- **Large fleets**: more forks and the `free` strategy for independent work (topic 61).
- **Rate-limited APIs** and licence servers: `throttle`.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| All servers restarted at once → outage | No `serial` | `serial` + LB drain |
| Rollout continued after the canary failed | No `max_fail_percentage` / `any_errors_fatal` | `max_fail_percentage: 0` |
| A "run_once" migration ran several times | `run_once` + `serial` = once per batch | Separate play without `serial`, or idempotent migration |
| `serial: "10%"` on 5 hosts gives batches of 1 | Percent rounds down, minimum 1 | Expected; use explicit numbers for small groups |
| `free` strategy broke a task that reads another host's result | Hosts no longer in step | Use `linear` for dependent plays |
| Control node overloaded / SSH errors with high forks | Too many processes/connections (also `MaxStartups`/`MaxSessions` on a bastion) | Lower forks; tune bastion sshd; ControlPersist |
| `throttle` seems ignored | Set higher than forks, or forks already lower | Throttle ≤ forks |
| Handlers ran per batch, not at the end | Expected with `serial` | That's what you want for rolling restarts |

---

## Quick recap

- **`forks`** = parallel hosts per task (control node limit, default 5). **`throttle`** = per-task limit.
- **Strategies**: `linear` (everyone in step, default), `free` (each host at its own pace), `host_pinned`.
- **`serial`** = batches that each run the whole play (+ handlers): number, `"%"`, or ramp-up list `[1, 5, "25%"]`.
- **`max_fail_percentage`** (per batch) and `any_errors_fatal` stop bad rollouts; `order` picks the canary.
- **`run_once`** = once **per batch**; `ansible_play_batch` shows the current batch.

### Interview questions

1. **Q: What's the difference between `forks` and `serial`?**
   A: `forks` limits how many hosts run a task at the same time; `serial` splits hosts into batches that each complete the whole play before the next batch starts. Forks is about speed; serial is about rollout safety.
2. **Q: How do you do a canary rollout with Ansible?**
   A: `serial: [1, "25%", "100%"]` (or similar) so one host goes first, with `max_fail_percentage: 0` so a failure stops the rollout, ideally with health checks and LB drain per host.
3. **Q: What does `throttle` do?**
   A: Limits the number of hosts that run a particular task concurrently, independent of forks and serial.

### Scenario question (answer out loud first, then compare)

> *"You need to patch and reboot 200 web servers behind a load balancer with at most 10% capacity loss, stop if more than 2 servers fail, and run the database migration exactly once before any web server gets the new code."*

**Model answer:**
1. **Play 1** (`hosts: db_primary` or `localhost` delegated): run the migration once (idempotent, guarded by a schema version check), no `serial`.
2. **Play 2** `hosts: web`, `serial: [1, "5%", "10%"]` (canary, then ≤ 10% at a time), `max_fail_percentage` small (per batch; for an absolute "more than 2 total" rule, also track failures and `meta: end_play`, or use `any_errors_fatal` for zero tolerance).
3. Per host: drain from the LB (`delegate_to`), patch, reboot (`ansible.builtin.reboot`), health check with `uri` + `until`, re-enable (topic 46/49); `block`/`rescue` to leave failed hosts drained and fail them.
4. forks ≥ batch size (e.g. 20) so each batch runs in parallel; `order: sorted` (or by AZ) so batches spread across availability zones.

**Likely follow-ups:**
- *Why a separate play for the migration?* `run_once` in a serial play runs once per batch.
- *How would you spread batches across AZs?* Order hosts by AZ (e.g. constructed groups or `order` on sorted names that include the AZ), or run per-AZ plays.
