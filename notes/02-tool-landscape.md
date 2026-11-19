# 02 · The Tool Landscape: Ansible vs Puppet/Chef/Salt vs Terraform

> **Level:** ★ foundation
>
> **You will learn:** the two different jobs automation tools do (**provisioning** vs **configuration management**), how Ansible compares with Puppet, Chef and Salt, why **Ansible and Terraform are not competitors**, how they hand work to each other, where cloud-init, Packer, Dockerfiles and Kubernetes fit, and how to answer "Terraform or Ansible?" in an interview.
>
> **You need:** [01 · What is configuration management?](01-what-is-configuration-management.md)
>
> **Part 0 · Foundations** · Next: [03 · What is Ansible?](03-what-is-ansible.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Automation has two big jobs:

- **Provisioning** = *create* the infrastructure: VMs (virtual machines), networks, load balancers, databases, DNS (Domain Name System) records. Terraform's job.
- **Configuration management** = *set up the inside* of servers that already exist: packages, users, config files, services. Ansible's job.

**Picture it.** Building a house: **Terraform is the construction company** (foundation, walls, plumbing). **Ansible is the interior team** (furniture, paint, Wi-Fi, keys for each tenant). You need both, in that order.

**The one thing to know.** "Terraform **or** Ansible?" is the wrong question. Usual answer: **Terraform creates the servers, then Ansible configures them.** Terraform keeps a **state** file (its memory of what it built) and talks to cloud **APIs** (Application Programming Interfaces). Ansible keeps **no state** and talks to servers over **SSH** (Secure Shell).

**Try it now:** open 3 DevOps job ads. For each tool named, write "provision", "configure", "build image" or "orchestrate containers". You'll usually see Terraform + Ansible, or Terraform + Kubernetes + Helm.

**You've got it if you can:** answer "Terraform or Ansible?" in two sentences.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Two jobs, two kinds of tools

| | **Provisioning** | **Configuration management** |
|---|---|---|
| **Question it answers** | "Does this server/network/database **exist**?" | "Is everything **inside** this server set up correctly?" |
| **Talks to** | Cloud APIs (AWS = Amazon Web Services, Azure, GCP = Google Cloud Platform), over HTTPS | The servers themselves, over SSH (or WinRM, Windows Remote Management, for Windows) |
| **Examples of what it manages** | EC2 (Elastic Compute Cloud) instance = an AWS VM, VPC (Virtual Private Cloud), security group (an AWS firewall), RDS (Relational Database Service) database, S3 (Simple Storage Service) bucket | `nginx` package, `/etc/nginx/nginx.conf`, `deploy` user, `chronyd` service |
| **Main tools** | Terraform, OpenTofu, CloudFormation, Pulumi, Bicep | Ansible, Puppet, Chef, Salt |
| **Remembers what it did?** | Yes: a **state** file | Ansible: no. Puppet/Chef: a central server keeps reports |

The line isn't perfect. Ansible *can* create cloud VMs (it has `amazon.aws.ec2_instance`), and Terraform *can* run commands on servers (provisioners, which Terraform itself calls a last resort). Each is **good** at one job and **okay** at the other.

### The configuration management tools

| Tool | Year | Language you write | Agent on servers? | Model | Today |
|---|---|---|---|---|---|
| **CFEngine** | 1993 | Its own language | Yes | Pull | Niche. The grandparent of all of them |
| **Puppet** | 2005 | Puppet DSL (Domain-Specific Language) | Yes (`puppet agent`) + a Puppet server | Pull, every 30 min | Still in big enterprises |
| **Chef** | 2009 | Ruby | Yes (`chef-client`) + Chef server | Pull | Shrinking. Now owned by Progress |
| **Salt** (SaltStack) | 2011 | YAML + Jinja2 | Yes (`salt-minion`), agentless mode exists | Push over a message bus | Owned by Broadcom (via VMware). Smaller community |
| **Ansible** | 2012 | YAML (+ Jinja2 templates) | **No**: SSH + Python | **Push** (pull possible) | **The most used** CM tool. Owned by Red Hat (IBM) |

**Why Ansible won the popularity contest:**

1. **Agentless.** No agent to install, upgrade, secure or monitor. If you can SSH to it, you can manage it. This also means you can manage a server **on day one**, before anything is installed.
2. **YAML, not a programming language.** Ops people, developers and even network engineers can read a playbook.
3. **Ordered tasks.** Tasks run top to bottom, like a script. Easy to understand. Easy to do multi-step things: "drain server from load balancer → update → health check → add back".
4. **Batteries included.** Thousands of modules for Linux, Windows, network devices, cloud, Kubernetes.
5. **Red Hat backing** and RHEL integration (RHEL System Roles, Ansible Automation Platform).

**Where Ansible is weaker:** with no agent, nothing fixes drift *between* runs unless you schedule runs. Very large fleets (10,000+ servers) need extra design (topic 61). And no state file means Ansible can't easily answer "what did I create that's no longer in my code?"

### Ansible vs Terraform in detail

This comparison is asked in almost every DevOps interview. Learn this table.

| | **Terraform** | **Ansible** |
|---|---|---|
| **Main job** | Provision infrastructure | Configure servers, deploy apps, orchestrate steps |
| **Style** | Declarative: describe the end result, Terraform works out the order | Mostly declarative **modules**, run in the **order you write** (procedural flow) |
| **State** | Keeps a **state file** of everything it created | **Stateless**: checks the real server every run |
| **Deletes things removed from code?** | **Yes**: remove a resource block → plan shows destroy | **No**: delete a task and the package stays installed. You must write `state: absent` |
| **Preview** | `terraform plan` (always) | `--check --diff` (works if modules support it, topic 20) |
| **Talks to** | Cloud/provider APIs | SSH/WinRM to hosts (and APIs, via modules) |
| **Language** | HCL (HashiCorp Configuration Language) | YAML + Jinja2 |
| **Agent** | No | No |
| **Good at** | VPCs, VMs, databases, IAM (Identity and Access Management), DNS, Kubernetes clusters | Packages, files, users, services, rolling deploys, patching, ad hoc fixes |
| **Bad at** | Anything inside the OS (operating system) | Tracking and destroying cloud resources cleanly |

The "deletes things" row is the deepest difference. Terraform's state lets it say "this bucket was in the code, now it isn't → delete it". Ansible has no memory. If you remove the `nginx` task from your playbook, Ansible simply stops talking about nginx, and nginx stays installed on every server. To remove it, you must *say so* with a task that has `state: absent`.

### How they work together

```
  Git repo
  ├── terraform/          ① terraform apply
  │                          creates VPC, subnets, 3 VMs, load balancer
  │                          outputs: the VMs' IP addresses (or tags them role=web)
  │                                     │
  │                                     ▼  hand-off
  └── ansible/            ② ansible-playbook -i <inventory built from ①> site.yml
                             connects to the 3 VMs over SSH
                             installs packages, writes config, deploys the app
```

The hand-off is the **inventory**: Ansible's list of servers to manage (topic 10). There are two common ways to build it from Terraform's work:

1. **From Terraform outputs**: Terraform prints the IPs (`terraform output -json`), a small script turns them into an inventory file.
2. **From cloud tags (better)**: Terraform tags each VM (`role = "web"`). Ansible's **dynamic inventory** plugin (topic 42) asks AWS "list all VMs with tag role=web". No file to keep in sync.

The full pattern, including running both in one CI (Continuous Integration) pipeline, is in topic 43 and Project 3 (topic 62). Your Terraform notes cover the Terraform side in [topic 52](../../Terraform/notes/52-provisioners-and-alternatives.md).

### The other tools on the map

| Tool | Job | Relation to Ansible |
|---|---|---|
| **cloud-init** ([Linux 104](../../linux-mastery-rhel/notes/104-cloud-init.md)) | Runs once at a VM's **first boot** (create user, add SSH key, install a few packages) | Often used to prepare a VM *just enough* for Ansible to connect. Ansible does the rest, and keeps doing it later |
| **Packer** | Builds VM images (e.g. AWS AMIs, Amazon Machine Images) | Packer can **run Ansible** to configure the image. Then servers boot already configured (immutable) |
| **Dockerfile** ([Docker 14](../../Docker/notes/14-dockerfile-basics.md)) | Configures a container image | For containers, the Dockerfile replaces most CM work. You don't SSH into containers |
| **Kubernetes + Helm** | Runs and configures containers in a cluster | Ansible may install Kubernetes nodes or apply manifests, but day-to-day app config lives in Kubernetes |
| **AWS SSM** (Systems Manager) | Runs commands on EC2 through an AWS agent | Cloud-specific alternative. Ansible can even connect *through* SSM instead of SSH |

### Mutable vs immutable

- **Mutable infrastructure** = servers are changed in place for their whole life. Ansible runs every week to patch and re-converge them.
- **Immutable infrastructure** = servers are never changed. To change one, build a new image and replace the server. Containers work this way.

Ansible fits **both**: it manages long-lived mutable servers, and it is also used *inside* Packer to build immutable images.

---

## Why it exists

Different problems need different tools. Using one tool for everything hurts:

- **Terraform for configuration** → `remote-exec` provisioners that run once at creation, never re-run, can't detect drift inside the server, and make `terraform apply` fail for app reasons.
- **Ansible for all infrastructure** → no state, so deleted resources stay running and cost money; no proper plan before creating 30 cloud resources.
- **Bash scripts for both** → not idempotent, no preview, no inventory, no parallelism.

So mature teams split the work: **Terraform owns "what exists", Ansible owns "what's inside", Git owns both.**

---

## Analogy

**Opening a new restaurant branch.**

- **Terraform = the building contractor.** Builds the kitchen, dining room, wiring, water. Keeps the blueprint (state). If the blueprint says "no back door", the contractor bricks it up.
- **Ansible = the head chef with a setup checklist.** Walks into any kitchen, checks "knives sharpened, ovens at 180°C, menu printed, staff trained", and fixes what's missing. Has no blueprint of the building. Removing "fryer" from the checklist doesn't remove the fryer from the kitchen.
- **Puppet/Chef = a robot assistant living in every kitchen** checking the list every 30 minutes. Great, but you must install and maintain a robot per kitchen.
- **Packer = a factory that builds whole pre-equipped kitchen modules** you drop in place.
- **cloud-init = the note taped to the door for opening day**: "the key is under the mat".

---

## How it works inside

How each tool talks to its targets:

```
 TERRAFORM                                    ANSIBLE
 ┌───────────┐  HTTPS API calls               ┌───────────┐   SSH (port 22)
 │ terraform │ ─────────────────▶ AWS API      │ ansible   │ ───────────────▶ server 1: runs a small
 │ + state   │   "create VM"      creates VM   │ (no state)│                  Python program, returns
 └───────────┘                                └───────────┘ ───────────────▶ server 2   JSON, deletes it
                                                                                             
 PUPPET                                                                                       
 ┌──────────────┐  agent asks every 30 min (HTTPS, port 8140)                                  
 │ puppetserver │ ◀────────────────────────── puppet agent (running on each server)          
 └──────────────┘ ──── compiled "catalog" ──▶ agent applies it locally                        
```

- **Terraform** never logs into your servers. It only calls cloud APIs and records results in its state.
- **Ansible** logs into each server with SSH, copies a tiny Python program (a **module**), runs it, reads its JSON (JavaScript Object Notation) result, deletes it. Nothing stays behind. Topic 13 traces this step by step.
- **Puppet** agents pull a compiled description ("catalog") from the server and apply it locally.

---

## Code / commands

The same goal ("one web server running nginx") split across Terraform and Ansible. Read it; you'll build the real thing in topic 62.

**① Terraform creates the VM and tags it** (HCL, from your [Terraform notes](../../Terraform/notes/33-project-2-web-server-on-aws.md)):

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0abc123"
  instance_type = "t3.micro"
  key_name      = "ops-key"          # SSH key Ansible will use
  tags = {
    Name = "web-1"
    role = "web"                     # Ansible finds servers by this tag
  }
}

