# 30 · Handlers

> **Level:** ★★ intermediate
>
> **You will learn:** what **handlers** are (tasks that run only when **notified** by a task that **changed** something), why restarts belong in handlers, **when** they run (at the end of each section of the play, once, in the order they're defined), **`listen`** topics, **`meta: flush_handlers`** to run them earlier, what happens to pending handlers when a play **fails** (`--force-handlers`, `force_handlers: true`), handlers with conditions and loops, chaining, and handlers inside roles.
>
> **You need:** [29 · Templates](29-templates.md) (the lb reload you did "by hand"), [27 · Idempotency](27-idempotency-in-depth.md) (why `restarted` in a task is bad), [15 · Inside ansible-playbook](15-inside-ansible-playbook.md).
>
> **Part 3 · Variables, facts and logic** · Next: [31 · Error handling](31-error-handling.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **handler** is a task that sits in a separate `handlers:` list and runs **only if** another task **notifies** it **and that task reported `changed`**. Typical use: restart a service only when its config file actually changed.

**Picture it.** A doorbell. Delivering a parcel (a change) rings it; walking past the door (no change) doesn't. However many parcels arrive during the day, the person inside answers the door **once**, at the end.

**The one thing to know.**

```yaml
- hosts: lb
  become: true
  tasks:
    - name: nginx.conf is rendered and validated
      ansible.builtin.template:
        src: lb-nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        validate: nginx -t -c %s
      notify: Reload nginx                # ring the bell IF this task changed

  handlers:
    - name: Reload nginx                  # name must match the notify text exactly
      ansible.builtin.systemd_service:
        name: nginx
        state: reloaded
```

**Try it now:** rewrite topic 29's `lb.yml` like this. Run twice: the first run shows `RUNNING HANDLER [Reload nginx]`, the second doesn't.

**You've got it if you can:** say when a notified handler runs, and how many times it runs if three tasks notify it.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### The rules

| Rule | Detail |
|---|---|
| **Defined** in the play's `handlers:` section (or a role's `handlers/main.yml`) | Same syntax as tasks |
| **Triggered** by `notify: <handler name>` on a task | `notify` can be a string or a **list** of names |
| **Only on change** | The notifying task must report `changed` (for that host). `ok` → no notification |
| **Per host** | Only hosts whose task changed get the handler |
| **Once** | Notified 5 times → runs **once** per host |
| **When** | At the end of each **section** of the play (after `pre_tasks`, after `roles`+`tasks`, after `post_tasks`), or at a `meta: flush_handlers` |
| **Order** | The order they're **written** in `handlers:`, not the order they were notified |
| **Failure** | If the host fails before the handler phase, its pending handlers **don't run** (unless forced) |

### Naming and `listen`

`notify` matches a handler by its `name` **or** by a `listen` topic:

```yaml
handlers:
  - name: Restart app
    ansible.builtin.systemd_service: { name: app, state: restarted }
    listen: app config changed

  - name: Clear app cache
    ansible.builtin.file: { path: /var/cache/app, state: absent }
    listen: app config changed
```

```yaml
tasks:
  - name: App config
    ansible.builtin.template: { src: app.yml.j2, dest: /etc/app/app.yml }
    notify: app config changed          # triggers BOTH handlers
```

- `listen` lets several handlers react to one "event", and lets one handler be triggered by different names.
- Handler names must be **unique** within the play. Use clear, action-style names ("Reload nginx").

### flush_handlers

Sometimes later tasks need the handler to have **already** run:

```yaml
tasks:
  - name: nginx config
    ansible.builtin.template: { src: nginx.conf.j2, dest: /etc/nginx/nginx.conf }
    notify: Reload nginx

  - name: Run pending handlers now
    ansible.builtin.meta: flush_handlers

  - name: Health check (needs the new config active)
    ansible.builtin.uri:
      url: "http://{{ ansible_host }}/health"
    delegate_to: localhost
```

`meta` is a special action that controls Ansible itself (others: `end_play`, `end_host`, `clear_facts`, `refresh_inventory`, `noop`).

### Failures and forced handlers

```
TASK 1: config changed → notify Restart app          (pending for web1)
TASK 2: FAILED on web1                               → web1 leaves the play
         → handler phase: web1 is gone, "Restart app" does NOT run for web1
```

The config is new but the service still runs the **old** config. Next run, the config task says `ok` (already correct) → **no notification** → the restart is lost forever. That's a real production trap.

Fixes:

| Option | Effect |
|---|---|
| `--force-handlers` (CLI) or `force_handlers: true` (play keyword, or `force_handlers = True` in ansible.cfg) | Run notified handlers even for hosts that failed later |
| `meta: flush_handlers` right after the important change | Handlers run before the risky later tasks |
| Make the restart self-healing | A later run compares the running config/version and restarts if they differ |

### Handlers with conditions, loops, and chaining

- Handlers accept `when`, `loop`, `become`, `delegate_to`... like any task.
- A **handler can notify another handler** (chaining). Chained handlers run in the same handler phase, still in definition order.
- `notify` inside a **loop**: the handler is notified if **any** item changed; it still runs once.
- Notifying a **non-existent** handler name is an **error** at run time: `The requested handler 'Restart ngnix' was not found in either the main handlers list nor in the listening handlers list`.

### Handlers in roles (preview, topic 35)

A role's `handlers/main.yml` handlers can be notified by the role's tasks, and by the play using `role_name : handler name` (or just the name if unique). Role handlers are imported into the play, so names should be prefixed (`nginx | Reload` style or `Reload nginx`) to avoid clashes between roles.

---

## Why it exists

Restarting a service has a cost: dropped connections, cold caches, a short outage. You want to restart **when its configuration changed**, not on every run. Without handlers you'd write `register` + `when: x is changed` after every config task (as in topic 29), and if three tasks change three files, you'd restart three times. Handlers give you **restart on change, once, at the right time**, and keep playbooks idempotent (topic 27: no `state: restarted` in normal tasks).

---

## Analogy

**A restaurant kitchen closing its order book at the end of each service.** During service, waiters drop notes in the "chef, re-season the soup" box whenever a customer complains (notify on change). At the end of the service (end of the play section), the chef checks the box: three notes, but the soup is re-seasoned **once**. No notes, no re-seasoning. If the kitchen catches fire mid-service (host fails), nobody empties the box: the soup is never fixed unless the manager insists (`--force-handlers`). If a dish must be tasted right after re-seasoning, the chef empties the box immediately (`flush_handlers`).

---

## How it works inside

```
 PLAY
 ├─ pre_tasks ......................... (tasks can notify)
 ├─ ► handler flush #1  (run notified handlers, in definition order, per host)
 ├─ roles + tasks ...................... (tasks can notify)
 │      template changed on web1 → mark "Reload nginx" pending for web1
 │      copy changed on web1     → "Reload nginx" already pending (still once)
 │      template ok on web2      → nothing for web2
 │      meta: flush_handlers     → ► run pending handlers now
 ├─ ► handler flush #2
 ├─ post_tasks .........................
 └─ ► handler flush #3
```

- A notification is just a **flag per host per handler**.
- Handlers run with the normal strategy (linear, forks): all hosts with the flag run it together.
- Failed/unreachable hosts are skipped in handler phases unless `force_handlers` is on.

---

## Code / commands

### The load balancer, with a handler

`~/ansible-lab/lb.yml` (replace topic 29's version):

```yaml
---
- name: Load balancer in front of the web servers
  hosts: lb
  become: true

  tasks:
    - name: nginx is installed
      ansible.builtin.package:
        name: nginx
        state: present

    - name: nginx.conf is rendered and validated
      ansible.builtin.template:
        src: lb-nginx.conf.j2
        dest: /etc/nginx/nginx.conf
        owner: root
        group: root
        mode: "0644"
        validate: nginx -t -c %s
        backup: true
      notify: Reload nginx

    - name: nginx is running and enabled
      ansible.builtin.systemd_service:
        name: nginx
        state: started
        enabled: true

  handlers:
    - name: Reload nginx
      ansible.builtin.systemd_service:
        name: nginx
        state: reloaded
```

Change `lb_algorithm` in `group_vars/lb.yml`, then:

```bash
ansible-playbook lb.yml
```
```
TASK [nginx.conf is rendered and validated] ********************************************
changed: [lb1]

TASK [nginx is running and enabled] ****************************************************
ok: [lb1]

RUNNING HANDLER [Reload nginx] *********************************************************
changed: [lb1]

PLAY RECAP *****************************************************************************
lb1    : ok=5  changed=2  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

- `RUNNING HANDLER [...]`: the handler phase after the tasks.
- Run again without changes: no `RUNNING HANDLER` line, `changed=0`.

### Several notifications, one run, definition order

`~/ansible-lab/handlers-demo.yml`:

```yaml
---
- name: Handler behaviour demo
  hosts: web1
  become: true
  tasks:
    - name: File A
      ansible.builtin.copy: { content: "a {{ stamp | default(1) }}\n", dest: /tmp/hd-a, mode: "0644" }
      notify: [Second handler, First handler]

    - name: File B
      ansible.builtin.copy: { content: "b {{ stamp | default(1) }}\n", dest: /tmp/hd-b, mode: "0644" }
      notify: First handler

    - name: A task after the changes
      ansible.builtin.debug: { msg: "handlers have NOT run yet" }

    - name: Flush now
      ansible.builtin.meta: flush_handlers

    - name: A task after the flush
      ansible.builtin.debug: { msg: "handlers HAVE run" }

  handlers:
    - name: First handler
      ansible.builtin.debug: { msg: "first (defined first)" }
    - name: Second handler
      ansible.builtin.debug: { msg: "second (defined second)" }
```

```bash
ansible-playbook handlers-demo.yml -e stamp=$RANDOM
```
```
TASK [File A] **************************************************************************
changed: [web1]
TASK [File B] **************************************************************************
changed: [web1]
TASK [A task after the changes] ********************************************************
ok: [web1] => { "msg": "handlers have NOT run yet" }
TASK [Flush now] ***********************************************************************
RUNNING HANDLER [First handler] ********************************************************
ok: [web1] => { "msg": "first (defined first)" }
RUNNING HANDLER [Second handler] *******************************************************
ok: [web1] => { "msg": "second (defined second)" }
TASK [A task after the flush] **********************************************************
ok: [web1] => { "msg": "handlers HAVE run" }
```

- `First handler` was notified twice but ran **once**.
- It ran **before** `Second handler` even though File A notified Second first: **definition order**.
- `-e stamp=$RANDOM` forces new file content (so `changed`) on every run of the demo.

### The lost-restart trap and `--force-handlers`

Add a failing task after File B:

```yaml
    - name: Something fails after the change
      ansible.builtin.command: /bin/false
```

```bash
ansible-playbook handlers-demo.yml -e stamp=$RANDOM
```
```
TASK [Something fails after the change] ************************************************
fatal: [web1]: FAILED! => {"changed": true, "cmd": ["/bin/false"], "rc": 1, ...}

NO MORE HOSTS LEFT *********************************************************************
PLAY RECAP *****************************************************************************
web1   : ok=3  changed=2  unreachable=0  failed=1  ...
```

No `RUNNING HANDLER`: the files changed but the handlers were lost.

```bash
ansible-playbook handlers-demo.yml -e stamp=$RANDOM --force-handlers
```
```
fatal: [web1]: FAILED! => {...}
RUNNING HANDLER [First handler] ********************************************************
ok: [web1] => { "msg": "first (defined first)" }
RUNNING HANDLER [Second handler] *******************************************************
ok: [web1] => { "msg": "second (defined second)" }
```

### A handler that checks before restarting

For services that can be broken by a bad config, validate in the handler chain:

```yaml
handlers:
  - name: Check sshd config
    ansible.builtin.command: /usr/sbin/sshd -t
    changed_when: false
    listen: sshd config changed

  - name: Restart sshd
    ansible.builtin.systemd_service:
      name: "{{ svc_ssh }}"
      state: restarted
    listen: sshd config changed
```

Both listen to the same topic; they run in definition order, so the check runs first. If it fails, the host fails and the restart doesn't happen.

---

## Hands-on lab

1. Rewrite `lb.yml` with the handler. Change the algorithm, run, see `RUNNING HANDLER`; run again, see nothing.
2. Run `handlers-demo.yml` and confirm: once per handler, definition order, flush point.
3. Reproduce the lost-restart trap. Then run again **without** `-e stamp` changes: File A/B are `ok`, so the handlers never run. That's how restarts get lost in real life. Fix with `--force-handlers`, then add `force_handlers: true` to the play.
4. Convert `files.yml` (topic 18): make the `sshd_config` `lineinfile` task `notify: sshd config changed`, and add the two handlers above. Change `MaxAuthTries`, run, and watch the check → restart chain.
5. Use `listen` to make one "app config changed" event trigger two handlers.
6. Misspell a handler name in `notify` and read the error.

**Done when:** no playbook of yours contains `state: restarted` outside a handler.

---

## Where you'll use it

- **Every service config**: nginx/Apache reload, app restarts, sshd restart, `systemctl daemon-reload` after unit file changes (`daemon_reload: true` in a handler).
- **Roles**: each role ships its own handlers (`handlers/main.yml`).
- **Deploys**: "new release unpacked" → handler restarts the app; `flush_handlers` before the health check.
- **Rolling updates**: with `serial`, handlers run at the end of **each batch**, so each batch restarts before the next starts (topic 45).

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| Handler never runs | Notifying task reported `ok` (nothing changed) | Expected. Force a change, or check why it didn't change |
| `The requested handler 'X' was not found` | Typo or wrong name in `notify` | Match `name` exactly, or use `listen` |
| Restart "lost" after a failed run | Host failed before the handler phase; next run sees no change | `force_handlers: true`, `flush_handlers` after critical changes |
| Health check fails right after the config task | Handler hasn't run yet (runs at end of section) | `meta: flush_handlers` before the check |
| Handlers run in an unexpected order | They run in **definition** order | Reorder the `handlers:` list |
| Handler ran on all hosts | It was notified on all hosts (all changed) | Expected; changes are per host |
| Two roles' handlers clash | Same handler name in both | Prefix names per role |
| Service restarted twice | Two different handlers both restart it (or one task restarts directly) | One handler per service action; `listen` for multiple triggers |
| Handler used `when: x is changed` with a registered var from a task | Fragile; registered vars are per task | Just use notify |

---

## Quick recap

- **Handlers** = tasks in `handlers:` that run only when a task **notifies** them **and changed**; per host; **once**; in **definition order**.
- They run at the end of each play section (`pre_tasks`, `roles`+`tasks`, `post_tasks`) or at **`meta: flush_handlers`**.
- **`listen`** = topics: one notify, several handlers.
- If a host fails first, pending handlers are **lost** unless `--force-handlers` / `force_handlers: true`.
- Restarts belong in handlers, never in normal tasks (`state: restarted` breaks idempotency).

### Interview questions

1. **Q: What is a handler and when does it run?**
   A: A task that runs only when notified by a task that reported a change. It runs once per host at the end of the current play section (or when handlers are flushed), in the order the handlers are defined.
2. **Q: Three tasks notify the same handler. How many times does it run?**
   A: Once per host.
3. **Q: What happens to notified handlers if the play fails afterwards, and how do you change that?**
   A: They don't run for the failed host, so the change may never be applied to the running service. Use `--force-handlers` or `force_handlers: true`, or flush handlers right after the critical change.

### Scenario question (answer out loud first, then compare)

> *"After a deploy, the new nginx config was on disk on 4 of 20 servers but those 4 were still serving the old config a week later. Nobody touched them. What probably happened, and how do you make it impossible?"*

**Model answer:**
1. During the deploy, the config template **changed** on those 4 and notified the reload handler, but a later task **failed** on them (or they became unreachable). Hosts that fail leave the play before the handler phase, so the reload never ran.
2. On every later run, the template reported `ok` (file already correct) → no notification → still no reload. The running process kept the old config.
3. Make it impossible:
   - `force_handlers: true` on deploy plays (or in `ansible.cfg`);
   - `meta: flush_handlers` immediately after config changes, before risky steps;
   - a verification task that compares what's **running** with what's **on disk** (e.g. nginx's config test output, the app's `/version` endpoint vs the deployed version) and triggers a reload/restart when they differ;
   - monitoring that alerts on version mismatch.

**Likely follow-ups:**
- *Why not just restart in every run?* Breaks idempotency and causes needless outages.
- *Do handlers run per batch with `serial`?* Yes: at the end of each batch's tasks, which is what you want for rolling restarts.
