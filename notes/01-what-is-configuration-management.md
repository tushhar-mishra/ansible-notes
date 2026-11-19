# 01 · What Is Configuration Management?

> **Level:** ★ foundation
>
> **You will learn:** what "configuration" means on a server, how teams managed it before tools existed and why that hurt (snowflake servers, runbooks, fragile scripts), what **configuration management** is, and the words you'll use in every later topic: **desired state**, **drift**, **idempotency**, **convergence**, **push vs pull**, **agent vs agentless**.
>
> **You need:** nothing. This is the start. (If you've read [Terraform topic 01](../../Terraform/notes/01-what-is-iac.md), some words will be familiar. Here they're applied to the *inside* of servers.)
>
> **Part 0 · Foundations** · Next: [02 · The tool landscape](02-tool-landscape.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A server is useful only after it's **configured**: the right packages installed, the right users created, config files written, services running, firewall ports open. Doing that **by hand** on one server is fine. On 50 servers it's slow, error-prone, and every server ends up slightly different.

**Configuration management** means you **write down how servers should be** (the *desired state*) in text files, keep those files in Git, and let a **tool** make every server match them, again and again.

**Picture it.** A hotel with 300 rooms. Instead of each cleaner deciding how a room should look, there's one **checklist** ("2 towels, bed made, TV on channel 1, minibar full"). A cleaner walks into each room, checks every item, and fixes only what's wrong. Rooms that are already perfect take 10 seconds.

**The one thing to know.** Good configuration management is **idempotent**: running it once or 100 times gives the same result. A second run on a correct server changes **nothing**.

**Try it now** (in your WSL2 Ubuntu terminal):

```bash
echo "PermitRootLogin no" >> /tmp/demo.conf   # run this 3 times
cat /tmp/demo.conf                              # 3 identical lines: NOT idempotent
grep -qxF "PermitRootLogin no" /tmp/demo.conf || echo "PermitRootLogin no" >> /tmp/demo.conf
                                                # "add it only if missing": idempotent
```

**You've got it if you can:** explain why "run the same checklist on every server, every day" is safer than "log in and fix things when someone notices".

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### What "configuration" means on a server

When a new Linux server boots for the first time, it's an empty shell. Before your app can run, someone has to set up everything **inside** it:

| Area | Examples | Linux topic |
|---|---|---|
| Packages | Install `nginx`, `python3`, `chrony`; remove `telnet` | [031 · Packages](../../linux-mastery-rhel/notes/031-packages-rpm-dnf-and-apt.md) |
| Users and access | Create `deploy` user, add SSH keys, sudo rules | [022](../../linux-mastery-rhel/notes/022-users-and-accounts.md), [028](../../linux-mastery-rhel/notes/028-sudo-and-su.md) |
| Files | `/etc/nginx/nginx.conf`, `/etc/ssh/sshd_config`, app config | [007](../../linux-mastery-rhel/notes/007-files-and-directories.md), [026](../../linux-mastery-rhel/notes/026-file-permissions-and-umask.md) |
| Services | `nginx` enabled and running, `postfix` stopped | [037 · systemd](../../linux-mastery-rhel/notes/037-systemd-services-and-targets.md) |
| Security | Firewall rules, SELinux booleans, SSH hardening | [071](../../linux-mastery-rhel/notes/071-firewalld.md), [072](../../linux-mastery-rhel/notes/072-selinux-fundamentals.md), [066](../../linux-mastery-rhel/notes/066-ssh-server-hardening.md) |
| System settings | Time sync, kernel parameters (`sysctl`), timezone, hostname | [069](../../linux-mastery-rhel/notes/069-time-sync-chrony.md), [055](../../linux-mastery-rhel/notes/055-kernel-modules-and-sysctl.md) |
| Scheduled jobs | Backups, log cleanup cron jobs | [039](../../linux-mastery-rhel/notes/039-job-scheduling-cron-and-timers.md) |

All of this together is the server's **configuration**.

### What configuration management is

**Configuration management (CM) = describing the configuration you want in files, and using a tool to make real servers match those files, now and later.**

Three parts, just like Infrastructure as Code (IaC):

1. **Code**: text files that say "nginx is installed", "user `deploy` exists", "port 443 is open".
2. **Version control** (Git): every change has an author, a date, a reason and a review.
3. **A tool**: connects to each server, checks what's there, and fixes only what's different. **Ansible** is the tool in these notes.

### The words you'll use forever

**Desired state.** What you *want* the server to look like: "nginx installed, version doesn't matter; nginx running; nginx starts on boot". You describe the *result*, not the commands.

**Actual state.** What the server *really* looks like right now.

**Drift.** When actual state slowly moves away from desired state. Someone SSHs in at 2 a.m., edits a config file to fix an incident, and never writes it down. Now this server differs from its 49 siblings.

**Convergence.** Bringing actual state back to desired state. A CM tool *converges* a server: it looks at the difference and fixes it.

**Idempotency** (say "eye-dem-POH-tent-see"). An action is **idempotent** if doing it many times has the same effect as doing it once.

| Action | Idempotent? | Why |
|---|---|---|
| `mkdir -p /opt/app` | ✅ Yes | "Make sure it exists". Already exists → nothing happens |
| `mkdir /opt/app` | ❌ No | Second run fails: "File exists" |
| `echo "x" >> file` | ❌ No | Every run adds another line |
| `printf 'x\n' > file` | ✅ Yes | File always ends up containing exactly `x` |
| `useradd deploy` | ❌ No | Second run: "user 'deploy' already exists" |
| `systemctl start nginx` | ✅ Yes | Already running → stays running |
| `systemctl restart nginx` | ❌ No | Restarts every time (drops connections every run) |

Idempotency is **the** most important property in configuration management. Topic 27 covers it in full depth. For now, remember: **a second run on a correct server must change nothing.**

**Push vs pull.**

- **Push**: a central machine *connects out* to servers and pushes changes when you run it. **Ansible is push** (by default).
- **Pull**: each server runs an **agent** (a background program) that regularly *asks* a central server "what should I look like?" and fixes itself. **Puppet** and **Chef** are pull.

**Agent vs agentless.**

- **Agent-based**: you must install and run special software on every server before the tool can manage it.
- **Agentless**: the tool uses something servers already have. Ansible uses **SSH** (Secure Shell, the encrypted remote-login protocol, [Linux topic 065](../../linux-mastery-rhel/notes/065-ssh.md)) and **Python**. Nothing extra to install, update or monitor on each server.

---

## Why it exists

### Life before configuration management

**Stage 1: by hand.** An admin SSHs into each server and types commands. On one server, fine. Problems appear fast:

| Problem | What it looks like in real life |
|---|---|
| **Snowflake servers** | Every server is unique, like a snowflake: hand-patched, hand-tuned over years. Nobody knows exactly what's on `web07`, so nobody dares to rebuild it. |
| **Drift** | 50 "identical" web servers, but `web23` has an old TLS (Transport Layer Security) setting and `web41` has debug logging on. |
| **No record** | "Who changed `sshd_config` on the database server?" No one knows. |
| **Slow** | A new server takes a day of copy-pasting from a wiki page. |
| **Human error** | Step 14 of 30 was skipped on 3 servers. Nobody noticed for a month. |
| **Bus factor** | Only Priya knows how the payment servers are set up. Priya is on holiday. |

**Stage 2: runbooks.** Teams wrote **runbooks**: wiki pages listing every step. Better, but people still skip steps, and the wiki goes out of date the day after it's written.

**Stage 3: shell scripts.** Teams turned runbooks into Bash scripts and copied them to servers with `scp` + `ssh` in a `for` loop. Much better, but scripts are hard to get right:

```bash
#!/usr/bin/env bash
# setup-web.sh: looks fine, but...
dnf install -y nginx
useradd deploy                                   # fails on the 2nd run
echo "deploy ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers   # adds a duplicate line every run
sed -i 's/^#Port 22/Port 2222/' /etc/ssh/sshd_config
systemctl restart sshd                           # restarts every run, even when nothing changed
```

Run it twice and you get errors, duplicate lines, and unneeded restarts. To make it safe you must add a check before **every** line ("does the user exist? is the line already there? did the file actually change?"). Multiply that by every kind of thing you manage, every Linux distribution, and every server, and the script becomes thousands of lines nobody wants to touch.

**That checking logic is exactly what a CM tool gives you, written once, tested by thousands of people.**

### What configuration management gives you

| Benefit | How |
|---|---|
| **Consistency** | Every server built from the same files is the same. |
| **Speed** | Configure 1 or 1,000 servers with one command. |
| **Safe re-runs** | Idempotent: run it every day; correct servers are untouched. |
| **Drift repair** | Each run puts manual changes back to the desired state. |
| **History and review** | `git log` shows who changed what. Changes go through Pull Requests (PRs). |
| **Preview** | "Dry run" mode shows what *would* change before anything changes. |
| **Documentation** | The code *is* the documentation of how servers are set up. |
| **Disaster recovery** | A server dies → build a new one and run the same code. |

---

## Analogy

**The hotel housekeeping checklist.**

- **By hand** = each cleaner does rooms "their way". Room 101 has 2 towels, room 102 has 3, room 103 has none because the cleaner forgot.
- **Runbook** = a printed checklist on the trolley. Better, but cleaners skip lines when they're in a hurry.
- **Shell script** = a robot that blindly does every step: "put 2 towels on the rack". In a room that already has 2 towels, now there are 4.
- **Configuration management** = a smart robot with the checklist: "the rack **has** 2 towels". It *looks first*, adds towels only if fewer than 2, and reports "room 101: OK, room 102: fixed 1 thing". That's **idempotency** and **convergence**.
- **Drift** = a guest moves the TV to the bathroom. Next time the robot visits, it notices and moves it back.
- **Push vs pull** = the manager sends the robot to rooms (push, Ansible) vs each room has its own robot that checks every 30 minutes (pull, Puppet/Chef).

---

## How it works inside

Every CM tool runs the same loop for each item it manages:

```
   ┌────────────────────┐      ┌───────────────────┐
   │ Desired state       │      │ Actual state       │
   │ (your files in Git) │      │ (look at server)   │
   └─────────┬──────────┘      └─────────┬─────────┘
             └──────────┬─────────────────┘
                        ▼
              ┌───────────────────┐
              │ Are they the same? │
              └───────┬───────────┘
            yes ◀─────┴─────▶ no
             │                 │
             ▼                 ▼
      report "ok"       make the change,
      (do nothing)      report "changed"
```

1. **Read** the desired state from your files ("user `deploy` exists with shell `/bin/bash`").
2. **Check** the actual state on the server (read `/etc/passwd`).
3. **Compare.**
4. **Same** → do nothing, report **ok**.
5. **Different** → make the smallest change that fixes it (`usermod -s /bin/bash deploy`), report **changed**.
6. **Error** → report **failed**.

Ansible prints exactly these words: `ok`, `changed`, `failed`. You'll see them in topic 12.

**Push vs pull, pictured:**

```
PUSH (Ansible)                               PULL (Puppet, Chef)

 your laptop / CI                             central server (holds the "catalog")
 ┌──────────────┐                             ┌──────────────┐
 │ ansible      │──SSH──▶ server 1            │ puppetserver │◀──every 30 min── agent on server 1
 │              │──SSH──▶ server 2            │              │◀──every 30 min── agent on server 2
 │              │──SSH──▶ server 3            │              │◀──every 30 min── agent on server 3
 └──────────────┘                             └──────────────┘
 runs when YOU run it                         runs on a timer, forever
```

| | Push (Ansible) | Pull (Puppet/Chef) |
|---|---|---|
| Who starts it | You or a CI pipeline | Each server's agent, on a timer |
| Needs on servers | SSH + Python (already there) | An agent installed and kept running |
| Drift fixed | When you next run it | Within ~30 minutes, automatically |
| Ordering ("do A on DB, then B on web") | Easy: you control the order | Hard: each server acts alone |
| Good for | Deploys, orchestration, one-off changes, smaller-to-large fleets | Very large fleets that must self-heal constantly |

Ansible can also do pull (`ansible-pull`, topic 55) when you need it.

---

## Code / commands

You haven't installed Ansible yet (topic 08). Here's the same goal three ways, so you can see what changes. Read it; you'll run the Ansible version in topic 14.

**Goal:** on a server, user `deploy` exists, `nginx` is installed, running and enabled at boot.

**1. By hand:**

```bash
ssh admin@web1
sudo useradd -m -s /bin/bash deploy
sudo dnf install -y nginx
sudo systemctl enable --now nginx
```

Now do it on 49 more servers, and remember next year exactly what you did.

**2. Idempotent shell script** (you must write every check yourself):

```bash
#!/usr/bin/env bash
set -euo pipefail
id deploy &>/dev/null || sudo useradd -m -s /bin/bash deploy   # create only if missing
rpm -q nginx &>/dev/null || sudo dnf install -y nginx           # install only if missing
systemctl is-enabled nginx &>/dev/null || sudo systemctl enable nginx
systemctl is-active nginx &>/dev/null || sudo systemctl start nginx
```

- Works, but only on RHEL-like systems (`rpm`, `dnf`). Ubuntu needs `dpkg`/`apt`.
- Doesn't tell you *what changed*. It doesn't check the user's shell. No preview mode.

**3. Ansible** (explained properly from topic 14 on):

```yaml
- name: Web server baseline
  hosts: web
  become: true
  tasks:
    - name: deploy user exists
      ansible.builtin.user:
        name: deploy
        shell: /bin/bash

    - name: nginx is installed
      ansible.builtin.package:
        name: nginx
        state: present

    - name: nginx is running and enabled
      ansible.builtin.systemd_service:
        name: nginx
        state: started
        enabled: true
```

- It describes **states** ("is installed", "is running"), not commands.
- `hosts: web` = do it on every server in the group `web` (1 or 1,000).
- `become: true` = use sudo on the server.
- `ansible.builtin.package` works on both RHEL (dnf) and Ubuntu (apt).

Sample output of the first run, then a second run:

```
PLAY RECAP *************************************************************
web1   : ok=4  changed=3  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```
```
PLAY RECAP *************************************************************
web1   : ok=4  changed=0  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

- `ok=4`: 4 steps finished successfully (3 tasks + 1 automatic "gather facts" step that collects info about the server). `ok` counts **every** successful step, including the ones that changed something.
- `changed=3` on the first run: user created, nginx installed, nginx started.
- `changed=0` on the second run: **idempotency in action**. Everything was already correct.

---

## Hands-on lab

No Ansible needed. You'll feel why idempotency matters, inside a throw-away container (Docker from your [Docker notes](../../Docker/notes/05-your-first-container.md)).

1. Start a throw-away Ubuntu container and get a shell in it:

   ```bash
   docker run -it --rm --name cm-lab ubuntu:24.04 bash
   ```

2. Inside it, write a **non-idempotent** setup script and run it three times:

   ```bash
   cat > /root/bad.sh <<'EOF'
   useradd deploy
   echo "MaxAuthTries 3" >> /etc/demo.conf
   EOF
   bash /root/bad.sh; bash /root/bad.sh; bash /root/bad.sh
   cat /etc/demo.conf
   ```

   Expect: `useradd: user 'deploy' already exists` twice, and **three** `MaxAuthTries 3` lines.

3. Write an **idempotent** version and run it three times:

   ```bash
   rm -f /etc/demo.conf; userdel deploy
   cat > /root/good.sh <<'EOF'
   id deploy &>/dev/null || { useradd deploy; echo "changed: user created"; }
   grep -qxF "MaxAuthTries 3" /etc/demo.conf 2>/dev/null \
     || { echo "MaxAuthTries 3" >> /etc/demo.conf; echo "changed: line added"; }
   echo "done"
   EOF
   bash /root/good.sh; bash /root/good.sh; bash /root/good.sh
   ```

   Expect: the first run prints two `changed:` lines, the next two runs print only `done`. You just wrote a tiny CM tool: it reports **changed** vs **ok**.

4. Simulate **drift**: `sed -i 's/3/10/' /etc/demo.conf`. Run `bash /root/good.sh` again and `cat /etc/demo.conf`. Notice the script added a *second* line instead of fixing the first one. "Make sure a line exists" is not the same as "make sure the setting has this value". Real modules (`ansible.builtin.lineinfile`, topic 18) handle this with a regular expression that finds and replaces the existing line.

5. Type `exit`. The container is deleted (`--rm`).

---

## Where you'll use it

- **Every server fleet** that isn't 100% containers: VMs (virtual machines) in AWS/Azure/GCP, bare-metal servers, on-prem data centres.
- **Baselines and hardening**: the same users, SSH settings, firewall rules and time sync on every server. Auditors love it.
- **Application deploys** on VMs: copy the new version, update config, restart in batches.
- **Kubernetes nodes and Jenkins agents**: they are still Linux servers that need packages, kernel settings and users.
- **Patching**: "update OpenSSL on all 400 servers, 20 at a time, and stop if 2 fail".
- **Network devices**: switches and routers (topic 44).
- **Interviews**: "What is idempotency?" and "Push vs pull?" are among the most common Ansible questions.

---

## When it breaks

CM doesn't remove problems; it changes which ones you get. The ones you'll meet in these notes:

| Problem | What it means | Where it's fixed |
|---|---|---|
| Someone edits servers by hand anyway | **Drift**: code and reality disagree | Re-run regularly, drift reports (topic 55) |
| Tasks use raw shell commands | Not idempotent: changes every run, can't preview | Topic 27 |
| One wrong variable reaches every server | A mistake now spreads to 500 servers in one run | Check/diff mode (20), batches (45), testing (51) |
| Passwords written in plain text in Git | Secrets leak | Vault (40) |
| "Works on my laptop" | Different tool versions on different machines | Execution environments (52) |

---

## Quick recap

- **Configuration** = everything *inside* a server: packages, users, files, services, security settings.
- **Configuration management** = desired state in files, in Git, applied by a tool that converges servers to match.
- **Idempotent** = running it again changes nothing if it's already correct. The second run should say `changed=0`.
- **Drift** = reality has moved away from the code; each CM run repairs it.
- **Push** (Ansible, agentless, over SSH) vs **pull** (Puppet/Chef, an agent on every server).

### Interview questions

1. **Q: What is configuration management?**
   A: Defining how servers should be configured (packages, files, users, services) in version-controlled code, and using a tool to make every server match it, consistently and repeatedly.
2. **Q: What is idempotency, and why does it matter?**
   A: Running an operation many times has the same effect as running it once. It makes re-runs safe, lets you run automation regularly to fix drift, and means "nothing changed" is a meaningful signal.
3. **Q: Push vs pull configuration management?**
   A: Push: a central node connects to servers and applies changes when triggered (Ansible). Pull: an agent on each server periodically fetches its configuration and applies it (Puppet, Chef). Push gives control over order and timing; pull self-heals continuously but needs agents.

### Scenario question (answer out loud first, then compare)

> *"We have 40 web servers set up by hand over three years. They're supposed to be identical but keep behaving differently. What would you do?"*

**Model answer:**
1. Admit the cause: snowflake servers and drift from manual changes.
2. Pick one known-good server and write its configuration as code (Ansible playbooks, later packaged as reusable **roles**, topic 35: packages, config files, users, services).
3. Run it first in **check + diff mode** (dry run that shows what would change) against all 40 to *see* the drift without changing anything.
4. Review the differences: some are bugs to fix, some are real needs to add to the code.
5. Apply in small batches (e.g. 5 servers at a time), checking health after each batch.
6. Stop manual changes: all changes go through Git + PR, and a scheduled check run reports new drift.

**Likely follow-ups:**
- *Why not just rebuild all 40 from an image?* That's also good (immutable infrastructure). You still need the configuration written down to *build* the image, and CM is often how you do that.
- *How do you know a re-run is safe?* Idempotent tasks: the second run reports `changed=0`.
