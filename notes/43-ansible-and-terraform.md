# 43 · Ansible and Terraform Together

> **Level:** ★★ intermediate
>
> **You will learn:** how to split work between **Terraform** (create infrastructure) and **Ansible** (configure what runs on it), the full lifecycle (provision → bootstrap → configure → day-2 operations → destroy), the **hand-off options** from Terraform to Ansible (generated inventory from `terraform output`, **tags + dynamic inventory**, the `cloud.terraform` collection's state inventory, Terraform writing the inventory file), waiting for new servers to be ready, ordering both in one pipeline, the **immutable** alternative (Packer + Ansible images), Ansible *calling* Terraform, and the anti-patterns (`remote-exec`/`local-exec` provisioners, both tools owning the same thing). You'll create servers with Terraform's Docker provider and configure them with Ansible.
>
> **You need:** [42 · Dynamic inventory](42-dynamic-inventory.md), [02 · Tool landscape](02-tool-landscape.md), your Terraform notes: [03 · What is Terraform](../../Terraform/notes/03-what-is-terraform.md), [15 · Outputs](../../Terraform/notes/15-outputs.md), [26 · Docker provider](../../Terraform/notes/26-docker-provider.md), [52 · Provisioners and alternatives](../../Terraform/notes/52-provisioners-and-alternatives.md).
>
> **Part 6 · Inventory at scale and the cloud** · Next: [44 · Beyond Linux servers](44-beyond-linux-servers.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** **Terraform owns what exists** (VMs, networks, firewalls, DNS, load balancers), with a state file and a plan. **Ansible owns what's inside** (packages, config, users, app deploys). Terraform hands Ansible a **list of servers**, ideally through **tags** that Ansible's dynamic inventory reads.

**Picture it.** A construction company builds the house and fits the door locks (Terraform). It gives the interior designer (Ansible) the address and the key (inventory + SSH key). The designer furnishes it, and comes back every month to keep it tidy.

**The one thing to know.** The pipeline order:

```
terraform plan → terraform apply          (VMs exist, tagged role=web, key installed)
        │
        ▼ hand-off: tags (dynamic inventory)  or  terraform output → inventory file
ansible-playbook site.yml                 (wait_for_connection → configure → deploy)
```

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = var.ami
  instance_type = "t3.small"
  key_name      = "ops"
  tags = { Name = "web-${count.index}", env = "prod", role = "web" }   # ← the hand-off
}
```

```yaml
# inventories/prod/aws_ec2.yml (topic 42)
keyed_groups:
  - { key: tags.role, prefix: role }        # → group role_web
```

**Try it now:** answer for your own projects: which resources belong in Terraform, which settings in Ansible? Anything managed by **both** is a bug waiting to happen.

**You've got it if you can:** name three ways to hand servers from Terraform to Ansible and say which you'd pick in AWS.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Who owns what

| Layer | Owner | Examples |
|---|---|---|
| Accounts, networks, security | **Terraform** | VPC, subnets, security groups, IAM roles, KMS keys |
| Compute and managed services | **Terraform** | EC2 instances, autoscaling groups, RDS, load balancers, DNS records |
| First-boot bootstrap | **cloud-init** (user data, set by Terraform) | Automation user + SSH key, install Python, hostname |
| OS configuration | **Ansible** | Packages, users, sudo, sshd, sysctl, time sync, firewalld, agents |
| App deployment and day-2 ops | **Ansible** | Releases, config files, rolling restarts, patching, rotations |
| Golden images (immutable) | **Packer** running **Ansible** | AMI with everything pre-installed |

**One owner per thing.** If Terraform manages a security group rule and Ansible also edits it, each run undoes the other ("flapping"). If Terraform writes an nginx config via user data and Ansible templates it later, you have two sources of truth.

### The lifecycle

```
 1. PROVISION   terraform apply      → instances, networks, LBs; tags env/role; outputs
 2. BOOTSTRAP   cloud-init (first boot only)
                                     → ansible user, SSH key, python3 (topic 07)
 3. CONFIGURE   ansible-playbook     → wait_for_connection, baseline, app (idempotent)
 4. DAY 2       ansible (scheduled/on demand) → patch, rotate, drift repair, deploys
                terraform (on change) → resize, add instances, new LB rules
 5. DESTROY     terraform destroy    → gone; dynamic inventory forgets them automatically
```

### Hand-off options

| # | Method | How | Pros | Cons |
|---|---|---|---|---|
| 1 | **Tags + dynamic inventory** | Terraform tags resources; `aws_ec2`/`azure_rm`/`gcp_compute` plugin groups by tag | ✅ Always current, no files, works with autoscaling | Needs cloud API access + credentials for Ansible |
| 2 | **Generated inventory from outputs** | `terraform output -json` → a script/template → `hosts.yml` | Simple, works anywhere (even on-prem) | A file to regenerate after every apply; stale if forgotten |
| 3 | **Terraform writes the inventory** | `local_file` resource rendering `templatefile(...)` | Inventory updated in the same apply | Inventory becomes a Terraform artifact; same staleness if apply isn't run |
| 4 | **Read Terraform state** | `cloud.terraform.terraform_state` inventory plugin (reads resources from state), or the `ansible/ansible` Terraform provider (`ansible_host`/`ansible_group` resources) + `cloud.terraform.terraform_provider` inventory plugin | No extra files; uses what Terraform knows | Ansible needs access to the state (remote backend credentials); couples the tools |
| 5 | ⚠️ **Terraform runs Ansible** (`local-exec` provisioner calling `ansible-playbook`) | In the resource | One command | Runs only at creation, failures taint resources, no re-runs for drift, slow applies. Terraform itself calls provisioners a **last resort** |
| 6 | ⚠️ `remote-exec` | Terraform SSHs in and runs commands | | Same problems, plus not idempotent |

**Recommendation:** cloud → **#1 (tags)**. On-prem or simple labs → **#2/#3**. Avoid #5/#6.

### Waiting for new servers

A freshly created VM needs a minute to boot, run cloud-init and start sshd. Start the play with:

```yaml
- name: Configure new servers
  hosts: role_web
  gather_facts: false              # facts need SSH + Python, not ready yet
  tasks:
    - name: Wait for SSH and Python
      ansible.builtin.wait_for_connection:
        timeout: 300
    - name: Wait for cloud-init to finish
      ansible.builtin.command: cloud-init status --wait
      changed_when: false
    - name: Now gather facts
      ansible.builtin.setup:
```

### Host keys for brand-new servers

New instances have new host keys (topic 05). Options: `StrictHostKeyChecking=accept-new` for the first contact (common for freshly created cloud hosts in a pipeline), host keys from cloud-init's console output collected by Terraform, SSH certificates, or no SSH at all (SSM connection, topic 44).

### The immutable alternative: Packer + Ansible

```
 packer build  ──▶ temporary VM ──▶ Ansible provisioner runs your roles ──▶ snapshot = AMI "web-2026-11-19"
 terraform apply (launch template uses the new AMI) ──▶ autoscaling group replaces instances
 Ansible at runtime: only small, instance-specific steps (or none)
```

- Pros: fast scale-out (instances boot ready), identical servers, no drift.
- Cons: every change = a new image + a rollout; harder for quick fixes.
- Many teams mix: **images** for the heavy, slow-changing parts; **Ansible at boot** (or `ansible-pull`, topic 55) for the last mile.

### Ansible calling Terraform

Sometimes one Ansible playbook orchestrates everything (e.g. in AWX): `cloud.terraform.terraform` (or `community.general.terraform`) runs `terraform plan/apply` as a task, then `meta: refresh_inventory` reloads the dynamic inventory, then configuration plays follow. Fine for orchestration, but Terraform still owns the resources and the state.

---

## Why it exists

Each tool is excellent at one job and awkward at the other (topic 02). Terraform's state and plans are perfect for creating and destroying infrastructure, but it can't keep the inside of 200 servers converged over months. Ansible is perfect for that, but has no state to know which cloud resources it created. Combining them, with a clean hand-off, gives you planned infrastructure **and** continuously managed servers.

---

## Analogy

**Building and running a hotel.** The **builder** (Terraform) follows architectural plans, keeps the blueprints (state), and is the only one allowed to move walls. The **hotel manager** (Ansible) gets the list of rooms from the building's directory board (tags/inventory), furnishes them, and keeps them clean every day. The builder doesn't arrange furniture; the manager doesn't knock down walls. A **prefab hotel** (Packer images) arrives with rooms already furnished; the manager only adds fresh flowers.

---

## How it works inside

A typical CI pipeline (topics 53–54 implement it):

```
 PR opened:   terraform fmt/validate/plan  +  ansible-lint  +  ansible-playbook --check (existing hosts)
 merge:       terraform apply  (tags: env, role; outputs)
                  │
                  ▼
              ansible-inventory --graph      (dynamic: new hosts appear by tag)
              ansible-playbook site.yml      (wait_for_connection → baseline → app)
                  │
                  ▼
              smoke tests (uri to the LB)
 schedule:    ansible-playbook site.yml --check --diff  (drift report)
```

Terraform and Ansible keep **separate** state: Terraform's state file; Ansible's "state" is the real servers. The tags are the contract between them.

---

## Code / commands

### Lab: Terraform creates containers, Ansible configures them

Terraform's Docker provider (your [Terraform topic 26](../../Terraform/notes/26-docker-provider.md)) creates two new "servers" from the lab images on the lab network. Ansible then configures them. Make sure the lab images exist (`~/ansible-lab/nodes/reset.sh` built them).

`~/infra/terraform/main.tf`:

```hcl
terraform {
  required_providers {
    docker = {
      source  = "kreuzwerker/docker"
      version = "~> 3.6"
    }
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}

provider "docker" {}

variable "app_count" {
  type    = number
  default = 2
}

resource "docker_container" "app" {
  count      = var.app_count
  name       = "tf-app${count.index + 1}"
  image      = "ansible-lab/alma10"
  hostname   = "tf-app${count.index + 1}"
  privileged = true
  cgroupns_mode = "host"
  restart    = "unless-stopped"

  networks_advanced {
    name         = "ansible-lab"
    ipv4_address = "172.30.0.${51 + count.index}"
  }
  volumes {
    host_path      = "/sys/fs/cgroup"
    container_path = "/sys/fs/cgroup"
    read_only      = false
  }
  tmpfs = { "/run" = "", "/run/lock" = "", "/tmp" = "" }

  labels {
    label = "role"
    value = "app"
  }
  labels {
    label = "env"
    value = "lab"
  }
}

output "app_hosts" {
  value = { for c in docker_container.app : c.name => c.network_data[0].ip_address }
}

resource "local_file" "inventory" {
  filename = "${path.module}/../inventories/tf/hosts.yml"
  content = templatefile("${path.module}/inventory.yml.tftpl", {
    hosts = { for c in docker_container.app : c.name => c.network_data[0].ip_address }
  })
}
```

- `labels` = Docker's equivalent of cloud **tags**: the hand-off contract.
- `output "app_hosts"`: names → IPs, for option #2.
- `local_file` + `templatefile`: option #3, Terraform writes an inventory file.
- If the provider rejects an attribute (provider versions differ), check its docs; the systemd settings mirror the Compose file from topic 09.

`~/infra/terraform/inventory.yml.tftpl`:

```
# Generated by Terraform. Do not edit.
all:
  children:
    tf_app:
      hosts:
%{ for name, ip in hosts ~}
        ${name}:
          ansible_host: ${ip}
%{ endfor ~}
```

(`%{ for }` / `${}` is Terraform's template syntax, not Jinja2.)

```bash
cd ~/infra/terraform
terraform init
terraform apply -auto-approve
terraform output -json app_hosts
cat ../inventories/tf/hosts.yml
```
```
Apply complete! Resources: 3 added, 0 changed, 0 destroyed.
Outputs:
app_hosts = {
  "tf-app1" = "172.30.0.51"
  "tf-app2" = "172.30.0.52"
}
```
```yaml
# Generated by Terraform. Do not edit.
all:
  children:
    tf_app:
      hosts:
        tf-app1:
          ansible_host: 172.30.0.51
        tf-app2:
          ansible_host: 172.30.0.52
```

Give that environment the connection settings (copy `inventories/lab/group_vars/all/main.yml` connection lines into `inventories/tf/group_vars/all.yml`), then configure:

`~/infra/playbooks/tf-app.yml`:

```yaml
---
- name: Configure servers created by Terraform
  hosts: tf_app
  gather_facts: false
  become: true
  tasks:
    - name: Wait until they accept SSH and run Python
      ansible.builtin.wait_for_connection:
        timeout: 120

    - name: Gather facts now
      ansible.builtin.setup:

    - name: Apply the web role
      ansible.builtin.include_role:
        name: nginx_web
      vars:
        nginx_web_sites:
          - { name: app, title: "Configured by Ansible after Terraform on {{ inventory_hostname }}" }
```

```bash
cd ~/infra
ansible-playbook -i inventories/tf playbooks/tf-app.yml
curl -s 172.30.0.51:8081/
```
```
Configured by Ansible after Terraform on tf-app1
```

### The same hand-off with labels (dynamic, option #1)

Add to the Docker dynamic inventory from topic 42 (`inventories/docker/lab.docker.yml`) a group from the `role` label:

```yaml
keyed_groups:
  - key: docker_config.Labels['role'] | default('none')
    prefix: role
```

Relax its filter (or add a second config file) so `tf-app*` containers are included, then:

```bash
ansible-inventory -i inventories/docker --graph role_app
```

No generated file: the label set by Terraform **is** the inventory.

### Scale and destroy

```bash
cd ~/infra/terraform
terraform apply -auto-approve -var app_count=3      # tf-app3 appears, inventory file regenerated
cd .. && ansible-playbook -i inventories/tf playbooks/tf-app.yml   # only tf-app3 shows changes
cd terraform && terraform destroy -auto-approve      # containers gone, inventory file removed
```

---

## Hands-on lab

1. Run the Terraform → Ansible flow above. Curl both new servers.
2. Run the playbook again: `changed=0` (idempotent; Terraform didn't touch the inside).
3. Scale to 3 with Terraform, re-run Ansible. Then delete the generated `hosts.yml` by hand and run Ansible: it fails until Terraform regenerates it. That's the weakness of file-based hand-offs.
4. Switch to the label-based dynamic inventory and repeat the scale-up. No file involved.
5. Introduce an ownership conflict on purpose: in Terraform add `env = ["NGINX_PORT=9999"]` to the container and in Ansible keep port 8081. Discuss why mixing owners is confusing (Terraform's change forces a **container replacement**, which wipes Ansible's work until the next run). Remove it.
6. `terraform destroy`, then check `ansible-inventory -i inventories/docker --graph role_app` is empty.
7. Write the CI sequence for this lab as a list of shell commands (you'll implement it in topic 53).

**Done when:** you can rebuild the whole "app tier" with `terraform apply && ansible-playbook ...` and tear it down with `terraform destroy`.

---

## Where you'll use it

- **Almost every cloud team**: Terraform for AWS/Azure/GCP resources, Ansible for OS and apps.
- **On-prem**: Terraform for vSphere/Proxmox VMs, Ansible for configuration.
- **Golden images**: Packer + Ansible, rolled out by Terraform launch templates.
- **AWX/AAP workflows**: a job template running Terraform, then configuration jobs (topic 56).
- **Interviews**: "How do Terraform and Ansible work together?" is one of the most common questions.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `UNREACHABLE` right after `terraform apply` | Instances still booting / cloud-init running | `wait_for_connection` with `gather_facts: false` first; `cloud-init status --wait` |
| `Host key verification failed` on new instances | New host keys, CI has no TTY | `accept-new` for fresh hosts, or SSH certificates / SSM |
| New instances not configured | Static inventory not regenerated | Tags + dynamic inventory; or regenerate in the same pipeline step |
| Config changes undone after the next Terraform apply | Terraform recreated the instance (e.g. changed user data/AMI) | Expect replacement; run Ansible after every apply; or bake config into images |
| Terraform and Ansible "fight" over a setting | Both manage it | One owner; remove it from the other tool |
| `terraform apply` hangs/fails because of Ansible | `local-exec` provisioner running Ansible | Move Ansible to a separate pipeline step |
| Ansible can't read Terraform state (state inventory plugin) | No backend credentials | Prefer tags; or grant read-only state access |
| Wrong environment configured | Tags missing/typo (`Env` vs `env`) | Enforce tags in Terraform (variables + validation, default_tags); `strict: true` in the inventory |

---

## Quick recap

- **Terraform** creates/changes/destroys infrastructure (state, plans); **Ansible** configures and operates what runs on it; **cloud-init** does the minimal first-boot bootstrap; **Packer + Ansible** bakes images.
- Hand-off: **tags + dynamic inventory** (best in cloud), `terraform output` → inventory, Terraform-written inventory file, Terraform state inventory. ⚠️ Avoid `local-exec`/`remote-exec` provisioners for configuration.
- Pipeline: plan → apply → `wait_for_connection` → configure → smoke test; scheduled check runs for drift.
- **One owner per resource or setting.**

### Interview questions

1. **Q: How do Terraform and Ansible work together?**
   A: Terraform provisions infrastructure and tags it (or outputs addresses); Ansible discovers those hosts through a dynamic inventory (or a generated inventory) and configures them. They run as separate pipeline steps, each owning its own layer.
2. **Q: Why not run Ansible from a Terraform provisioner?**
   A: Provisioners run only at creation, failures leave resources tainted, there's no way to re-run for drift without recreating, and they slow and complicate applies. HashiCorp recommends them only as a last resort.
3. **Q: How do you make Ansible wait for newly created servers?**
   A: Start the play with `gather_facts: false` and `ansible.builtin.wait_for_connection`, optionally wait for cloud-init, then gather facts.

### Scenario question (answer out loud first, then compare)

> *"Your company runs 300 EC2 instances built by Terraform. Configuration is done by Ansible from a laptop with a static inventory that's always out of date. Design the target setup."*

**Model answer:**
1. **Terraform** (in CI with remote state, topic 35 of your Terraform notes) creates instances with mandatory tags (`env`, `role`, `owner`) via `default_tags` and validation; user data/cloud-init installs only the automation user/key or relies on SSM.
2. **Ansible** in a repo with `inventories/<env>/aws_ec2.yml` (filters by `env`, `keyed_groups` by `role`), `group_vars/role_*.yml`, roles, pinned collections.
3. **Pipelines**: Terraform job → Ansible job (`wait_for_connection`, baseline, app) for the changed environment; scheduled `--check --diff` drift reports; patching playbook with `serial`.
4. **Access**: runners inside the VPC or SSM connection; OIDC roles for AWS API and secrets; no laptop runs against prod.
5. **Optionally**: Packer + Ansible images for faster autoscaling, with Ansible for the last mile; AWX/AAP for RBAC and schedules (topic 56).

**Likely follow-ups:**
- *How do you migrate from the static inventory?* Tag existing instances (Terraform import or tagging script), build the dynamic inventory, compare `--list-hosts` outputs between old and new, then switch.
- *Who updates the load balancer when instances change?* Terraform for target group membership with ASGs; Ansible only for in-host config.
