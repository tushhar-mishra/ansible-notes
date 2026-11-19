# 66 · Interview Question Bank

> **Level:** ★★ intermediate
>
> **You will learn:** the Ansible questions that come up in DevOps/SRE interviews, grouped like these notes (basics, inventory and connection, playbooks and modules, variables and logic, roles and collections, secrets, cloud and Terraform, execution control, quality, CI/CD and platform, production and security), each with a short, accurate answer and a pointer to the topic with the full explanation. Senior design scenarios are in topic 67.
>
> **You need:** the course. Use this page for revision: cover the answer, say yours out loud, then compare.
>
> **Part 12 · Production operations** · Next: [67 · Senior scenario interviews](67-senior-scenario-interviews.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Interviewers test three things: do you know the **concepts** (idempotency, push/agentless, precedence), can you **use** the tool (modules, roles, vault, loops, handlers), and have you **run it in production** (CI, secrets, batching, debugging). Short, precise answers with one example beat long recitations.

**How to use this page.** Read a question, answer aloud in 2–3 sentences, then compare. Mark the ones you missed and re-read their topic.

**The ten you must nail:**

1. What is idempotency? (topic 27)
2. Ansible vs Terraform? (02, 43)
3. Push vs pull, agentless? (01, 03)
4. What happens when you run a playbook? (13, 15)
5. Variable precedence? (25)
6. Handlers? (30)
7. Roles and their structure? (35)
8. Vault? (40)
9. Rolling updates with zero downtime? (45, 49)
10. How do you debug a failing playbook? (58)

> 📖 **That's the core.** Everything below is the **full bank**.

---

## What it is

### Basics

| Q | A | Topic |
|---|---|---|
| What is Ansible? | An agentless, push-based automation tool: it connects over SSH (or WinRM/APIs), runs idempotent modules, and is driven by YAML playbooks. | 03 |
| What is configuration management? | Defining server configuration as version-controlled code and using a tool to make servers match it repeatedly. | 01 |
| What is idempotency? | Running something again on an already-correct system changes nothing and reports `ok`; the second run of a playbook should show `changed=0`. | 01, 27 |
| Why is Ansible called agentless? | Targets need only SSH and Python (Windows: WinRM/SSH + PowerShell); modules are copied, executed and removed per task. | 03, 13 |
| Push vs pull? | Push: central node connects to hosts when you run it (Ansible). Pull: an agent on each host fetches config periodically (Puppet/Chef; Ansible via `ansible-pull`). | 01, 55 |
| Control node vs managed node? | Control node runs Ansible (Linux/macOS/WSL, Python 3.13+ for core 2.22); managed nodes are targets (SSH + Python 3.9+). | 03, 07 |
| ansible-core vs ansible package? | ansible-core = engine + `ansible.builtin`; `ansible` = a specific core plus ~85 collections. | 03, 08 |
| What's a module? | A unit of work (usually Python) that checks state, changes only what's needed, and returns JSON. | 13, 16 |
| What's a task, play, playbook? | Task = one module call; play = hosts + tasks + settings; playbook = YAML file of plays. | 14 |
| What is YAML and a common YAML pitfall in Ansible? | A data format of maps/lists/scalars; values starting with `{{` must be quoted; tabs are invalid; `yes/no` become booleans. | 04 |
| What does the PLAY RECAP show? | Per host: ok (includes changed), changed, unreachable, failed, skipped, rescued, ignored. | 14 |

### Inventory and connection

| Q | A | Topic |
|---|---|---|
| What is an inventory? | The list of hosts and groups (static INI/YAML, directory, or dynamic plugin), optionally with variables. | 10 |
| What are `all` and `ungrouped`? | Built-in groups: every host; hosts in no other group. | 10 |
| Static vs dynamic inventory? | Static is a file you maintain; dynamic is generated at run time by a plugin from a source of truth (e.g. AWS tags). | 42 |
| What are `keyed_groups` and `compose`? | Inventory plugin options: create groups from values (tags) and set host variables from expressions (e.g. `ansible_host`). | 42 |
| How does Ansible connect to Linux hosts? | Through the OpenSSH client (`ssh` connection plugin), honouring `~/.ssh/config`, keys and agents. | 05, 11 |
| What is host key checking? Should you disable it? | Verifying server identity against `known_hosts`; no: pre-load verified keys or use `accept-new`/certificates. | 05 |
| Where does Ansible look for `ansible.cfg`? | `ANSIBLE_CONFIG`, `./ansible.cfg`, `~/.ansible.cfg`, `/etc/ansible/ansible.cfg`: first found wins. | 11 |
| Why might `ansible.cfg` be ignored in WSL? | Under `/mnt/c` the directory is world-writable; Ansible ignores configs there for security. | 11 |
| How do you reach hosts behind a bastion? | `ansible_ssh_common_args: '-o ProxyJump=user@bastion'` or SSH config. | 05, 11 |
| What are connection plugins? Examples? | Code that reaches targets: `ssh`, `local`, `winrm`/`psrp`, `network_cli`, `community.docker.docker`, `amazon.aws.aws_ssm`. | 11, 44 |
| How do you manage hosts without Python? | `raw` module to bootstrap Python (with `gather_facts: false`). | 07 |

### Playbooks, modules and ad hoc

| Q | A | Topic |
|---|---|---|
| What's an ad hoc command? | `ansible PATTERN -m MODULE -a ARGS`: one module, many hosts, no playbook. | 12 |
| `command` vs `shell` vs `raw`? | `command` runs without a shell (safer); `shell` uses `/bin/sh` (pipes, redirects); `raw` needs no Python. All report changed; prefer modules. | 12 |
| What is an FQCN and why use it? | `namespace.collection.module` (e.g. `ansible.builtin.copy`): unambiguous and lint-compliant. | 16 |
| How do you read module documentation offline? | `ansible-doc MODULE` (`-s` for a snippet, `-l` to list). | 16 |
| `present` vs `latest` for packages? | `present` installs if missing; `latest` upgrades whenever a newer version exists (not idempotent over time). | 17 |
| What does `append: true` do in the user module? | Adds supplementary groups without removing existing ones. | 17 |
| `copy` vs `template`? | `template` renders Jinja2 with host variables first; both are checksum-based. | 18, 29 |
| Why does `lineinfile` sometimes add duplicates? | Its regexp doesn't match the current line form; match the key in all forms. | 18 |
| What does `validate` do? | Tests a temporary copy of the new file (`%s`) before replacing the real one. | 18 |
| What is `become`? | Privilege escalation (default sudo to root) after logging in as a normal user. | 19 |
| Why can `become_user: postgres` fail? | Temp files are owned by the login user; needs ACLs (`acl` package) or pipelining. | 19 |
| What do `--check` and `--diff` do? | Dry run predicting changes; show before/after diffs. Commands without `creates`/`removes` are skipped in check mode. | 20 |
| How do you limit a run to some hosts? | `--limit` (patterns like `web:!web3`, `@file`), and `--list-hosts` to preview. | 12, 20 |

### Variables, facts and logic

| Q | A | Topic |
|---|---|---|
| Where can variables be defined? | Role defaults/vars, inventory (`group_vars`/`host_vars`), play vars/vars_files/vars_prompt, block/task vars, include_vars, set_fact/register, role/include params, extra vars. | 21, 25 |
| Explain variable precedence briefly. | Role defaults lowest; inventory (all < group < host); facts; play vars; role vars; block/task; include_vars; set_fact/register; params; extra vars (`-e`) always win. | 25 |
| Where should a role put user-tunable values? | `defaults/main.yml`. | 25, 35 |
| What are facts? | Host data gathered by `setup` at play start, under `ansible_facts`. | 23 |
| `ansible_facts['x']` vs `ansible_x`? | Injected `ansible_x` variables are deprecated (2.20) and off by default in 2.24. | 23, 63 |
| What are magic variables? | Ansible-provided: `inventory_hostname`, `groups`, `group_names`, `hostvars`, `ansible_play_hosts`, `playbook_dir`... | 23 |
| How do you use another host's data? | `hostvars['host']`; facts only if gathered this run or cached. | 23 |
| What is `register`? | Saves a task's result per host; skipped tasks still register (`skipped: true`). | 26 |
| What changed about `when` in 2.19? | Conditionals must evaluate to booleans; strings/lists cause errors. | 26 |
| How do you make `shell` idempotent? | Prefer a module; else `creates`/`removes`, `changed_when`, `failed_when`, or check-then-act. | 27 |
| `changed_when` vs `creates`? | `creates` decides whether the command runs; `changed_when` only changes how the result is reported. | 27 |
| How do you loop over a dict? | `loop: "{{ d \| dict2items }}"` with `item.key`/`item.value`. | 28 |
| `with_items` vs `loop`? | `loop` is the recommended form; `with_items` flattens one level and is legacy style. | 28 |
| How do you retry until something works? | `until` + `retries` + `delay`. | 28 |
| What are handlers? | Tasks run only when notified by a changed task; once per host, at the end of the play section (or at `flush_handlers`), in definition order. | 30 |
| What happens to handlers if the play fails? | Pending handlers don't run for failed hosts unless `force_handlers`. | 30 |
| How do you do try/catch? | `block` / `rescue` / `always`; `ansible_failed_task`/`ansible_failed_result` in rescue. | 31 |
| `ignore_errors` vs `failed_when`? | `ignore_errors` hides any failure of the task; `failed_when` precisely defines failure. Prefer the latter. | 27, 31 |
| What does `any_errors_fatal` do? | Stops the play for all hosts if any host fails a task. | 31 |
| Default value for an undefined variable? | `{{ x \| default('v') }}`; `default(omit)` to drop a module option; `mandatory` to require it. | 22 |
| How do you compare versions? | The `version` test: `x is version('2.0', '>=')`. | 22 |
| How do you merge dicts? | `combine` filter (`recursive=true` for deep merge). | 32 |
| Where do lookups run? | On the control node. | 32 |

### Roles, collections and reuse

| Q | A | Topic |
|---|---|---|
| What's a role and its structure? | A reusable unit: `tasks/`, `handlers/`, `defaults/`, `vars/`, `templates/`, `files/`, `meta/`. | 35 |
| `roles:` vs `import_role` vs `include_role`? | Static before tasks / static in place / dynamic at run time (loops, conditions once). | 35 |
| `import_tasks` vs `include_tasks`? | Static (parse time, keywords inherited, no loops) vs dynamic (run time, loops, runtime file names). | 34 |
| What is `meta/argument_specs.yml`? | Role input declarations validated automatically before the role runs. | 36 |
| How do you support multiple OSes in a role? | OS-specific vars files loaded with `include_vars` + `first_found`; generic tasks. | 36 |
| What is a collection? | A versioned package of modules, plugins, roles and playbooks (`namespace.collection`). | 37 |
| How do you pin collections? | `requirements.yml` with versions; install with `ansible-galaxy collection install -r`. | 37 |
| Two versions installed: which wins? | First found in the search path (playbook-adjacent, `collections_path`, site-packages). | 37 |
| Have you written a custom module? | (Describe `AnsibleModule`, argument spec, check mode, idempotent compare-then-change, `exit_json`.) | 38 |
| What are tags, `always`, `never`? | Labels to run subsets; `always` always runs; `never` only when explicitly requested. | 33 |

### Secrets

| Q | A | Topic |
|---|---|---|
| How do you store secrets? | Ansible Vault (encrypted files/strings) or external secret managers via lookups; hide with `no_log`. | 40, 41 |
| What are vault IDs? | Labels for different vault passwords (e.g. per environment). | 40 |
| How do you give the vault password in CI? | A password script reading a CI secret/env var, or AWX vault credentials. | 40, 53 |
| What changed in 2.22 regarding secrets? | Vault values, `no_log` values and prompted passwords are masked as `$REDACTED$`; `register_secret` marks other values. | 40, 63 |
| Vault vs HashiCorp Vault/AWS Secrets Manager? | Encrypted-in-Git with a shared password vs central service with per-identity access, audit, rotation, dynamic secrets. | 41 |
| Why avoid secrets on command lines? | Visible in `ps`/audit logs even with `no_log`. | 41 |

### Cloud, Terraform and beyond Linux

| Q | A | Topic |
|---|---|---|
| Ansible vs Terraform? | Terraform provisions infrastructure via APIs with state and plans; Ansible configures inside servers and orchestrates; stateless. Used together. | 02, 43 |
| How do they hand off? | Tags + dynamic inventory (best), or Terraform outputs → inventory file. | 43 |
| Why not run Ansible via Terraform provisioners? | Run only at creation, taint on failure, no drift handling: a last resort. | 43 |
| How do you wait for new servers? | `gather_facts: false` + `wait_for_connection` (+ cloud-init wait). | 43, 47 |
| How does Ansible manage Windows? | WinRM/PSRP/SSH with PowerShell modules (`ansible.windows`), `runas`. | 44 |
| How does it manage network devices? | Modules run on the control node; `network_cli`/`httpapi`/`netconf`; resource modules with `merged`/`replaced`/`overridden`. | 44 |
| How do you deploy to Kubernetes with Ansible? | `kubernetes.core.k8s` (manifests/templates, `wait`), `helm` module; from localhost with a kubeconfig. | 44 |

### Execution control and performance

| Q | A | Topic |
|---|---|---|
| `forks` vs `serial`? | Parallel hosts per task vs batches that complete the whole play before the next batch. | 45 |
| Strategies? | `linear` (default, in step), `free` (each host at its own pace), `host_pinned`. | 45 |
| How do you do a canary rollout? | `serial: [1, "25%", "100%"]` + `max_fail_percentage: 0`. | 45 |
| What is `delegate_to`? | Run a task on another host with the current host's variables (e.g. LB drain). | 46 |
| `run_once` gotcha? | With `serial`, it runs once per batch. | 45 |
| How do you handle long tasks? | `async` + `poll`, or `poll: 0` + `async_status`. | 47 |
| How do you speed Ansible up? | Measure; pipelining, ControlPersist, forks, fact caching/subsets, lists not loops, free strategy, closer control nodes. | 48 |
| What does pipelining do? Requirement? | Sends modules over stdin, ~6 round trips → 1; sudo must not require a TTY. | 48 |
| How do you scale to thousands of hosts? | Tune per task, shard runs, execution near targets (mesh/runners), free strategy, images, pull mode. | 61 |

### Quality, CI/CD and platform

| Q | A | Topic |
|---|---|---|
| What does ansible-lint check? | Syntax, YAML, best practices (FQCN, names, idempotency smells, permissions), schemas; profiles up to `production`. | 50 |
| How do you test roles? | Molecule: create → converge → idempotence → verify → destroy, on all OSes. | 51 |
| What is an execution environment? | A container image with ansible-core, collections, Python and system deps; built with ansible-builder; run with navigator/AWX. | 52 |
| How do you run Ansible from GitHub Actions safely? | Self-hosted runner in the network (or SSM/OIDC), secrets in environments, ssh-agent, known_hosts, approvals, concurrency. | 53 |
| …from Jenkins? | Declarative pipeline, EE Docker agent, `sshagent`/`withCredentials`, `input` approval, `lock`. | 54 |
| How do you detect drift? | Scheduled `--check --diff` with JSON output; idempotent playbooks required. | 55 |
| What is `ansible-pull`? | Hosts clone a repo and apply a playbook to themselves (pull mode). | 55 |
| What is AWX/AAP? | A control plane: UI/API, RBAC, credentials, job templates, surveys, workflows, schedules; AAP adds hub, EDA, mesh, support. | 56 |
| What is Event-Driven Ansible? | Rulebooks: event sources → conditions → actions (e.g. launch job templates). | 56 |

### Production and security

| Q | A | Topic |
|---|---|---|
| How do you do a zero-downtime deploy? | `serial` + LB drain via `delegate_to` + deploy + health check + enable; `block`/`rescue` rollback; `max_fail_percentage`. | 49 |
| How do you avoid running against the wrong environment? | Separate inventories, no default prod inventory, environment guard asserts, prod creds only in CI/AWX. | 60, 65 |
| How do you secure the automation user? | Dedicated key-only account, short-lived certificates, source restrictions, per-env credentials; protect the key (NOPASSWD is normal). | 19, 60 |
| How do you debug a failing playbook? | Classify (parse/connection/data/module), narrow (`-l`, `--start-at-task`), `-vvv`, inspect vars, task debugger, fix, add a test. | 58 |
| UNREACHABLE vs FAILED? | Couldn't connect/authenticate vs module ran and failed. | 12, 59 |
| What changed in ansible-core 2.19? | Templating overhaul: boolean conditionals, no nested templates, untrusted templates, native types. | 63 |
| How do you upgrade Ansible? | Porting guides, branch + pins/EE, lint, Molecule, staging check/run, canary, rollback ready. | 63 |

---

## Why it exists

Interview time is short. A curated bank lets you check your knowledge quickly and spot gaps, and the topic pointers send you straight to the full explanation.

---

## Analogy

**Flashcards before an exam**: short prompts, short answers, and you only re-read the chapters for the cards you got wrong.

---

## How it works inside

Interviewers usually go **broad then deep**: a concept question (idempotency), then "how do you achieve it with `shell`?", then "how do you prove it in CI?". Follow-ups in these notes' scenario questions (end of every topic) mirror that pattern: practise those too.

---

## Code / commands

Short code answers interviewers like:

```yaml
# idempotent command
- ansible.builtin.command: /opt/app/bin/init
  args: { creates: /var/lib/app/.initialized }

# precise failure
- ansible.builtin.command: grep -q pattern /etc/file
  register: r
  changed_when: false
  failed_when: r.rc not in [0, 1]

# canary rollout
- hosts: web
  serial: [1, "25%", "100%"]
  max_fail_percentage: 0

# handler
  notify: Reload nginx
handlers:
  - name: Reload nginx
    ansible.builtin.systemd_service: { name: nginx, state: reloaded }

# try/catch
- block: [...]
  rescue: [...]
  always: [...]
```

---

## Hands-on lab

1. Go through every table covering the answer column; mark misses.
2. For each miss, re-read the topic's ⚡ Core and quick recap.
3. Explain the ten must-nail questions out loud to a friend (or record yourself) in under two minutes each.
4. Do topic 67's scenarios after this bank.

**Done when:** you answer 90% correctly without looking.

---

## Where you'll use it

- **Before interviews** (and as a refresher before on-call rotations).
- **Interviewing others**: pick questions by level.

---

## When it breaks

| Problem | Fix |
|---|---|
| Memorised answers fall apart on follow-ups | Learn the "why" from the topic; do the labs |
| Answers too long | Lead with one sentence, add one example, stop |
| Unsure about a version-specific detail | Say what you know and how you'd verify (porting guide, `ansible-doc`) |

---

## Quick recap

- Know the concepts (idempotency, agentless push, precedence), the tools (modules, roles, vault, handlers, loops), and production practice (CI, secrets, batching, debugging).
- Answer briefly with an example; expect follow-ups.

### Interview questions

(This whole page.)

### Scenario question (answer out loud first, then compare)

> *"In 60 seconds: explain Ansible to a developer who has only used Docker."*

**Model answer:** "Docker builds an image once and runs it; you replace containers instead of changing them. Ansible changes **existing machines**: you describe how servers should look in YAML (packages, files, services), and Ansible connects over SSH and makes each server match, only changing what's different, so running it again is safe. It's for the VMs and servers around your containers (and even the container hosts, Kubernetes nodes and cloud setup), and it runs from CI like your Docker builds do."

**Likely follow-ups:**
- *So why not just use containers everywhere?* Not everything is containerised (databases on VMs, network devices, Kubernetes nodes themselves, Windows); and images themselves are often built with Ansible.