output "web_ip" {
  value = aws_instance.web.public_ip
}
```

```
$ terraform apply
...
Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
Outputs:
web_ip = "54.210.12.34"
```

**② Ansible configures it.** A simple inventory file listing that IP (topic 10):

```ini
[web]
web-1 ansible_host=54.210.12.34 ansible_user=ec2-user
```

- `[web]`: a group called `web`.
- `web-1`: the name Ansible uses for this host.
- `ansible_host=`: the real address to connect to. `ansible_user=`: the SSH login user.

And a playbook:

```yaml
- name: Configure web servers
  hosts: web
  become: true
  tasks:
    - name: nginx is installed
      ansible.builtin.dnf:
        name: nginx
        state: present
    - name: nginx is running
      ansible.builtin.systemd_service:
        name: nginx
        state: started
        enabled: true
```

```
$ ansible-playbook -i inventory.ini web.yml

PLAY [Configure web servers] *******************************************

TASK [Gathering Facts] *************************************************
ok: [web-1]

TASK [nginx is installed] **********************************************
changed: [web-1]

TASK [nginx is running] ************************************************
changed: [web-1]

PLAY RECAP *************************************************************
web-1  : ok=3  changed=2  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

- `PLAY [...]`: Ansible starts the play named in your file.
- `Gathering Facts`: an automatic first step that collects information about the server (OS, IPs). `ok` = it worked and changed nothing.
- `changed: [web-1]`: the task made a change on `web-1`.
- `PLAY RECAP`: the totals per host. `ok=3` counts every successful task, including the 2 that changed something.

