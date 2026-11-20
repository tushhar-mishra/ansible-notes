# 46 · Delegation and Local Actions

> **Level:** ★★ intermediate
>
> **You will learn:** how to run a task **on a different machine** than the host it's "for" with **`delegate_to`** (the load balancer, the control node, a DNS server), what stays the same and what changes when you delegate (variables vs connection), **`delegate_to: localhost`** / `connection: local` and ⚠️ `local_action`, **`delegate_facts`**, **`run_once` + `delegate_to`**, and the classic **load-balancer drain/enable** pattern. You'll take `web1` out of the `lb1` load balancer and put it back.
>
> **You need:** [45 · Strategies, forks and serial](45-strategies-forks-serial.md), [29 · Templates](29-templates.md) (the lb1 config), [30 · Handlers](30-handlers.md), [19 · become](19-become.md) (the localhost trap).
>
> **Part 7 · Execution control and speed** · Next: [47 · Async tasks and long jobs](47-async-and-polling.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Sometimes a task **about** web1 must **run somewhere else**: remove web1 from the load balancer (runs on lb1), check web1's URL (runs on the control node), add a DNS record for web1 (runs on the DNS server or via an API). `delegate_to: <host>` does exactly that: the task executes on `<host>`, but with **web1's variables**.

**Picture it.** You (web1) are moving house. You don't go to the post office yourself; you **delegate** the change of address to a friend (the delegated host), who fills in the form with **your** details.

**The one thing to know.**

```yaml
- name: Take this host out of the load balancer
  ansible.builtin.command: "/usr/local/bin/lb-disable {{ inventory_hostname }}"
  delegate_to: lb1                  # RUNS on lb1, with web1's variables

- name: Check the page from the control node
  ansible.builtin.uri:
    url: "http://{{ hostvars[inventory_hostname]['ansible_host'] }}/"
  delegate_to: localhost
  become: false                     # don't try sudo on your laptop
```

**Try it now:** `ansible -i inventories/lab web -m ansible.builtin.command -a hostname` vs the same task delegated to `lb1` in a playbook: the second prints `lb1` for both.

**You've got it if you can:** say which variables a delegated task sees and which host it connects to.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### What changes and what doesn't

When web1's task has `delegate_to: lb1`:

| Thing | Comes from |
|---|---|
| Where the module **runs** (SSH connection, user, key, port, Python, become) | **lb1**'s connection variables (`ansible_host`, `ansible_user`... if lb1 is in the inventory) |
| `inventory_hostname`, `group_names`, facts, `group_vars`/`host_vars` values used in the task | **web1** (the original host) |
| Output line | `ok: [web1 -> lb1]` |
| Registered result | Stored for **web1** |

⚠️ Watch out for **connection variables in task arguments**: `{{ ansible_host }}` in a delegated task can resolve to the **delegated** host's address. To get the original host's address unambiguously, use `{{ hostvars[inventory_hostname]['ansible_host'] }}`.

If the delegate isn't in the inventory, Ansible connects to that name/IP with default settings.

### Delegating to the control node

| Form | Use |
|---|---|
| `delegate_to: localhost` | ✅ Normal form. Runs on the control node |
| `connection: local` (play or task) | The whole play/task runs locally (APIs, Kubernetes, topic 44) |
| ⚠️ `local_action: module args` | Old shorthand for `delegate_to: localhost`; harder to read, discouraged |

Remember topic 19: with play-level `become: true`, a task delegated to localhost tries **sudo on your machine**. Add `become: false`.

### delegate_facts

Normally, facts gathered by a delegated task are stored for the **original** host. With `delegate_facts: true`, they're stored for the **delegated** host:

```yaml
- name: Gather facts of the DB servers from the web play
  ansible.builtin.setup:
  delegate_to: "{{ item }}"
  delegate_facts: true
  loop: "{{ groups['db'] }}"
  run_once: true
```

Now `hostvars['db1']['ansible_facts']` is filled (topic 23's "other host's facts" problem).

### run_once + delegate_to

"Do this once, on that specific machine":

```yaml
- name: Announce the deploy once
  ansible.builtin.uri:
    url: https://chat.example.com/hooks/abc
    method: POST
    body_format: json
    body: { text: "Deploying {{ app_version }} to {{ ansible_play_hosts | length }} hosts" }
  run_once: true
  delegate_to: localhost
  become: false
```

(Remember: with `serial`, `run_once` runs once **per batch**, topic 45.)

### The load-balancer drain pattern

```
 for each web host (serial: 1):
   pre_tasks:   disable host in LB      ← delegate_to: <lb>
                wait for connections to drain
   tasks:       deploy / patch / restart
                health check            ← delegate_to: localhost (or the LB)
   post_tasks:  enable host in LB       ← delegate_to: <lb>
```

How you "disable" depends on the load balancer:

| Load balancer | Drain with |
|---|---|
| HAProxy | `community.general.haproxy` module (runtime API socket): `state: disabled` / `enabled`, `drain: true` |
| nginx (open source) | Mark the server `down` in the upstream and reload (lab below); or nginx Plus API |
| AWS ALB/NLB | `community.aws.elb_target` / target group modules from the control node (`state: absent`/`present`, deregistration delay) |
| F5 BIG-IP | `f5networks.f5_modules.bigip_pool_member` (`state: disabled`/`enabled`) |
| Kubernetes | Not needed: readiness probes and rolling Deployments do it |

---

## Why it exists

Real operations involve **several machines per step**: the server being changed, plus load balancers, monitoring, DNS, ticketing and chat systems, cloud APIs. Without delegation you'd split everything into separate plays and pass data between them awkwardly. With it, one task "about" web1 can act on whatever system needs to know about web1, in the right order, inside the same rolling batch.

---

## Analogy

**A hospital ward round.** The doctor works patient by patient (host by host). For patient Ali, some actions happen at the bedside (normal tasks), but others are delegated: "pharmacy, prepare Ali's medicine" (task runs at the pharmacy, using Ali's chart), "lab, run Ali's blood test". The output says "Ali → pharmacy". The doctor doesn't leave Ali's context; the work just happens elsewhere.

---

## How it works inside

```
 task (host=web1) with delegate_to: lb1
   │
   ├─ variables: build web1's vars (as always)
   │     + "delegated vars": lb1's connection settings (ansible_host, ansible_user, port, become...)
   ├─ render task args with web1's vars          ← inventory_hostname = web1
   ├─ connection plugin: connect to lb1 (lb1's address/user/key)
   ├─ run the module on lb1
   └─ result stored as web1's result; printed as [web1 -> lb1]
```

Delegated tasks still respect forks: with `serial: 1`, only one host's delegated task hits the LB at a time; without serial, many hosts could delegate to the LB **at once** (consider `throttle: 1` for LB changes).

---

## Code / commands

### Lab: drain web1 from the lab load balancer

The lb1 config from topic 29 has lines like:

```
        server 172.30.0.11:80;  # web1
        server 172.30.0.12:80;  # web2
```

nginx skips upstream servers marked `down`. We'll toggle that marker, delegated to lb1.

> ⚠️ **Corrected design.** An earlier version edited `/etc/nginx/nginx.conf` with `replace`. But the lb role (topic 29) owns that file through `template`, which breaks the "one owner per file" rule (topics 27, 43): the next run of `lb.yml` silently re-renders the file and **puts a drained host back into rotation**. It also reloaded nginx without testing the config first.
>
> Fix: split ownership. The lb role's template only `include`s one small file per backend, and **only `lb_member` writes those files**. (With HAProxy or a cloud LB, use the runtime API instead: `community.general.haproxy`, or target-group deregistration. Then the drain state never lives in a config file at all.)

**One-time change to topic 29's template** (`roles/lb/templates/nginx.conf.j2`): replace the `{% for host in groups[lb_backend_group] %} server … {% endfor %}` loop with:

```jinja
    upstream backend {
{% if lb_algorithm == 'least_conn' %}
        least_conn;
{% endif %}
        include /etc/nginx/upstream.d/*.conf;   # one file per backend, owned by lb_member
    }
```

and have the lb role create the directory and **seed** a file for any backend that has none yet (`force: false`, so it never overwrites a drain):

```yaml
- name: Upstream member directory
  ansible.builtin.file: { path: /etc/nginx/upstream.d, state: directory, mode: "0755" }

- name: Seed a member file for new backends (never overwrite: lb_member owns the state)
  ansible.builtin.copy:
    dest: "/etc/nginx/upstream.d/{{ item }}.conf"
    content: "server {{ hostvars[item]['ansible_host'] }}:{{ hostvars[item]['http_port'] | default(80) }};\n"
    force: false
    mode: "0644"
  loop: "{{ groups[lb_backend_group] }}"
  notify: Reload nginx
```

`roles/lb_member/tasks/disable.yml`:

```yaml
---
- name: Mark this host down in the load balancer upstream
  ansible.builtin.copy:
    dest: "/etc/nginx/upstream.d/{{ inventory_hostname }}.conf"
    content: "server {{ lb_member_address }}:{{ lb_member_port }} down;  # drained by Ansible\n"
    mode: "0644"
  delegate_to: "{{ lb_member_lb_host }}"
  become: true
  register: __lb_member_disable

- name: Test the nginx config before reloading
  ansible.builtin.command: nginx -t
  changed_when: false
  delegate_to: "{{ lb_member_lb_host }}"
  become: true
  when: __lb_member_disable is changed

- name: Reload the load balancer now
  ansible.builtin.systemd_service:
    name: nginx
    state: reloaded
  delegate_to: "{{ lb_member_lb_host }}"
  become: true
  when: __lb_member_disable is changed
```

`roles/lb_member/tasks/enable.yml`: the same three tasks, with `content: "server {{ lb_member_address }}:{{ lb_member_port }};\n"` and `register: __lb_member_enable`, and the task name `Mark this host up in the load balancer upstream`.

`roles/lb_member/defaults/main.yml`:

```yaml
---
lb_member_lb_host: "{{ groups['lb'][0] }}"
lb_member_address: "{{ hostvars[inventory_hostname]['ansible_host'] }}"
lb_member_port: "{{ hostvars[inventory_hostname]['http_port'] | default(80) }}"
```

- **Test the fix:** drain web1, run `lb.yml`, then `curl` the LB several times. web1 must stay out of rotation. With the old `replace` design it came back.

- `lb_member_address` uses `hostvars[inventory_hostname]` so it's **web1's** IP even inside a task delegated to lb1.
- The reload happens immediately (not via a handler), because the next steps depend on the LB already skipping this host.
- The drain state is now a file on lb1 with exactly one writer, so a re-run of any other role can't undo it.
- If two pipelines can drain at once (two deploys racing), use `throttle: 1` on these tasks, or a lock (topic 49), so the LB never ends up with every backend `down`.

`playbooks/drain-demo.yml`:

```yaml
---
- name: Drain and re-enable web servers one at a time
  hosts: web
  serial: 1
  gather_facts: false
  tasks:
    - name: Drain
      ansible.builtin.include_role:
        name: lb_member
        tasks_from: disable

    - name: While drained, the LB only uses the other server
      ansible.builtin.uri:
        url: "http://{{ hostvars[groups['lb'][0]]['ansible_host'] }}/"
        return_content: true
      register: lb_page
      loop: [1, 2, 3, 4]
      delegate_to: localhost
      become: false

    - name: Show which backends answered
      ansible.builtin.debug:
        msg: "{{ lb_page.results | map(attribute='content') | map('trim') | unique | list }}"

    - name: Pause (pretend we deploy here)
      ansible.builtin.pause:
        seconds: 2

    - name: Re-enable
      ansible.builtin.include_role:
        name: lb_member
        tasks_from: enable
```

```bash
cd ~/infra
ansible-playbook -i inventories/lab playbooks/lb.yml          # make sure lb1 is configured (topic 30/39)
ansible-playbook -i inventories/lab playbooks/drain-demo.yml
```
```
TASK [lb_member : Mark this host down in the load balancer upstream] ********************
changed: [web1 -> lb1]

TASK [lb_member : Reload the load balancer now] ****************************************
changed: [web1 -> lb1]

TASK [While drained, the LB only uses the other server] ********************************
ok: [web1 -> localhost] => (item=1)
...
TASK [Show which backends answered] ****************************************************
ok: [web1] => { "msg": ["Hello from web2"] }
...
TASK [lb_member : Mark this host up in the load balancer upstream] **********************
changed: [web1 -> lb1]
...
(then the same for web2: only "web1 is the canary" answers while web2 is drained)
```

- `[web1 -> lb1]`: web1's task executed on lb1.
- While web1 was drained, all four requests were served by web2.

### Facts of other hosts with delegate_facts

```yaml
- name: Web play that needs DB facts
  hosts: web
  gather_facts: false
  tasks:
    - name: Gather DB facts once
      ansible.builtin.setup:
        gather_subset: [network]
      delegate_to: "{{ item }}"
      delegate_facts: true
      loop: "{{ groups['db'] }}"
      run_once: true

    - name: Use them
      ansible.builtin.debug:
        msg: "DB address: {{ hostvars['db1']['ansible_facts']['default_ipv4']['address'] }}"
```

### A local report, written once

```yaml
- name: Fleet report on the control node
  ansible.builtin.copy:
    dest: "{{ playbook_dir }}/../reports/fleet.txt"
    content: |
      {% for h in ansible_play_hosts_all %}
      {{ h }}: {{ hostvars[h]['ansible_facts']['distribution'] | default('?') }}
      {% endfor %}
    mode: "0644"
  run_once: true
  delegate_to: localhost
  become: false
```

---

## Hands-on lab

1. Create the `lb_member` role and `drain-demo.yml`; run it and confirm each host is drained in turn.
2. Run the drain task alone for web1 (`--tags`, or a play with only the disable part), then `curl -s 172.30.0.31` several times: only web2 answers. Re-enable.
3. Replace `hostvars[inventory_hostname]['ansible_host']` with plain `ansible_host` in `lb_member_address` and run with `-v`: check which IP the regexp actually used. Restore it.
4. Remove `become: false` from the `uri` task and add `become: true` at the play level: read the localhost sudo error. Fix it.
5. Use `delegate_facts` to fill db1's facts from a web play.
6. Write the fleet report with `run_once` + `delegate_to: localhost`.

**Done when:** you can drain/enable a host in a load balancer as part of a per-host workflow.

---

## Where you'll use it

- **Rolling deploys/patching**: LB drain/enable around each host (topic 49).
- **Monitoring**: put a host into maintenance mode in Nagios/Zabbix/Datadog/PagerDuty before changes, and back after.
- **DNS/IPAM**: register/deregister hosts in DNS or NetBox.
- **Cloud**: deregister an instance from a target group, snapshot its volume (control node).
- **Notifications and reports**: once per run from the control node.
- **Cluster ops**: run a command on the primary database for each replica being added.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Missing sudo password` on `[web1 -> localhost]` | Play `become: true` applied to a localhost delegation | `become: false` on the task |
| Delegated task used the wrong IP | `ansible_host` resolved to the delegate's address | `hostvars[inventory_hostname]['ansible_host']` |
| `UNREACHABLE` to the delegate | Delegate not in inventory (no connection vars) or wrong user/key | Add it to the inventory with its connection vars |
| LB config corrupted / reloaded many times | Many hosts delegating to the LB at once | `serial: 1` or `throttle: 1` on LB tasks |
| Host stays drained after a failure | Enable step skipped because the host failed | Put enable in `always:` (only if healthy) or a final cleanup play; alert |
| Facts of the delegate stored on the wrong host | Missing `delegate_facts: true` | Add it |
| `local_action` syntax confusing / linter warnings | Old shorthand | `delegate_to: localhost` |
| `run_once` + `delegate_to` ran several times | `serial` batches | Separate play, or accept once-per-batch |

---

## Quick recap

- **`delegate_to: X`** runs the task **on X** with the **original host's variables**; output shows `[host -> X]`; results belong to the original host.
- Use `hostvars[inventory_hostname][...]` for the original host's connection values inside delegated tasks.
- **`delegate_to: localhost`** for APIs/checks on the control node (add `become: false`); ⚠️ `local_action` is old style.
- **`delegate_facts: true`** stores facts for the delegate. **`run_once` + `delegate_to`** = once, there (per batch with `serial`).
- LB drain/enable around each host, with `serial` and `throttle` protecting the LB.

### Interview questions

1. **Q: What does `delegate_to` do?**
   A: Runs a task on a different host than the current one while keeping the current host's variables and context; the result is recorded for the current host.
2. **Q: How do you take a server out of a load balancer during a rolling update?**
   A: In a `serial` play, a task delegated to the load balancer (or an API call from localhost) disables the current host before the change and re-enables it after a health check.
3. **Q: What's `delegate_facts` for?**
   A: To store facts gathered by a delegated `setup` task on the delegated host instead of the current one, e.g. to collect facts about database servers from a web play.

### Scenario question (answer out loud first, then compare)

> *"During a rolling deploy with LB drain, the play failed on web7 after it was drained. The next morning, capacity was down by one server. How do you make the playbook handle this properly?"*

**Model answer:**
1. Wrap the per-host work in `block`/`rescue`/`always` (topic 31): `block` = drain, deploy, health check; `rescue` = roll back to the previous release and re-check health; `always` = re-enable in the LB **only if** the health check (old or new version) passes; otherwise leave it drained and alert.
2. Make the failure visible: `rescue` ends with `fail` so the pipeline is red, and a notification (delegated to localhost) names the drained host.
3. Use `max_fail_percentage: 0` (or a small value) so the rollout stops instead of draining more hosts.
4. Add a final play (or a scheduled check) that lists any backends marked down/disabled and reports them.

**Likely follow-ups:**
- *Why not always re-enable in `always`?* An unhealthy server would get traffic again and serve errors.
- *How do you avoid two hosts editing the LB config at the same time?* `serial: 1` for the play or `throttle: 1` on LB tasks; an LB API with atomic per-member operations is even better.
