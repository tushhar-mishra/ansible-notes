# 17 · Essential Modules I: Packages, Services, Users

> **Level:** ★ foundation
>
> **You will learn:** the modules you'll use in almost every playbook, in depth: **packages** (`package`, `dnf`, `apt`: lists, versions, `present`/`latest`/`absent`, cache refresh, repositories and keys), **services** (`systemd_service`, `service_facts`), **users and groups** (`user`, `group`: `append`, passwords as hashes and why they break idempotency, locking, removing), **SSH keys** (`ansible.posix.authorized_key`), and **cron** (`cron`). Each with its gotchas and output.
>
> **You need:** [16 · Modules, FQCNs and ansible-doc](16-modules-fqcn-and-ansible-doc.md), [06 · Linux recap II](06-linux-recap-admin-tasks.md), [14 · First playbook](14-first-playbook.md).
>
> **Part 2 · Playbooks** · Next: [18 · Essential modules II: files](18-modules-files.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Five module families cover most server setup: install software, run services, manage users and groups, put SSH keys in place, schedule jobs. Each takes a **desired state** and changes only what's different.

**Picture it.** A building manager's checklist: "these appliances installed", "heating on and set to start every morning", "these tenants have keys", "bins collected every Tuesday".

**The one thing to know.**

```yaml
- name: Packages are installed
  ansible.builtin.package:
    name: [git, tree, curl]
    state: present

- name: Service is running and starts at boot
  ansible.builtin.systemd_service:
    name: nginx
    state: started
    enabled: true

- name: alice exists and is in devs (keeping her other groups)
  ansible.builtin.user:
    name: alice
    groups: devs
    append: true
```

**Try it now:** run the lab playbook below twice. The second run must show `changed=0`. If the password task changes every time, you've met this topic's biggest trap.

**You've got it if you can:** explain `present` vs `latest`, `started` vs `restarted`, and why `append: true` matters.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### 1. Packages

| Module | Use when | Key options |
|---|---|---|
| `ansible.builtin.package` | Same task must work on RHEL **and** Ubuntu, basic needs | `name`, `state`, `use` (force a backend) |
| `ansible.builtin.dnf` | RHEL family; you need dnf features | `name`, `state`, `enablerepo`, `disablerepo`, `security`, `bugfix`, `exclude`, `update_cache`, `autoremove`, `lock_timeout` |
| `ansible.builtin.apt` | Debian family; you need apt features | `name`, `state`, `update_cache`, `cache_valid_time`, `upgrade` (`dist`, `safe`…), `autoremove`, `deb` (install a .deb file), `install_recommends` |

**`state` values:**

| State | Means | Idempotent? | Production use |
|---|---|---|---|
| `present` (or `installed`) | Installed, any version | ✅ Yes | ✅ The normal choice |
| `latest` | Newest version from the repos | ⚠️ Changes whenever the repo gets an update | Only in deliberate patch playbooks |
| `absent` (or `removed`) | Not installed | ✅ Yes | Removing unwanted software |

**Pass a list, not a loop.** `name: [git, tree, curl]` installs all in **one** transaction: faster, and dependency resolution sees everything at once.

**Pin versions** for software where versions matter:

```yaml
- name: Exact nginx version (RHEL family)
  ansible.builtin.dnf:
    name: nginx-1.26.3
    state: present
- name: Exact nginx version (Debian family)
  ansible.builtin.apt:
    name: nginx=1.24.0-2ubuntu7
    state: present
```

Version syntax differs: dnf uses `name-version`, apt uses `name=version`.

**Repositories and keys:**

```yaml
- name: Docker repo (RHEL family)
  ansible.builtin.yum_repository:
    name: docker-ce
    description: Docker CE stable
    baseurl: https://download.docker.com/linux/rhel/$releasever/$basearch/stable
    gpgcheck: true
    gpgkey: https://download.docker.com/linux/rhel/gpg

- name: Docker repo (Debian family)
  ansible.builtin.deb822_repository:
    name: docker
    types: deb
    uris: https://download.docker.com/linux/ubuntu
    suites: noble
    components: stable
    signed_by: https://download.docker.com/linux/ubuntu/gpg
```

- `yum_repository` writes `/etc/yum.repos.d/docker-ce.repo` (the name is historic; it works with dnf).
- `deb822_repository` writes `/etc/apt/sources.list.d/docker.sources` in the modern deb822 format, and can fetch the signing key itself (`signed_by: URL`). ⚠️ The old way, `apt_key`, is deprecated (it trusted keys for **all** repositories).
- After adding an apt repo, refresh the cache (`apt: update_cache: true`) before installing from it.

### 2. Services

`ansible.builtin.systemd_service` (alias `ansible.builtin.systemd`):

| Option | Values | Meaning |
|---|---|---|
| `name` | unit name | `nginx`, `nginx.service`, `backup.timer` |
| `state` | `started`, `stopped`, `restarted`, `reloaded` | Now. `restarted`/`reloaded` **always** act (non-idempotent): use them in **handlers** (topic 30) |
| `enabled` | `true` / `false` | At boot |
| `daemon_reload` | `true` | Run `systemctl daemon-reload` first (after changing unit files) |
| `masked` | `true` / `false` | Prevent the unit from starting at all |
| `scope` | `system`, `user`, `global` | Manage a user's own services |

`ansible.builtin.service_facts` collects the state of every service into `ansible_facts['services']`, which you can check with `when` (topic 26).

**Service names differ** between families: `sshd`/`ssh`, `crond`/`cron`, `httpd`/`apache2`, `chronyd`/`chrony`. Handle it with `when` (this topic) or variables (topics 21, 24).

### 3. Users and groups

`ansible.builtin.group`: `name`, `gid`, `system`, `state`.

`ansible.builtin.user` options you'll use:

| Option | Meaning | Gotcha |
|---|---|---|
| `name` | Login name | required |
| `uid`, `group` (primary), `groups` (supplementary) | IDs and groups | `groups` **replaces** the list unless `append: true` |
| `append` | Add to `groups` without removing others | Default **false** |
| `shell`, `home`, `create_home`, `comment` | Account details | |
| `system` | Create a system account (low UID, no aging) | For service accounts |
| `password` | **Hashed** password (never plain text) | See below |
| `update_password` | `always` (default) or `on_create` | `on_create` = set only when the user is new |
| `password_lock` | Lock the password (`usermod -L`) | Key login may still work |
| `expires` | Account expiry (epoch seconds), `-1` = never | |
| `state` + `remove` | `absent` + `remove: true` = delete user **and** home | Like `userdel -r` |
| `generate_ssh_key` | Create a key pair for this user | For service accounts that SSH elsewhere |

**Passwords and idempotency.** The `password` option wants a **hash** like the ones in `/etc/shadow` ([Linux 023](../../linux-mastery-rhel/notes/023-passwords-and-aging.md)). The usual way is the `password_hash` **filter** (a Jinja2 function you apply with `|`; topic 22):

```yaml
password: "{{ 'S3cret!' | password_hash('sha512') }}"
```

`password_hash` uses a **random salt** each time, so every run produces a **different hash** → the module sees "different" → **`changed` every run**. Two fixes:

1. `update_password: on_create`: set the password only when the user is created. ✅ Most common.
2. A fixed salt: `password_hash('sha512', 'fixedsalt123')`, with the salt kept secret too. Works, but weaker practice.

And the plain-text password itself belongs in **Vault** (topic 40), never in Git.

### 4. SSH authorized keys

`ansible.posix.authorized_key` (from the **ansible.posix** collection):

```yaml
- name: alice can log in with her key
  ansible.posix.authorized_key:
    user: alice
    key: "{{ lookup('file', 'files/alice.pub') }}"
    state: present
```

- `lookup('file', 'files/alice.pub')`: read a file **on the control node** and use its contents (lookups: topic 32). The path is relative to the playbook.
- `exclusive: true`: the file must contain **only** these keys (removes others). Powerful for access control, dangerous if you forget a key: you can lock out the automation user itself.
- It creates `~/.ssh` and sets `0700`/`0600` modes for you (topic 05's rules).

### 5. Cron

`ansible.builtin.cron`:

```yaml
- name: Nightly backup at 02:30
  ansible.builtin.cron:
    name: nightly-backup               # identifies the entry: keep it stable
    user: root
    minute: "30"
    hour: "2"
    job: /usr/local/sbin/backup.sh >> /var/log/backup.log 2>&1
```

- The module writes a marker comment `#Ansible: nightly-backup` above the line, and uses `name` to find it next time. Change `name` and you get a **second** entry.
- `cron_file: backup` writes `/etc/cron.d/backup` instead of a user's crontab.
- For new work, systemd **timers** are often better ([Linux 039](../../linux-mastery-rhel/notes/039-job-scheduling-cron-and-timers.md)); you'd install them with the file modules (topic 18) + `systemd_service`.

---

## Why it exists

Each of these replaces a family of hand-written shell checks (topic 01):

| By hand you'd need | The module handles |
|---|---|
| `rpm -q x \|\| dnf install x`, plus apt for Ubuntu | `package` (+ family detection) |
| `systemctl is-active x \|\| systemctl start x`, same for enable | `systemd_service` |
| `id alice` then useradd **or** usermod with the right flags, group merging | `user` |
| `grep -qF "$key" authorized_keys \|\| echo >>`, plus mkdir and chmod | `authorized_key` |
| Editing crontabs without duplicating lines | `cron` |

---

## Analogy

**A hotel's opening checklist.**

- **Packages** = furniture in each room ("bed, desk, lamp: present"). `latest` would be "replace the furniture whenever a newer model comes out": exciting for a showroom, risky for a hotel full of guests.
- **Services** = utilities: heating `started` and `enabled` (comes on every morning). `restarted` = switching it off and on at every inspection: guests notice.
- **Users** = staff badges. `groups` without `append` = "your badge opens **only** these doors" (all others revoked).
- **Authorized keys** = the key cards that open a staff door; `exclusive` = cancel every card not on this list.
- **Cron** = the cleaning rota; the `name` is the rota line's label.

---

## How it works inside

What `ansible.builtin.package` does with `name: [git, tree]`, `state: present`:

```
 control node                                        target
 ─────────────                                       ──────
 package action plugin:
   which package manager? ── uses fact pkg_mgr ─────▶ (gathered with facts, or detected now)
   → "dnf" on web1, "apt" on web2
   runs ansible.builtin.dnf / ansible.builtin.apt  ─▶ dnf module:
     with name=[git, tree] state=present               ├ ask dnf: which of these are installed?
                                                       ├ none missing → changed=false
                                                       └ some missing → one transaction → changed=true
```

And `ansible.builtin.user` with `name: alice, groups: devs, append: true`:

```
 read /etc/passwd, /etc/group (via getent/pwd)
   alice missing?            → useradd ... -G devs                       → changed
   alice exists:
     shell/home/uid differ?  → usermod -s / -d / -u ...                  → changed
     devs not in her groups? → usermod -a -G devs alice                  → changed
     all the same            →                                           → ok
```

---

## Code / commands

### The lab playbook

`~/ansible-lab/base.yml`:

```yaml
---
- name: Base packages, users and jobs on every server
  hosts: all
  become: true

  tasks:
    - name: Base tools are installed
      ansible.builtin.package:
        name: [git, tree, curl, sudo]
        state: present

    - name: cron is installed (RedHat family calls it cronie)
      ansible.builtin.package:
        name: cronie
        state: present
      when: ansible_facts['os_family'] == 'RedHat'

    - name: cron is installed (Debian family)
      ansible.builtin.package:
        name: cron
        state: present
      when: ansible_facts['os_family'] == 'Debian'

    - name: crond is running and enabled (RedHat family)
      ansible.builtin.systemd_service:
        name: crond
        state: started
        enabled: true
      when: ansible_facts['os_family'] == 'RedHat'

    - name: cron is running and enabled (Debian family)
      ansible.builtin.systemd_service:
        name: cron
        state: started
        enabled: true
      when: ansible_facts['os_family'] == 'Debian'

    - name: Group devs exists
      ansible.builtin.group:
        name: devs
        gid: 2000

    - name: alice exists, in devs, keeping her other groups
      ansible.builtin.user:
        name: alice
        uid: 2001
        shell: /bin/bash
        groups: devs
        append: true

    - name: alice can log in with the lab key
      ansible.posix.authorized_key:
        user: alice
        key: "{{ lookup('file', 'nodes/ansible_key.pub') }}"     # the copy from topic 09, relative to the playbook
        state: present

    - name: bob exists with an initial password (set only once)
      ansible.builtin.user:
        name: bob
        uid: 2002
        shell: /bin/bash
        password: "{{ 'ChangeMe-123' | password_hash('sha512') }}"
        update_password: on_create

    - name: Old account 'temp1' is gone
      ansible.builtin.user:
        name: temp1
        state: absent
        remove: true

    - name: alice's cleanup job runs every hour
      ansible.builtin.cron:
        name: alice-tmp-cleanup
        user: alice
        minute: "15"
        job: find /home/alice/tmp -type f -mtime +7 -delete 2>/dev/null
```

- `sudo` is in the base list as a harmless example of "make sure it's there".
- `uid: 2001` / `gid: 2000`: fixed IDs keep file ownership consistent across servers (important with shared storage like NFS).
- The plain password is in the playbook **only for the lab**. In real life it comes from Vault (topic 40).

### First and second run

```bash
ansible-playbook base.yml
```
```
TASK [Base tools are installed] ********************************************************
changed: [web1]
changed: [db1]
ok: [web2]
changed: [lb1]
...
TASK [alice exists, in devs, keeping her other groups] *********************************
changed: [web1]
changed: [web2]
changed: [db1]
changed: [lb1]
...
TASK [Old account 'temp1' is gone] *****************************************************
ok: [web1]
...
PLAY RECAP *****************************************************************************
db1    : ok=10  changed=8  unreachable=0  failed=0  skipped=2  rescued=0  ignored=0
lb1    : ok=10  changed=8  unreachable=0  failed=0  skipped=2  rescued=0  ignored=0
web1   : ok=10  changed=7  unreachable=0  failed=0  skipped=2  rescued=0  ignored=0
web2   : ok=10  changed=6  unreachable=0  failed=0  skipped=2  rescued=0  ignored=0
```

- web2 already had some tools (from topic 14), so some tasks are `ok` there.
- `temp1` is `ok` everywhere if you deleted it in topic 12; it's `changed` only if it still existed.
- `skipped=2` everywhere: each host skips the two tasks for the other family.

```bash
ansible-playbook base.yml          # second run
```
```
PLAY RECAP *****************************************************************************
db1    : ok=10  changed=0  ...
lb1    : ok=10  changed=0  ...
web1   : ok=10  changed=0  ...
web2   : ok=10  changed=0  ...
```

`changed=0` everywhere, **thanks to `update_password: on_create`**.

### The password trap, live

Change bob's task to `update_password: always` and run twice:

```
TASK [bob exists with an initial password (set only once)] *****************************
changed: [web1]
changed: [web2]
changed: [db1]
changed: [lb1]
```

Every run: `changed`. A new random salt makes a new hash each time. Put `on_create` back.

### Look at the result JSON (-v)

```bash
ansible-playbook base.yml -v -l web1 2>&1 | grep -A3 'alice exists'
```
```
TASK [alice exists, in devs, keeping her other groups] *********************************
ok: [web1] => {"append": true, "changed": false, "comment": "", "group": 2001, "groups": "devs", "home": "/home/alice", "move_home": false, "name": "alice", "shell": "/bin/bash", "state": "present", "uid": 2001}
```

- These are the module's documented **return values** (`ansible-doc ansible.builtin.user`).
- `"group": 2001`: alice's **primary** group (a private group `alice` created with her, GID matching her UID on this system).

### Check services across the fleet

```bash
ansible all -m ansible.builtin.service_facts -o 2>/dev/null | head -c 300; echo
ansible all -b -m ansible.builtin.systemd_service -a "name=sshd" -l alma | grep -E 'SUCCESS|ActiveState'
```

### Update packages on purpose (a patch task)

```yaml
- name: Security updates only (RHEL family)
  ansible.builtin.dnf:
    name: "*"
    state: latest
    security: true
  when: ansible_facts['os_family'] == 'RedHat'

- name: Upgrade packages (Debian family)
  ansible.builtin.apt:
    upgrade: safe
    update_cache: true
  when: ansible_facts['os_family'] == 'Debian'
```

- This belongs in a separate **patching** playbook, run on purpose, in batches (topic 45), not in your normal baseline.

---

## Hands-on lab

1. Write `base.yml`, run it twice, confirm `changed=0` on the second run.
2. Prove alice's key works: `ssh -i ~/.ssh/ansible_key alice@172.30.0.12 'id'` → shows `devs`.
3. **The `append` trap**: on web1, add alice to `wheel` by hand: `ssh ansible@172.30.0.11 'sudo usermod -aG wheel alice'`. Change the task to `append: false` and run with `-l web1`. She loses `wheel`. Check: `ssh ansible@172.30.0.11 'id alice'`. Set `append: true` back.
4. **The password trap**: switch bob to `update_password: always`, run twice, count `changed`. Switch back.
5. **Drift**: stop cron on lb1 by hand (`sudo systemctl stop cron`), run the playbook, see one `changed` on lb1.
6. **Cron**: log in to db1 and run `sudo crontab -l -u alice`. Find the `#Ansible: alice-tmp-cleanup` marker. Rename the task's `name:` and run: now there are **two** entries. Fix it by adding a task with the **old** name and `state: absent`, run, then remove that task.
7. Remove bob everywhere with an ad hoc command, then run the playbook again: he's recreated with the initial password (because he's new again).

**Done when:** the playbook is idempotent and you can explain each of the traps in steps 3, 4 and 6.

---

## Where you'll use it

- **Every baseline role** (topic 57): base packages, admin users and keys, service accounts, scheduled jobs.
- **Onboarding/offboarding** engineers: add or remove a user + key on every server; `exclusive: true` for strict access control.
- **Patching playbooks**: `dnf: name='*' state=latest security=true`, `apt: upgrade=safe`, in batches.
- **App deploys**: install a pinned package version, then ensure the service is started (restarts via handlers).
- **Third-party software**: add the vendor repo + key, then install.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `No package matching 'x' is available` (apt) | Stale/missing apt cache, or a new repo not refreshed | `update_cache: true` (+ `cache_valid_time`) before installing |
| `No package x available` (dnf) | Wrong name for this family, repo disabled, or needs EPEL (Extra Packages for Enterprise Linux) | Check the name; `enablerepo`; add the repo |
| `Could not get lock /var/lib/dpkg/lock-frontend` | Another apt process running (often `unattended-upgrades` on a fresh VM) | `lock_timeout` (apt) or wait; retry with `until` (topic 28) |
| User lost supplementary groups | `groups:` without `append: true` | Add `append: true` (or list all groups) |
| Password task `changed` every run | Random salt in `password_hash` | `update_password: on_create` |
| `ERROR! couldn't resolve module/action 'ansible.posix.authorized_key'` | Collection missing | `ansible-galaxy collection install ansible.posix` |
| Automation user locked out after a run | `authorized_key` with `exclusive: true` didn't include the automation key | Always include it; test on one host first |
| `Could not find the requested service crond` on Ubuntu | Name differs per family | Per-family task/variable |
| Service task fails: `Unable to start service x: Job for x.service failed` | The service itself is broken (bad config) | `journalctl -u x` on the host; fix the config, then re-run |
| `userdel: user alice is currently used by process 1234` | Removing a logged-in user | `force: true` (careful) or stop their processes first |
| Duplicate cron entries | The task's `name` was changed | Remove the old one with `state: absent` and the old name |

---

## Quick recap

- **Packages**: `package` for portable basics, `dnf`/`apt` for features; pass **lists**; `present` (normal), `latest` (patching only), `absent`; pin versions where it matters; repos with `yum_repository`/`deb822_repository` (⚠️ not `apt_key`).
- **Services**: `systemd_service` with `state: started` + `enabled: true`; `restarted` only in handlers; names differ per family.
- **Users**: `append: true` with `groups`; passwords as hashes with `update_password: on_create`; `state: absent` + `remove: true` to delete.
- **Keys**: `ansible.posix.authorized_key`; careful with `exclusive`.
- **Cron**: `name` identifies the entry; changing it duplicates.

### Interview questions

1. **Q: How do you install packages on both RHEL and Ubuntu with one task?**
   A: `ansible.builtin.package`, which picks dnf or apt based on the host. For distro-specific options or different package names, use per-family tasks or variables.
2. **Q: Why might a `user` task with a password report `changed` every run?**
   A: `password_hash` generates a new random salt each time, so the hash differs. Use `update_password: on_create` (or a fixed salt).
3. **Q: Why shouldn't you use `state: restarted` in a normal task?**
   A: It restarts the service on every run (not idempotent) and causes unnecessary outages. Use `state: started` and restart through a handler only when config changes.

### Scenario question (answer out loud first, then compare)

> *"Security asks you to remove an engineer who left the company from all 400 servers within an hour, and to make sure nobody else has unexpected SSH keys for the shared `deploy` account. How do you do it with Ansible, safely?"*

**Model answer:**
1. **Remove the person**: a playbook task `ansible.builtin.user: name=<user> state=absent remove=true` (add `force: true` if processes may be running), plus removal from any sudoers drop-in. Run first with `--check` on all hosts, then for real, in batches with forks raised.
2. **Lock down `deploy`**: keep the list of **approved** public keys in Git, and use `ansible.posix.authorized_key` with `exclusive: true` so any other key is removed. Make sure the **automation's own key** is in the approved list, or you lock yourself out.
3. Test on one host first (`-l host`), check you can still connect, then roll out.
4. Report: unreachable hosts need follow-up (they still have the old account). Rerun later or schedule a drift run.
5. Longer term: central identity (LDAP/IdM/SSSD, [Linux 030](../../linux-mastery-rhel/notes/030-central-identity-nss-and-sssd.md)) or SSH certificates make offboarding instant without touching every server.

**Likely follow-ups:**
- *How do you prove it worked?* An ad hoc `getent passwd <user>` across all hosts (should fail everywhere), and `wc -l ~deploy/.ssh/authorized_keys`.
- *What if a host was unreachable?* It keeps the old access: track it, fix connectivity, re-run. Firewall or isolate it if it stays unreachable.
