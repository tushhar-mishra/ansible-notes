# 61 · Scaling to Many Servers

> **Level:** ★★★ advanced
>
> **You will learn:** what limits Ansible as fleets grow from tens to thousands of hosts (control-node CPU/RAM per fork, SSH connections and bastion limits, latency, facts, inventory APIs, output volume), how to **estimate run time** with simple arithmetic, and the techniques that scale: right-sized **forks**, the connection and fact tuning from topic 48, **sharding** runs by region/group and running them in parallel, putting **execution close to the targets** (regional runners, AAP **instance groups** and **automation mesh** with execution and hop nodes, Kubernetes container groups), **pull mode** for huge uniform fleets, doing less at run time (images, smaller plays), reducing log volume, and the **organisational** side (collections, EEs, team boundaries). You'll create a 20-node container fleet and measure.
>
> **You need:** [60 · Security](60-security.md), [48 · Performance tuning](48-performance-tuning.md), [45 · forks/strategies](45-strategies-forks-serial.md), [55 · ansible-pull](55-ansible-pull-and-drift.md), [56 · AAP](56-awx-and-aap.md), [42 · Docker dynamic inventory](42-dynamic-inventory.md).
>
> **Part 12 · Production operations** · Next: [62 · Project 3: Terraform → Ansible on AWS in CI](62-project-terraform-ansible-aws.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A run takes roughly **(hosts ÷ forks) × tasks × time per task**. You scale by: cutting **time per task** (pipelining, ControlPersist, closer to targets), raising **parallelism** (more forks, more control nodes running **shards** in parallel), and doing **fewer tasks** (images, smaller targeted plays, no needless facts). At the very top end, let hosts configure **themselves** (pull mode).

**Picture it.** Delivering 5,000 parcels: more vans (forks), depots in every city (regional execution nodes), faster drop-offs (pipelining), pre-packed boxes (images), or letting each household collect from a locker (pull).

**The one thing to know.**

```
 estimated time ≈ ceil(hosts / forks) × tasks × seconds_per_task

 5,000 hosts, forks 50, 40 tasks, 1.5 s/task  → 100 × 40 × 1.5 s = 6,000 s ≈ 1 h 40 min
 + pipelining (0.6 s/task)                    → 100 × 40 × 0.6 s = 2,400 s ≈ 40 min
 + 5 regional nodes in parallel (1,000 each)  →  20 × 40 × 0.6 s =   480 s ≈  8 min
```

**Try it now:** plug your own fleet's numbers into the formula; which factor is cheapest to improve?

**You've got it if you can:** explain why adding forks stops helping at some point, and what to do next.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### What limits a single control node

| Limit | Why | Symptom |
|---|---|---|
| **CPU** | Each fork is a Python process templating and parsing results | Control node at 100% CPU, slow progress |
| **Memory** | Each fork holds variables/facts; big inventories and facts multiply | Swapping, OOM kills |
| **File descriptors / processes** | Forks × (SSH master + sessions + pipes) | `Too many open files` |
| **SSH via bastions** | `MaxStartups`/`MaxSessions` on the bastion, its CPU | `Connection reset`, `ssh_exchange_identification` errors |
| **Latency** | Round trips × distance (topic 48) | Time per task grows with distance |
| **Linear waiting** | Every task waits for the slowest host | Long tails |
| **Facts** | Gathering on thousands of hosts every run | Minutes spent before task 1 |
| **Inventory APIs** | Cloud APIs rate-limit; big responses | Slow start, throttling errors |
| **Output volume** | One line per host per task (millions of lines) | Huge logs, slow callbacks, CI log limits |
| **Results in memory** | `register` on thousands of hosts, big outputs | Memory growth |

Rule of thumb: a few hundred forks on a large control node is the practical ceiling; beyond that, **add control nodes** instead of forks.

### Techniques

| # | Technique | How |
|---|---|---|
| 1 | Right-size **forks** | Raise until CPU/RAM/bastion limits, measure (often 50–200 on a big node) |
| 2 | **Cut per-task time** | Pipelining, ControlPersist, `gather_subset`/fact cache (Redis for shared caches), lists instead of loops (topic 48) |
| 3 | **Shard** the run | Split by region/AZ/group: `-l eu_west`, `-l us_east`... run shards **in parallel** jobs, each with its own forks |
| 4 | **Execute close to targets** | Regional CI runners / control nodes; AAP **instance groups** and **automation mesh** |
| 5 | **Container groups** | AWX/AAP spawn job pods on Kubernetes on demand (elastic capacity) |
| 6 | **Strategy** | `free`/`host_pinned` for independent hosts (no waiting for the slowest) |
| 7 | **Do less at run time** | Bake images (Packer + Ansible), keep runtime plays small and targeted (tags, separate playbooks) |
| 8 | **Pull mode** | `ansible-pull` on every host for uniform baselines (topic 55) |
| 9 | **Event-driven** instead of "run everything every hour" | Run only where something changed (EDA, topic 56) |
| 10 | **Less output** | `display_ok_hosts = False`, `display_skipped_hosts = False`, a compact callback, JSON to storage instead of console |
| 11 | **Resilience** | Expect some unreachable hosts at scale: `max_fail_percentage` thresholds, retry lists of failed hosts, don't let one bad host block the fleet |
| 12 | **Inventory caching** | Dynamic inventory cache; regional inventory files |

### Automation mesh (AAP)

```
                 ┌─────────────────────────┐
                 │ Control plane            │  (UI/API, scheduling, DB)
                 └───────────┬─────────────┘
                             │ receptor (mesh protocol over TLS)
          ┌──────────────────┼──────────────────────┐
          ▼                  ▼                      ▼
   ┌─────────────┐   ┌──────────────┐        ┌────────────┐
   │ execution   │   │ hop node     │        │ execution  │
   │ node: EU    │   │ (DMZ relay)  │──────▶ │ node: DMZ  │
   └──────┬──────┘   └──────────────┘        └─────┬──────┘
          │ SSH                                      │ SSH
     EU servers                                  DMZ servers
```

- **Execution nodes** run jobs (EEs) **near** the servers they manage.
- **Hop nodes** relay mesh traffic into restricted networks without running jobs.
- **Instance groups** map job templates/inventories to the right nodes ("DMZ inventory jobs run on the DMZ execution node").
- Capacity per node is computed from CPU/RAM (forks it can handle); the controller schedules jobs accordingly.

### Organisational scaling

Large estates are also a **people** problem:

| Practice | Why |
|---|---|
| Shared content in versioned **collections** (topic 37) | Many teams reuse without forking |
| Team-specific **EEs** (topic 52) | Independent dependency upgrades |
| AWX/AAP **organisations, RBAC, instance groups** (topic 56) | Isolation and self-service |
| Conventions: layout, naming, lint profile, Molecule (topics 39, 50, 51) | Anyone can read anyone's code |
| Inventories per team/environment, dynamic from a source of truth | No giant hand-edited host file |

---

## Why it exists

A playbook that takes 2 minutes on 20 hosts can take hours on 5,000 if nothing changes, and long runs get skipped, which brings back drift and manual changes. Scaling is about keeping runs short enough to stay part of everyday operations, and keeping the system manageable when dozens of people and teams use it.

---

## Analogy

**A national parcel network.** One central warehouse with more vans eventually jams its own loading docks (single control node). The network instead builds regional depots (execution nodes) with local vans (forks), relays for remote islands (hop nodes), pre-packs standard boxes at the factory (images), and lets some customers collect from lockers (pull mode). The head office (control plane) only plans and tracks.

---

## How it works inside

```
 single node:            5,000 hosts ──(forks 50)──▶ 100 waves per task × 40 tasks
 sharded/regional:       EU node    1,000 hosts ─┐
                         US node    1,000 hosts ─┤ in parallel, each forks 50 → 20 waves per task
                         APAC node  1,000 hosts ─┤
                         ...                     ┘
 pull:                   each host runs its own play on a timer; central = Git + reporting
```

---

## Code / commands

### Lab: a 20-node fleet

Add a `fleet` service to `~/ansible-lab/nodes/compose.yaml` (no fixed IP, scalable):

```yaml
  fleet:
    <<: *node
    profiles: [fleet]
    image: ansible-lab/ubuntu2404
    networks: [lab]
```

```bash
cd ~/ansible-lab/nodes
docker compose --profile fleet up -d --scale fleet=20
docker ps --filter "label=com.docker.compose.service=fleet" -q | wc -l       # 20
```

Use the Docker dynamic inventory (topic 42) so you don't need IPs or SSH; the `svc_fleet` group appears automatically:

```bash
cd ~/infra
ansible-inventory -i inventories/docker --graph svc_fleet | head
```

(If your Docker inventory's filter only includes certain services, extend it to include `fleet`.)

### A benchmark play

`playbooks/fleet-bench.yml`:

```yaml
---
- name: Fleet benchmark
  hosts: svc_fleet
  gather_facts: "{{ bench_facts | default(true) | bool }}"
  tasks:
    - name: Ten small tasks
      ansible.builtin.command: "true"
      changed_when: false
      loop: "{{ range(10) | list }}"
    - name: One file
      ansible.builtin.copy:
        content: "fleet\n"
        dest: /tmp/fleet.txt
        mode: "0644"
```

### Measure

```bash
for f in 5 20 40; do
  echo "forks=$f"; /usr/bin/time -f "%e s" ansible-playbook -i inventories/docker playbooks/fleet-bench.yml -f $f >/dev/null
done
/usr/bin/time -f "%e s" ansible-playbook -i inventories/docker playbooks/fleet-bench.yml -f 40 -e bench_facts=false >/dev/null
/usr/bin/time -f "%e s" env ANSIBLE_STRATEGY=free ansible-playbook -i inventories/docker playbooks/fleet-bench.yml -f 40 -e bench_facts=false >/dev/null
```

Example results (yours will differ):

| Setup | Time |
|---|---|
| forks 5 | 74 s |
| forks 20 | 23 s |
| forks 40 | 19 s (diminishing: control node CPU now the limit) |
| forks 40, no facts | 15 s |
| + free strategy | 13 s |

Watch the control node while it runs: `top` (many `ansible-playbook` processes, CPU near 100% at high forks).

### Sharding: run groups in parallel

Split the fleet into two shards with the `constructed` plugin (by container name parity, as a stand-in for "regions"):

```yaml
# inventories/docker/99-shards.yml
plugin: ansible.builtin.constructed
strict: false
groups:
  shard_a: inventory_hostname is search('[13579]$')
  shard_b: inventory_hostname is search('[02468]$')
```

```bash
( ansible-playbook -i inventories/docker playbooks/fleet-bench.yml -l 'svc_fleet:&shard_a' -f 20 >/tmp/a.log; echo "A rc=$?" ) &
( ansible-playbook -i inventories/docker playbooks/fleet-bench.yml -l 'svc_fleet:&shard_b' -f 20 >/tmp/b.log; echo "B rc=$?" ) &
wait
```

On one control node this mostly helps when you're limited by **waiting** (latency) rather than CPU; with shards on **different** machines (regional runners/execution nodes), it scales almost linearly.

### Less output at scale

```ini
[defaults]
display_ok_hosts = False          # only print changed/failed/unreachable
display_skipped_hosts = False
```

```bash
ANSIBLE_DISPLAY_OK_HOSTS=False ansible-playbook -i inventories/docker playbooks/fleet-bench.yml -f 40
```

Store full results as JSON (topic 55) instead of scrolling millions of lines.

### Retry only the failed hosts

```bash
ANSIBLE_STDOUT_CALLBACK=ansible.posix.json ansible-playbook -i inventories/docker playbooks/fleet-bench.yml > run.json
jq -r '.stats | to_entries[] | select(.value.failures > 0 or .value.unreachable > 0) | .key' run.json > failed.txt
ansible-playbook -i inventories/docker playbooks/fleet-bench.yml -l @failed.txt
```

### Clean up

```bash
cd ~/ansible-lab/nodes && docker compose --profile fleet down
```

---

## Hands-on lab

1. Create the 20-node fleet and run the benchmark with forks 5/20/40, facts on/off, linear/free. Record your table.
2. Watch `top` on the control node during the forks 40 run; note when CPU becomes the bottleneck.
3. Shard with `constructed` groups and run two shards in parallel; compare with one run.
4. Make 3 fleet containers "fail" (`docker stop` them mid-run), then use the JSON + `jq` + `@failed.txt` retry pattern.
5. Turn on `display_ok_hosts = False` and compare log sizes (`wc -l`).
6. Using the formula, estimate the run time of `site.yml` for 2,000 hosts in 4 regions and propose an architecture (forks, runners/execution nodes, facts, pull vs push).

**Done when:** you can predict roughly how long a run will take at scale and name the next bottleneck after each improvement.

---

## Where you'll use it

- **Large enterprises and cloud fleets** (thousands of VMs across regions and networks).
- **AAP design**: execution nodes per region/zone, hop nodes into DMZs, instance groups per team.
- **Patch days**: sharded, batched runs with failure thresholds and retry lists.
- **Interviews (senior)**: "How would you run Ansible against 10,000 servers?"

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| Control node at 100% CPU, little speed-up from more forks | CPU-bound | More/bigger control nodes; shard across machines |
| OOM kills of `ansible-playbook` | Too many forks × big facts/registers | Lower forks, `gather_subset`, avoid huge `register`s, more RAM |
| `Too many open files` | FD limit | Raise `ulimit -n` for the runner user |
| Bastion refusing connections at high forks | `MaxStartups`/`MaxSessions` | Tune the bastion, multiple bastions, execution nodes inside networks |
| Cloud inventory throttling | API rate limits | Inventory cache, fewer regions per source, filters |
| Runs take hours because a few hosts are slow | Linear strategy waits | `free` strategy, timeouts, async, isolate slow hosts |
| CI job hits log size limits | Massive output | `display_ok_hosts = False`, JSON artifacts |
| Every run has some unreachable hosts | Normal churn at scale | Thresholds, retry lists, alert on persistent ones |

---

## Quick recap

- Run time ≈ **(hosts ÷ forks) × tasks × time per task**. Improve the cheapest factor first.
- Single control node limits: CPU, RAM, FDs, SSH/bastion limits, latency, facts, output.
- Scale with: right-sized forks, topic 48 tuning, **sharding** across parallel jobs, **execution close to targets** (regional runners, AAP **instance groups**, **automation mesh** execution/hop nodes, container groups), `free` strategy, images, **pull mode**, event-driven runs, less output, retry lists.
- Organise people too: collections, EEs, RBAC, conventions.

### Interview questions

1. **Q: How would you run Ansible against 10,000 servers?**
   A: Tune per-task cost (pipelining, ControlPersist, fact caching/subsets), shard the inventory by region/group and run shards in parallel on control/execution nodes close to the servers (AAP automation mesh/instance groups or regional runners), use reasonable forks per node, free strategy where safe, minimise runtime work with images, reduce output, and handle unreachable hosts with thresholds and retries. For uniform baselines, consider pull mode.
2. **Q: Why doesn't raising forks scale forever?**
   A: Each fork costs CPU, memory and connections on the control node (and bastions); eventually those, not the network, become the bottleneck.
3. **Q: What is automation mesh?**
   A: AAP's overlay network (receptor) connecting the control plane to execution nodes (which run jobs near targets) and hop nodes (which relay into restricted networks).

### Scenario question (answer out loud first, then compare)

> *"Nightly configuration runs against 6,000 servers in 3 regions now take 5 hours from a single Jenkins agent and often don't finish before business hours. Redesign it."*

**Model answer:**
1. **Measure** with `profile_tasks`/`profile_roles` on a sample; find facts, slow tasks, latency.
2. **Tune** (topic 48): pipelining, ControlPersist, Redis fact cache with `gathering = smart`, lists not loops.
3. **Distribute**: one runner/execution node **per region** (or AAP execution nodes via mesh), each running its region's shard in parallel with forks sized to the node (e.g. 100).
4. **Do less**: split the nightly run into a fast **drift check** (`--check`, compact output) plus targeted remediation only where drift is found; move slow, static setup into images.
5. **Resilience**: `max_fail_percentage` per shard, retry lists for failures, alerts for hosts unreachable for >24 h.
6. Estimate: 2,000 hosts/region ÷ 100 forks = 20 waves × 40 tasks × ~0.5 s ≈ 7 min per region, in parallel.

**Likely follow-ups:**
- *Why not pull mode for all of it?* Possible for the baseline; but you lose central orchestration and must distribute secrets to every host; hybrid is common.
- *How do you keep the shards consistent?* Same Git commit, same EE tag, same playbook; only `--limit`/inventory differs.
