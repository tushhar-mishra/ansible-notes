# 57 · Project 2: Baseline + Web Stack with Roles

> **Level:** ★★★ advanced
>
> **You will build:** a complete, production-shaped configuration for the whole lab fleet (AlmaLinux 10 + Ubuntu 24.04) using roles: a **`baseline`** hardening role for every host (admin users and keys, validated sudo rules, SSH hardening drop-in with validate-then-restart handlers, sysctl and firewall that **auto-skip inside containers**, time sync), a **`postgres`** role on `db1` (install, init once, listen address, `pg_hba` rules, database and user with a **vaulted** password), the **web tier** (`nginx_web` + an app config file holding the DB connection), the **load balancer** (`lb_nginx`), all wired together by `site.yml`, verified end to end (web → DB connection, LB → web), idempotent, linted, with a Molecule test for `baseline`.
>
> **You need:** everything up to [56](56-awx-and-aap.md), especially [36 · Roles II](36-roles-advanced.md), [40 · Vault](40-ansible-vault.md), [30 · Handlers](30-handlers.md), [19 · become (unprivileged)](19-become.md), [23 · facts](23-facts-and-magic-variables.md), [51 · Molecule](51-testing-with-molecule.md), [39 · Project layout](39-project-layout.md).
>
> **Part 11 · Projects** · Next: [58 · Debugging Ansible](58-debugging-ansible.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A real environment = a **baseline** every server gets + **service roles** per group + **wiring** between them (web servers need the DB's address and password; the LB needs the web servers). All values come from inventory variables and vault; roles stay generic.

**Picture it.** Building a small town: every house gets the same foundations, locks and smoke alarms (baseline); then the bank (database), shops (web) and the roundabout that directs traffic (load balancer) are built from standard plans, and the roads between them come from the town map (inventory).

**The one thing to know.**

```yaml
# playbooks/site.yml
- ansible.builtin.import_playbook: baseline.yml     # all hosts
- ansible.builtin.import_playbook: db.yml           # db: postgres
- ansible.builtin.import_playbook: web.yml          # web: nginx_web + app config (needs db facts/vars)
- ansible.builtin.import_playbook: lb.yml           # lb: lb_nginx (needs web hosts)
- ansible.builtin.import_playbook: verify.yml       # end-to-end checks
```

**Try it now:** after building the roles, `./nodes/reset.sh` (fresh lab) and run `site.yml` once from zero: it must converge without errors, then `changed=0` on the second run.

**You've got it if you can:** rebuild the whole fleet from nothing with one command, and explain where every value comes from.

> 📖 **That's the core.** On a first pass, follow the build steps once. Everything below is the **full project**.

---

## What it is

### Requirements

| # | Requirement |
|---|---|
| R1 | Every host: admin users with SSH keys; sudo only via validated drop-ins; root login and password login disabled; safe kernel network settings; time sync; firewall allowing only needed ports |
| R2 | Things that would affect the **WSL2 host** when run in a privileged container (sysctl apply, firewall, time sync) are **skipped automatically** in containers and applied on VMs/real servers |
| R3 | `db1`: PostgreSQL installed and initialised **once**, listening on its IP, accepting only the web servers, with database `shop` and user `shop` whose password is **vaulted** |
| R4 | Web servers: nginx site (topic 36 role) + `/etc/shop/app.env` (mode `0600`) with the DB host, name, user and password |
| R5 | `lb1` balances across the web servers (topic 29/30 role) |
| R6 | An end-to-end check proves: each web server can log in to the DB; the LB serves pages |
| R7 | Second run of `site.yml`: `changed=0`; `ansible-lint` production profile passes; `baseline` has a Molecule test |

### The repository after this project

```
~/infra/
├── inventories/lab/
│   ├── hosts.yml
│   ├── group_vars/
│   │   ├── all/{main.yml, vault.yml, baseline.yml}
│   │   ├── db.yml
│   │   └── web.yml
│   └── host_vars/
├── playbooks/{site,baseline,db,web,lb,verify}.yml
└── roles/
    ├── baseline/      ← new
    ├── postgres/      ← new
    ├── nginx_web/     (topic 36)
    ├── shop_app/      (topic 49, unchanged)
    ├── lb_nginx/      (topics 29/30/39)
    └── lb_member/     (topic 46)
```

---

## Why it exists

Topics 14–56 each taught one tool. Real work combines them: roles with defaults and argument specs, OS differences through vars files, secrets through vault, handlers that validate before restarting, cross-host data through inventory/facts, and verification at the end. Building one coherent environment from scratch is also exactly what interviewers mean by "tell me about an Ansible project you've built".

---

## Analogy

**Opening a new branch of a restaurant chain from the franchise manual.** Every branch gets the same safety fit-out (baseline). The kitchen (database) is set up from its manual, with the safe's combination delivered sealed (vault). The dining room (web) gets the kitchen's address and the pass-through code. The host stand (load balancer) is told which tables exist. On opening day, an inspector checks that orders actually reach the kitchen (end-to-end verify).

---

## How it works inside

```
 site.yml
  ├─ baseline.yml  hosts: all          → baseline role (users, sudo, sshd, sysctl*, firewall*, time*)   (* VMs only)
  ├─ db.yml        hosts: db           → postgres role (install, initdb once, conf, hba for web IPs, db/user)
  ├─ web.yml       hosts: web          → nginx_web role + app.env template (db host = hostvars[db1].ansible_host)
  ├─ lb.yml        hosts: lb           → lb_nginx role (backends = groups['web'])
  └─ verify.yml    hosts: web, lb      → psql login from each web server; curl through the LB
 values: inventories/lab/group_vars (+ vault.yml) → roles' defaults overridden per environment
```

---

## Code / commands

### 1. Inventory values

`inventories/lab/group_vars/all/baseline.yml`:

```yaml
---
baseline_admins:
  - name: alice
    uid: 2001
    key: "{{ lookup('ansible.builtin.file', playbook_dir ~ '/../nodes/ansible_key.pub') }}"
    sudo: true
    password_hash: "{{ vault_alice_password_hash }}"   # sudo asks for a password, so she needs one
  - name: bob
    uid: 2002
    key: "{{ lookup('ansible.builtin.file', playbook_dir ~ '/../nodes/ansible_key.pub') }}"
    sudo: false
baseline_sshd_settings:
  PermitRootLogin: "no"
  PasswordAuthentication: "no"
  MaxAuthTries: "3"
  X11Forwarding: "no"
baseline_firewall_allowed_tcp: [22]
baseline_ntp_servers: [pool.ntp.org]
```

(For the lab, copy `~/ansible-lab/nodes/ansible_key.pub` to `~/infra/nodes/ansible_key.pub` so the lookup finds it. Real teams store each admin's own public key.)

`inventories/lab/group_vars/db.yml`:

```yaml
---
postgres_listen_address: "{{ ansible_host }}"
postgres_allowed_clients: "{{ groups['web'] | map('extract', hostvars, 'ansible_host') | list }}"
postgres_users:
  - name: shop
    password: "{{ vault_postgres_shop_password }}"
postgres_databases:
  - name: shop
    owner: shop
baseline_firewall_allowed_tcp: [22, 5432]
```

`inventories/lab/group_vars/web.yml` (add to what's there):

```yaml
---
http_port: 8080
baseline_firewall_allowed_tcp: [22, 80, 8080, 8081]
shop_db_host: "{{ hostvars[groups['db'][0]]['ansible_host'] }}"
shop_db_name: shop
shop_db_user: shop
shop_db_password: "{{ vault_postgres_shop_password }}"
```

Add the secret to the vault:

```bash
ansible-vault edit inventories/lab/group_vars/all/vault.yml
# add:  vault_postgres_shop_password: "Shop-DB-Lab-2026"
```

- The DB password exists **once** (vault) and is referenced by both the DB role (to create the user) and the web config (to connect).
- `postgres_allowed_clients` is computed from the inventory: add a web server and the DB allows it on the next run.

### 2. The `baseline` role

```
roles/baseline/
├── defaults/main.yml
├── meta/{main.yml, argument_specs.yml}
├── tasks/{main.yml, users.yml, ssh.yml, kernel.yml, firewall.yml, time.yml}
├── handlers/main.yml
├── templates/{sshd-baseline.conf.j2, sudoers.j2, chrony.conf.j2}
└── vars/{RedHat.yml, Debian.yml}
```

`defaults/main.yml`:

```yaml
---
baseline_admins: []
baseline_sshd_settings:
  PermitRootLogin: "no"
  PasswordAuthentication: "no"
baseline_sysctl:
  net.ipv4.conf.all.accept_redirects: 0
  net.ipv4.conf.all.send_redirects: 0
  net.ipv4.tcp_syncookies: 1
baseline_firewall_allowed_tcp: [22]
baseline_ntp_servers: [pool.ntp.org]
baseline_packages: [curl, tree, vim, rsync]
# Auto-detect containers: never touch the shared kernel, firewall or clock from inside one
baseline_is_container: "{{ ansible_facts['virtualization_type'] | default('') in ['docker', 'container', 'podman', 'containerd'] }}"
baseline_manage_kernel: "{{ not baseline_is_container }}"
baseline_manage_firewall: "{{ not baseline_is_container }}"
baseline_manage_time: "{{ not baseline_is_container }}"
```

`vars/RedHat.yml`:

```yaml
---
__baseline_admin_group: wheel
__baseline_ssh_service: sshd
__baseline_time_package: chrony
__baseline_time_service: chronyd
__baseline_chrony_conf: /etc/chrony.conf
__baseline_firewall: firewalld
```

`vars/Debian.yml`:

```yaml
---
__baseline_admin_group: sudo
__baseline_ssh_service: ssh
__baseline_time_package: chrony
__baseline_time_service: chrony
__baseline_chrony_conf: /etc/chrony/chrony.conf
__baseline_firewall: ufw
```

`tasks/main.yml`:

```yaml
---
- name: OS-specific variables
  ansible.builtin.include_vars: "{{ ansible_facts['os_family'] }}.yml"
  tags: [always]

- name: Base packages are installed
  ansible.builtin.package:
    name: "{{ baseline_packages }}"
    state: present
  tags: [baseline, baseline_packages]

- name: Users
  ansible.builtin.import_tasks: users.yml
  tags: [baseline, baseline_users]

- name: SSH
  ansible.builtin.import_tasks: ssh.yml
  tags: [baseline, baseline_ssh]

- name: Kernel settings
  ansible.builtin.include_tasks: kernel.yml
  when: baseline_manage_kernel | bool
  tags: [baseline, baseline_kernel]

- name: Firewall
  ansible.builtin.include_tasks: firewall.yml
  when: baseline_manage_firewall | bool
  tags: [baseline, baseline_firewall]

- name: Time sync
  ansible.builtin.include_tasks: time.yml
  when: baseline_manage_time | bool
  tags: [baseline, baseline_time]
```

- `include_tasks` + `when` (evaluated once, topic 34): on containers these files aren't even loaded.

`tasks/users.yml`:

```yaml
---
- name: Admin users exist
  ansible.builtin.user:
    name: "{{ item.name }}"
    uid: "{{ item.uid | default(omit) }}"
    shell: /bin/bash
    groups: "{{ [__baseline_admin_group] if item.sudo | default(false) else omit }}"
    append: true
    password: "{{ item.password_hash | default(omit) }}"   # a SHA-512 crypt hash, never plaintext
    state: "{{ item.state | default('present') }}"
    remove: "{{ (item.state | default('present')) == 'absent' }}"
  loop: "{{ baseline_admins }}"
  loop_control: { label: "{{ item.name }}" }

- name: Admin SSH keys
  ansible.posix.authorized_key:
    user: "{{ item.name }}"
    key: "{{ item.key }}"
    state: present
  loop: "{{ baseline_admins | selectattr('key', 'defined') | list }}"
  loop_control: { label: "{{ item.name }}" }
  when: (item.state | default('present')) == 'present'

- name: Sudo drop-in for admins (validated)
  ansible.builtin.template:
    src: sudoers.j2
    dest: /etc/sudoers.d/baseline-admins
    owner: root
    group: root
    mode: "0440"
    validate: /usr/sbin/visudo -cf %s
```

`templates/sudoers.j2`:

```jinja
{{ ansible_managed | comment }}
{% for u in baseline_admins if (u.sudo | default(false)) and (u.state | default('present')) == 'present' %}
{{ u.name }} ALL=(ALL) ALL
{% endfor %}
```

- Sudo rules come from data; removing `sudo: true` from a user removes their line.
- ⚠️ **Corrected:** `ALL=(ALL) ALL` makes sudo ask for the **user's password**. An earlier version created admins with SSH keys only and no password, so they could log in but **never** use sudo. Each sudo admin now gets a vaulted `password_hash` (create one with `mkpasswd -m sha-512` or `openssl passwd -6`, then `ansible-vault encrypt_string` it as `vault_alice_password_hash`). The alternatives are a conscious `NOPASSWD:` (effectively root on key compromise, topic 19), or short-lived SSH certificates plus a PAM module such as `pam_ssh_agent_auth` (topic 60).

`tasks/ssh.yml`:

```yaml
---
- name: SSH hardening drop-in
  ansible.builtin.template:
    src: sshd-baseline.conf.j2
    dest: /etc/ssh/sshd_config.d/40-baseline.conf
    owner: root
    group: root
    mode: "0644"
    validate: /usr/sbin/sshd -t -f %s
  notify: baseline sshd changed
```

`templates/sshd-baseline.conf.j2`:

```jinja
{{ ansible_managed | comment }}
{% for key, value in baseline_sshd_settings | dictsort %}
{{ key }} {{ value }}
{% endfor %}
```

- `40-` sorts **before** distro drop-ins like `50-redhat.conf`; sshd uses the **first** value it reads for most settings, so ours win.
- `dictsort`: stable order → stable file → idempotent.

`tasks/kernel.yml`:

```yaml
---
- name: Kernel settings (persisted and applied)
  ansible.posix.sysctl:
    name: "{{ item.key }}"
    value: "{{ item.value }}"
    sysctl_file: /etc/sysctl.d/90-baseline.conf
    sysctl_set: true
    state: present
  loop: "{{ baseline_sysctl | dict2items }}"
```

`tasks/firewall.yml`:

```yaml
---
- name: firewalld (RedHat family)
  when: __baseline_firewall == 'firewalld'
  block:
    - name: firewalld installed and running
      ansible.builtin.package: { name: firewalld, state: present }
    - name: firewalld service
      ansible.builtin.systemd_service: { name: firewalld, state: started, enabled: true }
    - name: Allowed TCP ports
      ansible.posix.firewalld:
        port: "{{ item }}/tcp"
        permanent: true
        immediate: true
        state: enabled
      loop: "{{ baseline_firewall_allowed_tcp }}"

- name: ufw (Debian family)
  when: __baseline_firewall == 'ufw'
  block:
    - name: ufw installed
      ansible.builtin.package: { name: ufw, state: present }
    - name: Allowed TCP ports
      community.general.ufw: { rule: allow, port: "{{ item }}", proto: tcp }
      loop: "{{ baseline_firewall_allowed_tcp }}"
    - name: Default deny incoming, then enable
      community.general.ufw: { state: enabled, policy: deny, direction: incoming }
```

- Ports are allowed **before** the default-deny/enable step, so SSH (22) is never cut.

`tasks/time.yml`:

```yaml
---
- name: chrony installed
  ansible.builtin.package: { name: "{{ __baseline_time_package }}", state: present }
- name: chrony configured
  ansible.builtin.template:
    src: chrony.conf.j2
    dest: "{{ __baseline_chrony_conf }}"
    mode: "0644"
  notify: baseline restart chrony
- name: chrony running
  ansible.builtin.systemd_service: { name: "{{ __baseline_time_service }}", state: started, enabled: true }
```

`templates/chrony.conf.j2`:

```jinja
{{ ansible_managed | comment }}
{% for s in baseline_ntp_servers %}
pool {{ s }} iburst
{% endfor %}
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
```

`handlers/main.yml`:

```yaml
---
- name: Check sshd config
  ansible.builtin.command: /usr/sbin/sshd -t
  changed_when: false
  listen: baseline sshd changed

- name: Restart sshd
  ansible.builtin.systemd_service:
    name: "{{ __baseline_ssh_service }}"
    state: restarted
  listen: baseline sshd changed

- name: baseline restart chrony
  ansible.builtin.systemd_service:
    name: "{{ __baseline_time_service }}"
    state: restarted
```

### 3. The `postgres` role (AlmaLinux)

```
roles/postgres/
├── defaults/main.yml
├── tasks/main.yml
└── handlers/main.yml
```

`defaults/main.yml`:

```yaml
---
postgres_listen_address: localhost
postgres_port: 5432
postgres_allowed_clients: []
postgres_databases: []
postgres_users: []
postgres_data_dir: /var/lib/pgsql/data
```

`tasks/main.yml`:

```yaml
---
- name: Supported platform
  ansible.builtin.assert:
    that: ansible_facts['os_family'] == 'RedHat'
    fail_msg: "This lab role supports the RedHat family only"
    quiet: true

- name: Packages (server, client, Python driver for the modules, acl for become_user)
  ansible.builtin.package:
    name: [postgresql-server, postgresql, python3-psycopg2, acl]
    state: present

- name: Cluster is initialised (once)
  ansible.builtin.command: postgresql-setup --initdb
  args:
    creates: "{{ postgres_data_dir }}/PG_VERSION"

- name: Listen address
  ansible.builtin.lineinfile:
    path: "{{ postgres_data_dir }}/postgresql.conf"
    regexp: '^#?\s*listen_addresses\s*='
    line: "listen_addresses = 'localhost,{{ postgres_listen_address }}'"
  notify: Restart postgresql

- name: Client access for each allowed address
  community.postgresql.postgresql_pg_hba:
    dest: "{{ postgres_data_dir }}/pg_hba.conf"
    contype: host
    databases: all
    users: all
    source: "{{ item }}/32"
    method: scram-sha-256
  loop: "{{ postgres_allowed_clients }}"
  notify: Reload postgresql

- name: PostgreSQL is running and enabled
  ansible.builtin.systemd_service:
    name: postgresql
    state: started
    enabled: true

- name: Apply config changes before creating objects
  ansible.builtin.meta: flush_handlers

- name: Users exist with their passwords
  community.postgresql.postgresql_user:
    name: "{{ item.name }}"
    password: "{{ item.password }}"
    state: present
  loop: "{{ postgres_users }}"
  loop_control: { label: "{{ item.name }}" }
  become: true
  become_user: postgres
  no_log: true

- name: Databases exist, owned by their app user
  community.postgresql.postgresql_db:
    name: "{{ item.name }}"
    owner: "{{ item.owner | default(omit) }}"
    state: present
  loop: "{{ postgres_databases }}"
  become: true
  become_user: postgres
```

- Users first, then databases **owned** by them: the owner has full rights in its database (since PostgreSQL 15, ordinary users can't create objects in another owner's `public` schema).

- `creates:` makes `initdb` run only once (topic 27).
- `become_user: postgres` uses the database's local peer authentication; `acl` is installed so becoming an unprivileged user works (topic 19).
- `no_log` hides the password; vault values are also masked by 2.22.
- `community.postgresql` must be in `collections/requirements.yml` (add it).

`handlers/main.yml`:

```yaml
---
- name: Restart postgresql
  ansible.builtin.systemd_service: { name: postgresql, state: restarted }
- name: Reload postgresql
  ansible.builtin.systemd_service: { name: postgresql, state: reloaded }
```

### 4. The web app config

Keep `nginx_web` generic: the app-specific wiring (DB connection file, client tools) goes in the web playbook (or, as it grows, its own small `shop_config` role):

`playbooks/web.yml`:

```yaml
---
- name: Web tier
  hosts: web
  become: true
  roles:
    - nginx_web
  tasks:
    - name: App config directory
      ansible.builtin.file:
        path: /etc/shop
        state: directory
        mode: "0750"

    - name: App environment file with DB connection (secret)
      ansible.builtin.copy:
        dest: /etc/shop/app.env
        content: |
          DB_HOST={{ shop_db_host }}
          DB_NAME={{ shop_db_name }}
          DB_USER={{ shop_db_user }}
          DB_PASSWORD={{ shop_db_password }}
        owner: root
        group: root
        mode: "0600"
      no_log: true

    - name: PostgreSQL client for connection checks
      ansible.builtin.package:
        name: "{{ 'postgresql' if ansible_facts['os_family'] == 'RedHat' else 'postgresql-client' }}"
        state: present
```

### 5. The playbooks

`playbooks/baseline.yml`:

```yaml
---
- name: Baseline for every host
  hosts: all
  become: true
  roles:
    - baseline
```

`playbooks/db.yml`:

```yaml
---
- name: Database
  hosts: db
  become: true
  roles:
    - postgres
```

`playbooks/lb.yml` (from topic 39): `hosts: lb`, role `lb_nginx`.

`playbooks/verify.yml`:

```yaml
---
- name: End-to-end checks from the web servers
  hosts: web
  become: true
  gather_facts: false
  tasks:
    - name: Web server can log in to the database
      ansible.builtin.command: psql -h {{ shop_db_host }} -U {{ shop_db_user }} -d {{ shop_db_name }} -tAc "select 1"
      environment:
        PGPASSWORD: "{{ shop_db_password }}"
      register: dbcheck
      changed_when: false
      failed_when: dbcheck.stdout | trim != '1'
      no_log: true

- name: End-to-end checks through the load balancer
  hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: The load balancer serves the site
      ansible.builtin.uri:
        url: "http://{{ hostvars[groups['lb'][0]]['ansible_host'] }}/"
        return_content: true
      register: lbpage
      until: lbpage.status == 200
      retries: 5
      delay: 2
```

- The password goes to `psql` through the **environment**, not the command line (topic 41).

`playbooks/site.yml`:

```yaml
---
- name: Gather facts on every host first (cross-host data)
  hosts: all
  gather_facts: true
  tasks: []

- ansible.builtin.import_playbook: baseline.yml
- ansible.builtin.import_playbook: db.yml
- ansible.builtin.import_playbook: web.yml
- ansible.builtin.import_playbook: lb.yml
- ansible.builtin.import_playbook: verify.yml
```

### 6. Run it from zero

```bash
cd ~/ansible-lab/nodes && ./reset.sh            # brand-new fleet
cd ~/infra
ansible-galaxy collection install -r collections/requirements.yml -p ./collections    # incl. community.postgresql
ansible-lint
ansible-playbook -i inventories/lab playbooks/site.yml --check --diff     # plan (some tasks fail in check mode on a fresh fleet: expected, topic 20)
ansible-playbook -i inventories/lab playbooks/site.yml
ansible-playbook -i inventories/lab playbooks/site.yml                    # second run
```
```
PLAY RECAP *****************************************************************************
db1        : ok=21  changed=0  unreachable=0  failed=0  skipped=4  rescued=0  ignored=0
lb1        : ok=17  changed=0  unreachable=0  failed=0  skipped=3  rescued=0  ignored=0
localhost  : ok=1   changed=0  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
web1       : ok=30  changed=0  unreachable=0  failed=0  skipped=4  rescued=0  ignored=0
web2       : ok=31  changed=0  unreachable=0  failed=0  skipped=3  rescued=0  ignored=0
```

- `changed=0` everywhere on the second run (R7).
- `skipped`: kernel/firewall/time includes on containers, and per-family tasks.

Spot checks:

```bash
ssh ansible@172.30.0.11 'sudo sshd -T | grep -Ei "^(permitrootlogin|passwordauthentication|maxauthtries)"'
ssh ansible@172.30.0.11 'sudo cat /etc/sudoers.d/baseline-admins; id alice'
ssh ansible@172.30.0.21 'sudo -u postgres psql -tAc "select datname from pg_database" | grep shop'
ssh -i ~/.ssh/ansible_key alice@172.30.0.12 'sudo -n true 2>&1 | head -1'   # alice needs her password for sudo (no NOPASSWD)
```

### 7. Molecule for `baseline`

Create `roles/baseline/molecule/default/` like topic 51 (alma10 + ubuntu2404 platforms). Its `verify.yml` checks:

```yaml
- name: Verify baseline
  hosts: all
  gather_facts: false
  tasks:
    - name: Effective sshd settings
      ansible.builtin.command: sshd -T
      register: sshd_t
      changed_when: false
    - name: Root and password login are off
      ansible.builtin.assert:
        that:
          - "'permitrootlogin no' in sshd_t.stdout"
          - "'passwordauthentication no' in sshd_t.stdout"
    - name: Sudoers drop-in is valid and safe
      ansible.builtin.stat: { path: /etc/sudoers.d/baseline-admins }
      register: sudoers
      failed_when: sudoers.stat.mode != '0440'
```

Give the scenario `baseline_admins` in `provisioner.inventory.group_vars` (with a test public key), and run `molecule test`. The container auto-detection means kernel/firewall/time are skipped in Molecule too; test those parts on a VM-based scenario.

---

## Hands-on lab

1. Build both roles, the inventory values and the playbooks. Run `site.yml` from a fresh lab twice (second run `changed=0`).
2. Run `verify.yml` alone; then change the DB password in the vault **only** (re-run `site.yml`): the postgres user's password and `app.env` both update, and verify passes.
3. Add a third web server (`tf-app1` from topic 43, or a new Compose service) to `web`: run `site.yml` and confirm the DB's `pg_hba.conf` allows it and the LB lists it.
4. Remove `sudo: true` from bob... (he has none) → give it to him, run, check the sudoers file; take it away again, run, check it's gone.
5. On the VM from topic 09 (add it to the inventory as `vm1` in group `alma`): run `baseline.yml -l vm1` and see the kernel, firewall and time tasks **run** there (not a container).
6. Break the SSH template on purpose (`MaxAuthTries three` via `baseline_sshd_settings`): validation stops it; nobody is locked out.
7. Run `ansible-lint` and `molecule test` for `baseline`; fix findings.

**Done when:** one command builds the whole fleet from zero, a second run changes nothing, and verify proves the pieces work together.

---

## Where you'll use it

- This **is** the shape of most real Ansible codebases: a baseline role for every server, service roles per group, environment values in inventory, secrets in vault or a secret manager, `site.yml` composing it all, CI running lint/tests/plans.
- Use it as your **portfolio project** for interviews: explain the role design, the container auto-skip, the validate-then-restart handlers, the vaulted password shared by two roles, and the end-to-end verify.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| Locked out after the SSH task | Drop-in disabled something you rely on (e.g. `AllowUsers` without the automation user) | `validate` only checks syntax; keep the automation user/keys in mind; test in Molecule/VM first; serial rollout |
| `Failed to set permissions on the temporary files ... unprivileged user` on postgres tasks | `become_user: postgres` without ACL support | `acl` package (the role installs it) or pipelining |
| `psycopg2` import error | Python driver missing on db1 | `python3-psycopg2` (the role installs it) |
| `no pg_hba.conf entry for host "172.30.0.12"` | Web IP not in `postgres_allowed_clients`, or reload not applied | Check the computed list; handler/flush |
| `FATAL: password authentication failed for user "shop"` | Password mismatch, or password hashed differently | Same vault variable for both roles; re-run db play |
| Container's host clock/firewall changed | Container auto-detection failed | Check `ansible_facts['virtualization_type']` on that host; set the `baseline_manage_*` flags explicitly |
| Second run shows changes | Non-idempotent bits (unsorted templates, `latest`, commands) | `dictsort`/`sort`, `present`, `creates`/`changed_when` |
| `hostvars[groups['db'][0]]` errors in web play | DB host not in inventory/limit or facts missing | The first "gather facts on all" play; use inventory addresses |

---

## Quick recap

- A full environment = **baseline** (all hosts) + **service roles** per group + **wiring** through inventory variables, `groups`/`hostvars`, and one vaulted secret shared by producer and consumer.
- Make roles environment-aware **through data**: container auto-detection flags (`baseline_manage_*`) protect the WSL2 host in the lab and apply fully on real servers.
- Safety patterns: validated sudoers and sshd, validate-then-restart handlers, `creates` for one-time init, `no_log` for secrets, ports allowed before default-deny.
- `site.yml` composes plays in dependency order and ends with **end-to-end verification**; second run `changed=0`; lint + Molecule.

### Interview questions

1. **Q: Describe an Ansible project you built.**
   A: (Use this project.) A baseline hardening role for a mixed RHEL/Ubuntu fleet with data-driven users/sudo/SSH settings and OS-specific vars; a PostgreSQL role with one-time init and access rules computed from the inventory; web and load-balancer roles; secrets in vault; validate-then-restart handlers; an end-to-end verify play; idempotent, linted and tested with Molecule; run from CI.
2. **Q: How do you pass a database password from the DB role to the web servers' config?**
   A: Keep one vaulted variable (e.g. `vault_postgres_shop_password`) in group_vars for all; reference it both in the DB role's user creation and in the web config template, with `no_log` on tasks that handle it.
3. **Q: How do you avoid locking yourself out when hardening SSH with Ansible?**
   A: Use a drop-in file with `validate: sshd -t -f %s`, a handler that tests the full config before restarting, never remove the automation user's access, test in Molecule/VMs, and roll out in batches with a connection check after each.

### Scenario question (answer out loud first, then compare)

> *"Your baseline role must now meet a CIS benchmark (hundreds of settings) for RHEL 9/10 and Ubuntu 24.04. How do you extend the design without turning the role into a monster?"*

**Model answer:**
1. **Don't reinvent**: evaluate existing, maintained content (e.g. community/vendor CIS/STIG hardening roles or RHEL System Roles for parts like SELinux/firewall/crypto policies), and wrap them with your defaults.
2. If building in-house: split by **benchmark section** (`tasks/1_filesystems.yml`, `5_ssh.yml`...), each guarded by a **control toggle** variable (`baseline_cis_5_2_4: true`) with documented defaults, so exceptions are explicit data in `group_vars`, not code changes.
3. Data-driven implementations (dicts of sysctl/sshd/auditd settings rendered by a few generic tasks) instead of one task per control.
4. Verification: a **check-mode compliance run** (drift report, topic 55) + a scanner (OpenSCAP) in CI against Molecule/VMs per OS, mapping failures back to controls.
5. Roll out by control groups, starting report-only, then enforce in batches.

**Likely follow-ups:**
- *How do you handle a control that breaks an application?* A per-group or per-host exception variable with a justification comment and a ticket link, reviewed in a PR.
- *How do you prove compliance to auditors?* Stored check-mode results and scanner reports per run, plus the Git history of exceptions.
