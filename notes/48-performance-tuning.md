# 48 · Performance Tuning

> **Level:** ★★★ advanced
>
> **You will learn:** how to make Ansible fast on big fleets, **measuring first** (`profile_tasks`, `timer`, `time`, `-vvv` timings) and then pulling the right levers: **forks**, **SSH pipelining**, **ControlPersist**, **fact gathering** (`gather_facts: false`, `gather_subset`, fact caching with `gathering = smart`), **task design** (lists instead of loops, one template instead of many line edits, `synchronize`/archives for many files), **strategies** and **async**, dynamic-inventory caching, package mirrors, where to run the control node, and the trade-offs of each (including third-party accelerators like Mitogen). You'll benchmark the lab before and after.
>
> **You need:** [47 · Async](47-async-and-polling.md), [45 · forks and strategies](45-strategies-forks-serial.md), [13 · Inside a module run](13-inside-a-module-run.md) (the ~6 SSH round trips), [05 · ControlPersist](05-linux-recap-ssh.md), [23 · Facts](23-facts-and-magic-variables.md).
>
> **Part 7 · Execution control and speed** · Next: [49 · Project 1: zero-downtime rolling deploy](49-project-rolling-deploy.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Ansible's time goes into **(1)** connecting and copying modules for every task on every host, **(2)** gathering facts, **(3)** waiting for the slowest host, **(4)** the work itself. Measure, then cut the biggest one.

**Picture it.** A delivery company: fewer trips per parcel (pipelining), keep the van's engine running between stops (ControlPersist), more vans (forks), don't re-inspect every house each visit (fact caching), deliver one box instead of 20 envelopes (lists instead of loops).

**The one thing to know.** The usual quick wins, in `ansible.cfg`:

```ini
[defaults]
forks = 25
gathering = smart
fact_caching = ansible.builtin.jsonfile
fact_caching_connection = ~/.ansible/facts_cache
fact_caching_timeout = 7200
callbacks_enabled = ansible.posix.profile_tasks, ansible.posix.timer

[ssh_connection]
pipelining = True
ssh_args = -C -o ControlMaster=auto -o ControlPersist=600s -o ServerAliveInterval=30
```

**Try it now:** time the same playbook before and after these settings (Hands-on lab).

**You've got it if you can:** explain what pipelining removes from a module run, and its one requirement on the target.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Where the time goes

| Cost | Per | Typical size | Lever |
|---|---|---|---|
| SSH connection setup (handshake + auth) | host (×tasks without ControlPersist) | 0.1–1 s (much more via bastions/long distance) | **ControlPersist**, keepalive |
| Module round trips (mkdir, put, chmod, run, rm) | task × host | ~6 round trips | **Pipelining** (→ 1) |
| Fact gathering | play × host | 1–10 s | `gather_facts: false`, **`gather_subset`**, **fact cache** |
| Waiting for the slowest host | task (linear) | varies | `free` strategy, fix the slow host, async |
| Too few parallel hosts | task | hosts/forks waves | **forks** |
| Loops (one module run per item) | item × host | N × round trips | **Pass lists** to modules |
| Large/many file transfers | task | size/number | `synchronize`, archives, mirrors |
| Package downloads | host | network | Local mirror/proxy, `cache_valid_time` |
| Dynamic inventory API calls | run | 1–30 s | Inventory **cache** |

### The levers in detail

**1. forks** (topic 45). Default 5 is low for big fleets. 20–50 is common; more needs a bigger control node (each fork ≈ a Python process + SSH). Watch CPU/RAM and bastion SSH limits (`MaxStartups`, `MaxSessions`).

**2. ControlPersist.** Keeps one SSH master connection per host open between tasks (topic 05). Default `60s`; raise it (e.g. `600s`) so masters survive long tasks and gaps. Set `control_path_dir` to a short path if you hit "path too long" errors.

**3. Pipelining.** Sends the module to the remote Python over **stdin** in the **same** SSH session, instead of mkdir → sftp → chmod → run → rm:

| | Without pipelining | With pipelining |
|---|---|---|
| Round trips per task | ~5–6 | 1 |
| Temp files on target | Yes | No (also fixes the become-unprivileged problem, topic 19) |
| Requirement | — | sudo must **not** require a TTY (`Defaults requiretty` must be off for the automation user; it's off by default on modern distros) |

Enable with `pipelining = True` in `[ssh_connection]` (or `ANSIBLE_PIPELINING=True`). Modules that need to transfer files (`copy`, `template`) still transfer the file content; only the module itself is piped.

**4. Facts.**

| Option | Effect |
|---|---|
| `gather_facts: false` on plays that don't use facts | Removes the whole step |
| `gather_subset: [min]` / `['!all', 'network']` | Collect only what you need |
| `gather_timeout: 10` | Don't let a hung NFS mount or slow hardware query stall the run |
| `gathering = smart` + `fact_caching = jsonfile`/`redis` | Gather only if not cached (or expired) |
| `gathering = explicit` | Never gather unless a play says `gather_facts: true` |

**5. Task design.**

| Slow | Fast |
|---|---|
| `package` in a loop of 30 names | `package: name: "{{ list }}"` (one transaction) |
| 50 `lineinfile` tasks on one file | One `template` |
| `copy` of a directory with 2,000 files | `ansible.posix.synchronize` (rsync), or `archive` + `unarchive` |
| `command` per user/item to check state | One command that lists everything + filters (`register` + Jinja2) |
| `include_tasks` in a big loop | `import_tasks` where possible; loops inside one module call |
| `become: true` on every task, including reads | Only where needed (each become adds a sudo step) |
| `shell` pipelines for data processing on targets | Process on the control node with filters |

**6. Strategy.** `free` lets fast hosts finish without waiting (topic 45). Long jobs: `async` (topic 47).

**7. Inventory.** Cache dynamic inventories (topic 42); filter server-side; limit regions.

**8. Where the control node runs.** Close to the targets (same region/VPC). A control node in Europe managing servers in Asia pays ~250 ms per round trip × thousands of round trips.

**9. Third-party accelerators.** **Mitogen for Ansible** replaces the execution path and was famous for 2–7× speed-ups, but it's a third-party project that historically lagged behind new ansible-core releases. Check its compatibility with your ansible-core version before considering it, and prefer the built-in levers first.

### Measure, don't guess

| Tool | Shows |
|---|---|
| `ansible.posix.profile_tasks` callback | Time per task (slowest first) |
| `ansible.posix.timer` | Total run time |
| `ansible.posix.profile_roles` | Time per role |
| `time ansible-playbook ...` | Wall-clock time |
| `-vvv` | Each SSH command (count round trips) |
| `ANSIBLE_DEBUG=1` (very verbose) | Internal timings; rarely needed |

---

## Why it exists

Ansible's agentless design costs round trips. On 5 hosts nobody notices; on 2,000 hosts a 40-task playbook can take hours if untuned, which makes people skip runs, which brings back drift. A tuned setup turns "the nightly run takes 3 hours" into "15 minutes", keeping automation usable at scale.

---

## Analogy

**Running errands.** Without ControlPersist, you start the car, drive to the shop, park, switch off the engine, walk in... for **every** item on your list. With ControlPersist, you keep the engine running between shops. Pipelining = instead of five trips (bring the bag, open it, unpack, do the job, take the bag home), you hand over everything in one go. Forks = more family members running errands at the same time. Fact caching = not re-measuring every room of the house each time you redecorate one.

---

## How it works inside

Without pipelining (topic 13), per task per host:

```
 echo ~  →  mkdir tmp  →  sftp put AnsiballZ  →  chmod  →  python3 file  →  rm tmp     (≈6 round trips)
```

With pipelining:

```
 ssh host '/bin/sh -c "sudo -H -S -n -u root /bin/sh -c '\''echo BECOME-SUCCESS-x; /usr/bin/python3'\''"'  <  AnsiballZ (stdin)
                                                                                                 (1 round trip)
```

With ControlPersist, each of those round trips reuses an already-authenticated master connection (no new handshake).

---

## Code / commands

### A benchmark playbook

`playbooks/bench.yml` (20 cheap tasks, so overhead dominates):

```yaml
---
- name: Benchmark
  hosts: all
  become: true
  tasks:
    - name: Many small tasks
      ansible.builtin.file:
        path: "/tmp/bench/{{ item }}"
        state: directory
        mode: "0755"
      loop: "{{ range(1, 21) | list }}"

    - name: Packages, one by one (slow on purpose)
      ansible.builtin.package:
        name: "{{ item }}"
        state: present
      loop: [tree, curl, rsync]
```

### Baseline vs tuned

```bash
cd ~/infra
rm -f ~/.ansible/cp/*                                     # no warm SSH masters
# Baseline: forks 5, no pipelining, facts every time
time ANSIBLE_PIPELINING=False ANSIBLE_FORKS=5 ANSIBLE_GATHERING=implicit \
  ansible-playbook -i inventories/lab playbooks/bench.yml >/dev/null
```

```bash
# Tuned: pipelining + forks + smart facts with cache (warm the cache once first)
ansible -i inventories/lab all -m ansible.builtin.setup >/dev/null
time ANSIBLE_PIPELINING=True ANSIBLE_FORKS=20 ANSIBLE_GATHERING=smart \
  ANSIBLE_CACHE_PLUGIN=ansible.builtin.jsonfile ANSIBLE_CACHE_PLUGIN_CONNECTION=~/.ansible/facts_cache \
  ansible-playbook -i inventories/lab playbooks/bench.yml >/dev/null
```

Example results on the lab (your numbers will differ; local containers have tiny latency, so gains are smaller than over a real network):

| Setup | Time |
|---|---|
| Baseline | 34 s |
| + pipelining | 19 s |
| + forks 20 | 16 s |
| + fact cache (smart) | 13 s |
| + `package` with a list instead of a loop | 11 s |

Count round trips per task with `-vvv`:

```bash
ANSIBLE_PIPELINING=False ansible -i inventories/lab web1 -b -m ansible.builtin.ping -vvv 2>&1 | grep -c 'SSH: EXEC'   # ~5–6
ANSIBLE_PIPELINING=True  ansible -i inventories/lab web1 -b -m ansible.builtin.ping -vvv 2>&1 | grep -c 'SSH: EXEC'   # ~1–2
```

### Simulate a slow network to see the real effect

On the WSL2 host, add 50 ms latency to the lab bridge (Linux traffic control, [Linux topic 095](../../linux-mastery-rhel/notes/095-tcp-ip-deep-dive.md)):

```bash
br=$(docker network inspect ansible-lab -f '{{.Id}}' | cut -c1-12)
sudo tc qdisc add dev br-$br root netem delay 50ms
# re-run baseline and tuned → the difference grows a lot
sudo tc qdisc del dev br-$br root netem
```

### Profile a real playbook

```bash
ANSIBLE_CALLBACKS_ENABLED=ansible.posix.profile_tasks,ansible.posix.profile_roles,ansible.posix.timer \
  ansible-playbook -i inventories/lab playbooks/site.yml
```
```
Playbook run took 0 days, 0 hours, 0 minutes, 21 seconds
===============================================================================
nginx_web : Packages are installed --------------------------------------- 4.12s
Gathering Facts ---------------------------------------------------------- 3.05s
lb_nginx : nginx.conf is rendered and validated -------------------------- 1.10s
...
===============================================================================
nginx_web ---------------------------------------------------------------- 7.81s
lb_nginx ----------------------------------------------------------------- 3.40s
```

Fix the top items first.

---

## Hands-on lab

1. Run the baseline and tuned benchmarks; fill in your own table.
2. Count `SSH: EXEC` lines with and without pipelining.
3. Change the package loop to a list and re-measure.
4. Add 50 ms latency with `tc`, repeat 1, remove the latency.
5. Add the tuned settings to `~/infra/ansible.cfg` permanently (keep `profile_tasks` on).
6. Find the three slowest tasks in `site.yml` with `profile_tasks` and improve at least one.
7. Check that pipelining works with `become` on all lab hosts (no `requiretty` errors).

**Done when:** you can explain where your playbook's time goes and have cut it with measured changes.

---

## Where you'll use it

- **Large fleets** (hundreds to thousands of hosts) and **slow networks** (bastions, cross-region, VPN).
- **CI pipelines** with time limits.
- **AWX/AAP** job templates (forks per template, fact cache in the controller, topic 56).
- **Scaling design** (topic 61): forks, batching, pull mode, multiple execution nodes.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `sudo: sorry, you must have a tty to run sudo` with pipelining | `Defaults requiretty` in sudoers | Remove it for the automation user, or disable pipelining |
| `ControlPath too long` | Long home/user/host names in the socket path | `control_path_dir = /tmp/.ansible-cp` (short), or `control_path = %(directory)s/%%C` |
| `mux_client_request_session: session request failed` / `Too many open sessions` | Too many sessions over one master (sshd `MaxSessions`) or bastion limits | Raise `MaxSessions`/`MaxStartups` on the bastion, lower forks |
| Control node runs out of memory | Forks too high, huge fact sets | Lower forks, `gather_subset`, more RAM |
| Facts wrong after enabling the cache | Stale cache | Lower `fact_caching_timeout`, `--flush-cache` |
| No speed-up from pipelining on `copy`-heavy plays | File content still transferred | `synchronize`/archives for many files |
| Runs still slow with high forks | Linear waiting on one slow host | Find the host (`-v` timings), `free` strategy, async |
| Mitogen errors after an ansible-core upgrade | Unsupported core version | Remove it; use built-in tuning |

---

## Quick recap

- **Measure first**: `profile_tasks`, `profile_roles`, `timer`, `time`, `-vvv`.
- **Connections**: `pipelining = True` (needs no `requiretty`), **ControlPersist** longer (e.g. 600 s), forks 20–50 within control-node limits.
- **Facts**: `gather_facts: false`, `gather_subset`, `gathering = smart` + **fact cache**, `gather_timeout`.
- **Tasks**: lists not loops, templates not many line edits, `synchronize`/archives for many files, become only where needed.
- **Elsewhere**: `free` strategy/async for slow hosts, inventory caching, mirrors, control node close to targets.

### Interview questions

1. **Q: How do you speed up Ansible on a large fleet?**
   A: Measure with profile callbacks, then: enable pipelining, tune ControlPersist, raise forks, reduce or cache fact gathering, pass lists to modules instead of looping, use `free` strategy/async where appropriate, cache dynamic inventory, and run the control node close to the hosts.
2. **Q: What does pipelining do, and what can break it?**
   A: It sends modules over the existing SSH session's stdin instead of copying temp files, cutting round trips from ~6 to 1. It fails if sudo requires a TTY (`requiretty`).
3. **Q: What is `gathering = smart`?**
   A: Gather facts only if they aren't already in the fact cache (or have expired), instead of every play.

### Scenario question (answer out loud first, then compare)

> *"Your 60-task baseline playbook takes 2.5 hours across 1,500 servers in three regions, run from one Jenkins agent in one region. Leadership wants it under 20 minutes. What's your plan?"*

**Model answer:**
1. **Measure**: `profile_tasks`/`profile_roles` + `timer` on a representative subset; look for fact gathering, loops, slow hosts, and per-region latency.
2. **Quick wins**: pipelining, ControlPersist 600 s, forks 50+ on a bigger agent, `gather_subset` + jsonfile/redis fact cache with `gathering = smart`.
3. **Fix tasks**: package lists instead of loops, templates instead of line edits, remove needless `command` probes, `become` only where needed.
4. **Distribute**: run one job **per region** on agents/execution nodes **in** each region (parallel, low latency), or AAP with instance groups / automation mesh (topic 61).
5. **Reduce work**: bake slow-changing parts into images; for the nightly drift check, `--check` only on changed roles or tags.
6. Re-measure after each step; keep the `profile_tasks` output in CI to catch regressions.

**Likely follow-ups:**
- *Why not just raise forks to 1,500?* Control node CPU/RAM/file descriptors and SSH limits; diminishing returns once you're latency-bound.
- *Risk of the fact cache?* Stale facts; use a sensible timeout and refresh on change-heavy runs.
