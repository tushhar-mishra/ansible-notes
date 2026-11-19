# 56 · AWX and Ansible Automation Platform

> **Level:** ★★ intermediate
>
> **You will learn:** why teams put a **control plane** in front of Ansible (role-based access, a credentials vault, schedules, an audit trail, self-service forms, an API), the building blocks of **AWX / Automation Controller** (organizations, teams, **projects**, **inventories** and inventory sources, **credentials** and credential plugins, **job templates**, **surveys**, **workflows** with approval nodes, schedules, notifications, **execution environments**, instance groups), what **Red Hat Ansible Automation Platform (AAP)** adds (platform gateway, private **Automation Hub**, **Event-Driven Ansible**, automation mesh, **Lightspeed**, support), how AWX and AAP differ (and AWX's current release status), and how to manage the platform itself **as code** with the `awx.awx` / `ansible.controller` collections. You'll install AWX on kind and run your `~/infra` site playbook from it.
>
> **You need:** [55 · Drift detection](55-ansible-pull-and-drift.md), [52 · Execution environments](52-execution-environments.md), [41 · Secret managers](41-external-secrets.md), [42 · Dynamic inventory](42-dynamic-inventory.md), [44 · kind](44-beyond-linux-servers.md), [Kubernetes notes: Operators/CRDs](../../CKA-Certification-Course-2025/INDEX.md).
>
> **Part 10 · The platform layer** · Next: [57 · Project 2: baseline + web stack](57-project-baseline-and-web-stack.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Running `ansible-playbook` from CI works, but large organisations also need: "the network team may run **these** playbooks on **those** devices but can't see the SSH keys", "patch every Sunday at 02:00", "a form for the help desk to reset a service", "who ran what, when, with which result?". **AWX** (open source) and **Automation Controller** (its supported version inside Red Hat **AAP**) provide a web UI, REST API, RBAC, credential storage, schedules, workflows and a complete job history around your existing playbooks.

**Picture it.** Your playbooks are the recipes; AWX/AAP is the **restaurant's kitchen management system**: who may cook which dish, locked spice cabinet (credentials), the shift schedule, order tickets (jobs) and a full log of every order.

**The one thing to know.** The core objects and how they connect:

```
 Project (Git repo) ──┐
 Inventory ───────────┼──▶ Job Template ──(launch / schedule / API / workflow)──▶ Job (run in an EE)
 Credentials ─────────┤        + survey (form), limit, tags, extra vars
 Execution Env ───────┘        + RBAC: who can see / run it
```

**Try it now:** if you have 4 CPUs / 8 GB RAM free, follow the Hands-on lab to install AWX on kind. Otherwise read the Code section to see the objects.

**You've got it if you can:** explain what a job template combines and why credentials in AWX are safer than keys on laptops.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### AWX, Automation Controller, AAP

| Name | What | Support |
|---|---|---|
| **AWX** | Open-source web app + API for running Ansible centrally; the **upstream** of Automation Controller. Deployed on Kubernetes with the **AWX Operator** | Community. ⚠️ AWX releases were **paused in mid-2024** while the project is being re-architected; check the project's current status before building production on it |
| **Automation Controller** | Red Hat's supported product built from AWX | Red Hat subscription |
| **Ansible Automation Platform (AAP)** | Controller + **platform gateway** (single UI/login) + **private Automation Hub** + **Event-Driven Ansible** controller + **automation mesh** + execution environments + **Lightspeed** + certified/validated content + support | Red Hat subscription. 2.6 is the last version installable via RPM (RHEL 9); from 2.7, **containerized** installs (on RHEL) or the **OpenShift operator** |

### The building blocks

| Object | What it is | Example |
|---|---|---|
| **Organization** | Top-level tenant | `Platform`, `Network` |
| **Team / User** | People, grouped | `web-ops` team |
| **RBAC roles** | Per-object permissions: admin, use, execute, read, update, approve... | `web-ops` may **execute** "Deploy shop", not edit it |
| **Project** | A Git repo (or archive) with playbooks, synced on demand/schedule/before each job | `infra` repo, branch `main` |
| **Inventory** | Hosts and groups: static, from **inventory sources** (cloud plugins, files in the project), or **constructed** | `prod` from `aws_ec2.yml` synced hourly |
| **Credential** | Stored secret, typed: **Machine** (SSH user/key/become), **Vault**, **Source Control**, cloud (AWS, Azure, GCP), **Container Registry**, custom types | `ansible-ssh-prod` (users can **use** it, never **see** it) |
| **Credential plugins** | Fetch secrets from external managers at launch: HashiCorp Vault, CyberArk, AWS Secrets Manager, Azure Key Vault... (topic 41) | Machine credential whose key comes from HashiCorp Vault |
| **Execution Environment** | The container image jobs run in (topic 52) | `registry.example.com/platform/infra-ee:1.0` |
| **Job Template** | Project + playbook + inventory + credentials + EE + options (limit, tags, verbosity, forks, extra vars, "prompt on launch") | "Deploy shop" = `playbooks/deploy.yml` on `prod` |
| **Survey** | A form on a job template that becomes extra vars | `app_version` (text, required), `env` (choice) |
| **Workflow Job Template** | A graph of job templates (and project/inventory syncs, **approval** nodes) with success/failure/always paths | Terraform → configure → smoke test → approval → prod |
| **Schedule** | Run a template/workflow/sync on a calendar | Patching Sundays 02:00; drift check nightly |
| **Notification** | Slack, Teams, email, webhook, PagerDuty on start/success/failure/approval | |
| **Instance groups / container groups** | Where jobs run: controller nodes, execution nodes (mesh), or Kubernetes pods | Jobs for the DMZ run on an execution node in the DMZ |
| **Activity stream / job history** | Who changed which object; full output of every job | Audit |
| **REST API, `awx` CLI, collections** | Everything in the UI is in the API | Launch jobs from CI; configure the platform as code |

### What happens when a job launches

1. The controller checks RBAC (may this user execute this template with these prompts?).
2. It syncs the **project** (if "update on launch") and inventory sources (if configured).
3. It collects credentials (decrypting from its database or calling credential plugins).
4. It starts the **EE** container on a chosen instance (pod or execution node) with the project files, inventory, and credentials injected (SSH key into an agent, vault password, cloud env vars), never written into the project.
5. `ansible-runner` runs `ansible-playbook` inside it and **streams events** back.
6. The job's output, extra vars (secrets masked), who launched it, and the result are stored.

### Event-Driven Ansible (EDA)

Run automation **in response to events** instead of schedules or humans:

```yaml
# rulebook.yml (for ansible-rulebook / EDA controller)
- name: Restart service on alert
  hosts: all
  sources:
    - ansible.eda.alertmanager:          # receive Prometheus Alertmanager webhooks
        host: 0.0.0.0
        port: 5000
  rules:
    - name: nginx down
      condition: event.alert.labels.alertname == "NginxDown" and event.alert.status == "firing"
      action:
        run_job_template:
          name: "Restart nginx"
          organization: Platform
          job_args:
            limit: "{{ event.alert.labels.instance }}"
```

- **Sources**: webhooks, Kafka, Alertmanager, cloud event streams...
- **Rules**: conditions on event data.
- **Actions**: run a job template/workflow (in the controller), run a playbook, post an event.
- Runs in **decision environments** (container images, like EEs).

### Lightspeed

Red Hat Ansible **Lightspeed**: AI assistance for writing Ansible (task suggestions in VS Code from natural-language task names, playbook generation and explanation) and an intelligent assistant inside AAP for platform questions. Treat suggestions like any code: lint, review and test them.

### AWX/AAP as code

Clicking objects together in a UI creates the same "snowflake" problem as configuring servers by hand. Manage the platform's objects from Git with the **`awx.awx`** (community) or **`ansible.controller`** (certified) collection, or the "infra.controller_configuration"-style roles:

```yaml
- name: Job template is defined
  awx.awx.job_template:
    name: Deploy shop
    organization: Platform
    project: infra
    playbook: playbooks/deploy.yml
    inventory: lab
    credentials: [ansible-ssh-lab, vault-lab]
    execution_environment: infra-ee
    ask_variables_on_launch: false
    survey_enabled: true
    survey_spec: "{{ lookup('ansible.builtin.file', 'surveys/deploy.json') }}"
    state: present
```

---

## Why it exists

| Need | CLI/CI alone | AWX/AAP |
|---|---|---|
| Who may run what, on which hosts | Repo access ≈ all-or-nothing | Fine-grained RBAC per template/inventory/credential |
| Credentials | Secrets in CI, keys on laptops | Encrypted store, **use without see**, external secret plugins |
| Self-service for non-Ansible people | Teach them CI | Job templates + surveys (forms) |
| Schedules | Cron in CI | Built-in schedules with history |
| Multi-step orchestration with approvals | Pipeline YAML | Visual workflows with approval nodes |
| Audit | CI logs, scattered | Central job history + activity stream |
| Running close to targets in many networks | Self-hosted runners per network | Automation mesh execution/hop nodes |
| Reacting to events | Custom glue | Event-Driven Ansible |

Many organisations use **both**: CI for lint/test/PR plans, AWX/AAP for running approved automation in production (CI calls AWX's API to launch job templates).

---

## Analogy

**An airport operations centre.** Pilots (playbooks) still fly the planes. The operations centre (AWX/AAP) decides which crew may fly which route (RBAC), keeps the keys to the planes in a locked office (credentials), publishes the timetable (schedules), runs multi-leg journeys with checkpoints (workflows with approvals), logs every flight (job history), and reacts automatically to weather alerts (Event-Driven Ansible).

---

## How it works inside

```
 User / API / schedule / workflow / EDA rule
        │ launch "Deploy shop" (survey: app_version=1.2)
        ▼
 Controller (web + API + task manager, PostgreSQL)
   ├─ RBAC check
   ├─ project sync (git) → project files
   ├─ inventory (static or synced from plugins)
   ├─ credentials (DB, encrypted; or credential plugins → HashiCorp/CyberArk/AWS SM)
   └─ dispatch to an instance group:
          Kubernetes pod (container group)   or   execution node via automation mesh (receptor)
             └─ EE container: ansible-runner → ansible-playbook  (SSH key in agent, vault pw injected)
                   → events streamed back → stored, shown live in the UI
```

AWX on Kubernetes runs as pods (web, task, PostgreSQL) managed by the **AWX Operator** (a Kubernetes operator: a controller that installs and maintains an app from a custom resource).

---

## Code / commands

### Lab: AWX on kind (needs ~4 CPUs, 8 GB RAM free)

**1. Cluster and operator** (operator version pinned; check the AWX Operator releases page for the current one):

```bash
kind create cluster --name awx
mkdir -p ~/awx && cd ~/awx
cat > kustomization.yaml <<'EOF'
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: awx
resources:
  - github.com/ansible/awx-operator/config/default?ref=2.19.1
  - awx.yaml
images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.19.1
EOF
cat > awx.yaml <<'EOF'
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-demo
spec:
  service_type: clusterip
EOF
kubectl create namespace awx
kubectl apply -k .
kubectl -n awx get pods -w            # wait until awx-demo-web/-task/-postgres are Running (several minutes)
```

- `kind: AWX`: a **custom resource**; the operator watches it and creates the deployments, services and database.

**2. Log in:**

```bash
kubectl -n awx get secret awx-demo-admin-password -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n awx port-forward svc/awx-demo-service 8043:80
```

Open `http://localhost:8043`, user `admin`.

**3. Let AWX reach the lab and your Git repo:**

```bash
docker network connect ansible-lab awx-control-plane     # the kind node joins the lab network
kind load docker-image infra-ee:1.0 --name awx           # make your EE available to the cluster
# Git: reuse the git daemon from topic 55 (git://172.30.0.1/infra)
```

**4. Create objects** (UI, or as code below):

| Object | Settings |
|---|---|
| Credential **Machine** `ansible-ssh-lab` | Username `ansible`, SSH private key = `~/.ssh/ansible_key`, privilege escalation method `sudo` |
| Credential **Vault** `vault-lab` | Vault password = lab vault password, Vault identifier `lab` |
| Execution Environment `infra-ee` | Image `infra-ee:1.0`, pull = **Never** |
| Project `infra` | SCM type Git, URL `git://172.30.0.1/infra`, branch `main`, EE `infra-ee` |
| Inventory `lab` | Source: **from project**, file `inventories/lab` |
| Job Template `Site` | Inventory `lab`, project `infra`, playbook `playbooks/site.yml`, credentials `ansible-ssh-lab` + `vault-lab`, EE `infra-ee` |

Launch **Site**. The job output looks exactly like `ansible-playbook`, plus host summaries, filters and a stored history.

Since the lab containers' host keys aren't known to AWX, set `ANSIBLE_HOST_KEY_CHECKING=False` in the job template's **environment variables** for the lab only (or add your `known_hosts` handling to the EE). Never do that for production.

**5. A survey and a schedule:**

- Job template **Deploy shop** → playbook `playbooks/deploy.yml` → Survey: `app_version` (text, required, default `1.1`).
- Job template **Drift check** → playbook `playbooks/site.yml`, Job Type **Check**, Show changes (diff) on → Schedule: daily at 03:00.

**6. A workflow with approval:**

Workflow "Release": `Project sync` → `Deploy shop (lab)` → **Approval** ("Promote to prod?") → `Deploy shop (prod)` → on failure: `Notify` (a debug job or a webhook notification).

### Configuration as code with `awx.awx`

```bash
cd ~/infra
ansible-galaxy collection install awx.awx -p ./collections
export CONTROLLER_HOST=http://localhost:8043 CONTROLLER_USERNAME=admin CONTROLLER_PASSWORD='<admin password>'
```

`playbooks/awx-config.yml`:

```yaml
---
- name: AWX objects as code
  hosts: localhost
  connection: local
  gather_facts: false
  vars:
    org: Default
  tasks:
    - name: Execution environment
      awx.awx.execution_environment:
        name: infra-ee
        image: infra-ee:1.0
        pull: never

    - name: Project
      awx.awx.project:
        name: infra
        organization: "{{ org }}"
        scm_type: git
        scm_url: git://172.30.0.1/infra
        scm_branch: main
        default_environment: infra-ee
        update_project: true
        wait: true

    - name: Machine credential (key read from the control node, never committed)
      awx.awx.credential:
        name: ansible-ssh-lab
        organization: "{{ org }}"
        credential_type: Machine
        inputs:
          username: ansible
          ssh_key_data: "{{ lookup('ansible.builtin.file', lookup('ansible.builtin.env', 'HOME') ~ '/.ssh/ansible_key') }}"
          become_method: sudo
      no_log: true

    - name: Inventory and its source from the project
      awx.awx.inventory:
        name: lab
        organization: "{{ org }}"

    - name: Inventory source
      awx.awx.inventory_source:
        name: lab-from-project
        inventory: lab
        source: scm
        source_project: infra
        source_path: inventories/lab
        update_on_launch: true

    - name: Job template
      awx.awx.job_template:
        name: Site
        organization: "{{ org }}"
        project: infra
        inventory: lab
        playbook: playbooks/site.yml
        credentials: [ansible-ssh-lab]
        execution_environment: infra-ee
        extra_vars: {}
        state: present

    - name: Launch it and wait
      awx.awx.job_launch:
        job_template: Site
        wait: true
      register: job

    - name: Result
      ansible.builtin.debug:
        msg: "Job {{ job.id }} finished with status {{ job.status }}"
```

```bash
ansible-playbook playbooks/awx-config.yml
```

Run it twice: the object tasks report `ok` (idempotent, like any module), and the platform's configuration now lives in Git.

### Launch from CI

```bash
# e.g. a GitHub Actions/Jenkins step after merge
awx --conf.host https://aap.example.com --conf.token "$AAP_TOKEN" \
  job_templates launch "Deploy shop" --extra_vars '{"app_version": "1.2"}' --monitor
```

CI keeps doing lint/tests/plans; the controller holds production credentials and RBAC.

---

## Hands-on lab

1. Install AWX on kind (or, if resources are tight, read through and do steps 5–6 later on a bigger machine).
2. Create the objects by hand once, launch **Site**, and explore the job output (host filters, task view, event details).
3. Recreate them with `awx-config.yml`; delete one object in the UI and re-run the playbook to restore it.
4. Add a survey to a **Deploy shop** template and launch it with version 1.2.
5. Create a **Check** job template with diff and schedule it daily: that's topic 55's drift report, in the platform.
6. Create a **workflow** with an approval node and test approve/deny paths.
7. Create a user `helpdesk` with **execute** permission on one template only; log in as that user and confirm they can't see credentials or edit templates.
8. Clean up: `kind delete cluster --name awx`.

**Done when:** you can explain each AWX object, configure them as code, and run your playbooks through the platform with RBAC.

---

## Where you'll use it

- **Enterprises with many teams**: AAP is common wherever Red Hat is the Linux vendor.
- **Self-service ops**: help desk/developers run approved actions through forms without SSH access.
- **Scheduled operations**: patching windows, drift checks, certificate renewals, backups.
- **Event-driven remediation**: restart services, open tickets, scale out on alerts (EDA).
- **Regulated environments**: RBAC + audit trail + credential isolation.
- **Interviews**: "What is Ansible Tower/AWX/AAP?" (Tower is the old name of Automation Controller).

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| Project sync fails | Git URL unreachable from the controller/pods, missing Source Control credential | Network/credentials; check the sync job output |
| Job `UNREACHABLE` | Execution pod/node can't reach targets | Instance/container groups placed in the right network; mesh execution node near the targets |
| `couldn't resolve module` in jobs | The EE lacks the collection | Rebuild the EE or add `collections/requirements.yml` to the project (installed at sync) |
| Image pull errors for the EE | Registry auth or pull policy | Container Registry credential; `pull: missing/never` for local images |
| Host key errors | known_hosts not in the EE/job | Bake trusted known_hosts into the EE, use SSH certificates; disable checking only in labs |
| Users can see secrets in extra vars | Secrets passed as plain extra vars/surveys | Use credentials (custom credential types) or password-type survey fields |
| Pods pending on kind | Not enough CPU/RAM | More resources, or a smaller test (read-only) |
| Configuration drift in the platform itself | Objects created by hand | Manage with `awx.awx`/`ansible.controller` from Git |
| AWX upgrade path unclear | AWX release pause / operator changes | Check upstream status; for supported production, AAP |

---

## Quick recap

- **AWX** (open source, upstream; releases paused since mid-2024) and **Automation Controller/AAP** (supported) put a web UI, API, **RBAC**, credentials, schedules, workflows and audit around your playbooks.
- Objects: **projects** (Git), **inventories** (+ sources), **credentials** (use without see; external plugins), **execution environments**, **job templates** (+ **surveys**), **workflows** (+ approvals), schedules, notifications, instance/container groups.
- Jobs run `ansible-runner` inside an **EE**, credentials injected at launch, events streamed and stored.
- AAP adds a gateway, private **Automation Hub**, **Event-Driven Ansible** (rulebooks: sources → rules → actions), automation mesh, **Lightspeed**, certified content and support.
- Configure the platform **as code** with `awx.awx`/`ansible.controller`; CI launches job templates via the API.

### Interview questions

1. **Q: What is AWX / Ansible Automation Platform and why use it?**
   A: A control plane around Ansible: web UI and REST API, RBAC, encrypted credential storage, job templates with surveys, workflows, schedules, notifications and full job history. AWX is the open-source upstream; AAP is Red Hat's supported platform adding Automation Hub, Event-Driven Ansible, mesh and Lightspeed.
2. **Q: What's in a job template?**
   A: A project (Git repo) and playbook, an inventory, credentials, an execution environment, and run options (limit, tags, verbosity, extra vars, prompts, survey).
3. **Q: What is Event-Driven Ansible?**
   A: Rulebooks that listen to event sources (webhooks, Alertmanager, Kafka...), evaluate conditions and trigger actions such as launching job templates, for automated responses to events.

### Scenario question (answer out loud first, then compare)

> *"Your company wants the help desk to restart a specific application service on production servers, without giving them SSH access or seeing any credentials, with every action logged. Design it on AAP."*

**Model answer:**
1. A small **operational playbook** in Git: assert the target is in the app's group, restart the service with `serial` and a health check, notify on failure (topics 31, 45).
2. **Job template** "Restart app" using that playbook, the prod inventory (or a constructed inventory with only the app hosts), a **Machine credential** for the automation user, and the standard EE.
3. **Survey**: a dropdown of allowed hosts or groups (not free text), and a reason/ticket number field (stored in the job's extra vars).
4. **RBAC**: a `helpdesk` team gets **execute** on this template only; no access to credentials, inventories, or other templates.
5. **Notifications** to the app team's channel on start/failure; **audit** through job history (who, when, which hosts, result, ticket number).
6. Everything defined **as code** with `ansible.controller` and reviewed through PRs.

**Likely follow-ups:**
- *How do you stop them restarting all servers at once?* The playbook enforces `serial: 1` and a `limit` from the survey; an assert rejects "all".
- *What if the credentials must come from CyberArk?* Use a credential plugin so the controller fetches the key at launch time.
