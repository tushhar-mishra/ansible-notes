# 49 · Project 1: Zero-Downtime Rolling Deploy

> **Level:** ★★★ advanced
>
> **You will build:** a production-style deployment of a versioned web app to `web1` and `web2` behind the `lb1` load balancer, **one server at a time**, with **no failed requests** while it runs: preflight checks, a release artifact built once, per-host **drain** from the load balancer, release directories with an atomic **`current` symlink switch**, a **health check** against the new version, automatic **rollback** on failure, **re-enable** only when healthy, cleanup of old releases, and a live traffic probe that proves zero downtime. It combines topics 25–48.
>
> **You need:** [48 · Performance](48-performance-tuning.md), [46 · Delegation (lb_member role)](46-delegation.md), [45 · serial](45-strategies-forks-serial.md), [31 · block/rescue/always](31-error-handling.md), [30 · Handlers](30-handlers.md), [29 · Templates (lb1)](29-templates.md), [36 · Roles II](36-roles-advanced.md), the `~/infra` project ([39](39-project-layout.md)).
>
> **Part 7 · Execution control and speed** · Next: [50 · Linting](50-linting.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A rolling deploy updates servers **in small batches** so the service never goes down: take one server out of the load balancer, update it, check it, put it back, then the next one. If anything fails, roll that server back and **stop**.

**Picture it.** Changing the tyres of a bus **while it's driving** by lifting one wheel at a time... the bus (service) has enough wheels left to keep rolling.

**The one thing to know.** The skeleton:

```yaml
- hosts: web
  serial: 1
  max_fail_percentage: 0
  pre_tasks:   [ preflight asserts, remember current version, DRAIN from LB ]
  tasks:
    - block:   [ ship release, unpack, switch symlink, reload, HEALTH CHECK new version ]
      rescue:  [ switch symlink back, reload, fail loudly ]
      always:  [ ENABLE in LB only if healthy ]
  post_tasks:  [ check through the LB ]
```

**Try it now:** run the traffic probe in one terminal and the deploy in another; the probe should show only `200`.

**You've got it if you can:** explain why each step is where it is, and what happens if the health check fails on web1.

> 📖 **That's the core.** On a first pass, follow the build steps once. Everything below is the **full project**.

---

## What it is

### Requirements

| # | Requirement | How |
|---|---|---|
| R1 | No failed user requests during the deploy | `serial: 1` + LB drain before changes, enable after health |
| R2 | A bad release never reaches more than one server | `max_fail_percentage: 0` + health check in each batch |
| R3 | A failed server returns to the previous version automatically | `block`/`rescue` rollback via symlink |
| R4 | The pipeline fails visibly when a rollback happened | `fail` at the end of `rescue` |
| R5 | Instant rollback possible at any time | Releases kept side by side; `current` symlink |
| R6 | The artifact is built once and identical everywhere | Build on the control node (`run_once`), checksum |
| R7 | Re-running with the same version changes nothing | Idempotent tasks (`creates`, `file: state=link`, templates) |
| R8 | Disk doesn't fill with old releases | Keep the newest N |

### Architecture

```
                         ┌───────────────────────┐
     users / probe ────▶ │ lb1 nginx :80          │
                         │ upstream backend {     │
                         │   server web1:8080;    │  ← "down" while web1 is updated
                         │   server web2:8080; }  │
                         └───────┬───────┬───────┘
                                 │       │
               ┌─────────────────▼─┐   ┌─▼─────────────────┐
               │ web1 nginx :8080   │   │ web2 nginx :8080   │
               │ root /opt/shop/    │   │ root /opt/shop/    │
               │      current ──────┼─┐ │      current       │
               │ releases/1.0/      │ │ │ releases/1.0/      │
               │ releases/1.1/ ◀────┼─┘ │ releases/1.1/      │
               └────────────────────┘   └────────────────────┘
```

### The release directory pattern

```
/opt/shop/
├── releases/
│   ├── 1.0/   index.html  VERSION
│   └── 1.1/   index.html  VERSION
└── current -> releases/1.1          ← nginx serves this
```

Switching versions = changing **one symlink**, which is atomic (a rename). Rolling back = pointing it back. No copying during the switch.

---

## Why it exists

"Stop everything, copy the new version, start everything" means downtime on every release, and a bad release breaks **all** servers at once. Rolling deploys with health checks and automatic rollback are the standard for VM-based services (Kubernetes Deployments do the same thing for containers). Building it yourself in Ansible teaches every piece: batching, delegation, handlers, error handling, idempotency.

---

## Analogy

**Repainting the lanes of a busy motorway at night.** Close **one lane** at a time with cones (drain), repaint it (deploy), inspect it (health check), reopen it only if it's good (enable), then move to the next lane. If the paint is wrong, scrape it back to the old markings (rollback) and stop the job (fail) before touching the other lanes. Traffic never stops.

---

## How it works inside

```
 deploy.yml -e app_version=1.1
 │
 ├─ PLAY 1 (localhost): build artifact dist/shop-1.1.tar.gz once (if missing), compute sha256
 │
 └─ PLAY 2 (hosts: web, serial: 1, max_fail_percentage: 0)
     batch [web1]:
       pre_tasks  ─ assert inputs; read current version (readlink); DRAIN web1  (delegate → lb1)
       tasks      ─ block:
                      copy artifact → unarchive → releases/1.1 (creates)
                      current → releases/1.1          (notify reload)
                      flush_handlers                  (reload nginx now)
                      health: GET web1:8080/version == "1.1"  (until/retries)
                    rescue:
                      current → previous release, reload, re-check
                      set rollback flag, fail
                    always:
                      health OK (new or old)? → ENABLE web1 (delegate → lb1)
       post_tasks ─ GET lb1:80 works; prune old releases
     batch [web2]: same
     any failure → max_fail_percentage 0 → web2 never touched
```

---

## Code / commands

All files live in `~/infra`.

### 1. The app source

```
~/infra/app/
├── index.html.j2
└── VERSION.j2
```

`app/index.html.j2`:

```jinja
Shop version {{ app_version }}
```

(Plain text keeps the probe output and `curl` checks simple. A real app would be a built artifact from CI.)

`app/VERSION.j2`:

```jinja
{{ app_version }}
```

### 2. The app role: `roles/shop_app`

`roles/shop_app/defaults/main.yml`:

```yaml
---
shop_app_version: ""                     # required, validated below
shop_app_root: /opt/shop
shop_app_port: 8080
shop_app_artifact: ""                    # local path to the tarball (set by the playbook)
shop_app_keep_releases: 3
```

`roles/shop_app/meta/argument_specs.yml`:

```yaml
---
argument_specs:
  main:
    short_description: Deploy a shop release with a current symlink
    options:
      shop_app_version:
        type: str
        required: true
        description: Version to deploy (e.g. 1.1).
      shop_app_artifact:
        type: path
        required: true
        description: Local tarball containing a directory named after the version.
      shop_app_root: { type: path, default: /opt/shop }
      shop_app_port: { type: int, default: 8080 }
      shop_app_keep_releases: { type: int, default: 3 }
```

`roles/shop_app/tasks/main.yml`:

```yaml
---
- name: Release directories exist
  ansible.builtin.file:
    path: "{{ shop_app_root }}/releases"
    state: directory
    mode: "0755"

- name: Release {{ shop_app_version }} is unpacked
  ansible.builtin.unarchive:
    src: "{{ shop_app_artifact }}"
    dest: "{{ shop_app_root }}/releases/"
    creates: "{{ shop_app_root }}/releases/{{ shop_app_version }}/VERSION"

- name: nginx site for the app
  ansible.builtin.template:
    src: shop.conf.j2
    dest: /etc/nginx/conf.d/shop.conf
    mode: "0644"
  notify: Reload nginx (shop)

- name: current points to {{ shop_app_version }}
  ansible.builtin.file:
    src: "{{ shop_app_root }}/releases/{{ shop_app_version }}"
    dest: "{{ shop_app_root }}/current"
    state: link
    force: true
  notify: Reload nginx (shop)
```

`roles/shop_app/templates/shop.conf.j2`:

```jinja
{{ ansible_managed | comment }}
server {
    listen {{ shop_app_port }};
    root {{ shop_app_root }}/current;
    index index.html;
    location = /version {
        default_type text/plain;
        alias {{ shop_app_root }}/current/VERSION;
    }
}
```

`roles/shop_app/handlers/main.yml`:

```yaml
---
- name: Check nginx config (shop)
  ansible.builtin.command: nginx -t
  changed_when: false
  listen: Reload nginx (shop)

- name: Reload nginx now (shop)
  ansible.builtin.systemd_service:
    name: nginx
    state: reloaded
  listen: Reload nginx (shop)
```

`roles/shop_app/tasks/prune.yml`:

```yaml
---
- name: List releases
  ansible.builtin.find:
    paths: "{{ shop_app_root }}/releases"
    file_type: directory
  register: __shop_releases

- name: Remove all but the newest {{ shop_app_keep_releases }} (never the current one)
  ansible.builtin.file:
    path: "{{ item.path }}"
    state: absent
  loop: "{{ (__shop_releases.files | sort(attribute='mtime', reverse=true))[shop_app_keep_releases:] }}"
  loop_control:
    label: "{{ item.path | basename }}"
  when: item.path | basename != shop_app_version
```

### 3. Point the load balancer at port 8080

In `inventories/lab/group_vars/web.yml` set:

```yaml
http_port: 8080
```

The lb1 template (topic 29) uses `hostvars[host]['http_port']`, so the upstream becomes `server 172.30.0.1x:8080;`. Re-apply it:

```bash
ansible-playbook -i inventories/lab playbooks/lb.yml
```

(The upstream points at 8080 before the app exists there; deploy version 1.0 right away, step 5.)

### 4. The deploy playbook

`playbooks/deploy.yml`:

```yaml
---
- name: Build the release artifact once (control node)
  hosts: localhost
  connection: local
  gather_facts: false
  vars:
    app_dist: "{{ playbook_dir }}/../dist"
  tasks:
    - name: Version given
      ansible.builtin.assert:
        that: app_version is defined and app_version is match('^[0-9]+\\.[0-9]+$')
        fail_msg: "Run with -e app_version=X.Y"

    - name: Staging directory for this version
      ansible.builtin.file:
        path: "{{ app_dist }}/build/{{ app_version }}"
        state: directory
        mode: "0755"

    - name: Render the release files
      ansible.builtin.template:
        src: "{{ playbook_dir }}/../app/{{ item }}.j2"
        dest: "{{ app_dist }}/build/{{ app_version }}/{{ item }}"
        mode: "0644"
      loop: [index.html, VERSION]

    - name: Package the release (directory named after the version)
      ansible.builtin.command:
        cmd: "tar -czf {{ app_dist }}/shop-{{ app_version }}.tar.gz -C {{ app_dist }}/build {{ app_version }}"
        creates: "{{ app_dist }}/shop-{{ app_version }}.tar.gz"

- name: Rolling deploy of the shop
  hosts: web
  become: true
  serial: 1
  max_fail_percentage: 0
  vars:
    shop_app_version: "{{ app_version }}"
    shop_app_artifact: "{{ playbook_dir }}/../dist/shop-{{ app_version }}.tar.gz"
    shop_health_url: "http://{{ hostvars[inventory_hostname]['ansible_host'] }}:{{ http_port }}/version"

  pre_tasks:
    - name: Remember the release currently live on this host
      ansible.builtin.command: readlink /opt/shop/current
      register: previous_link
      changed_when: false
      failed_when: false

    - name: Store the previous version (empty on first deploy)
      ansible.builtin.set_fact:
        shop_previous_version: "{{ previous_link.stdout | basename if previous_link.rc == 0 else '' }}"

    - name: Drain this host from the load balancer
      ansible.builtin.include_role:
        name: lb_member
        tasks_from: disable

    - name: Give in-flight requests a moment to finish
      ansible.builtin.pause:
        seconds: 2

  tasks:
    - name: Deploy, check, or roll back
      block:
        - name: Deploy the release
          ansible.builtin.include_role:
            name: shop_app

        - name: Apply the new config/symlink now
          ansible.builtin.meta: flush_handlers

        - name: Health check - the new version answers
          ansible.builtin.uri:
            url: "{{ shop_health_url }}"
            return_content: true
          register: health
          until: health.status == 200 and (health.content | trim) == app_version
          retries: 10
          delay: 2
          delegate_to: localhost
          become: false

        - name: Mark healthy
          ansible.builtin.set_fact:
            shop_healthy: true

      rescue:
        - name: Roll back to {{ shop_previous_version }}
          ansible.builtin.file:
            src: "/opt/shop/releases/{{ shop_previous_version }}"
            dest: /opt/shop/current
            state: link
            force: true
          when: shop_previous_version | length > 0

        - name: Reload nginx after rollback
          ansible.builtin.systemd_service:
            name: nginx
            state: reloaded

        - name: Health check - the old version answers
          ansible.builtin.uri:
            url: "{{ shop_health_url }}"
            return_content: true
          register: health_old
          until: health_old.status == 200
          retries: 10
          delay: 2
          delegate_to: localhost
          become: false
          when: shop_previous_version | length > 0

        - name: Healthy on the old version
          ansible.builtin.set_fact:
            shop_healthy: "{{ health_old is succeeded and shop_previous_version | length > 0 }}"

        - name: Fail loudly after rolling back
          ansible.builtin.fail:
            msg: >-
              Deploy of {{ app_version }} failed on {{ inventory_hostname }}
              ({{ ansible_failed_task.name }}); rolled back to {{ shop_previous_version | default('nothing') }}.

      always:
        - name: Re-enable in the load balancer only if healthy
          ansible.builtin.include_role:
            name: lb_member
            tasks_from: enable
          when: shop_healthy | default(false) | bool

  post_tasks:
    - name: The site answers through the load balancer
      ansible.builtin.uri:
        url: "http://{{ hostvars[groups['lb'][0]]['ansible_host'] }}/"
      delegate_to: localhost
      become: false

    - name: Prune old releases
      ansible.builtin.include_role:
        name: shop_app
        tasks_from: prune
```

Key points:

- **Play 1** builds the artifact on the control node once; play 2 ships the same file everywhere (R6).
- `serial: 1` + `max_fail_percentage: 0`: one host at a time; any failure stops the rollout (R1, R2).
- `pre_tasks` record the previous release and **drain** the host before anything changes.
- `flush_handlers` inside the block applies the new symlink/config **before** the health check (topic 30).
- The health check compares the **content** of `/version` with the expected version: "200 OK from the old version" isn't success.
- `rescue` rolls back, reloads, re-checks, then **fails** (R3, R4).
- `always` re-enables only when healthy on **some** version: a broken host stays drained.
- On the very first deploy there's no previous version; a failure then leaves the host drained (nothing to roll back to).

### 5. First deploy and the zero-downtime probe

Start the probe in a **second terminal** (it runs until Ctrl+C, then prints counts):

```bash
while true; do curl -s -o /dev/null -w '%{http_code}\n' --max-time 2 http://172.30.0.31/; sleep 0.1; done | tee /tmp/probe.log
```

In the first terminal:

```bash
cd ~/infra
ansible-playbook -i inventories/lab playbooks/deploy.yml -e app_version=1.0
```

Then stop the probe and count:

```bash
sort /tmp/probe.log | uniq -c
```
```
    412 200
```

(During the **very first** deploy you may see `502`s before 1.0 exists on port 8080, because the LB was pointed at 8080 in step 3. From the second deploy on, expect only `200`.)

### 6. Upgrade to 1.1 with the probe running

```bash
ansible-playbook -i inventories/lab playbooks/deploy.yml -e app_version=1.1
```
```
PLAY [Rolling deploy of the shop] ******************************************************

TASK [lb_member : Mark this host down in the load balancer upstream] ********************
changed: [web1 -> lb1]
...
TASK [shop_app : current points to 1.1] ************************************************
changed: [web1]

RUNNING HANDLER [shop_app : Check nginx config (shop)] *********************************
ok: [web1]
RUNNING HANDLER [shop_app : Reload nginx now (shop)] ***********************************
changed: [web1]

TASK [Health check - the new version answers] ******************************************
ok: [web1 -> localhost]
...
TASK [lb_member : Mark this host up in the load balancer upstream] **********************
changed: [web1 -> lb1]
...
PLAY [Rolling deploy of the shop] ******************************************************
(same for web2)

PLAY RECAP *****************************************************************************
localhost  : ok=5   changed=3   ...
web1       : ok=17  changed=6   unreachable=0  failed=0  skipped=4  rescued=0  ignored=0
web2       : ok=17  changed=6   unreachable=0  failed=0  skipped=4  rescued=0  ignored=0
```

```bash
sort /tmp/probe.log | uniq -c          # only 200s
curl -s 172.30.0.31/                   # Shop version 1.1
```

### 7. Idempotency check

```bash
ansible-playbook -i inventories/lab playbooks/deploy.yml -e app_version=1.1
```

The release tasks report `ok` (already unpacked, link already correct). The drain/enable steps still run (they're **actions**, by design: each run is a controlled rollout), but nothing in the app changes.

### 8. A bad release: watch the rollback

Simulate a broken release whose `/version` file is wrong: in `app/VERSION.j2`, temporarily write `broken` instead of `{{ app_version }}`, then:

```bash
ansible-playbook -i inventories/lab playbooks/deploy.yml -e app_version=1.2; echo "exit=$?"
```
```
TASK [Health check - the new version answers] ******************************************
FAILED - RETRYING: [web1 -> localhost]: Health check - the new version answers (10 retries left).
...
fatal: [web1 -> localhost]: FAILED! => {"attempts": 10, "content": "broken\n", ...}

TASK [Roll back to 1.1] ****************************************************************
changed: [web1]
TASK [Reload nginx after rollback] *****************************************************
changed: [web1]
TASK [Health check - the old version answers] ******************************************
ok: [web1 -> localhost]
TASK [Fail loudly after rolling back] **************************************************
fatal: [web1]: FAILED! => {"msg": "Deploy of 1.2 failed on web1 (Health check - the new version answers); rolled back to 1.1."}

TASK [lb_member : Mark this host up in the load balancer upstream] **********************
changed: [web1 -> lb1]

NO MORE HOSTS LEFT *********************************************************************
PLAY RECAP *****************************************************************************
web1   : ok=14  changed=6  unreachable=0  failed=1  ...
exit=2
```

- web1 was rolled back to 1.1 and **re-enabled** (healthy on the old version).
- web2 was **never touched** (still 1.1).
- Exit code 2: the pipeline goes red.
- The probe still shows only `200`.

Restore `app/VERSION.j2`, delete `dist/shop-1.2.tar.gz` and `dist/build/1.2`, and the bad release directory on web1 (`/opt/shop/releases/1.2`) or let pruning handle it later.

### 9. Manual instant rollback (ops playbook)

`playbooks/ops/rollback.yml`:

```yaml
---
- name: Roll back the shop to a given version
  hosts: web
  become: true
  serial: 1
  vars:
    shop_health_url: "http://{{ hostvars[inventory_hostname]['ansible_host'] }}:{{ http_port }}/version"
  tasks:
    - name: Target version exists
      ansible.builtin.stat:
        path: "/opt/shop/releases/{{ rollback_to }}/VERSION"
      register: target
      failed_when: not target.stat.exists

    - name: Drain
      ansible.builtin.include_role: { name: lb_member, tasks_from: disable }

    - name: Switch current
      ansible.builtin.file:
        src: "/opt/shop/releases/{{ rollback_to }}"
        dest: /opt/shop/current
        state: link
        force: true

    - name: Reload nginx
      ansible.builtin.systemd_service: { name: nginx, state: reloaded }

    - name: Health
      ansible.builtin.uri: { url: "{{ shop_health_url }}", return_content: true }
      register: h
      until: (h.content | trim) == rollback_to
      retries: 10
      delay: 2
      delegate_to: localhost
      become: false

    - name: Enable
      ansible.builtin.include_role: { name: lb_member, tasks_from: enable }
```

```bash
ansible-playbook -i inventories/lab playbooks/ops/rollback.yml -e rollback_to=1.0
```

---

## Hands-on lab

1. Build everything above. Deploy 1.0, then 1.1 with the probe running: only `200`s.
2. Run the bad-release scenario; confirm web1 rolled back and web2 untouched; check the exit code.
3. Deploy 1.3 and 1.4; check that pruning keeps three releases and never deletes the current one.
4. Use the ops rollback playbook to go back to 1.3; then deploy 1.4 again.
5. Change `serial: 1` to `serial: "100%"` and deploy with the probe running: count the non-200 responses (both servers drained at once). Put it back.
6. Stop web2's nginx before a deploy (`ssh ansible@172.30.0.12 'sudo systemctl stop nginx'`). What happens to web2's batch? (Its health check fails, rollback also fails the health check, so it stays drained and the run fails.) Start nginx and re-run.
7. Add a Slack/Teams-style notification task (a `debug` delegated to localhost is fine) with `run_once` at the start and end of the deploy.

**Done when:** you can ship a new version with zero failed requests, and a broken version stops after one server and rolls back automatically.

---

## Where you'll use it

- **VM-based services** behind HAProxy/nginx/F5/AWS ALBs: this exact pattern (swap `lb_member` for the right module).
- **Patching** with reboots: same structure, `reboot` instead of release switch.
- **Config changes** that need restarts on a cluster.
- **Interviews**: "How would you do a zero-downtime deployment with Ansible?" → describe this playbook.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| Probe shows 502s during deploy | Host not drained before restart, or LB reload races | Drain + short pause before changes; `serial: 1` |
| Probe shows 502s after deploy | Host re-enabled before healthy | Enable only after the health check passes (`always` + `shop_healthy`) |
| Health check passes on the old version | Checking only status 200 | Compare the returned version |
| Rollout continued after a failure | No `max_fail_percentage` | `max_fail_percentage: 0` |
| Pipeline green after a rollback | `rescue` cleared the failure | `fail` at the end of `rescue` |
| `unarchive` re-extracts every time | No `creates` | `creates: .../VERSION` |
| Host stays out of the LB forever | Both new and old versions unhealthy | Expected safety; alert and investigate; the probe/LB keeps serving with the others |
| Pruning removed the live release | Sorting by name/mtime without excluding current | Exclude `current`'s target (as in `prune.yml`) |
| On real RHEL: 403 from nginx | SELinux context on `/opt/shop` | `setype: httpd_sys_content_t` on the release dirs or serve from `/usr/share/nginx` |

---

## Quick recap

- Rolling deploy = `serial` batches + **drain** → deploy → **health check** (correct version) → **enable**, with `max_fail_percentage: 0`.
- Releases side by side + atomic **`current` symlink** = fast deploys and instant rollbacks.
- `block` (deploy + check) / `rescue` (rollback + re-check + **fail**) / `always` (enable **only if healthy**).
- Build the artifact once on the control node; ship the same file everywhere.
- Prove zero downtime with a live probe; prove safety with a deliberately bad release.

### Interview questions

1. **Q: How do you do a zero-downtime deployment with Ansible?**
   A: A play with `serial` (e.g. 1 or a small percentage) and `max_fail_percentage: 0`; for each host: drain it from the load balancer (delegated task), deploy, apply config via flushed handlers, health-check the new version, re-enable; with `block`/`rescue` to roll back and fail on errors.
2. **Q: Why use a `current` symlink and release directories?**
   A: Switching or rolling back is one atomic symlink change, with no copying during the switch, and previous releases stay available for instant rollback.
3. **Q: Why fail the run after a successful rollback?**
   A: The release is still broken; a red pipeline makes sure someone investigates instead of silently running the old version.

### Scenario question (answer out loud first, then compare)

> *"Your rolling deploy works, but the app needs a database schema migration for version 2.0 that's not backward compatible with 1.x. How do you deploy it without downtime?"*

**Model answer:**
1. Use the **expand/contract** (parallel change) pattern: never make a breaking schema change in the same release as the code that needs it.
2. **Release A (expand)**: add new columns/tables in a backward-compatible way; deploy the migration once (separate play, `run_once`, no `serial`) before the rolling code deploy; 1.x keeps working.
3. **Release 2.0**: code that writes to both old and new structures (or reads new with fallback); roll out with the normal rolling deploy.
4. **Backfill** data if needed (async job).
5. **Release B (contract)**: once everything runs 2.0 and nothing uses the old structures, remove them in a later migration.
6. Rollback plans: code rollbacks stay safe because the schema is compatible with both versions at every step.

**Likely follow-ups:**
- *Where does the migration task run?* Once, from a dedicated play (delegated to one app host or a migration runner), guarded by a schema-version check so re-runs are idempotent.
- *What if the migration is long?* `async`/`poll` and online migration tools; schedule it before the code rollout.
