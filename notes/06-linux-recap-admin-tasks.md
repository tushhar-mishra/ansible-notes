# 06 · Linux Recap II: Users, sudo, Packages, Services, Files

> **Level:** ★ foundation
>
> **You will learn:** a short recap of the admin tasks you already do by hand on Linux (users and groups, sudo, packages and repositories, systemd services, files, ownership and permissions, SELinux, firewall, cron), the **Ansible module that replaces each command**, why the module is better than the command, the RHEL vs Ubuntu differences that bite automation, and how Ansible gets root rights through **sudo** (`become`).
>
> **You need:** [05 · Linux recap I: SSH](05-linux-recap-ssh.md). Full Linux background is linked in every row below.
>
> **Part 0 · Foundations** · Next: [07 · Python on the managed node](07-python-on-managed-nodes.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Ansible doesn't invent new admin tasks. Every module is a careful, idempotent version of something you already do by hand. If you know `useradd`, you understand `ansible.builtin.user`. If you know `systemctl enable --now`, you understand `ansible.builtin.systemd_service`.

**Picture it.** Your Linux commands are **verbs** ("add a user", "start a service"). Ansible modules are **descriptions** ("the user exists", "the service is running"). The module works out the verb, and only runs it when needed.

**The one thing to know.** The mapping you'll use most:

| By hand | Ansible module (FQCN) | Desired state you write |
|---|---|---|
| `useradd -m deploy` | `ansible.builtin.user` | `name: deploy`, `state: present` |
| `dnf install nginx` / `apt install nginx` | `ansible.builtin.package` (or `.dnf` / `.apt`) | `name: nginx`, `state: present` |
| `systemctl enable --now nginx` | `ansible.builtin.systemd_service` | `state: started`, `enabled: true` |
| `cp file /etc/x; chown; chmod` | `ansible.builtin.copy` | `src`, `dest`, `owner`, `mode` |
| `mkdir -p /opt/app` | `ansible.builtin.file` | `path`, `state: directory` |
| `sudo ...` | the `become: true` keyword | (Ansible runs the module through sudo) |

FQCN = **Fully Qualified Collection Name**: the module's full name, `<namespace>.<collection>.<module>`. Topic 16 explains it.

**Try it now:** in WSL2, run `sudo -l`. That's what Ansible relies on when you write `become: true`.

**You've got it if you can:** name the module for "user exists", "package installed", "service running and enabled", and say what `become` does.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

A recap of the Linux admin areas Ansible automates, each with: the manual commands, the module, and what the module does better.

### 1. Users and groups

**Recap** ([Linux 022](../../linux-mastery-rhel/notes/022-users-and-accounts.md), [023](../../linux-mastery-rhel/notes/023-passwords-and-aging.md), [024](../../linux-mastery-rhel/notes/024-groups.md)): users live in `/etc/passwd` (name, UID = user ID number, home, shell), password hashes in `/etc/shadow`, groups in `/etc/group`. A user has one **primary** group and any number of **supplementary** groups.

| By hand | Module | Example arguments |
|---|---|---|
| `groupadd -g 2000 devs` | `ansible.builtin.group` | `name: devs`, `gid: 2000` |
| `useradd -m -s /bin/bash -G devs alice` | `ansible.builtin.user` | `name: alice`, `shell: /bin/bash`, `groups: devs`, `append: true` |
| `usermod -aG wheel alice` | `ansible.builtin.user` | `groups: wheel`, `append: true` |
| `userdel -r bob` | `ansible.builtin.user` | `name: bob`, `state: absent`, `remove: true` |
| `passwd alice` | `ansible.builtin.user` | `password: "{{ hash }}"` (a **hash**, never plain text, topic 40) |
| append to `~/.ssh/authorized_keys` | `ansible.posix.authorized_key` | `user: alice`, `key: "ssh-ed25519 AAAA..."` |

⚠️ **`append: true` matters.** Without it, `groups: devs` means "alice is in **exactly** these supplementary groups", and Ansible will **remove** her from every other group (like `usermod -G` without `-a`).

Note: `authorized_key` lives in the **`ansible.posix`** collection, not in `ansible.builtin`. You install it separately if you use only ansible-core (topic 37).

### 2. sudo

**Recap** ([Linux 028](../../linux-mastery-rhel/notes/028-sudo-and-su.md)): `sudo` runs one command as root if a rule in `/etc/sudoers` or `/etc/sudoers.d/*` allows it. Files in `sudoers.d` must have **no dot** in the name, be owned by root, mode `0440`, and must be checked with `visudo -c`, because a broken sudoers file can lock everyone out.

How Ansible **uses** sudo: it logs in as a normal user (say `deploy`), then runs each module through `sudo` when you set **`become: true`**. That's called **privilege escalation**. The usual rule for an automation user:

```
# /etc/sudoers.d/ansible     (no dot in the name; mode 0440)
deploy ALL=(ALL) NOPASSWD: ALL
```

- `NOPASSWD`: Ansible doesn't have to supply a password (otherwise you pass `-K` / `--ask-become-pass`).
- Full details, tighter rules and the security trade-offs: topic 19.

How Ansible **manages** sudo rules on servers:

| By hand | Module |
|---|---|
| `visudo -f /etc/sudoers.d/ops` | `ansible.builtin.copy` or `ansible.builtin.template` with `validate: /usr/sbin/visudo -cf %s` and `mode: "0440"` |
| (simple rules) | `community.general.sudoers` |

`validate:` runs the check on a **temporary copy** first and only puts the file in place if it passes. Exactly the safe pattern from Linux 028, built in.

### 3. Packages and repositories

**Recap** ([Linux 031](../../linux-mastery-rhel/notes/031-packages-rpm-dnf-and-apt.md)): RHEL/AlmaLinux/Rocky use **RPM** (RPM Package Manager) packages with `dnf`; Ubuntu/Debian use **deb** packages with `apt`. Packages come from **repositories**, signed with GPG (GNU Privacy Guard) keys.

| By hand | Module | Notes |
|---|---|---|
| `dnf install -y nginx` | `ansible.builtin.dnf` | `state: present` |
| `apt-get update && apt-get install -y nginx` | `ansible.builtin.apt` | `update_cache: true`, `cache_valid_time: 3600` (refresh at most hourly) |
| either, distro-neutral | `ansible.builtin.package` | Calls dnf or apt for you. Only basic options |
| `dnf update -y` | `ansible.builtin.dnf` | `name: "*"`, `state: latest` |
| `dnf remove telnet` | `ansible.builtin.dnf` | `state: absent` |
| new `.repo` file | `ansible.builtin.yum_repository` | Name kept from the yum era; works with dnf |
| new apt source | `ansible.builtin.deb822_repository` | (Older: `apt_repository`) |
| `rpm --import key` | `ansible.builtin.rpm_key` | |
| `rpm -qa` / `dpkg -l` | `ansible.builtin.package_facts` | Gathers installed packages as data |

⚠️ `ansible.builtin.yum` is **gone**: since ansible-core 2.17 it's just a redirect to `dnf`. Write `dnf`.

**`present` vs `latest`:** `present` = "installed, any version" (idempotent and stable). `latest` = "upgrade every run if a newer version exists" (changes whenever the repo changes, so it can surprise you in production). Use `present` for normal installs and **pin versions** for critical software (`name: nginx-1.26.2`).

### 4. Services (systemd)

**Recap** ([Linux 037](../../linux-mastery-rhel/notes/037-systemd-services-and-targets.md), [098](../../linux-mastery-rhel/notes/098-systemd-unit-files.md)): **systemd** starts and supervises services. `start`/`stop` = now. `enable`/`disable` = at boot. After editing a unit file you run `systemctl daemon-reload`.

| By hand | Module arguments (`ansible.builtin.systemd_service`) |
|---|---|
| `systemctl start nginx` | `state: started` |
| `systemctl stop nginx` | `state: stopped` |
| `systemctl restart nginx` | `state: restarted` (⚠️ restarts **every** run: use in handlers, topic 30) |
| `systemctl reload nginx` | `state: reloaded` |
| `systemctl enable nginx` | `enabled: true` |
| `systemctl daemon-reload` | `daemon_reload: true` |
| `systemctl mask x` | `masked: true` |

- `ansible.builtin.systemd` is the older name of the same module; `systemd_service` is the current name.
- `ansible.builtin.service` is a generic module that works with systemd and older init systems. Fine too.

### 5. Files, ownership, permissions

**Recap** ([Linux 007](../../linux-mastery-rhel/notes/007-files-and-directories.md), [026](../../linux-mastery-rhel/notes/026-file-permissions-and-umask.md), [027](../../linux-mastery-rhel/notes/027-special-permissions-acls-and-attributes.md)): every file has an **owner**, a **group** and a **mode** (`rwx` for user/group/others, written in octal like `0644`). Directories need `x` to be entered.

| By hand | Module | Topic |
|---|---|---|
| `mkdir -p /opt/app` | `ansible.builtin.file` (`state: directory`) | 18 |
| `touch`, `ln -s`, `rm -rf` | `ansible.builtin.file` (`state: touch` / `link` / `absent`) | 18 |
| `chown app:app f; chmod 0640 f` | any file module: `owner`, `group`, `mode: "0640"` | 18 |
| `cp app.conf /etc/app/` | `ansible.builtin.copy` | 18 |
| edit one line with `sed -i` | `ansible.builtin.lineinfile` | 18 |
| `cat >> file <<EOF` block | `ansible.builtin.blockinfile` | 18 |
| fill a config from variables | `ansible.builtin.template` | 29 |
| `stat file` | `ansible.builtin.stat` | 18 |
| `scp server:/var/log/x .` | `ansible.builtin.fetch` | 18 |
| `curl -o` / `wget` | `ansible.builtin.get_url` | 18 |
| `tar xzf` | `ansible.builtin.unarchive` | 18 |
| `setfacl` | `ansible.posix.acl` | 18 |

### 6. Security: firewall and SELinux

| By hand ([Linux 071](../../linux-mastery-rhel/notes/071-firewalld.md), [072](../../linux-mastery-rhel/notes/072-selinux-fundamentals.md)–[074](../../linux-mastery-rhel/notes/074-selinux-booleans-ports-troubleshooting.md)) | Module |
|---|---|
| `firewall-cmd --permanent --add-service=http` + `--reload` | `ansible.posix.firewalld` (`permanent: true`, `immediate: true`) |
| `ufw allow 80/tcp` (Ubuntu) | `community.general.ufw` |
| `setenforce` / edit `/etc/selinux/config` | `ansible.posix.selinux` |
| `setsebool -P httpd_can_network_connect on` | `ansible.posix.seboolean` |
| `semanage fcontext -a -t ...` + `restorecon` | `community.general.sefcontext` + file modules' `setype` |
| `semanage port -a -t http_port_t -p tcp 8080` | `community.general.seport` |

### 7. System settings and scheduling

| By hand | Module |
|---|---|
| `hostnamectl set-hostname web1` | `ansible.builtin.hostname` |
| `timedatectl set-timezone UTC` | `community.general.timezone` |
| `sysctl -w` + `/etc/sysctl.d/` | `ansible.posix.sysctl` |
| `crontab -e` | `ansible.builtin.cron` |
| edit `/etc/fstab` + `mount` | `ansible.posix.mount` |
| `reboot` and wait | `ansible.builtin.reboot` |
| `pvcreate/vgcreate/lvcreate/mkfs` | `community.general.lvg`, `lvol`, `filesystem` |

### RHEL vs Ubuntu: names that differ

Automation that targets both families must handle these:

| Thing | RHEL / AlmaLinux / Rocky | Ubuntu / Debian |
|---|---|---|
| Package manager | `dnf` | `apt` |
| Admin group | `wheel` | `sudo` |
| SSH service name | `sshd` | `ssh` |
| Apache package / service | `httpd` | `apache2` |
| Firewall | `firewalld` | `ufw` (or nftables) |
| MAC (Mandatory Access Control) | SELinux | AppArmor |
| Auth log | `/var/log/secure` | `/var/log/auth.log` |
| Default cloud user | `ec2-user` (AWS), `cloud-user` | `ubuntu` |

Ansible knows which family a host belongs to through **facts** (`ansible_facts['os_family']` is `RedHat` or `Debian`, topic 23), so one playbook can pick the right names.

---

## Why it exists

Why not just run the same commands through Ansible's `command` module?

| Command | Problem in automation | Module fixes it by |
|---|---|---|
| `useradd alice` | Fails if she exists | Checking first; creating or **modifying** only what differs |
| `usermod -G devs alice` | Silently removes other groups | Explicit `append` choice |
| `echo rule >> /etc/sudoers` | Duplicates every run; one typo locks you out | Writing a whole file and validating it first |
| `systemctl restart nginx` | Restarts every run (drops connections) | `state: started` does nothing when already running |
| `sed -i 's/a/b/'` | Can't tell you if it changed anything | Reporting `changed` only when the file changed |
| `dnf`/`apt` | Different command per distro | `package` picks the right one |

Each module **checks → compares → changes only if needed → reports `changed` or `ok`**. That's the idempotency idea from topic 01, already written and tested for you.

---

## Analogy

**A restaurant kitchen's prep list.**

- Doing it by hand is the cook's **verbs**: "chop 2 onions", "boil water". Do them twice and you have 4 chopped onions.
- An Ansible module is a **station check**: "2 chopped onions are in the tray". The prep cook looks at the tray, chops more only if there are fewer than 2, and says "done, nothing needed" otherwise.
- `become` is the **key to the walk-in freezer**: a normal cook asks the head chef (sudo) to open it each time, and the logbook records who asked.

---

## How it works inside

What happens when a module like `ansible.builtin.user` runs with `become: true`, compared with doing it by hand:

```
 BY HAND                                  ANSIBLE (user module, become: true)
 ─────────                                ───────────────────────────────────
 ssh deploy@web1                          ssh deploy@web1 (same SSH, topic 05)
 sudo useradd -m -s /bin/bash alice       sudo -H -S -n -u root /bin/sh -c 'python3 .../AnsiballZ_user.py'
                                            │
                                            ├─ read /etc/passwd: does alice exist?
                                            │    no  → run useradd ...      → changed: true
                                            │    yes → compare shell, groups, home...
                                            │           same     → changed: false
                                            │           different → run usermod ... → changed: true
                                            └─ print JSON: {"changed": true, "name": "alice", "uid": 1001, ...}
```

Key insights:

1. **Modules usually call the same tools you'd call** (`useradd`, `usermod`, `dnf`, `systemctl`). They just decide *whether* to call them.
2. **sudo is used exactly like you use it**, with the flags `-n` (never prompt) or `-S` (read the password from input) so it works without a terminal.
3. The result is **data** (JSON), not text you have to parse. You'll use that data with `register` (topic 26).

---

## Code / commands

The same baseline by hand and as Ansible tasks (playbook syntax arrives in topic 14; for now just compare line by line).

**By hand on RHEL:**

```bash
sudo groupadd -f devs
sudo useradd -m -s /bin/bash -G devs alice
sudo dnf install -y chrony
sudo systemctl enable --now chronyd
sudo mkdir -p /opt/app && sudo chown alice:devs /opt/app && sudo chmod 0750 /opt/app
echo '%devs ALL=(root) /usr/bin/systemctl restart app' | sudo tee /etc/sudoers.d/devs
sudo chmod 0440 /etc/sudoers.d/devs && sudo visudo -c
```

**As Ansible tasks:**

```yaml
- name: devs group exists
  ansible.builtin.group:
    name: devs

- name: alice exists, in devs
  ansible.builtin.user:
    name: alice
    shell: /bin/bash
    groups: devs
    append: true

- name: chrony is installed
  ansible.builtin.package:
    name: chrony
    state: present

- name: chronyd is running and enabled
  ansible.builtin.systemd_service:
    name: chronyd
    state: started
    enabled: true

- name: /opt/app exists with the right owner and mode
  ansible.builtin.file:
    path: /opt/app
    state: directory
    owner: alice
    group: devs
    mode: "0750"

- name: devs may restart the app
  ansible.builtin.copy:
    dest: /etc/sudoers.d/devs
    content: "%devs ALL=(root) /usr/bin/systemctl restart app\n"
    owner: root
    group: root
    mode: "0440"
    validate: /usr/sbin/visudo -cf %s
```

- `name:` on each task is a human description, printed in the output. Write it as the **state** ("chrony is installed"), not the action.
- `validate: /usr/sbin/visudo -cf %s`: `%s` is replaced by the path of a temporary file. If `visudo` fails, the real file is never touched.
- `mode: "0750"`: quoted, as topic 04 explained.

Sample output from running these twice (first run, then second):

```
TASK [alice exists, in devs] ************************************
changed: [web1]
...
TASK [devs may restart the app] *********************************
changed: [web1]
```
```
TASK [alice exists, in devs] ************************************
ok: [web1]
...
TASK [devs may restart the app] *********************************
ok: [web1]
```

- First run: `changed`: Ansible made the change.
- Second run: `ok`: already correct, nothing touched.

---

## Hands-on lab

Do the baseline **by hand** in a throw-away AlmaLinux container, so the Ansible versions later feel familiar. (No systemd inside a plain container, so services wait until topic 09.)

1. Start the container:

   ```bash
   docker run -it --rm --name lab06 almalinux:10 bash
   ```

2. Users and groups:

   ```bash
   dnf install -y sudo shadow-utils procps-ng
   groupadd -f devs
   useradd -m -s /bin/bash -G devs alice
   id alice                                   # uid, gid, groups=...,devs
   useradd -m -s /bin/bash -G devs alice      # run again → "already exists": NOT idempotent
   ```

3. The `-G` trap:

   ```bash
   groupadd -f ops; usermod -aG ops alice; id alice     # in devs AND ops
   usermod -G devs alice; id alice                      # ops is GONE (no -a)
   ```

   Remember this when you see `append: true` in the user module.

4. A validated sudoers drop-in:

   ```bash
   echo '%devs ALL=(root) NOPASSWD: /usr/bin/id' > /tmp/devs
   visudo -cf /tmp/devs && install -m 0440 -o root -g root /tmp/devs /etc/sudoers.d/devs
   su - alice -c 'sudo -n /usr/bin/id -un'              # prints "root"
   ```

5. Packages: `dnf install -y chrony` twice. The second run says `Package chrony... is already installed. Nothing to do.` That's dnf being idempotent on its own, which the module builds on.

6. Fill in the module column yourself, then check against the tables above:

   | Manual step you just did | Ansible module |
   |---|---|
   | `groupadd -f devs` | ? |
   | `useradd -m -s /bin/bash -G devs alice` | ? |
   | `install -m 0440 ... /etc/sudoers.d/devs` with `visudo -cf` | ? (+ which argument?) |
   | `dnf install -y chrony` | ? (two possible answers) |

7. `exit`.

---

## Where you'll use it

- **Server baselines** (topic 57 project): users, sudo rules, time sync, packages, SSH hardening, firewall.
- **Onboarding/offboarding**: add or remove an engineer's account and key on every server in one run.
- **Patching**: `dnf`/`apt` with `state: latest`, in batches.
- **Compliance**: auditors ask "show me every server's sudo rules". With Ansible, the answer is a file in Git.
- **Mixed fleets**: RHEL and Ubuntu in one playbook, picking package and service names from facts.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Missing sudo password` | `become: true` but the user's sudo rule requires a password | `NOPASSWD` rule, or run with `-K` (topic 19) |
| `deploy is not in the sudoers file` | The automation user has no sudo rule | Add `/etc/sudoers.d/ansible` (by hand or cloud-init) |
| A user lost all their other groups | `groups:` without `append: true` | Add `append: true`, re-add groups |
| sudo broken for everyone after a run | sudoers written without `validate` | Always `validate: /usr/sbin/visudo -cf %s`; fix via console/rescue |
| `couldn't resolve module/action 'ansible.posix.firewalld'` | Collection not installed (only ansible-core) | `ansible-galaxy collection install ansible.posix` (topic 37) |
| `No package matching 'httpd' is available` on Ubuntu | Package names differ per distro | Pick names from `ansible_facts['os_family']` (topic 23) |
| Service task fails: `Could not find the requested service sshd` on Ubuntu | Service is called `ssh` there | Same: per-family variables |
| Every run upgrades something | `state: latest` | Use `present`; patch on purpose, in a separate playbook |
| File ends up with odd permissions | Unquoted `mode` | `mode: "0644"` (topic 04) |

---

## Quick recap

- Every module is an idempotent version of a command you know: `user`, `group`, `package`/`dnf`/`apt`, `systemd_service`, `file`, `copy`, `lineinfile`, `template`.
- Ansible logs in as a normal user and uses **sudo** for root work when you set **`become: true`**. The automation user needs a sudo rule (usually `NOPASSWD`).
- Watch the traps: `append: true` for groups, `present` vs `latest`, `restarted` restarts every run, `validate` for sudoers, quoted modes.
- Some modules live outside `ansible.builtin` (`ansible.posix`, `community.general`).
- RHEL and Ubuntu differ in package/service/group names; facts let one playbook handle both.

### Interview questions

1. **Q: Why use the `user` module instead of `command: useradd ...`?**
   A: The module checks the current state and only creates or modifies what differs, so it's idempotent, reports `changed` accurately, supports check mode, and handles modifications (shell, groups) that `useradd` alone can't.
2. **Q: What does `become: true` do?**
   A: Runs the task with privilege escalation, by default through `sudo` to root, after logging in as the normal SSH user.
3. **Q: `state: present` vs `state: latest` for packages?**
   A: `present` installs if missing and otherwise does nothing. `latest` upgrades whenever a newer version exists, so results depend on the repository on the day you run. Prefer `present` (or pinned versions) for predictable runs.

### Scenario question (answer out loud first, then compare)

> *"After an Ansible run that 'just added alice to the docker group', alice can no longer use sudo. What happened, and how do you fix and prevent it?"*

**Model answer:**
1. The task probably used `ansible.builtin.user` with `groups: docker` and **no `append: true`**. That makes `docker` her *only* supplementary group, removing her from `wheel`/`sudo` (exactly like `usermod -G` without `-a`).
2. Fix now: re-run with `groups: [wheel, docker]` (or `append: true` plus the groups), and tell alice to log in again, because group membership is fixed at login.
3. Prevent: always set `append: true` when adding groups, unless you really want to enforce the exact list. Review with `--check --diff` before applying (topic 20), and test roles with Molecule (topic 51).

**Likely follow-ups:**
- *When would you want `append: false`?* When the code is the full source of truth for group membership, e.g. removing ex-employees from admin groups.
- *How do you safely push a sudoers change to 300 servers?* A template/copy with `validate: visudo -cf %s`, mode 0440, in batches, after testing in staging.