**What happens when you delete things?**

| You remove from code... | Terraform | Ansible |
|---|---|---|
| The VM / the nginx task | Next `apply` **destroys** the VM | Next run **ignores** nginx; it stays installed |
| To really remove nginx with Ansible | n/a | Change the task to `state: absent`, run, *then* delete the task |

---

## Hands-on lab

A thinking lab (no installs yet). Sort each job into **Terraform**, **Ansible**, **Packer/Dockerfile**, **cloud-init** or **Kubernetes**, then check the answers.

| # | Job |
|---|---|
| 1 | Create a VPC with 3 subnets |
| 2 | Install security patches on 200 existing RHEL servers, 20 at a time |
| 3 | Create an S3 bucket and an IAM role |
| 4 | Make sure `/etc/ssh/sshd_config` has `PermitRootLogin no` on every server |
| 5 | Put the `ops` user's SSH key on a brand-new VM at its first boot so automation can log in |
| 6 | Build a golden AMI with the company's monitoring agent pre-installed |
| 7 | Restart the app on web servers one at a time, removing each from the load balancer first |
| 8 | Run 5 replicas of a containerised API with rolling updates |
| 9 | Create a DNS record pointing to the load balancer |
| 10 | Rotate the `deploy` user's SSH key on 500 servers |

