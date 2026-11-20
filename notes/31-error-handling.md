# 31 · Error Handling

> **Level:** ★★ intermediate
>
> **You will learn:** what Ansible does by default when a task fails, and every tool to change that: **`ignore_errors`** (and what it does *not* ignore), **`ignore_unreachable`**, **`failed_when`** (recap), **`block` / `rescue` / `always`** (Ansible's try / catch / finally, with `ansible_failed_task` and `ansible_failed_result`), **`assert`** and **`fail`** for preconditions, **`any_errors_fatal`**, **`max_fail_percentage`**, and the `meta` actions `end_host`, `end_play`, `clear_host_errors`. You'll build a deploy that **rolls itself back** when something goes wrong.
>
> **You need:** [30 · Handlers](30-handlers.md), [27 · Idempotency](27-idempotency-in-depth.md) (`failed_when`), [26 · register and when](26-register-and-when.md).
>
> **Part 3 · Variables, facts and logic** · Next: [32 · Advanced Jinja2 and lookups](32-advanced-jinja2-and-lookups.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** By default, when a task fails on a host, **that host stops** and the others continue. You can change this per task (`ignore_errors`, `failed_when`), per group of tasks (`block` / `rescue` / `always`), or for the whole play (`any_errors_fatal`, `max_fail_percentage`). And you can **fail early on purpose** with `assert`.

**Picture it.** A trapeze act: `block` is the act, `rescue` is the safety net (catch the fall, recover), `always` is the bow at the end that happens whatever.

**The one thing to know.**

```yaml
- name: Preconditions
  ansible.builtin.assert:
    that:
      - ansible_facts['os_family'] in ['RedHat', 'Debian']
      - app_version is defined
    fail_msg: "Unsupported OS or app_version missing"

- name: Deploy with automatic rollback
  block:
    - name: Switch to the new release
      ansible.builtin.file: { src: "/opt/app/releases/{{ app_version }}", dest: /opt/app/current, state: link }
    - name: Health check
      ansible.builtin.uri: { url: "http://{{ ansible_host }}:8080/health" }
      delegate_to: localhost
  rescue:
    - name: Roll back to the previous release
      ansible.builtin.file: { src: "/opt/app/releases/{{ previous_version }}", dest: /opt/app/current, state: link }
  always:
    - name: Clean temp files
      ansible.builtin.file: { path: /tmp/deploy-tmp, state: absent }
```

**Try it now:** run the lab playbook with `-e break_it=true` and watch `rescue` run and the recap show `rescued=1`.

**You've got it if you can:** explain what `rescue` does to the host's failure status, and why `ignore_errors` is usually the wrong tool.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Default behaviour

| Event | Default result |
|---|---|
| Task fails on host H | H gets `fatal: ... FAILED!`, is **removed from the rest of the play** (and later plays), others continue |
| Host unreachable | Same, reported as `UNREACHABLE!` |
| All hosts failed | `NO MORE HOSTS LEFT`, play ends |
| End of run | Exit code `2` (some failed) or, usually, `4` (some unreachable; older docs say `3`, see topic 12) |
| Pending handlers on H | **Not** run (topic 30) |

### The tools

| Tool | Scope | What it does |
|---|---|---|
| `failed_when: <cond>` | task | Redefine what "failed" means (topic 27) |
| `ignore_errors: true` | task/block/play | A **failed** result doesn't stop the host; shown as `...ignoring`, counted in `ignored` |
| `ignore_unreachable: true` | task/block/play | An **unreachable** result doesn't stop the host |
| `block` / `rescue` / `always` | group of tasks | try / catch / finally |
| `ansible.builtin.assert` | task | Fail if conditions aren't all true (clear messages) |
| `ansible.builtin.fail` | task | Fail with your message (usually with `when`) |
| `any_errors_fatal: true` | play/block | **One** host fails → stop the play for **all** hosts |
| `max_fail_percentage: N` | play | Abort if more than N% of hosts (in a batch) fail (topic 45) |
| `meta: end_host` | task | Stop this host gracefully (not a failure) |
| `meta: end_play` / `end_batch` | task | Stop the play / batch for everyone, gracefully |
| `meta: clear_host_errors` | task | Put failed/unreachable hosts back into the play |
| `force_handlers: true` | play | Run pending handlers despite failures (topic 30) |

### ⚠️ What `ignore_errors` does **not** ignore

The official rule: `ignore_errors` only applies when the task **ran** and returned `failed`. It is **not** meant to cover:

- undefined variables / template errors (depending on where they occur, e.g. in `loop` or `when`, they can still stop the host),
- syntax errors (the playbook never starts),
- unreachable hosts (use `ignore_unreachable`),
- problems before the module runs (e.g. a module that can't be found).

Never rely on `ignore_errors` to paper over a missing variable: fix it, or use `default`/`mandatory` (topic 22).

And it's dangerous: it hides **every** failure of that task, including the ones you didn't expect. Prefer `failed_when` with a precise condition (topic 27), or a `block`/`rescue` that actually handles the failure.

### block / rescue / always

```yaml
- name: A unit of work
  block:          # try
    - task 1
    - task 2
  rescue:         # catch: runs if ANY task in block failed (for that host)
    - recovery tasks
  always:         # finally: runs whether block succeeded, failed, or was rescued
    - cleanup tasks
```

| Situation | block | rescue | always | Host status after |
|---|---|---|---|---|
| All block tasks succeed | ✅ all run | skipped | runs | OK |
| Task 1 fails | stops at task 1 | runs | runs | **OK** (counted as `rescued=1`), continues with the play |
| Task 1 fails, a rescue task also fails | stops | stops at the failing rescue task | runs | **FAILED** |
| No rescue, task 1 fails | stops | — | runs | **FAILED** (always ran, then the host stops) |

Inside `rescue`, two special variables tell you what went wrong:

| Variable | Contains |
|---|---|
| `ansible_failed_task` | The task that failed (`ansible_failed_task.name`) |
| `ansible_failed_result` | Its result (`msg`, `rc`, `stderr`...) |

A rescued host is considered **successful**. If you want the run to still fail (after rolling back), end the rescue with a `fail` task: the rollback happened, **and** the pipeline goes red.

`block` is also simply a way to **group** tasks so they share `when`, `become`, `vars`, `tags`, `delegate_to`... (keywords set on the block apply to every task inside).

### assert

```yaml
- name: Preconditions for this playbook
  ansible.builtin.assert:
    that:
      - ansible_facts['os_family'] in ['RedHat', 'Debian']
      - ansible_facts['mounts'] | selectattr('mount', 'equalto', '/') | map(attribute='size_available') | first > 1073741824
      - app_env in ['dev', 'staging', 'production']
    fail_msg: "Preconditions not met on {{ inventory_hostname }}"
    success_msg: "Preconditions OK"
    quiet: true
```

- `that:` a list of conditions (each must be a boolean, topic 26). All must be true.
- The `mounts` expression finds `/` and checks more than 1 GiB free (the filters are explained in topic 32).
- `quiet: true`: less output on success.

Use asserts at the **start** of playbooks and roles: fail fast with a clear message, **before** anything changes.

### fail

```yaml
- name: Refuse to run on production without approval
  ansible.builtin.fail:
    msg: "Set approved=true to run against production"
  when:
    - app_env == 'production'
    - not (approved | default(false) | bool)
```

### any_errors_fatal

For clusters where a half-applied change is worse than no change (database schema, cluster upgrades):

```yaml
- hosts: db_cluster
  any_errors_fatal: true
```

If **any** host fails a task, Ansible finishes that task on the other hosts, then **stops the whole play** for everyone.

---

## Why it exists

Production automation must answer: "what if step 4 fails on 3 of 50 servers?". The default (failed hosts stop, others continue) is right for independent servers, but:

- some failures are **expected** and harmless (a probe returns "not found"),
- some changes must be **rolled back** if a later check fails (deploys),
- some must **stop everything** if anything fails (clusters),
- some problems should be caught **before** changing anything (wrong OS, no disk space, missing variables),
- cleanup must happen **no matter what** (re-enable a server in the load balancer, remove temp files, release a lock).

Without these tools you get half-deployed fleets and servers left out of the load balancer.

---

## Analogy

**Cooking a big family dinner.**

- `assert` = checking you have every ingredient **before** you start cooking.
- `block` = cooking the roast.
- `rescue` = the roast burns → order pizza (recovery), and the dinner still happens (`rescued`).
- `always` = whatever happened, turn off the oven and open the windows.
- `ignore_errors` = pretending the smoke alarm didn't go off. Sometimes it really is just toast; usually it isn't.
- `any_errors_fatal` = a wedding cake: if one tier collapses, you stop and rebuild the whole cake.

---

## How it works inside

```
 host web1 executing a block
   │
   ├─ block tasks, in order
   │     task fails → mark web1 "failed in block", skip remaining block tasks
   │
   ├─ rescue (only if failed in block)
   │     ansible_failed_task / ansible_failed_result available
   │     all rescue tasks OK → web1's failure is CLEARED → stats: rescued +1
   │     a rescue task fails → web1 stays failed
   │
   ├─ always (in every case)
   │
   └─ after the block: failed? → web1 leaves the play; else continue

 any_errors_fatal: after each task, if ANY host failed → stop scheduling new tasks for ALL hosts
 max_fail_percentage: after each task, failed/total in the batch > N% → stop
```

---

## Code / commands

### A deploy that rolls back

Prepare two "releases" on the web servers (reusing topic 27's archive name style):

`~/ansible-lab/errors.yml`:

```yaml
---
- name: Deploy with preconditions, rollback and cleanup
  hosts: web
  become: true
  vars:
    releases_dir: /opt/rb/releases
    current_link: /opt/rb/current
    new_version: "2.0"
    previous_version: "1.0"

  tasks:
    - name: Preconditions
      ansible.builtin.assert:
        that:
          - ansible_facts['os_family'] in ['RedHat', 'Debian']
          - new_version != previous_version
        fail_msg: "Unsupported OS or nothing to deploy"
        quiet: true

    - name: Release directories exist
      ansible.builtin.file:
        path: "{{ releases_dir }}/{{ item }}"
        state: directory
        mode: "0755"
      loop: ["{{ previous_version }}", "{{ new_version }}"]

    - name: Each release has a VERSION file
      ansible.builtin.copy:
        dest: "{{ releases_dir }}/{{ item }}/VERSION"
        content: "{{ item }}\n"
        mode: "0644"
      loop: ["{{ previous_version }}", "{{ new_version }}"]

    - name: Start from the previous release
      ansible.builtin.file:
        src: "{{ releases_dir }}/{{ previous_version }}"
        dest: "{{ current_link }}"
        state: link

    - name: Deploy the new release
      block:
        - name: Switch current to the new release
          ansible.builtin.file:
            src: "{{ releases_dir }}/{{ new_version }}"
            dest: "{{ current_link }}"
            state: link

        - name: Smoke test the new release
          ansible.builtin.command: cat {{ current_link }}/VERSION
          register: smoke
          changed_when: false
          failed_when: smoke.stdout != new_version or (break_it | default(false) | bool)

      rescue:
        - name: Explain what failed
          ansible.builtin.debug:
            msg: "Task '{{ ansible_failed_task.name }}' failed: {{ ansible_failed_result.msg | default('see result') }}"

        - name: Roll back to the previous release
          ansible.builtin.file:
            src: "{{ releases_dir }}/{{ previous_version }}"
            dest: "{{ current_link }}"
            state: link

        - name: Still fail the run (after rolling back) if asked
          ansible.builtin.fail:
            msg: "Deploy of {{ new_version }} failed and was rolled back"
          when: fail_after_rollback | default(false) | bool

      always:
        - name: Report the active release
          ansible.builtin.command: readlink -f {{ current_link }}
          register: active
          changed_when: false

        - name: Show it
          ansible.builtin.debug:
            msg: "{{ inventory_hostname }} now runs {{ active.stdout }}"
```

- `failed_when: ... or (break_it | default(false) | bool)`: a switch to simulate a failing smoke test.
- The rescue explains, rolls back, and optionally re-fails.
- `always` reports the final state whatever happened.

### Happy path

```bash
ansible-playbook errors.yml
```
```
TASK [Switch current to the new release] ***********************************************
changed: [web1]
changed: [web2]
TASK [Smoke test the new release] ******************************************************
ok: [web1]
ok: [web2]
TASK [Report the active release] *******************************************************
ok: [web1]
...
ok: [web1] => { "msg": "web1 now runs /opt/rb/releases/2.0" }
...
PLAY RECAP *****************************************************************************
web1   : ok=10  changed=6  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

### Failure, rescued

```bash
ansible-playbook errors.yml -e break_it=true
```
```
TASK [Smoke test the new release] ******************************************************
fatal: [web1]: FAILED! => {"changed": false, "cmd": ["cat", "/opt/rb/current/VERSION"], "failed_when_result": true, "rc": 0, "stdout": "2.0", ...}
fatal: [web2]: FAILED! => {...}

TASK [Explain what failed] *************************************************************
ok: [web1] => { "msg": "Task 'Smoke test the new release' failed: see result" }
...
TASK [Roll back to the previous release] ***********************************************
changed: [web1]
changed: [web2]

TASK [Still fail the run (after rolling back) if asked] ********************************
skipping: [web1]
skipping: [web2]

TASK [Show it] *************************************************************************
ok: [web1] => { "msg": "web1 now runs /opt/rb/releases/1.0" }
...
PLAY RECAP *****************************************************************************
web1   : ok=11  changed=3  unreachable=0  failed=0  skipped=1  rescued=1  ignored=0
web2   : ok=11  changed=3  unreachable=0  failed=0  skipped=1  rescued=1  ignored=0
```

- `fatal: ... FAILED!` is still printed, but `failed=0` and **`rescued=1`** in the recap: the rescue handled it. Exit code `0`.
- `failed_when_result: true`: the failure came from your `failed_when`, not from the command's rc.

### Failure, rolled back **and** reported

```bash
ansible-playbook errors.yml -e break_it=true -e fail_after_rollback=true; echo "exit=$?"
```
```
TASK [Still fail the run (after rolling back) if asked] ********************************
fatal: [web1]: FAILED! => {"changed": false, "msg": "Deploy of 2.0 failed and was rolled back"}
...
TASK [Show it] *************************************************************************
ok: [web1] => { "msg": "web1 now runs /opt/rb/releases/1.0" }
...
PLAY RECAP *****************************************************************************
web1   : ok=10  changed=3  unreachable=0  failed=1  skipped=0  rescued=0  ignored=0
exit=2
```

Rolled back, `always` still ran, and the pipeline fails (exit 2). This is the pattern for CI deploys.

### any_errors_fatal

```yaml
- name: All or nothing
  hosts: web
  any_errors_fatal: true
  gather_facts: false
  tasks:
    - name: Fails only on web2
      ansible.builtin.command: /bin/false
      when: inventory_hostname == 'web2'
    - name: Never reached by anyone
      ansible.builtin.debug: { msg: "you won't see this" }
```
```
TASK [Fails only on web2] **************************************************************
skipping: [web1]
fatal: [web2]: FAILED! => {...}

NO MORE HOSTS LEFT *********************************************************************
```

web1 didn't fail, but the play stopped for it too.

### ignore_unreachable

```yaml
- name: Best-effort cleanup on hosts that may be down
  ansible.builtin.file: { path: /tmp/x, state: absent }
  ignore_unreachable: true
```

---

## Hands-on lab

1. Run `errors.yml` three ways (happy, `break_it`, `break_it` + `fail_after_rollback`). Compare the recaps and exit codes.
2. Check the rollback on web1: `ssh ansible@172.30.0.11 readlink -f /opt/rb/current`.
3. Remove the `rescue:` section and run with `break_it=true`: the `always` tasks still run, then the hosts fail.
4. Add an `assert` that fails if `new_version` doesn't look like a version (`new_version is match('^[0-9]+\\.[0-9]+$')`) and run with `-e new_version=latest`.
5. Explore the edges of `ignore_errors`: add a task `debug: msg="{{ no_such_var }}"` with `ignore_errors: true`, then one with `loop: "{{ no_such_list }}"` and `ignore_errors: true`. Note which one stops the host and which is shown as `...ignoring`. The lesson either way: undefined variables are bugs to fix, not errors to ignore.
6. Run the `any_errors_fatal` demo, then remove the keyword and run again.
7. Stop web2's container (`docker stop ansible-lab-web2-1`), run a task with `ignore_unreachable: true` and one without; restart it.

**Done when:** you can build a deploy that rolls back on failure and still reports the failure to CI.

---

## Where you'll use it

- **Deploys**: `block` (switch release, restart, health check) → `rescue` (roll back, alert) → `always` (re-enable in the LB, release locks).
- **Preconditions** at the start of every role/playbook: OS, versions, disk space, required variables, "not production unless approved".
- **Clusters**: `any_errors_fatal` for schema changes and cluster-wide upgrades.
- **Rolling updates**: `max_fail_percentage` + `serial` to stop a bad rollout early (topic 45).
- **Best-effort tasks**: `ignore_unreachable` for cleanup on hosts that may be gone.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `ignore_errors` didn't ignore an error | It was a syntax error, an unreachable host, or a templating error outside the task's result (e.g. in `loop`) | Fix the variable/syntax; `ignore_unreachable` for connection failures |
| Real failures hidden for months (`ignored=N` in every recap) | Broad `ignore_errors` | Precise `failed_when`, or `rescue` that handles it |
| Pipeline green although the deploy failed | `rescue` cleared the failure | End `rescue` with `fail` when the run should be red |
| `rescue` didn't run | The failing task was outside the `block`, or the host was **unreachable** (rescue handles failures; unreachable hosts are handled only with `ignore_unreachable`) | Put the task inside the block; handle unreachability separately |
| `ansible_failed_result.msg` undefined in rescue | That failure type had no `msg` | `\| default(...)` |
| `assert` error: `Conditional result ... must have a boolean result` | Non-boolean expression in `that:` | Same fixes as `when` (topic 26) |
| Half the cluster upgraded, half not | No `any_errors_fatal` / no batching | `any_errors_fatal: true`, or `serial` + `max_fail_percentage: 0` |
| A server stayed out of the load balancer after a failure | Re-enable step was in `block`, not `always` | Move it to `always` |

---

## Quick recap

- Default: a failed host stops; others continue; exit 2/3.
- `failed_when` = redefine failure; `ignore_errors` = keep going after a **failed** result (not undefined vars, not unreachable); `ignore_unreachable` = keep going after unreachable.
- **`block` / `rescue` / `always`** = try / catch / finally; rescued hosts count as **success** (`rescued=1`) unless rescue ends with **`fail`**; use `ansible_failed_task` / `ansible_failed_result`.
- **`assert`** preconditions early; **`fail`** with `when` for custom stops.
- **`any_errors_fatal`** = all or nothing; **`max_fail_percentage`** with `serial`; `meta: end_host` / `end_play` / `clear_host_errors`.

### Interview questions

1. **Q: How do you implement try/catch in Ansible?**
   A: With `block`, `rescue` and `always`: `rescue` runs if any task in the `block` fails, `always` runs regardless. Inside rescue, `ansible_failed_task` and `ansible_failed_result` describe the failure.
2. **Q: Why is `ignore_errors` risky?**
   A: It hides every failure of the task, including unexpected ones, and it's not meant for undefined variables or unreachable hosts anyway. Prefer a precise `failed_when` or a handling `rescue`.
3. **Q: What does `any_errors_fatal` do?**
   A: If any host fails a task, Ansible stops the play for all hosts after that task, instead of letting the others continue.

### Scenario question (answer out loud first, then compare)

> *"Design the error handling for a production deploy to 30 web servers behind a load balancer. A failed health check on any server must roll that server back, and if more than 2 servers fail, the rollout must stop. The pipeline must go red on any failure."*

**Model answer:**
1. Play with `serial: 5` (batches) and `max_fail_percentage` set so >2 failures in total stops it (or `serial` small + `max_fail_percentage: 0` per batch, depending on policy).
2. Start with `assert` preconditions (variables, disk space, artifact exists).
3. Per host, a `block`:
   - disable the host in the LB (`delegate_to` the LB, topic 46),
   - deploy new release, switch symlink, restart via `flush_handlers`,
   - health check with `uri` + `until` retries.
4. `rescue`: switch back to the previous release, restart, re-check health, then `fail` with a clear message (so the pipeline is red and the host counts as failed for `max_fail_percentage`).
5. `always`: re-enable the host in the LB **only if** it's healthy (old or new version), and remove temp files.
6. `force_handlers: true` so restarts aren't lost.
7. The CI job fails on exit code 2; the summary shows which hosts were rolled back.

**Likely follow-ups:**
- *Why fail after a successful rollback?* Someone must look at it; a green pipeline would hide that version 2.0 is broken.
- *Where does the LB re-enable go if the host is unreachable?* It can't run on that host, but it's delegated to the LB, so put it in `always` with `ignore_unreachable` considerations, or handle it in a final play that checks all backends.
