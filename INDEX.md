# Ansible: From Zero to Production

> ✅ **Status: complete.** All 68 topics (Parts 0–12) written and reviewed, November 2026. The sample outputs were written, not captured from a real run, and the labs haven't been run end to end yet. Your own terminal output is the source of truth.

One learning path, in reading order. Each topic uses only ideas explained before it. When a later idea has to appear early, it is explained right there in a line or two, and a pointer like "(topic 24)" just tells you where the full story is. You never need to jump ahead to follow along.

**Every topic has two layers:**

1. **⚡ Core: the 10-minute version.** The idea in plain words, a mental model, the one command or YAML to know, a mini-lab, and a self-check.
2. **The full reference**, always in the same shape: What it is · Why it exists · Analogy · How it works inside · Code / commands (sample output explained line by line: `ok` / `changed` / `failed` / `skipped` / `unreachable`) · Hands-on lab · Where you'll use it · When it breaks · Quick recap + interview questions + a scenario question.

**Current Ansible (checked against PyPI, 19 November 2026):** targets **ansible-core 2.22** (2.22.0, 2 November 2026) and the **`ansible` community package 15** (15.0.0, 17 November 2026). The control node needs **Python 3.13+**. Examples use FQCNs (Fully Qualified Collection Names, e.g. `ansible.builtin.copy`), `loop`, `become`, `import_tasks`/`include_tasks`. Deprecated or discouraged patterns (`include:`, `sudo:`, short module names, `with_*` as a default, `yum`, bare variables in `when` that aren't booleans, `ansible_*` injected facts instead of `ansible_facts[...]`, `shell` where a module exists…) are marked ⚠️ where they come up and are never taught as the main way.

**Your lab:** Ansible installed in the **WSL2 Ubuntu** from your Docker notes (the control node). Managed nodes are **Docker containers running SSH and systemd** (AlmaLinux 10 + Ubuntu 24.04), started with one Compose file in topic 09, so every lab is free and resets in seconds. A Vagrant/VirtualBox VM option (your Linux topic 002 lab) is given for the few labs containers can't do well. The AWS capstone is marked 💰 and ends with `terraform destroy`.

**Related notes:** [Linux](../linux-mastery-rhel/INDEX.md) · [Docker](../Docker/INDEX.md) · [Kubernetes](../CKA-Certification-Course-2025/INDEX.md) · [Jenkins](../Jenkins/INDEX.md) · [GitHub Actions](../GitHub-Actions-Basics-To-Production/notes/INDEX.md) · [Terraform](../Terraform/INDEX.md). Topics link straight to the matching topic file wherever Ansible builds on them.

**Sample outputs** show the *shape* of real output. Host names, IPs, timings and versions are only examples. Your terminal is the source of truth.

**Levels:** ★ foundation · ★★ intermediate · ★★★ advanced.

## How to study with these notes

| Your goal | Read | Time |
|---|---|---|
| **First pass: get the whole picture** | Only the **⚡ Core** of each topic, in order, doing every mini-lab | ~12 hours |
| **Become job-ready** | Second pass: the full reference plus every Hands-on lab and the three projects | A few weeks |
| **Interview tomorrow** | Cores of Parts 0–4, then [66](notes/66-interview-question-bank.md) + [67](notes/67-senior-scenario-interviews.md) | An evening |
| **Fixing something right now** | [58](notes/58-debugging-ansible.md) → [59](notes/59-error-catalogue.md) → [68](notes/68-cheat-sheet.md) | Minutes |

---

## Part 0 · Foundations you bring

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 01 | ★ | [What is configuration management?](notes/01-what-is-configuration-management.md) | Life before it (hand-built "snowflake" servers, runbooks, fragile scripts), desired state, drift, idempotency at a glance, push vs pull, agent vs agentless |
| 02 | ★ | [The tool landscape: Ansible vs Puppet/Chef/Salt vs Terraform](notes/02-tool-landscape.md) | Configuration vs provisioning, how Ansible differs from agent-based tools, why Ansible and Terraform do different jobs and how they hand off to each other |
| 03 | ★ | [What is Ansible?](notes/03-what-is-ansible.md) | Control node vs managed nodes, agentless over SSH (Secure Shell), modules, ansible-core vs the `ansible` package vs Ansible Automation Platform, Red Hat/IBM, the mental model for everything after |
| 04 | ★ | [YAML for Ansible](notes/04-yaml-for-ansible.md) | YAML (YAML Ain't Markup Language) maps, lists, scalars, indentation rules, multi-line strings, quoting `"{{ }}"`, yes/no boolean traps, how to read a playbook's shape |
| 05 | ★ | [Linux recap I: SSH, keys and host keys](notes/05-linux-recap-ssh.md) | Key-based login, `ssh-agent`, `known_hosts` and host key checking, `~/.ssh/config`, jump hosts, exactly how Ansible uses each one |
| 06 | ★ | [Linux recap II: users, sudo, packages, services, files](notes/06-linux-recap-admin-tasks.md) | The admin tasks you do by hand (useradd, sudoers, dnf/apt, systemctl, chmod/chown) and the Ansible module that replaces each one |
| 07 | ★ | [Python on the managed node](notes/07-python-on-managed-nodes.md) | Why targets need Python, interpreter discovery, `ansible_python_interpreter`, the `raw` module for bootstrapping, minimal images without Python |

## Part 1 · Setup and first contact

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 08 | ★ | [Installing Ansible](notes/08-installing-ansible.md) | pipx vs venv vs distro packages, ansible-core vs `ansible`, `ansible --version` explained line by line, the `ansible-*` commands |
| 09 | ★ | [Lab setup: managed nodes in Docker](notes/09-lab-setup.md) | A Dockerfile for SSH + systemd nodes (AlmaLinux and Ubuntu), Compose file, SSH keys, first `ssh` by hand, VM alternative, reset in seconds |
| 10 | ★ | [Inventory basics](notes/10-inventory-basics.md) | INI vs YAML inventory, hosts, groups, `children`, `all`/`ungrouped`, ranges, `ansible-inventory --graph/--list` |
| 11 | ★ | [Connection settings and ansible.cfg](notes/11-connection-settings-and-ansible-cfg.md) | `ansible_host/user/port/ssh_private_key_file`, connection types (ssh, local, docker), where ansible.cfg is looked up, `ansible-config dump --only-changed` |
| 12 | ★ | [Ad hoc commands](notes/12-ad-hoc-commands.md) | `ansible <pattern> -m -a`, host patterns, `ping`, `command` vs `shell`, `copy`, packages, reading the colored output, return codes |
| 13 | ★★ | [What happens inside: one module run](notes/13-inside-a-module-run.md) | Connect → make temp dir → copy the module (AnsiballZ) → run with Python → JSON back → clean up; seen live with `-vvvv` and `ANSIBLE_KEEP_REMOTE_FILES` |

## Part 2 · Playbooks

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 14 | ★ | [Your first playbook](notes/14-first-playbook.md) | Plays, `hosts`, `tasks`, `name`, FQCN modules, running it twice, the PLAY RECAP line by line (ok/changed/unreachable/failed/skipped/rescued/ignored) |
| 15 | ★★ | [What happens inside ansible-playbook](notes/15-inside-ansible-playbook.md) | Full trace: parse YAML → load inventory → resolve variables → connect → gather facts → task by task across hosts (linear strategy, forks) → handlers → recap |
| 16 | ★ | [Modules, FQCNs and ansible-doc](notes/16-modules-fqcn-and-ansible-doc.md) | `ansible.builtin` vs other collections, finding a module, reading `ansible-doc`, parameters, return values, why short names are ⚠️ discouraged |
| 17 | ★ | [Essential modules I: packages, services, users](notes/17-modules-packages-services-users.md) | `package`/`dnf`/`apt`, `systemd_service`/`service`, `user`, `group`, `authorized_key`, `cron`; each mapped to its Linux topic |
| 18 | ★ | [Essential modules II: files and content](notes/18-modules-files.md) | `file`, `copy`, `lineinfile`, `blockinfile`, `replace`, `stat`, `fetch`, `get_url`, `unarchive`, `synchronize`; owner/mode/SELinux context |
| 19 | ★★ | [Privilege escalation: become](notes/19-become.md) | `become`, `become_user`, `become_method`, `-K`, sudoers rules for automation, becoming an unprivileged user (temp-file permission problems), security trade-offs |
| 20 | ★ | [Safe runs: check, diff, limit and friends](notes/20-check-diff-limit.md) | `--check`, `--diff`, `--limit`, `--syntax-check`, `--list-hosts/--list-tasks`, `--start-at-task`, `--step`; how to run a playbook against prod without fear |

## Part 3 · Variables, facts and logic

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 21 | ★ | [Variables basics](notes/21-variables-basics.md) | `vars`, `vars_files`, `-e`, naming rules, types (string, number, list, dict), when you must quote `"{{ }}"`, `debug` |
| 22 | ★ | [Jinja2 basics: expressions, filters, tests](notes/22-jinja2-basics.md) | Jinja2 (the templating language), `{{ }}` vs `{% %}`, common filters (`default`, `mandatory`, `lower`, `join`, `bool`), tests, lazy evaluation |
| 23 | ★ | [Facts and magic variables](notes/23-facts-and-magic-variables.md) | `gather_facts`, the `setup` module, `ansible_facts[...]`, custom facts (`facts.d`), `set_fact`, `hostvars`, `groups`, `inventory_hostname`, fact caching |
| 24 | ★ | [Inventory variables: group_vars and host_vars](notes/24-group-vars-and-host-vars.md) | Directory layout, `all`, parent/child groups, which group wins when a host is in two groups, `ansible-inventory --host` |
| 25 | ★★ | [Variable precedence](notes/25-variable-precedence.md) | Every place a variable can live, the precedence ladder as one table + diagram, the 5 rules you actually need, debugging "why is it this value?" |
| 26 | ★ | [register and when](notes/26-register-and-when.md) | Capturing results, the return structure (`rc`, `stdout`, `changed`, `failed`), conditionals, boolean-only conditionals (2.19+ rule), `is defined` |
| 27 | ★★ | [Idempotency in depth](notes/27-idempotency-in-depth.md) | What it really means, why it matters (safe re-runs, drift repair, check mode), how `command`/`shell` break it, fixes: `creates`/`removes`, `changed_when`, `failed_when`, a real module, the "second run = 0 changed" test |
| 28 | ★ | [Loops](notes/28-loops.md) | `loop`, `loop_control` (`label`, `index_var`, `pause`), loops over dicts, `until`/`retries`, ⚠️ `with_*` and when you still see it |
| 29 | ★★ | [Templates](notes/29-templates.md) | The `template` module, Jinja2 `if`/`for`, whitespace control, `ansible_managed`, `validate:` (e.g. `nginx -t`, `visudo -cf`), `backup` |
| 30 | ★★ | [Handlers](notes/30-handlers.md) | `notify`, why restarts should happen only on change, ordering, `listen`, `meta: flush_handlers`, `--force-handlers`, handlers that never run |
| 31 | ★★ | [Error handling](notes/31-error-handling.md) | `ignore_errors`, `failed_when`, `block`/`rescue`/`always`, `assert`, `fail`, `any_errors_fatal`, `max_fail_percentage`, `ignore_unreachable` |
| 32 | ★★ | [Advanced Jinja2: data wrangling and lookups](notes/32-advanced-jinja2-and-lookups.md) | `map`, `selectattr`, `combine`, `dict2items`, `json_query`, `to_nice_yaml`, `ipaddr`, lookups (`file`, `env`, `template`, `password`) |

## Part 4 · Organizing and reusing code

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 33 | ★ | [Tags](notes/33-tags.md) | `--tags`, `--skip-tags`, `always`/`never`, tag inheritance, when tags help and when they hide bugs |
| 34 | ★★ | [import vs include](notes/34-import-vs-include.md) | `import_tasks`/`include_tasks`, `import_playbook`, static vs dynamic, how tags/`when`/loops behave on each, ⚠️ the removed `include:` |
| 35 | ★ | [Roles I: structure and use](notes/35-roles-basics.md) | Role directory layout, `ansible-galaxy role init`, `defaults` vs `vars`, `roles:` vs `import_role`/`include_role`, where files/templates are found |
| 36 | ★★ | [Roles II: writing reusable roles](notes/36-roles-advanced.md) | Role design, `meta/argument_specs.yml` input validation, dependencies, multi-distro roles (`ansible_facts['os_family']`), role variables naming |
| 37 | ★★ | [Collections and Galaxy](notes/37-collections-and-galaxy.md) | What a collection is, `ansible-galaxy collection install`, `requirements.yml`, version pinning, collection paths, Galaxy vs Automation Hub, trusting third-party content |
| 38 | ★★★ | [Plugins and writing your own module](notes/38-plugins-and-custom-modules.md) | Plugin types (connection, callback, lookup, filter, inventory), a custom module in Python, a custom filter, putting them in a collection |
| 39 | ★★ | [Project layout for production](notes/39-project-layout.md) | `inventories/{dev,staging,prod}`, `group_vars`, `site.yml`, roles vs collections in a repo, `ansible.cfg`, one repo or many |

## Part 5 · Secrets

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 40 | ★★ | [Ansible Vault](notes/40-ansible-vault.md) | `ansible-vault create/edit/encrypt/encrypt_string`, vault IDs per environment, password files and scripts, `no_log`, the vars + vault file pattern |
| 41 | ★★★ | [External secret managers](notes/41-external-secrets.md) | HashiCorp Vault, AWS Secrets Manager/SSM (Systems Manager) Parameter Store lookups, secrets in CI, keeping secrets out of logs and `-v` output |

## Part 6 · Inventory at scale and the cloud

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 42 | ★★ | [Dynamic inventory](notes/42-dynamic-inventory.md) | Inventory plugins, `amazon.aws.aws_ec2`, `keyed_groups`, `compose`, the `constructed` plugin, caching, ⚠️ old inventory scripts |
| 43 | ★★ | [Ansible and Terraform together](notes/43-ansible-and-terraform.md) | Who does what, the handoff (outputs/tags → inventory), ordering in a pipeline, cloud-init vs Ansible, Packer images, ⚠️ why not `remote-exec`/`local-exec` |
| 44 | ★★ | [Beyond Linux servers](notes/44-beyond-linux-servers.md) | Connection plugins for Windows (WinRM/SSH), network devices (`network_cli`), cloud APIs, containers (`community.docker`) and Kubernetes (`kubernetes.core`) |

## Part 7 · Execution control and speed

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 45 | ★★ | [Strategies, forks and serial](notes/45-strategies-forks-serial.md) | `linear` vs `free` strategy, forks, `serial` batches for rolling changes, `throttle`, `order`, `run_once` |
| 46 | ★★ | [Delegation and local actions](notes/46-delegation.md) | `delegate_to`, `delegate_facts`, `connection: local`, `run_once` + delegate, load balancer drain/enable pattern |
| 47 | ★★ | [Async tasks and long jobs](notes/47-async-and-polling.md) | `async`/`poll`, fire-and-forget, `async_status`, SSH timeouts, rebooting (`ansible.builtin.reboot`) and waiting (`wait_for`) |
| 48 | ★★★ | [Performance tuning](notes/48-performance-tuning.md) | Pipelining, SSH ControlPersist, forks, fact caching/`gather_subset`, `profile_tasks` callback, measuring before tuning |
| 49 | ★★★ | [Project 1: zero-downtime rolling deploy](notes/49-project-rolling-deploy.md) | `serial` + `delegate_to` load balancer + health checks + `max_fail_percentage` on the lab containers |

## Part 8 · Quality and testing

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 50 | ★★ | [Linting: ansible-lint and yamllint](notes/50-linting.md) | Rules and profiles, fixing common findings, `--fix`, pre-commit hooks, ansible-dev-tools and the VS Code extension |
| 51 | ★★★ | [Testing with Molecule](notes/51-testing-with-molecule.md) | Scenarios, Docker/Podman driver, converge → idempotence → verify, testing several distros, Molecule in CI |
| 52 | ★★★ | [Execution environments and ansible-navigator](notes/52-execution-environments.md) | Why "works on my laptop" breaks, EEs (container images holding ansible-core + collections + Python deps), `ansible-builder`, `ansible-navigator run` |

## Part 9 · CI/CD

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 53 | ★★ | [Ansible in GitHub Actions](notes/53-ansible-in-github-actions.md) | Lint + check on PR, apply on merge, SSH key and `known_hosts` from secrets, environments with approval, OIDC (OpenID Connect) to AWS for dynamic inventory |
| 54 | ★★ | [Ansible in Jenkins](notes/54-ansible-in-jenkins.md) | Declarative pipeline stages, SSH credentials with `sshagent`/`withCredentials`, vault password credential, Docker/EE agents, the Ansible plugin vs plain `sh` |
| 55 | ★★ | [ansible-pull and drift detection](notes/55-ansible-pull-and-drift.md) | Pull mode for large or autoscaled fleets, scheduled `--check --diff` runs as drift reports |

## Part 10 · The platform layer

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 56 | ★★ | [AWX and Ansible Automation Platform](notes/56-awx-and-aap.md) | Why a control plane: job templates, credentials, RBAC (Role-Based Access Control), schedules, workflows, surveys, EEs, Automation Hub, Event-Driven Ansible, Lightspeed; AWX vs AAP |

## Part 11 · Projects

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 57 | ★★★ | [Project 2: baseline + web stack with roles](notes/57-project-baseline-and-web-stack.md) | Mixed AlmaLinux/Ubuntu fleet: hardening role (users, SSH, firewall, time sync), nginx + app + PostgreSQL roles, vault, handlers, templates, Molecule tests |

## Part 12 · Production operations

| # | Level | Topic | What you'll learn |
|---|---|---|---|
| 58 | ★★ | [Debugging Ansible](notes/58-debugging-ansible.md) | A method; `-v` to `-vvvv`, `debug`, `ansible-console`, the task `debugger`, `ANSIBLE_KEEP_REMOTE_FILES`, callback output formats |
| 59 | ★★ | [Error catalogue](notes/59-error-catalogue.md) | UNREACHABLE, permission denied, become/sudo failures, Python missing, undefined variable, YAML indentation, host key checking, module not found, conditional not boolean… meaning + fix for each |
| 60 | ★★★ | [Security best practices](notes/60-security.md) | Least-privilege automation users, narrow sudo, bastions, `no_log`, vault, signed collections, auditing who ran what |
| 61 | ★★★ | [Scaling to many servers](notes/61-scaling.md) | Hundreds to thousands of hosts: forks, batching, `free` strategy, fact caching, pull mode, AAP instance groups and automation mesh |
| 62 | ★★★ | [Project 3: Terraform → Ansible on AWS in CI](notes/62-project-terraform-ansible-aws.md) | 💰 Terraform builds EC2 instances, dynamic inventory finds them, Ansible configures them, GitHub Actions runs it all, then `terraform destroy` |
| 63 | ★★ | [Upgrades and what's changed](notes/63-upgrades-and-changes.md) | Upgrading ansible-core safely, the 2.19 templating changes, removed/deprecated features list, reading porting guides |
| 64 | ★★ | [Production checklist](notes/64-production-checklist.md) | Everything to check before a playbook touches prod |
| 65 | ★★★ | [Production scenarios](notes/65-production-scenarios.md) | Real incidents and how to handle them: half-applied run, wrong inventory, secret leaked in logs, slow runs, drift |
| 66 | ★★ | [Interview question bank](notes/66-interview-question-bank.md) | The common questions, grouped by topic, with short answers |
| 67 | ★★★ | [Senior scenario interviews](notes/67-senior-scenario-interviews.md) | Design and troubleshooting questions with model answers and follow-ups |
| 68 | ★ | [Cheat sheet](notes/68-cheat-sheet.md) | Commands, keywords, modules and fixes on one page |
