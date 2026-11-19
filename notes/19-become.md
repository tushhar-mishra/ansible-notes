# 19 · Privilege Escalation: become

> **Level:** ★★ intermediate
>
> **You will learn:** how Ansible gets root (or another user's) rights after logging in as a normal user: the keywords **`become`**, **`become_user`**, **`become_method`**, **`become_flags`**, the variables **`ansible_become_*`** and the options `-b` / `-K`; what Ansible really runs (`sudo -H -S -n -u root /bin/sh -c 'echo BECOME-SUCCESS-... ; python3 ...'`); passwords and prompts; the classic **"becoming an unprivileged user"** temp-file problem and its fixes (ACLs, pipelining); and the **security** trade-offs of the automation user's sudo rule.
>
> **You need:** [18 · Essential modules II](18-modules-files.md), [06 · Linux recap II](06-linux-recap-admin-tasks.md) (sudo basics), [Linux 028 · sudo and su](../../linux-mastery-rhel/notes/028-sudo-and-su.md), [Linux 027 · ACLs](../../linux-mastery-rhel/notes/027-special-permissions-acls-and-attributes.md).
>
> **Part 2 · Playbooks** · Next: [20 · Safe runs: check, diff, limit](20-check-diff-limit.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Ansible logs in as a normal user (`ansible`) and, when a task needs more rights, runs the module **through sudo**. That's **become**. By default it becomes `root`; `become_user` lets it become any other user (like `postgres`).

**Picture it.** You enter the office with your own badge (SSH as `ansible`). For the server room you ask security (sudo) to open the door; the logbook records your name.

**The one thing to know.**

```yaml
- name: Admin work
  hosts: db
  become: true                      # every task: sudo to root
  tasks:
    - name: postgres client is installed
      ansible.builtin.package:
        name: postgresql
        state: present

    - name: A file owned and written by alice
      ansible.builtin.copy:
        content: "hi\n"
        dest: /home/alice/hello.txt
      become_user: alice            # this task: sudo -u alice
```

```bash
ansible-playbook site.yml             # works if the sudo rule is NOPASSWD
ansible-playbook site.yml -K          # asks "BECOME password:" if sudo needs one
```

**Try it now:** `ansible web1 -a whoami` → `ansible`; `ansible web1 -a whoami -b` → `root`; `ansible web1 -a whoami -b --become-user alice` → `alice`.

**You've got it if you can:** explain what `-b`, `-K` and `become_user` each do, and why becoming a non-root user can fail with "Failed to set permissions on the temporary files".

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### The keywords

| Keyword (play/block/task) | Variable (inventory/vars) | CLI | Meaning | Default |
|---|---|---|---|---|
| `become` | `ansible_become` | `-b` / `--become` | Turn privilege escalation on | `false` |
| `become_user` | `ansible_become_user` | `--become-user` | Who to become | `root` |
| `become_method` | `ansible_become_method` | `--become-method` | Which tool | `sudo` |
| `become_flags` | `ansible_become_flags` | | Extra flags for the tool | sudo: `-H -S -n` |
| (no keyword) | `ansible_become_password` | `-K` / `--ask-become-pass` | Password for the tool | none |
| `become_exe` | `ansible_become_exe` | | Path to the tool | `sudo` |

Set them where they fit:

```yaml
- hosts: all
  become: true                 # play level: all tasks
  tasks:
    - name: Read-only check, no root needed
      ansible.builtin.command: id
      become: false            # task level overrides the play
      changed_when: false
```

**Least privilege**: put `become: true` on the tasks (or blocks) that need it, not on everything, when practical. In real playbooks most tasks need root, so play-level `become: true` is common and acceptable.

### Become methods

| Method | Tool | Notes |
|---|---|---|
| `sudo` (default) | `sudo` | Your user's sudo rules (topic 06, Linux 028) |
| `su` | `su` | Needs the **target** user's password (root's). Rare on RHEL 10/Ubuntu (root has no password) |
| `doas` | OpenBSD's simpler sudo | BSDs |
| `pbrun`, `pfexec`, `dzdo`, `ksu`, `pmrun` | Enterprise privilege tools | Some regulated environments |
| `machinectl` | systemd | Becoming a user **with a full systemd user session** (for `systemctl --user`) |
| `runas` | Windows | Windows hosts (topic 44) |
| `community.general.sudosu` | sudo + su | Edge cases |

### What Ansible actually runs

With `become: true` (sudo), the module execution from topic 13 becomes:

```
sudo -H -S -n  -u root /bin/sh -c 'echo BECOME-SUCCESS-qxklmwxbqfprkcpxvxzyjekcxk ; /usr/bin/python3 /home/ansible/.ansible/tmp/ansible-tmp-.../AnsiballZ_dnf.py'
 │    │  │  │    │       │              │                               │
 │    │  │  │    │       │              │                               └ the module, as root
 │    │  │  │    │       │              └ marker: Ansible watches for it to know sudo succeeded
 │    │  │  │    │       └ a shell to run both commands
 │    │  │  │    └ target user (become_user)
 │    │  │  └ -n: never prompt (fail instead), used when no password was given
 │    │  └ -S: read the password from stdin (used when a password was given with -K)
 │    └ -H: set HOME to the target user's home
 └ the become tool
```

- If a password is supplied, Ansible removes `-n`, adds a **unique prompt** (`-p "[sudo via ansible, key=...] password:"`), watches the output for it, and sends the password.
- `BECOME-SUCCESS-<random>`: printed first by the escalated shell. Seeing it proves the escalation worked; Ansible strips it before parsing the JSON.

### Passwords

| Situation | What to do |
|---|---|
| Automation user has `NOPASSWD` | Nothing. Most common in production |
| sudo needs a password, interactive run | `-K` (prompts once, used for all hosts) |
| sudo needs a password, CI | `ansible_become_password` from **Ansible Vault** or a secret manager (topics 40–41), never plain text |
| Different passwords per host | `ansible_become_password` in `host_vars` (vaulted) |

Since ansible-core 2.22, become passwords are registered as **secrets** and masked as `$REDACTED$` in output and logs.

### Becoming an unprivileged user: the temp-file problem

When you log in as `ansible` and `become_user: postgres`:

```
1. Ansible (as ansible) creates   /home/ansible/.ansible/tmp/ansible-tmp-XYZ/AnsiballZ_x.py   (mode 0700, owner ansible)
2. sudo -u postgres python3 that file
3. postgres can't read it  →  ✗ permission denied
```

Root can read anything, so `become_user: root` never hits this. For a **non-root** target user, Ansible tries these fixes, in order:

| Order | Method | Needs |
|---|---|---|
| 1 | `setfacl` to give the become user read access | The **`acl`** package (`setfacl`) on the target, and a filesystem with ACL support |
| 2 | `chown` the file to the become user | Login user must be root (rare) |
| 3 | `chmod +a` | macOS only |
| 4 | `chgrp` to a shared group | `ansible_common_remote_group` set to a group both users are in |
| 5 | World-readable temp files | ⚠️ Only if `allow_world_readable_tmpfiles = True` (insecure: any local user could read module arguments, maybe secrets) |

If none works:

```
fatal: [lb1]: FAILED! => {"msg": "Failed to set permissions on the temporary files Ansible needs to create when becoming an unprivileged user (rc: 1, err: chmod: invalid mode: 'A+user:alice:rx:allow'...). For information on working around this, see https://docs.ansible.com/..."}
```

**Fixes in practice:** install `acl` on targets (most common), or enable **pipelining** (topic 48), which sends the module through stdin so there's no temp file to share.

### Security: the automation user's sudo rule

Ansible runs `sudo /bin/sh -c '<anything>'`, so you **can't meaningfully restrict** which commands Ansible may run with sudo rules. `deploy ALL=(ALL) NOPASSWD: ALL` effectively means "this key is root on every server". So protect the key, not the sudo rule:

| Control | How |
|---|---|
| A **dedicated** automation user | Not a person's account; no password login (`usermod -p '*'`) |
| Key protection | Key only in CI secrets / AWX credentials; rotate it; passphrase + agent for humans |
| Restrict **where** the key works from | `from="10.0.5.0/24"` option in `authorized_keys`, firewall SSH to the CI/bastion network |
| Audit | sudo logs only show `COMMAND=/bin/sh -c echo BECOME-SUCCESS-...`, so keep **Ansible-side** logs (`log_path`, AWX job history, CI logs) |
| Short-lived credentials | SSH certificates with expiry, or AWX/AAP credential injection (topic 56) |
| Separate users per environment | A different key/user for prod vs staging |

---

## Why it exists

Logging in **directly as root** over SSH is a bad idea: no per-person accountability, root passwords/keys shared, and many distros disable root SSH login (`PermitRootLogin no`, [Linux 066](../../linux-mastery-rhel/notes/066-ssh-server-hardening.md)). The standard practice is "log in as a normal user, escalate with sudo". `become` makes Ansible follow exactly that practice, works with any escalation tool, and lets one playbook run some tasks as root and others as application users (`postgres`, `nginx`, `app`).

---

## Analogy

**A hotel maintenance worker.** They enter with their **staff card** (SSH as `ansible`). For guest rooms they ask reception for a **master key** (`become: true` → root). For one guest's safe they ask for **that guest's key** (`become_user: alice`). If the worker leaves their toolbox in their own locker (temp files in `~ansible`), the guest's key can't open that locker (permission problem): they either give the guest's key **read access** to the locker (ACL) or hand the tools through the door directly (pipelining).

---

## How it works inside

```
 task with become: true, become_user: alice (method sudo, no password)
 │
 ├─ 1. SSH as ansible (topic 05)
 ├─ 2. create temp dir as ansible: ~ansible/.ansible/tmp/ansible-tmp-XYZ (0700)
 ├─ 3. upload AnsiballZ_copy.py
 ├─ 4. target user is NOT root → make the file readable for alice:
 │       setfacl -m u:alice:r-x  ~ansible/.ansible/tmp/ansible-tmp-XYZ/ ...AnsiballZ_copy.py
 │       (fails? try chown / chgrp / world-readable, see table)
 ├─ 5. run: sudo -H -S -n -u alice /bin/sh -c 'echo BECOME-SUCCESS-abc ; python3 .../AnsiballZ_copy.py'
 │       ├ sudo checks /etc/sudoers.d/ansible: (ALL) NOPASSWD: ALL → allowed
 │       ├ prints BECOME-SUCCESS-abc   ← Ansible waits for this (timeout → error)
 │       └ module runs as alice, prints JSON
 └─ 6. remove temp dir (as ansible)
```

With **pipelining**, steps 2–4 and 6 disappear: `sudo -u alice /bin/sh -c 'echo BECOME-SUCCESS-abc ; python3'` reads the module from stdin.

---

## Code / commands

### Ad hoc

```bash
ansible web1 -a whoami                                  # ansible
ansible web1 -a whoami -b                               # root
ansible web1 -a whoami -b --become-user alice           # alice
ansible web1 -a "cat /etc/shadow" -o                    # FAILED: permission denied (no -b)
```
```
web1 | FAILED | rc=1 | (stdout)  (stderr) cat: /etc/shadow: Permission denied
```

### See the sudo command (-vvv)

```bash
ansible web1 -m ansible.builtin.ping -b -vvv 2>&1 | grep -o "sudo -H -S -n.*BECOME-SUCCESS-[a-z]*"
```
```
sudo -H -S -n  -u root /bin/sh -c '"'"'echo BECOME-SUCCESS-yzlqjduvhxyrnsmpfvxdxbmlrjkqwqgo
```

### A playbook mixing users

`~/ansible-lab/become.yml`:

```yaml
---
- name: Become in practice
  hosts: web2,lb1
  become: true

  tasks:
    - name: acl is installed (needed to become unprivileged users)
      ansible.builtin.package:
        name: acl
        state: present

    - name: alice has a tmp dir she owns
      ansible.builtin.file:
        path: /home/alice/tmp
        state: directory
        owner: alice
        group: alice
        mode: "0750"

    - name: A note written AS alice (owner set by the process itself)
      ansible.builtin.copy:
        content: "written by alice\n"
        dest: /home/alice/tmp/note.txt
        mode: "0640"
      become_user: alice

    - name: Who am I, without become
      ansible.builtin.command: whoami
      become: false
      register: me
      changed_when: false

    - name: Show it
      ansible.builtin.debug:
        var: me.stdout
```

- `hosts: web2,lb1`: a pattern with a comma = a list of hosts.
- The `copy` task runs as **alice**, so the file is created by her and owned by her, without any `owner:`.
- `register: me` + `changed_when: false`: save the result; it's a read-only command so never "changed" (topics 26–27).
- `debug: var: me.stdout`: prints `ansible`.

### The unprivileged-user failure, live

On lb1, remove `acl` and try becoming alice:

```bash
ansible lb1 -b -m ansible.builtin.package -a "name=acl state=absent"
ansible lb1 -b --become-user alice -m ansible.builtin.command -a whoami
```
```
lb1 | FAILED! => {
    "msg": "Failed to set permissions on the temporary files Ansible needs to create when becoming an unprivileged user (rc: 1, err: chmod: invalid mode: 'A+user:alice:rx:allow'\nTry 'chmod --help' for more information.\n}). For information on working around this, see https://docs.ansible.com/ansible-core/2.22/playbook_guide/playbooks_privilege_escalation.html#risks-of-becoming-an-unprivileged-user"
}
```

- Ansible tried `setfacl` (missing), then `chown` (not root), then the macOS `chmod +a` (invalid on Linux), and gave up.
- Fix 1: install `acl` (the playbook above). Fix 2: pipelining:

```bash
ANSIBLE_PIPELINING=True ansible lb1 -b --become-user alice -m ansible.builtin.command -a whoami
```
```
lb1 | CHANGED | rc=0 >>
alice
```

### Password prompts

Make sudo require a password for a test user (lab only) and see the errors:

```bash
ansible web1 -b -m ansible.builtin.shell -a \
 "echo 'ansible:LabPass-1' | chpasswd && echo 'ansible ALL=(ALL) ALL' > /etc/sudoers.d/ansible"
ansible web1 -b -a whoami
```
```
web1 | FAILED! => {
    "msg": "Missing sudo password"
}
```

```bash
ansible web1 -b -K -a whoami
```
```
BECOME password: ********
web1 | CHANGED | rc=0 >>
root
```

Restore the lab rule (and the locked password) with your password once more, or just `./reset.sh` in `nodes/`:

```bash
ansible web1 -b -K -m ansible.builtin.copy -a "content='ansible ALL=(ALL) NOPASSWD: ALL\n' dest=/etc/sudoers.d/ansible mode=0440 validate='/usr/sbin/visudo -cf %s'"
```

### Become on the control node: a trap

Tasks with `connection: local` or `delegate_to: localhost` (topic 46) **also** honour `become`. With play-level `become: true`, Ansible tries `sudo` **on your laptop/CI runner**:

```
fatal: [web1 -> localhost]: FAILED! => {"msg": "Missing sudo password"}
```

Add `become: false` to local tasks.

---

## Hands-on lab

1. Run the three `whoami` ad hoc commands (as ansible, root, alice).
2. Write and run `become.yml`. Check ownership: `ssh ansible@172.30.0.31 'sudo ls -l /home/alice/tmp'` → `alice alice`.
3. Reproduce the unprivileged-user failure on lb1 (remove `acl`), read the message, fix it with pipelining once and with `acl` once.
4. Reproduce `Missing sudo password` on web1, run with `-K`, then restore NOPASSWD (or `./reset.sh`).
5. Look at the sudo log on web1 after a run: `ssh ansible@172.30.0.11 'sudo journalctl _COMM=sudo -n 3 --no-pager'`. Note how uninformative `COMMAND=/bin/sh -c echo BECOME-SUCCESS-...` is. That's why you keep Ansible-side logs.
6. Optional hardening: on db1, restrict the lab key to your WSL2 bridge address. Find it (`ip -4 addr show | grep 172.30.0.1`), then prefix the key in `~ansible/.ssh/authorized_keys` with `from="172.30.0.1" ` using `lineinfile` or `authorized_key`'s `key_options: 'from="172.30.0.1"'`. Confirm Ansible still connects.

**Done when:** you can explain each line of the `sudo -H -S -n -u ...` command and fix the unprivileged-become error two ways.

---

## Where you'll use it

- **Every production playbook**: log in as an automation user, escalate to root.
- **Database work**: `become_user: postgres` for `community.postgresql.*` modules and `psql` commands.
- **App deploys**: run build/migrate steps as the app user so files get the right owner.
- **Locked-down environments**: sudo with a password from Vault, or enterprise tools (`pbrun`, `dzdo`).
- **Security reviews**: explaining what the automation user can do and how its access is protected.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Missing sudo password` | sudo rule needs a password, none given | `-K`, `ansible_become_password` (vaulted), or a NOPASSWD rule |
| `Incorrect sudo password` | Wrong password | Check the vaulted value / the user's password |
| `Timeout (12s) waiting for privilege escalation prompt` | sudo printed something unexpected (custom prompt, lecture, PAM message) or hung | `-vvvv`; `Defaults lecture=never`; increase `timeout`; check PAM. Since 2.19 this is reported as **unreachable** (use `ignore_unreachable`, not `ignore_errors`, if you must continue) |
| `ansible is not in the sudoers file` / `not allowed to execute '/bin/sh'` | No rule, or a rule restricted to specific commands | Give the automation user a full rule (Ansible always runs `/bin/sh`) |
| `sudo: a terminal is required to read the password` | `requiretty` in sudoers | Remove `Defaults requiretty` for the automation user, or enable pipelining with care |
| `Failed to set permissions on the temporary files ... becoming an unprivileged user` | Non-root `become_user` without ACL support | Install `acl`, or enable pipelining (topic 48) |
| File owned by root when you expected the app user | `become_user` not set (defaults to root) | Set `become_user` or `owner:` |
| `Missing sudo password` on `[web1 -> localhost]` | Play-level `become` applied to a local task | `become: false` on delegated/local tasks |
| `su: Authentication failure` | `become_method: su` needs root's password | Use sudo, or provide root's password via vault |
| `Privilege escalation timeout` only on some hosts | Slow LDAP/SSSD lookups during sudo | Fix name service; raise `timeout` |

---

## Quick recap

- **become** = run modules through sudo (or su, doas, pbrun...) after logging in as a normal user. Default target: `root`.
- Keywords `become`, `become_user`, `become_method`, `become_flags`; variables `ansible_become_*`; CLI `-b`, `--become-user`, `-K`.
- Ansible runs `sudo -H -S -n -u USER /bin/sh -c 'echo BECOME-SUCCESS-... ; python3 module'`, so sudo rules can't usefully restrict commands. Protect the **key** instead.
- Becoming a **non-root** user needs **`acl`** on the target or **pipelining**.
- Passwords: NOPASSWD, `-K`, or vaulted `ansible_become_password` (masked in 2.22+).

### Interview questions

1. **Q: What does `become` do, and how is it different from `remote_user`?**
   A: `remote_user` (or `ansible_user`) is who Ansible logs in as over SSH. `become` escalates after login (by default via sudo to root) for the tasks that need it.
2. **Q: Why might `become_user: postgres` fail when `become_user: root` works?**
   A: The module's temp files belong to the login user; root can read them but `postgres` can't. Ansible needs ACLs (`setfacl`) or pipelining to share them.
3. **Q: How do you provide a sudo password in CI?**
   A: As `ansible_become_password` from Ansible Vault or a secret manager, injected at run time, never stored in plain text. Or use a NOPASSWD rule for a tightly protected automation user.

### Scenario question (answer out loud first, then compare)

> *"An auditor says: 'Your Ansible user has `NOPASSWD: ALL`. That's root without a password. Restrict it to only the commands Ansible needs.' How do you respond?"*

**Model answer:**
1. Explain the mechanics: Ansible executes every module as `sudo /bin/sh -c 'python3 <module>'`. Command-restricted sudo rules either break Ansible or are trivially bypassed (allowing `/bin/sh` is allowing everything).
2. So the real control is **who can use the key**, not the sudo rule. Propose:
   - dedicated non-human user, no password, key-only;
   - key stored only in the CI secret store / AWX credential vault, never on laptops; rotated regularly; ideally short-lived SSH certificates;
   - `from=` restriction in `authorized_keys` and firewall rules so SSH is only accepted from the CI runners/bastion;
   - separate users/keys per environment;
   - full audit trail on the Ansible side (AWX job logs or CI logs with who triggered what, playbook in Git with reviews).
3. If a password is mandated: sudo with a password injected from Vault (`ansible_become_password`). It adds a secret to protect but satisfies policy.

**Likely follow-ups:**
- *Why not log in as root directly?* No accountability, root SSH is usually disabled, and the same key problem exists, worse.
- *How do you see what Ansible did as root?* sudo logs only show the wrapper; rely on Ansible job logs, `--diff` output saved per run, and auditd rules on sensitive files ([Linux 077](../../linux-mastery-rhel/notes/077-auditd.md)).
