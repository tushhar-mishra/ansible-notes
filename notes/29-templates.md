# 29 · Templates

> **Level:** ★★ intermediate
>
> **You will learn:** how the **`template`** module turns a **Jinja2 file** (`.j2`) plus variables into a real config file on each host, the template **statements** (`{% if %}`, `{% for %}` with the `loop` variable, `{% set %}`), **whitespace control** (`trim_blocks`, `lstrip_blocks`, `{%-  -%}`), the **`ansible_managed`** header, **`validate`** for whole configs, `backup`, where templates are found (`templates/`), and how to build cluster configs from other hosts' data (`groups` + `hostvars`). You'll template a real **nginx load balancer** on `lb1` that spreads traffic across `web1` and `web2`.
>
> **You need:** [28 · Loops](28-loops.md), [22 · Jinja2 basics](22-jinja2-basics.md), [23 · Facts and magic variables](23-facts-and-magic-variables.md), [18 · copy and validate](18-modules-files.md).
>
> **Part 3 · Variables, facts and logic** · Next: [30 · Handlers](30-handlers.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **template** is a config file with blanks and small bits of logic. Ansible fills it in **per host**, on the control node, and copies the result to the host. It's `copy` with variables.

**Picture it.** A wedding invitation design with `{{ guest_name }}` and "`{% if plus_one %}`and guest`{% endif %}`". One design, a personalised card per guest.

**The one thing to know.** `templates/motd.j2`:

```jinja
{{ ansible_managed | comment }}
Welcome to {{ inventory_hostname }} ({{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }})
{% if 'web' in group_names %}
This is a WEB server. Peers:
{% for h in groups['web'] if h != inventory_hostname %}
  - {{ h }} ({{ hostvars[h]['ansible_host'] }})
{% endfor %}
{% endif %}
```

```yaml
- name: Message of the day
  ansible.builtin.template:
    src: motd.j2
    dest: /etc/motd
    mode: "0644"
```

**Try it now:** create both, run, then `ssh ansible@172.30.0.11 cat /etc/motd`.

**You've got it if you can:** write a template with an `if` and a `for` loop, and explain where it's rendered.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### The `template` module

| Option | Meaning |
|---|---|
| `src` | Template file on the control node. Relative paths are searched in **`templates/`** next to the playbook/role, then the playbook dir |
| `dest` | Path on the target |
| `owner`, `group`, `mode`, `setype`... | As for `copy` (topic 18) |
| `validate` | Check the rendered file before it replaces the real one (`%s` = temp file) |
| `backup` | Keep the previous version |
| `trim_blocks` | Remove the newline after a `{% %}` tag (default **true**) |
| `lstrip_blocks` | Remove spaces/tabs before a `{% %}` tag on its line (default false) |
| `newline_sequence` | `\n` (default) or `\r\n` for Windows files |
| `output_encoding` | Default `utf-8` |

Everything `copy` does (checksum comparison, atomic replace, diff support) also applies: template is idempotent as long as the **rendered** content doesn't change between runs.

### Template syntax

All Jinja2 from topic 22 works (`{{ }}`, filters, tests), plus **statements** that are mostly used in files:

**if / elif / else**

```jinja
{% if app_env == 'production' %}
log_level = warn
{% elif app_env == 'staging' %}
log_level = info
{% else %}
log_level = debug
{% endif %}
```

**for** (with the special `loop` variable)

```jinja
{% for user in app_users %}
{{ loop.index }}. {{ user.name }}{{ ',' if not loop.last }}
{% else %}
(no users)
{% endfor %}
```

| `loop.` variable | Value |
|---|---|
| `loop.index` / `loop.index0` | 1-based / 0-based position |
| `loop.first`, `loop.last` | Booleans |
| `loop.length` | Number of items |
| `loop.revindex` | Items left (1-based) |

- `{% else %}` inside a `for`: printed when the list is **empty**.
- Filter inline: `{% for h in groups['web'] if h != inventory_hostname %}`.
- Loop over a dict: `{% for key, value in settings.items() %}` (in templates, `.items()` is fine and common).

**set** (a variable local to the template)

```jinja
{% set workers = ansible_facts['processor_vcpus'] * 2 %}
worker_processes {{ workers }};
```

**Comments**: `{# not printed #}`.

### Whitespace control

Statements leave blank lines behind unless you control them:

| Tool | Effect |
|---|---|
| `trim_blocks: true` (module default) | The newline right after `%}` is removed |
| `lstrip_blocks: true` (module option) | Leading whitespace before `{%` on a line is removed, so you can indent tags |
| `{%-` / `-%}` | Strip whitespace before / after this tag, including newlines |
| `{{-` / `-}}` | Same for expressions |
| First line `#jinja2: lstrip_blocks: True, trim_blocks: True` | Set options inside the template file itself |

Rule of thumb: render once with `--check --diff` and adjust until the output looks like a hand-written file.

### `ansible_managed`

A variable with a standard "don't edit this by hand" text, set in `ansible.cfg` (`ansible_managed = ...`; default `Ansible managed`). Use it at the top of every template with the **`comment`** filter, which wraps it in the file's comment style:

```jinja
{{ ansible_managed | comment }}            {# → "#\n# Ansible managed\n#" #}
{{ ansible_managed | comment('c') }}       {# C-style /* */ #}
{{ ansible_managed | comment(decoration='; ') }}   {# INI files with ; comments #}
```

⚠️ Don't put dates, user names or hostnames of the control node in `ansible_managed` (old configs used `{file} modified on %Y-%m-%d by {uid}`): the file would change every run and break idempotency (topic 27).

### Template vs copy vs lineinfile

| Use | When |
|---|---|
| `template` | You **own** the whole file and parts depend on variables/facts. ✅ The default for config you manage |
| `copy` | Static file, no variables |
| `lineinfile`/`blockinfile` | Small change to a file **someone else** owns (topic 18) |

### Templates that use other hosts

The most powerful pattern: generate a host's config from **other hosts' data**:

```jinja
upstream web_backend {
{% for host in groups['web'] %}
    server {{ hostvars[host]['ansible_host'] }}:{{ hostvars[host]['http_port'] | default(80) }};
{% endfor %}
}
```

- `groups['web']`: every web host in the inventory (even ones not in this play).
- `hostvars[host]['ansible_host']`: their inventory address, always available.
- Facts of other hosts need gathering first (topic 23).

Add a web server to the inventory, re-run, and the load balancer config updates itself.

---

## Why it exists

Real config files mix fixed text with values that differ per host (IP, memory, role, environment) and lists that change over time (backends, users, allowed networks). Managing that with `lineinfile` is fragile; keeping a copy per host doesn't scale. Templates let you keep **one** readable file in Git that produces the correct config for every host, and they make generated configs **reviewable** with `--diff`.

---

## Analogy

**A mail-merge letter with optional paragraphs.** The template is the master letter. Fields (`{{ name }}`) are filled from the address book (variables, facts, `hostvars`). Optional paragraphs (`{% if %}`) appear only for some recipients. A list of items (`{% for %}`) prints one line per order item. Before posting, a proof-reader checks each letter (`validate`). The printer (control node) does all of it; the post office (target) only receives the finished letter.

---

## How it works inside

```
 template: src=lb-nginx.conf.j2 dest=/etc/nginx/nginx.conf validate='nginx -t -c %s'
   │
 CONTROL NODE (template action plugin)
   ├─ 1. find src in templates/ (role → playbook dir)
   ├─ 2. render with Jinja2 using THIS host's variables (+ hostvars, groups, facts)
   │       undefined variable → task fails here, nothing sent
   ├─ 3. write the result to a local temp file
   └─ 4. hand over to the copy logic (topic 18):
 TARGET
   ├─ 5. compare checksum with dest → same? ok (no transfer)
   ├─ 6. different: upload to temp → run validate on it → fail? stop, real file untouched
   ├─ 7. backup (if asked) → atomic rename into place → set owner/mode
   └─ 8. changed: true (+ diff in --diff mode)
```

The target never sees Jinja2. It only receives finished text.

---

## Code / commands

### 1. A message of the day for every host

`~/ansible-lab/templates/motd.j2`:

```jinja
{{ ansible_managed | comment }}
Welcome to {{ inventory_hostname }} ({{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }})
Groups: {{ group_names | join(', ') }}
{% if 'web' in group_names %}
This is a WEB server. Peers:
{% for h in groups['web'] if h != inventory_hostname %}
  - {{ h }} ({{ hostvars[h]['ansible_host'] }})
{% else %}
  (no peers)
{% endfor %}
{% endif %}
{% if ansible_facts['memtotal_mb'] < 2048 %}
NOTE: low-memory host
{% endif %}
```

Rendered on web1:

```
#
# Ansible managed
#
Welcome to web1 (AlmaLinux 10.1)
Groups: alma, app, web
This is a WEB server. Peers:
  - web2 (172.30.0.12)
```

On db1 (not in `web`): only the first three lines.

### 2. A real load balancer on lb1

`~/ansible-lab/group_vars/lb.yml` (create or extend):

```yaml
---
lb_listen_port: 80
lb_backend_group: web
lb_algorithm: round_robin        # or least_conn
```

`~/ansible-lab/templates/lb-nginx.conf.j2`:

```jinja
{{ ansible_managed | comment }}
user www-data;
worker_processes {{ ansible_facts['processor_vcpus'] }};
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    access_log /var/log/nginx/access.log;
    error_log  /var/log/nginx/error.log;

    upstream backend {
{% if lb_algorithm == 'least_conn' %}
        least_conn;
{% endif %}
{% for host in groups[lb_backend_group] %}
        server {{ hostvars[host]['ansible_host'] }}:{{ hostvars[host]['http_port'] | default(80) }};  # {{ host }}
{% endfor %}
    }

    server {
        listen {{ lb_listen_port }} default_server;
        location / {
            proxy_pass http://backend;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

- `user www-data;`: Ubuntu's nginx user (lb1 is Ubuntu). A multi-distro role would take this from a variable.
- `$host`, `$proxy_add_x_forwarded_for`: **nginx** variables, not Jinja2: Jinja2 only touches `{{ }}`, `{% %}` and `{# #}`, so `$...` passes through untouched.
- The `for` writes one `server` line per web host, with a comment naming it.

`~/ansible-lab/lb.yml`:

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
      register: lb_conf

    - name: nginx is running and enabled
      ansible.builtin.systemd_service:
        name: nginx
        state: started
        enabled: true

    - name: Reload nginx when its config changed
      ansible.builtin.systemd_service:
        name: nginx
        state: reloaded
      when: lb_conf is changed
```

- `validate: nginx -t -c %s`: nginx tests the **rendered temp file** as a full config. A broken config never reaches `/etc/nginx/nginx.conf`.
- The last task reloads only **when the config changed**. That's exactly the job **handlers** do, more cleanly (topic 30).

```bash
ansible-playbook lb.yml --diff
```
```
TASK [nginx.conf is rendered and validated] ********************************************
--- before: /etc/nginx/nginx.conf
+++ after: /home/you/.ansible/tmp/ansible-local-.../tmpab12/lb-nginx.conf.j2
@@ -1,85 +1,31 @@
-user www-data;
-worker_processes auto;
...
+    upstream backend {
+        server 172.30.0.11:80;  # web1
+        server 172.30.0.12:80;  # web2
+    }
...
changed: [lb1]

TASK [Reload nginx when its config changed] ********************************************
changed: [lb1]
```

Test the load balancer from your WSL2 shell:

```bash
for i in 1 2 3 4; do curl -s 172.30.0.31; done
```
```
web1 is the canary
Hello from web2
web1 is the canary
Hello from web2
```

Round-robin across both web servers. (Page texts from topic 24's `group_vars`/`host_vars`.)

### 3. Whitespace: before and after

Without `lstrip_blocks`, indented tags leave spaces. Compare:

```jinja
allowed:
    {% for net in allowed_nets %}
    - {{ net }}
    {% endfor %}
```

With the default (`trim_blocks: true`, `lstrip_blocks: false`) the lines holding only a tag become lines of **spaces**. Add `lstrip_blocks: true` to the task (or `#jinja2: lstrip_blocks: True` as the template's first line) and they disappear.

### 4. Render a template locally to test it

```bash
ansible localhost -m ansible.builtin.debug -a "msg={{ lookup('ansible.builtin.template', 'templates/motd.j2') }}" -e '{"inventory_hostname": "test"}'
```

The **template lookup** renders a template on the control node and returns the text: handy for quick tests (facts won't exist for `localhost` unless gathered; for full tests run the playbook with `--check --diff`).

---

## Hands-on lab

1. Create `templates/motd.j2` and a play for `all` that templates it to `/etc/motd`. Check two hosts.
2. Create the load balancer (`group_vars/lb.yml`, `templates/lb-nginx.conf.j2`, `lb.yml`), run it, and curl `172.30.0.31` several times.
3. Switch `lb_algorithm` to `least_conn`, run with `--check --diff` first, then for real. Only the template and the reload change.
4. Break the template on purpose (remove a `;` after `server ...`). Run: `failed to validate`, and nginx keeps serving with the old config (curl still works).
5. **Scale out**: add a fake backend to `[web]` in `inventory.ini`: `web3 ansible_host=172.30.0.13`. Run **only** `lb.yml` (web3 doesn't exist, but lb.yml never connects to it; it only reads its inventory data). The upstream now lists three servers; curl shows nginx skipping the dead one after errors. Remove web3 and re-run.
6. Add `{% set workers = ansible_facts['processor_vcpus'] * 2 %}` and use it; add a `{# comment #}`; check the output with `--diff`.
7. Run `lb.yml` twice: second run `changed=0` and **no reload**.

**Done when:** your load balancer config is generated entirely from the inventory, and re-runs don't reload nginx.

---

## Where you'll use it

- **Every config you own**: nginx/Apache vhosts, app `.env`/`.yml`/`.ini` files, systemd units, logrotate, chrony, sshd drop-ins, sudoers.
- **Cluster configs** from inventory data: load balancer backends, database replica lists, Kafka/Zookeeper/etcd member lists, Prometheus scrape targets, `/etc/hosts`.
- **Reports**: an HTML/CSV summary of the fleet rendered on the control node (`delegate_to: localhost`, `run_once: true`).
- **Kubernetes manifests** rendered from variables before applying (`kubernetes.core.k8s` with `template:`).

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Could not find or access 'x.j2'` | Not in `templates/` (role/playbook) | Move it; check the search paths in the error |
| `AnsibleUndefinedVariable: 'x' is undefined` | Variable missing for this host | Define it, or `\| default(...)` in the template |
| `'dict object' has no attribute 'ansible_facts'` / empty for other hosts | Template uses `hostvars[h]['ansible_facts']` but those hosts' facts weren't gathered | Gather on all first, or use inventory data (`ansible_host`) |
| `failed to validate` | The rendered config is invalid (good: real file untouched) | Read `stderr`, fix the template |
| Validation fails for an included **fragment** (e.g. a single vhost) | `nginx -t -c %s` needs a full config | Validate the main config in a handler, or template the full file |
| File `changed` every run | Run-time values (dates, `now()`, random) or unordered data (iterating a set) | Remove them; sort lists (`\| sort`) for stable output |
| Blank lines / odd indentation in output | Whitespace from tags | `lstrip_blocks`, `{%- -%}` |
| `$var` or `${var}` broken | Rarely; Jinja2 ignores `$`. If the file contains `{{` literally (e.g. Go templates), escape with `{% raw %}...{% endraw %}` | `{% raw %}` blocks |
| Template works for RHEL but not Ubuntu | Hard-coded distro values (`user nginx;` vs `www-data`) | Take them from variables per OS group |

---

## Quick recap

- `template` = render a `.j2` file from **`templates/`** with this host's variables **on the control node**, then copy (checksum, atomic, `validate`, `backup`, diff).
- Syntax: `{{ expr }}`, `{% if/elif/else %}`, `{% for x in list if cond %}` with `loop.index/first/last` and `{% else %}`, `{% set %}`, `{# comment #}`.
- Whitespace: `trim_blocks` (default on), `lstrip_blocks`, `{%- -%}`.
- Start every template with `{{ ansible_managed | comment }}`; never put dates in it.
- Build cluster configs from `groups[...]` + `hostvars[...]`; gather other hosts' facts first if you use them.

### Interview questions

1. **Q: What's the difference between `copy` and `template`?**
   A: `template` renders a Jinja2 file with the host's variables on the control node before copying; `copy` copies a file as is. Both compare checksums and are idempotent.
2. **Q: How do you generate a load balancer config that lists all web servers?**
   A: In the template, loop over `groups['web']` and print each host's address from `hostvars[host]['ansible_host']` (or its facts, if gathered).
3. **Q: How do you prevent a broken generated config from being applied?**
   A: Use `validate:` with the service's config test (e.g. `nginx -t -c %s`, `visudo -cf %s`), so the file is only replaced if the test passes.

### Scenario question (answer out loud first, then compare)

> *"You manage an HAProxy/nginx load balancer config with a template that lists backends from `groups['web']`. During an autoscaling event, a new web server joined, the LB config was regenerated and reloaded, but traffic went to the new server before its app was deployed, causing errors. How do you redesign this?"*

**Model answer:**
1. Problem: membership in `groups['web']` meant "should receive traffic", but a server should only get traffic **after** it's configured and healthy.
2. Fix the ordering: deploy/configure the new web server first, verify health (`uri` + `until`), **then** update the LB. In one playbook: a play on `web` (deploy + health check), then a play on `lb` (template + reload).
3. Add a readiness filter in the template: only include hosts that are ready, e.g. a host variable/fact `web_ready: true` set after the health check (`set_fact` with `cacheable: true` or a custom fact), or a separate group like `web_in_service`.
4. Use the LB's own health checks (nginx `max_fails`/`fail_timeout`, HAProxy `check`) as a second safety net.
5. For rolling changes, drain/enable backends per host with `delegate_to` the LB (topics 46, 49).

**Likely follow-ups:**
- *Why not rely only on LB health checks?* They react after errors start; ordering prevents them.
- *How do you keep the template output stable?* Sort the hosts (`groups['web'] | sort`), avoid timestamps, so the config only changes when membership changes.