<details>
<summary>Answers</summary>

1 Terraform · 2 Ansible (`serial: 20`, topic 45) · 3 Terraform · 4 Ansible · 5 cloud-init · 6 Packer (often running Ansible inside) · 7 Ansible (`serial` + `delegate_to`, topics 45–46) · 8 Kubernetes · 9 Terraform · 10 Ansible
</details>

**Bonus:** look at the [Terraform topic 02](../../Terraform/notes/02-iac-tool-landscape.md) diagram of provisioning vs configuration. It's the same picture from the other side.

---

## Where you'll use it

- **Architecture decisions**: "Should this be in Terraform or Ansible?" Rule of thumb: if it has an API endpoint and a lifecycle (create/delete), Terraform. If it lives inside an OS, Ansible.
- **Pipelines**: a common CI/CD (Continuous Integration / Continuous Delivery) job is `terraform apply` → `ansible-playbook` (topics 43, 53, 54).
- **Golden images**: Packer + Ansible to bake AMIs.
- **Migration projects**: replacing Puppet/Chef agents with agentless Ansible.
- **Interviews**: the comparison table above.

---

## When it breaks

| Problem | Cause | Fix |
|---|---|---|
| Terraform `remote-exec` fails halfway, resource marked "tainted" | Using Terraform for configuration | Move config to Ansible or cloud-init (topic 43) |
| Removed a package from the playbook, still installed everywhere | Ansible is stateless: removing a task removes nothing | Add `state: absent` first, run, then delete the task |
| Cloud VMs created by Ansible keep costing money after "removal" | No state, nothing destroyed them | Provision with Terraform; or write explicit `state: absent` tasks |
| Ansible can't reach new VMs right after `terraform apply` | VM still booting / SSH not ready / no key | Wait with `ansible.builtin.wait_for_connection` (topic 47); put the key in via `key_name` or cloud-init |
| Inventory file has old IPs after Terraform recreated servers | Static file out of sync | Dynamic inventory from tags (topic 42) |

