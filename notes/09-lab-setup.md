# 09 · Lab Setup: Managed Nodes in Docker

> **Level:** ★ foundation
>
> **You will learn:** how to build a free, disposable lab of **managed nodes** as Docker containers that behave like small servers (real **systemd**, real **sshd**, a sudo-enabled `ansible` user, Python), on AlmaLinux 10 and Ubuntu 24.04; the Compose file that starts them with fixed IP addresses; an SSH config so you (and Ansible) can log in with your key; a reset script that rebuilds everything in seconds; and the VM alternative.
>
> **You need:** [08 · Installing Ansible](08-installing-ansible.md), [05 · SSH recap](05-linux-recap-ssh.md) (the `~/.ssh/ansible_key` you made there), Docker Engine in WSL2 with systemd enabled ([Docker topic 04](../../Docker/notes/04-installing-docker.md)), Compose basics ([Docker topic 35](../../Docker/notes/35-compose-basics.md)).
>
> **Part 1 · Setup and first contact** · Next: [10 · Inventory basics](10-inventory-basics.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Ansible needs servers to manage. Instead of paying for VMs, you run **4 containers that pretend to be servers**: each boots systemd (so services work), runs sshd (so Ansible can log in), and has a user `ansible` with passwordless sudo and your public key.

**Picture it.** A **flight simulator**: not a real plane, but every switch and gauge works the same way, and a crash costs nothing. Press reset and you're back on the runway.

**The one thing to know.** The lab you'll use in every topic from now on:

| Name | OS | IP | Group (topic 10) |
|---|---|---|---|
| `web1` | AlmaLinux 10 | 172.30.0.11 | web |
| `web2` | Ubuntu 24.04 | 172.30.0.12 | web |
| `db1` | AlmaLinux 10 | 172.30.0.21 | db |
| `lb1` | Ubuntu 24.04 | 172.30.0.31 | lb |
| `bare1` (optional) | Ubuntu 24.04, **no Python** | 172.30.0.41 | (for topic 07 practice) |

```bash
cd ~/ansible-lab/nodes && ./reset.sh        # build + start everything, wait for SSH
ssh ansible@172.30.0.11 'hostname; sudo whoami'   # → web1, root
```

**Try it now:** follow the Hands-on lab below once. After that, `./reset.sh` gives you a clean lab any time.

**You've got it if you can:** SSH to all four nodes with no password and run `sudo whoami` on each.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

A small set of files in `~/ansible-lab/nodes/`:

```
~/ansible-lab/
├── requirements.txt          (topic 08)
└── nodes/
    ├── ansible_key.pub       copy of your public key, baked into the images
    ├── Dockerfile.alma       AlmaLinux 10 + systemd + sshd + sudo + python3
    ├── Dockerfile.ubuntu     Ubuntu 24.04 + the same
    ├── compose.yaml          the 4 (+1) nodes, fixed IPs on network 172.30.0.0/24
    └── reset.sh              rebuild, restart, wait for SSH
```

Plus an SSH config block in `~/.ssh/config` for the lab's IP range.

### Why containers *can* act as servers here

A normal container runs **one** process (your app) as PID 1 ([Docker topic 12](../../Docker/notes/12-pid1-signals-and-shutdown.md)). A server runs **systemd** as PID 1, which starts sshd, journald and everything else. We run systemd as PID 1 inside the container. That needs:

| Setting | Why |
|---|---|
| `CMD ["/usr/sbin/init"]` (or `/lib/systemd/systemd`) | Start systemd as PID 1 |
| `privileged: true` | systemd needs to manage cgroups and mounts. ⚠️ **Lab only**: a privileged container is effectively root on your WSL2 host |
| `cgroup: host` + `/sys/fs/cgroup` mounted read-write | systemd needs to create its cgroup tree ([Linux 097](../../linux-mastery-rhel/notes/097-namespaces-and-cgroups.md)) |
| `tmpfs` on `/run`, `/run/lock`, `/tmp` | systemd expects these to be fresh, memory-backed filesystems at boot |
| `STOPSIGNAL SIGRTMIN+3` | The signal that tells systemd "shut down cleanly" (instead of SIGTERM) |

This is the same pattern Molecule (the role-testing tool, topic 51) uses to test roles in containers.

### What containers can't do well (use a VM instead)

| Lab topic | Problem in a container | Use |
|---|---|---|
| Kernel settings (`sysctl` for some keys), kernel modules | Shared kernel with the host | VM |
| Disks, partitions, LVM | No real block devices | VM |
| Firewalls (`firewalld`, `ufw`) | Partly works with `privileged`, but can touch the host's netfilter | VM if anything looks odd |
| SELinux enforcing | Containers on an Ubuntu (WSL2) host have no SELinux | VM (AlmaLinux) |
| `ansible.builtin.reboot` | "Reboot" = container stops | Works if `restart: unless-stopped` brings it back, but timing differs |

Those topics say so and give the VM steps.

---

## Why it exists

You need **several** targets, of **two** distro families, that you can **break and rebuild** freely. Cloud VMs cost money and take minutes; local VMs take gigabytes and minutes. Containers take seconds and megabytes. For 90% of Ansible learning (users, packages, files, templates, services, roles, rolling updates), they behave exactly like servers.

---

## Analogy

**A model railway** of your production network. Four little stations (nodes) on a track (Docker network), each with a working signal box (sshd) and a stationmaster who obeys you (passwordless sudo). You can derail trains on purpose, then pick everything up and put it back in place with one button (`reset.sh`).

---

## How it works inside

What happens when you run `./reset.sh`:

```
reset.sh
 ├─ docker compose down            stop + remove old node containers (all changes lost, by design)
 ├─ docker compose up -d --build   build images if Dockerfiles changed, start 4 containers
 │     each container:
 │       PID 1 = systemd
 │        ├─ sshd-keygen / ssh-keygen -A  → NEW host keys (every reset = "new servers")
 │        ├─ sshd listening on :22
 │        └─ systemd-journald, etc.
 ├─ empty ~/.ssh/known_hosts_lab   forget old host keys, so new ones are accepted (accept-new)
 └─ loop: ssh -o BatchMode=yes ansible@IP true   until each node answers
```

And the network path from Ansible to a node:

```
 WSL2 Ubuntu (control node)                Docker bridge "ansible-lab" 172.30.0.0/24
 ┌─────────────────────┐                   ┌──────────────────────────────────────────┐
 │ ansible / ssh       │── 172.30.0.1 ───▶ │ web1 .11   web2 .12   db1 .21   lb1 .31   │
 │ ~/.ssh/ansible_key  │   (bridge gw)     │  sshd:22   sshd:22    sshd:22   sshd:22   │
 └─────────────────────┘                   └──────────────────────────────────────────┘
```

On Docker Engine **inside WSL2**, the WSL2 Linux host can reach container IPs directly, just like a server on a LAN (Local Area Network). No port publishing needed. (With Docker **Desktop**, container IPs aren't reachable from your shell; that's one reason your Docker notes install Docker Engine in WSL2.)

---

## Code / commands

### `Dockerfile.alma`

```dockerfile
FROM almalinux:10
RUN dnf -y install systemd openssh-server sudo python3 procps-ng iproute hostname \
 && dnf clean all \
 && systemctl enable sshd \
 && rm -f /etc/ssh/ssh_host_*
RUN useradd -m -s /bin/bash ansible \
 && usermod -p '*' ansible \
 && echo 'ansible ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/ansible \
 && chmod 0440 /etc/sudoers.d/ansible \
 && install -d -m 0700 -o ansible -g ansible /home/ansible/.ssh
COPY --chown=ansible:ansible --chmod=0600 ansible_key.pub /home/ansible/.ssh/authorized_keys
STOPSIGNAL SIGRTMIN+3
CMD ["/usr/sbin/init"]
```

- `systemd openssh-server sudo python3`: what a minimal server needs for Ansible: init, SSH, privilege escalation, Python (topic 07).
- `systemctl enable sshd`: works at build time (it only creates symlinks), so sshd starts at boot.
- `rm -f /etc/ssh/ssh_host_*`: delete host keys made during the build, so **each container generates its own** at boot (AlmaLinux's `sshd-keygen` units do that). Otherwise all nodes from this image would share one identity.
- `usermod -p '*'`: no usable password, but the account isn't "locked", so key login works.
- `/etc/sudoers.d/ansible` with `NOPASSWD: ALL`, mode `0440`: the standard automation-user rule from topic 06 (topic 19 discusses tightening it).
- `COPY --chmod=0600`: your public key becomes `authorized_keys` with the permissions sshd insists on (topic 05).

### `Dockerfile.ubuntu`

```dockerfile
FROM ubuntu:24.04
ARG PYTHON=python3
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update \
 && apt-get install -y --no-install-recommends systemd systemd-sysv openssh-server sudo iproute2 procps ${PYTHON} \
 && apt-get clean \
 && rm -f /etc/ssh/ssh_host_* \
 && systemctl disable ssh.socket \
 && systemctl enable ssh.service \
 && mkdir -p /etc/systemd/system/ssh.service.d \
 && printf '[Service]\nExecStartPre=\nExecStartPre=/usr/bin/ssh-keygen -A\nExecStartPre=/usr/sbin/sshd -t\n' \
      > /etc/systemd/system/ssh.service.d/10-hostkeys.conf
RUN useradd -m -s /bin/bash ansible \
 && usermod -p '*' ansible \
 && echo 'ansible ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/ansible \
 && chmod 0440 /etc/sudoers.d/ansible \
 && install -d -m 0700 -o ansible -g ansible /home/ansible/.ssh
COPY --chown=ansible:ansible --chmod=0600 ansible_key.pub /home/ansible/.ssh/authorized_keys
STOPSIGNAL SIGRTMIN+3
CMD ["/lib/systemd/systemd"]
```

- `ARG PYTHON=python3`: build argument. The `bare1` node builds with `PYTHON=""`, so it has **no Python** (for practising topic 07's `raw` bootstrap).
- `DEBIAN_FRONTEND=noninteractive`: stop apt from asking questions during the build.
- `apt-get clean` deletes downloaded `.deb` files but we **keep** `/var/lib/apt/lists`, so the nodes can install packages straight away (like a real server). Production images often delete the lists to save space; then the first Ansible task must refresh the cache (`update_cache: true`, topic 17).
- `systemd-sysv`: provides `/sbin/init` and friends on Ubuntu.
- Ubuntu 24.04 starts SSH through **socket activation** (`ssh.socket` starts `ssh.service` on the first connection). We switch to a normally running `ssh.service`, which is simpler to reason about.
- The drop-in file (`ssh.service.d/10-hostkeys.conf`) generates host keys at boot (`ssh-keygen -A`) before the config test. The empty `ExecStartPre=` first **clears** the package's own list ([Linux 098](../../linux-mastery-rhel/notes/098-systemd-unit-files.md) explains drop-ins).

### `compose.yaml`

```yaml
name: ansible-lab

x-node: &node                      # shared settings for every node (YAML anchor, topic 04)
  privileged: true
  cgroup: host
  volumes:
    - /sys/fs/cgroup:/sys/fs/cgroup:rw
  tmpfs:
    - /run
    - /run/lock
    - /tmp
  restart: unless-stopped

services:
  web1:
    <<: *node
    build: { context: ., dockerfile: Dockerfile.alma }
    image: ansible-lab/alma10
    hostname: web1
    networks: { lab: { ipv4_address: 172.30.0.11 } }

  web2:
    <<: *node
    build: { context: ., dockerfile: Dockerfile.ubuntu }
    image: ansible-lab/ubuntu2404
    hostname: web2
    networks: { lab: { ipv4_address: 172.30.0.12 } }

  db1:
    <<: *node
    build: { context: ., dockerfile: Dockerfile.alma }
    image: ansible-lab/alma10
    hostname: db1
    networks: { lab: { ipv4_address: 172.30.0.21 } }

  lb1:
    <<: *node
    build: { context: ., dockerfile: Dockerfile.ubuntu }
    image: ansible-lab/ubuntu2404
    hostname: lb1
    networks: { lab: { ipv4_address: 172.30.0.31 } }

  bare1:
    <<: *node
    profiles: [bare]               # only started with: docker compose --profile bare up -d
    build:
      context: .
      dockerfile: Dockerfile.ubuntu
      args: { PYTHON: "" }
    image: ansible-lab/ubuntu2404-bare
    hostname: bare1
    networks: { lab: { ipv4_address: 172.30.0.41 } }

networks:
  lab:
    name: ansible-lab
    ipam:
      config:
        - subnet: 172.30.0.0/24
```

- `x-node: &node` + `<<: *node`: define the systemd settings once, merge them into every service. Compose ignores top-level keys starting with `x-`.
- `ipv4_address`: fixed IPs, so your inventory (topic 10) never changes.
- `restart: unless-stopped`: nodes come back after a "reboot" or a WSL restart.
- `profiles: [bare]`: `bare1` starts only when you ask for it.

### `reset.sh`

```bash
#!/usr/bin/env bash
# Rebuild the lab from scratch: every node is a brand-new "server".
set -euo pipefail
cd "$(dirname "$0")"

docker compose --profile bare down --remove-orphans
docker compose up -d --build
: > ~/.ssh/known_hosts_lab                 # new containers = new host keys

echo "Waiting for SSH on the nodes..."
for ip in 172.30.0.11 172.30.0.12 172.30.0.21 172.30.0.31; do
  for _ in $(seq 1 60); do
    ssh -o BatchMode=yes -o ConnectTimeout=2 "ansible@$ip" true 2>/dev/null && break
    sleep 1
  done
  printf '  %-12s %s\n' "$ip" "$(ssh -o BatchMode=yes "ansible@$ip" hostname 2>/dev/null || echo 'NOT READY')"
done
```

- `set -euo pipefail`: stop on errors ([Linux 045](../../linux-mastery-rhel/notes/045-robust-scripts.md)).
- `--profile bare down`: also removes `bare1` if you started it. `up` without the profile starts only the four main nodes; add `bare1` separately when you need it (lab step 7).
- `: > file`: empty the lab's known_hosts file.
- `BatchMode=yes`: never prompt (fail instead), like Ansible in CI.

### The SSH config block (`~/.ssh/config`)

```
Host 172.30.0.*
    User ansible
    IdentityFile ~/.ssh/ansible_key
    IdentitiesOnly yes
    UserKnownHostsFile ~/.ssh/known_hosts_lab
    StrictHostKeyChecking accept-new
```

- Applies to every lab IP.
- `IdentitiesOnly yes`: offer only this key (avoids "Too many authentication failures").
- `UserKnownHostsFile ~/.ssh/known_hosts_lab`: lab host keys live in their **own** file, so resets never pollute your real `known_hosts`.
- `StrictHostKeyChecking accept-new`: trust a node the first time, refuse if its key later changes. Safe enough for a lab you rebuild yourself (topic 05).

### Sample output

```bash
./reset.sh
```
```
[+] Running 5/5
 ✔ Network ansible-lab  Created
 ✔ Container ansible-lab-web1-1  Started
 ✔ Container ansible-lab-web2-1  Started
 ✔ Container ansible-lab-db1-1   Started
 ✔ Container ansible-lab-lb1-1   Started
Waiting for SSH on the nodes...
  172.30.0.11  web1
  172.30.0.12  web2
  172.30.0.21  db1
  172.30.0.31  lb1
```

```bash
docker compose ps --format 'table {{.Name}}\t{{.Image}}\t{{.Status}}'
```
```
NAME                 IMAGE                    STATUS
ansible-lab-db1-1    ansible-lab/alma10       Up 40 seconds
ansible-lab-lb1-1    ansible-lab/ubuntu2404   Up 40 seconds
ansible-lab-web1-1   ansible-lab/alma10       Up 40 seconds
ansible-lab-web2-1   ansible-lab/ubuntu2404   Up 40 seconds
```

Check that a node really boots like a server:

```bash
ssh ansible@172.30.0.11 'ps -p 1 -o comm=; systemctl is-system-running; systemctl is-active sshd'
```
```
systemd
running
active
```

- `systemd`: PID 1 is systemd, not a shell.
- `running`: all units started fine. (`degraded` means some unit failed; usually harmless in a container. See which with `systemctl --failed`.)

---

## Hands-on lab

1. **Prerequisites.** Docker Engine works in WSL2 (`docker run --rm hello-world`), systemd is enabled in WSL2 (`ps -p 1 -o comm=` prints `systemd`), and the host uses cgroup v2:

   ```bash
   stat -fc %T /sys/fs/cgroup        # must print: cgroup2fs
   ```

2. **Key.** If you skipped topic 05: `ssh-keygen -t ed25519 -f ~/.ssh/ansible_key -C ansible-lab -N ""`.

3. **Files.** Create `~/ansible-lab/nodes/`, copy your public key in, and create the four files above:

   ```bash
   mkdir -p ~/ansible-lab/nodes && cd ~/ansible-lab/nodes
   cp ~/.ssh/ansible_key.pub .
   # create Dockerfile.alma, Dockerfile.ubuntu, compose.yaml, reset.sh (copy from above)
   chmod +x reset.sh
   ```

4. **SSH config.** Append the `Host 172.30.0.*` block to `~/.ssh/config` and `chmod 600 ~/.ssh/config`.

5. **Start.** `./reset.sh`. The first build downloads images and packages (a few minutes); later resets take seconds.

6. **Verify each node:**

   ```bash
   for ip in 172.30.0.11 172.30.0.12 172.30.0.21 172.30.0.31; do
     ssh ansible@$ip 'echo "$(hostname): $(. /etc/os-release; echo $PRETTY_NAME), python=$(python3 -V 2>&1), sudo=$(sudo -n whoami)"'
   done
   ```
   ```
   web1: AlmaLinux 10.1 (Heliotrope Lion), python=Python 3.12.9, sudo=root
   web2: Ubuntu 24.04.3 LTS, python=Python 3.12.3, sudo=root
   db1: AlmaLinux 10.1 (Heliotrope Lion), python=Python 3.12.9, sudo=root
   lb1: Ubuntu 24.04.3 LTS, python=Python 3.12.3, sudo=root
   ```

7. **The bare node** (no Python):

   ```bash
   docker compose --profile bare up -d --build bare1
   ssh ansible@172.30.0.41 'python3 -V || echo NO PYTHON'
   ```

8. **Feel the reset.** On web1, break something: `ssh ansible@172.30.0.11 'sudo touch /etc/I-WAS-HERE'`. Run `./reset.sh`. Check again: the file is gone and web1 has a **new host key** (look in `~/.ssh/known_hosts_lab`).

9. **Optional: a VM node** for the topics that need a real kernel. Use the AlmaLinux VM from [Linux topic 002](../../linux-mastery-rhel/notes/002-build-your-practice-lab.md). On it, create the same `ansible` user, sudoers file and `authorized_keys` (the `RUN` lines above, run by hand with `sudo`), then note its IP for the inventory. Snapshot it before destructive labs.

**Done when:** step 6 prints four lines ending in `sudo=root`, with no password prompts.

---

## Where you'll use it

- **Every lab in these notes** from topic 10 on.
- **At work**: the same pattern is how teams test roles locally (Molecule with Docker/Podman, topic 51) and in CI.
- **Demos and interviews**: you can show a working multi-node Ansible setup on a laptop in two minutes.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| Container exits at once; `docker logs ansible-lab-web1-1` mentions cgroups | Host isn't on cgroup v2, or `/sys/fs/cgroup` not mounted rw | Check `stat -fc %T /sys/fs/cgroup` = `cgroup2fs`; enable systemd in WSL2 (`/etc/wsl.conf` → `[boot] systemd=true`, then `wsl --shutdown`) |
| `ssh: connect to host 172.30.0.11 port 22: No route to host` | Using Docker **Desktop** (container IPs not reachable) or the network wasn't created | Use Docker Engine in WSL2; `docker network inspect ansible-lab` |
| `Connection refused` right after start | sshd not up yet | Wait; `reset.sh` waits for you. Check `docker exec ansible-lab-web1-1 systemctl status sshd` |
| `Permission denied (publickey)` | Image built with a different/old `ansible_key.pub` | Copy the right `.pub` into `nodes/` and `./reset.sh` (the `--build` flag rebuilds) |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | Containers recreated, old keys remembered | `./reset.sh` empties `known_hosts_lab`; or `: > ~/.ssh/known_hosts_lab` |
| `systemctl is-system-running` → `degraded` | A unit that doesn't fit containers failed (e.g. a kernel-related one) | `systemctl --failed`; usually safe to ignore in the lab |
| `sudo: unable to resolve host web1` | Missing hosts entry in odd setups | Harmless warning; Compose normally sets `/etc/hosts` |
| `Pool overlaps with other one on this host` | 172.30.0.0/24 already used by another Docker network | Change the subnet and IPs consistently, or remove the other network |

---

## Quick recap

- The lab = 4 containers running **systemd + sshd + python3**, user `ansible` with **NOPASSWD sudo** and your key: `web1`/`db1` (AlmaLinux 10), `web2`/`lb1` (Ubuntu 24.04), on `172.30.0.0/24`. Optional `bare1` has no Python.
- systemd in Docker needs `privileged`, cgroup v2 mounted rw, tmpfs on `/run` and `/tmp`, and `SIGRTMIN+3` to stop. **Lab only.**
- Each node generates its **own host keys** at boot; the lab keeps them in `~/.ssh/known_hosts_lab` with `accept-new`.
- `./reset.sh` = brand-new servers in seconds. Use a VM for kernel, disk, SELinux and firewall labs.

### Interview questions

1. **Q: How would you test Ansible code locally without cloud servers?**
   A: Run containers (or small VMs) that act as servers: systemd + sshd + Python, managed over SSH, or use Molecule with the Docker/Podman driver. Rebuild them for every test so runs start from a known state.
2. **Q: What's different about running systemd inside a container?**
   A: systemd must be PID 1 and needs access to cgroups (rw cgroup v2 mount, often privileged), tmpfs for `/run`, and the right stop signal. Kernel-level features remain shared with the host.
3. **Q: Why remove host keys from a server image?**
   A: Otherwise every server built from the image has the same SSH identity, which breaks host key verification's purpose. Keys should be generated at first boot.

### Scenario question (answer out loud first, then compare)

> *"Your team bakes a golden VM image with Packer and boots 200 servers from it. Security notices all 200 have the same SSH host key fingerprint. What happened, why is it bad, and how do you fix it?"*

**Model answer:**
1. Host keys were generated during the image build and baked in, so every clone shares them.
2. It's bad because one leaked key lets an attacker impersonate **any** of the 200 servers, and host key checking can't tell them apart.
3. Fix the image: delete `/etc/ssh/ssh_host_*` (and reset `machine-id`) at the end of the build, so keys are generated at first boot (cloud-init or sshd-keygen does it).
4. Fix the running servers: regenerate host keys with Ansible (remove keys, `ssh-keygen -A`, restart sshd) in batches, then update known_hosts / SSH certificates for all clients.

**Likely follow-ups:**
- *What else must be unique per clone?* `/etc/machine-id`, hostnames, any baked-in credentials or TLS keys.
- *How would clients learn the new keys safely?* SSH host certificates signed by an internal CA, or collect fingerprints through a trusted channel (cloud console output) instead of TOFU (trust on first use).