---

## Quick recap

- **Provisioning** (create infrastructure through APIs) ≠ **configuration management** (set up the inside of servers).
- Terraform: declarative, **state file**, API calls, destroys what you remove. Ansible: modules in order, **stateless**, SSH, removes nothing unless told `state: absent`.
- Ansible vs Puppet/Chef/Salt: **agentless push over SSH** vs agents pulling on a timer.
- They work **together**: Terraform builds, Ansible configures. The hand-off is the **inventory** (outputs or tags).
- cloud-init = first boot; Packer = images; Dockerfile/Kubernetes = containers.

### Interview questions

1. **Q: Terraform vs Ansible: when would you use each?**
   A: Terraform to provision infrastructure through cloud APIs, with a state file and a plan. Ansible to configure the OS and deploy apps on servers over SSH. Typically Terraform creates the VMs, then Ansible configures them.
2. **Q: Why is Ansible called agentless, and why does that matter?**
   A: It needs only SSH and Python on the target, which nearly every Linux server already has. No agent to install, patch, secure or monitor, and you can manage a server from its first minute.
3. **Q: If you remove a task from a playbook, what happens to the thing it created?**
   A: Nothing. Ansible doesn't keep state. To remove it you must write a task with `state: absent`.

### Scenario question (answer out loud first, then compare)

> *"Your team uses Ansible's cloud modules to create EC2 instances and configure them in one big playbook. What problems do you expect, and what would you change?"*

**Model answer:**
1. No state: instances removed from the playbook keep running (cost, security risk). Nothing shows "what exists but isn't in code".
2. No real plan before creating cloud resources; re-runs can create duplicates if the module's matching (by name/tag) isn't set up right.
3. Mixing jobs makes the playbook slow and risky: a config change re-runs cloud calls.
4. Change: move VPC/instances/security groups to **Terraform** (state, plan, destroy). Tag instances by role. Use Ansible's **`amazon.aws.aws_ec2` dynamic inventory** to find them, and keep Ansible for OS configuration and deploys. Run both in one pipeline: `terraform apply` → `ansible-playbook`.

**Likely follow-ups:**
- *Is there ever a good reason to create cloud resources with Ansible?* Small one-off tasks, or orchestration that mixes API calls and server steps (e.g. snapshot a disk before patching). For long-lived infrastructure, use Terraform.
- *Where does cloud-init fit?* First-boot bootstrap (user + SSH key) so Ansible can connect.
