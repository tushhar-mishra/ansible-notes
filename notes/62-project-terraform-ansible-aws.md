# 62 · Project 3: Terraform → Ansible on AWS in CI

> **Level:** ★★★ advanced · 💰 small AWS cost (a few cents to a few dollars if you destroy promptly)
>
> **You will build:** the full production pattern end to end: **GitHub Actions** authenticates to AWS with **OIDC** (no stored keys), **Terraform** (remote state in S3 with locking) creates two **Amazon Linux 2023** instances with **no inbound ports**, an instance profile for **SSM**, and an S3 bucket for SSM file transfer, tagging them `env=demo, role=web`; Ansible finds them with the **`aws_ec2` dynamic inventory**, connects through **AWS Systems Manager** (the `aws_ssm` connection plugin, so no SSH keys, no bastion), configures nginx, verifies, and a manual workflow **destroys** everything.
>
> **You need:** [61 · Scaling](61-scaling.md), [43 · Ansible + Terraform](43-ansible-and-terraform.md), [42 · aws_ec2 inventory](42-dynamic-inventory.md), [53 · GitHub Actions](53-ansible-in-github-actions.md), [44 · connection plugins](44-beyond-linux-servers.md), and your Terraform notes: [29 · AWS basics](../../Terraform/notes/29-aws-basics.md), [30 · AWS provider and auth](../../Terraform/notes/30-aws-provider-and-auth.md), [35 · remote backends](../../Terraform/notes/35-remote-backends.md), [36 · state locking](../../Terraform/notes/36-state-locking.md), [57 · Terraform in GitHub Actions](../../Terraform/notes/57-github-actions.md); GitHub Actions [36 · OIDC](../../GitHub-Actions-Basics-To-Production/notes/36-oidc-cloud-authentication.md).
>
> **Part 12 · Production operations** · Next: [63 · Upgrades and what's changed](63-upgrades-and-changes.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** One pipeline: **plan** on pull requests, **apply + configure + verify** on `main` (after approval), **destroy** on demand. Credentials come from **OIDC** (short-lived AWS role), servers are reached through **SSM** (no SSH at all), and the hand-off between Terraform and Ansible is **tags** read by the dynamic inventory.

**Picture it.** A remote-controlled factory: the pipeline signs in with a temporary badge (OIDC), the builders put up two workshops with no outside doors (no inbound ports), stick role labels on them (tags), and the interior team walks in through the internal service corridor (SSM) to fit them out.

**The one thing to know.**

```
 GitHub Actions ──OIDC──▶ AWS role (temporary credentials)
   ├─ terraform apply  → EC2 ×2 (tags env=demo, role=web), IAM instance profile (SSM), S3 bucket
   ├─ ansible-inventory (aws_ec2: tag:env=demo → group role_web)
   ├─ ansible-playbook  (connection amazon.aws.aws_ssm → nginx)
   └─ verify            (curl from inside each instance via SSM)
```

**Try it now:** read the whole project first; then do the one-time bootstrap (OIDC role + state bucket) and run it.

**You've got it if you can:** explain why this design needs no SSH keys, no open ports and no stored AWS secrets.

> 📖 **That's the core.** Follow the build steps carefully; **always run the destroy workflow at the end**.

---

## What it is

### Architecture

```
 GitHub repo (infra-aws)
 ├── terraform/            ← EC2, IAM instance profile, S3 (SSM transfer), SG (egress only)
 ├── ansible/
 │   ├── inventories/demo/aws_ec2.yml + group_vars/all.yml
 │   ├── playbooks/{web.yml, verify.yml}
 │   ├── requirements.txt, collections/requirements.yml, ansible.cfg
 └── .github/workflows/{deploy.yml, destroy.yml}

 AWS account
 ├── S3 bucket: Terraform state (with S3 native locking)
 ├── IAM OIDC provider for token.actions.githubusercontent.com
 ├── IAM role "gha-infra-aws" (trusted only by this repo's main branch / environment)
 └── (created by Terraform) 2 × t3.micro AL2023 in the default VPC, public IP for outbound only,
     security group with NO inbound rules, instance profile with AmazonSSMManagedInstanceCore,
     S3 bucket for SSM file transfers
```

### Why SSM instead of SSH

| | SSH | SSM (`amazon.aws.aws_ssm` connection) |
|---|---|---|
| Inbound port | 22 open (to runners/bastion) | **None** |
| Credentials | SSH key/cert to distribute and protect | **IAM** (the same OIDC role) |
| Bastion | Often needed for private subnets | Not needed |
| Audit | sshd logs on each host | CloudTrail + Session Manager logs |
| Requirements | sshd, key | SSM agent (preinstalled on Amazon Linux), instance profile, `session-manager-plugin` + `boto3` on the control node, an S3 bucket for file transfer |

(Older documentation names the plugin `community.aws.aws_ssm`; current `amazon.aws` versions ship it as `amazon.aws.aws_ssm`. Check `ansible-doc -t connection -l | grep ssm` for your installed version.)

### Costs and safety

- 2 × `t3.micro` + tiny S3 usage: cents per hour; free-tier eligibility depends on your account (check AWS's current free-tier terms). **Destroy** when done.
- The OIDC role should have only the permissions this project needs, and trust only this repository (and ideally only the `main` branch or the `production` environment).
- Set an AWS **budget alert** before you start ([Terraform topic 29](../../Terraform/notes/29-aws-basics.md)).

---

## Why it exists

Topics 43 and 53 showed the pieces with local labs. In production on AWS you'd face the real questions: how does CI get AWS credentials without storing keys, how does Ansible reach instances in a network with no inbound access, how does Terraform's output become Ansible's inventory, and how is it all torn down. This project answers all of them in one working repository you can show in interviews.

---

## Analogy

**A secure building contractor working remotely.** The contractor's laptop gets a one-day pass from the client's security desk (OIDC). The builder erects two units with no external doors (no inbound ports) but connected to the building's internal service corridor (SSM). Each unit has a label on the door (tags) so the fit-out team knows which is which (dynamic inventory). When the project ends, both units are dismantled (destroy).

---

## How it works inside

```
 push to main
   │
   ├─ job deploy (environment: demo, needs approval if configured)
   │    ├─ OIDC: GitHub issues a JWT → aws-actions/configure-aws-credentials → sts:AssumeRoleWithWebIdentity
   │    │         → AWS_ACCESS_KEY_ID/SECRET/SESSION_TOKEN valid ~1 hour (env vars for later steps)
   │    ├─ terraform init (S3 backend) → plan → apply
   │    ├─ wait until SSM agents are online (aws ssm describe-instance-information)
   │    ├─ ansible-inventory -i inventories/demo  → boto3 DescribeInstances (tag filters)
   │    ├─ ansible-playbook web.yml
   │    │     connection aws_ssm: boto3 StartSession → session-manager-plugin → shell on instance
   │    │     files (modules) go through the S3 bucket
   │    └─ ansible-playbook verify.yml
   │
 workflow_dispatch "destroy"
   └─ OIDC → terraform destroy
```

---

## Code / commands

### 0. One-time bootstrap (from your laptop, with admin credentials)

Do this once, outside the pipeline (a small separate Terraform config, or the console):

1. **State bucket** (versioned, encrypted, private): e.g. `yourname-infra-aws-tfstate`.
2. **OIDC provider** for `https://token.actions.githubusercontent.com` (audience `sts.amazonaws.com`).
3. **IAM role** `gha-infra-aws` with this trust policy (replace account ID, org/user and repo):

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
      "StringLike":   { "token.actions.githubusercontent.com:sub": "repo:YOUR_USER/infra-aws:*" }
    }
  }]
}
```

   For a lab, attach permissions covering EC2, IAM (role/instance profile for this project), S3 (state + transfer bucket) and SSM. In production, write a least-privilege policy scoped by resource names/tags, and narrow `sub` to `repo:...:environment:demo`.

4. GitHub repo settings: environment **`demo`**, variables `AWS_ROLE_ARN`, `AWS_REGION` (e.g. `eu-west-1`), `TF_STATE_BUCKET`.

### 1. Terraform

`terraform/versions.tf`:

```hcl
terraform {
  required_version = ">= 1.10"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 6.0" }
  }
  backend "s3" {
    key          = "infra-aws/demo.tfstate"
    encrypt      = true
    use_lockfile = true          # S3-native state locking (Terraform 1.10+)
    # bucket and region passed with -backend-config in CI
  }
}

provider "aws" {
  region = var.region
  default_tags {
    tags = { project = "infra-aws", env = "demo", managed_by = "terraform" }
  }
}

variable "region" { type = string }
variable "instance_count" {
  type    = number
  default = 2
}
```

`terraform/main.tf`:

```hcl
data "aws_ssm_parameter" "al2023" {
  name = "/aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64"
}

data "aws_vpc" "default" { default = true }

resource "aws_security_group" "web" {
  name        = "infra-aws-demo-web"
  description = "No inbound; all outbound (SSM, package repos)"
  vpc_id      = data.aws_vpc.default.id
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

resource "aws_iam_role" "ssm" {
  name = "infra-aws-demo-ssm"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{ Effect = "Allow", Principal = { Service = "ec2.amazonaws.com" }, Action = "sts:AssumeRole" }]
  })
}

resource "aws_iam_role_policy_attachment" "ssm_core" {
  role       = aws_iam_role.ssm.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
}

resource "aws_s3_bucket" "ssm_transfer" {
  bucket_prefix = "infra-aws-demo-ssm-"
  force_destroy = true
}

resource "aws_iam_role_policy" "ssm_bucket" {
  name = "ssm-transfer-bucket"
  role = aws_iam_role.ssm.id
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject", "s3:ListBucket"]
      Resource = [aws_s3_bucket.ssm_transfer.arn, "${aws_s3_bucket.ssm_transfer.arn}/*"]
    }]
  })
}

resource "aws_iam_instance_profile" "ssm" {
  name = "infra-aws-demo-ssm"
  role = aws_iam_role.ssm.name
}

resource "aws_instance" "web" {
  count                  = var.instance_count
  ami                    = data.aws_ssm_parameter.al2023.value
  instance_type          = "t3.micro"
  iam_instance_profile   = aws_iam_instance_profile.ssm.name
  vpc_security_group_ids = [aws_security_group.web.id]
  metadata_options {
    http_tokens = "required"            # IMDSv2 only
  }
  tags = {
    Name = "infra-aws-demo-web-${count.index + 1}"
    role = "web"
  }
}

output "instance_ids" { value = aws_instance.web[*].id }
output "ssm_bucket"   { value = aws_s3_bucket.ssm_transfer.bucket }
```

- `default_tags` adds `env = "demo"` (and more) to **every** resource: the dynamic inventory filters on it.
- No `key_name`, no inbound rules: SSH is impossible by design.
- `AmazonSSMManagedInstanceCore` lets the SSM agent register; the extra bucket policy lets Ansible move module files through S3.

### 2. Ansible

`ansible/requirements.txt`:

```text
ansible-core==2.22.0
boto3
botocore
```

`ansible/collections/requirements.yml`:

```yaml
---
collections:
  - name: amazon.aws
    version: ">=10.0.0,<11.0.0"     # pin a current major that includes the aws_ssm connection and aws_ec2 inventory
```

`ansible/ansible.cfg`:

```ini
[defaults]
collections_path = ./collections:~/.ansible/collections
interpreter_python = auto_silent
forks = 20
callbacks_enabled = ansible.posix.profile_tasks
host_key_checking = True
```

(Add `ansible.posix` to requirements if you keep `profile_tasks`.)

`ansible/inventories/demo/aws_ec2.yml`:

```yaml
plugin: amazon.aws.aws_ec2
regions:
  - eu-west-1              # your region (keep it in sync with the AWS_REGION variable)
filters:
  instance-state-name: running
  tag:env: demo
  tag:project: infra-aws
hostnames:
  - instance-id
keyed_groups:
  - key: tags.role
    prefix: role
compose:
  ansible_host: instance_id
strict: true
```

- `hostnames: instance-id` + `ansible_host: instance_id`: the SSM plugin connects by **instance ID**, not IP.

`ansible/inventories/demo/group_vars/all.yml`:

```yaml
---
ansible_connection: amazon.aws.aws_ssm
ansible_aws_ssm_region: "{{ lookup('ansible.builtin.env', 'AWS_REGION') }}"
ansible_aws_ssm_bucket_name: "{{ lookup('ansible.builtin.env', 'SSM_BUCKET') }}"
ansible_python_interpreter: /usr/bin/python3
```

`ansible/playbooks/web.yml`:

```yaml
---
- name: Wait for SSM and configure the web instances
  hosts: role_web
  gather_facts: false
  become: true
  tasks:
    - name: Instance is reachable through SSM
      ansible.builtin.wait_for_connection:
        timeout: 600

    - name: Gather facts
      ansible.builtin.setup:

    - name: nginx is installed
      ansible.builtin.dnf:
        name: nginx
        state: present

    - name: Home page shows the instance
      ansible.builtin.copy:
        dest: /usr/share/nginx/html/index.html
        content: "Hello from {{ inventory_hostname }} ({{ ansible_facts['distribution'] }} {{ ansible_facts['distribution_version'] }}) configured by Ansible via SSM\n"
        mode: "0644"

    - name: nginx running and enabled
      ansible.builtin.systemd_service:
        name: nginx
        state: started
        enabled: true
```

`ansible/playbooks/verify.yml`:

```yaml
---
- name: Verify from inside each instance
  hosts: role_web
  gather_facts: false
  tasks:
    - name: Local HTTP check
      ansible.builtin.uri:
        url: http://127.0.0.1/
        return_content: true
      register: page
      failed_when: "'configured by Ansible via SSM' not in page.content"
```

### 3. Workflows

`.github/workflows/deploy.yml`:

```yaml
name: deploy

on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  id-token: write          # needed for OIDC

concurrency:
  group: infra-aws-demo
  cancel-in-progress: false

jobs:
  plan:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-24.04
    environment: demo
    steps:
      - uses: actions/checkout@v5
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
      - uses: hashicorp/setup-terraform@v3
      - name: Terraform plan
        working-directory: terraform
        run: |
          terraform init -backend-config="bucket=${{ vars.TF_STATE_BUCKET }}" -backend-config="region=${{ vars.AWS_REGION }}"
          terraform plan -var region=${{ vars.AWS_REGION }} -no-color | tee plan.txt
      - name: Lint Ansible
        working-directory: ansible
        run: |
          pip install -r requirements.txt ansible-lint
          ansible-galaxy collection install -r collections/requirements.yml -p ./collections
          ansible-lint playbooks/

  deploy:
    if: github.event_name == 'push'
    runs-on: ubuntu-24.04
    environment: demo
    steps:
      - uses: actions/checkout@v5
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_wrapper: false

      - name: Terraform apply
        working-directory: terraform
        run: |
          terraform init -backend-config="bucket=${{ vars.TF_STATE_BUCKET }}" -backend-config="region=${{ vars.AWS_REGION }}"
          terraform apply -auto-approve -var region=${{ vars.AWS_REGION }}
          echo "SSM_BUCKET=$(terraform output -raw ssm_bucket)" >> "$GITHUB_ENV"
          echo "INSTANCE_IDS=$(terraform output -json instance_ids | jq -r 'join(" ")')" >> "$GITHUB_ENV"

      - name: Wait for SSM agents to register
        run: |
          for i in $(seq 1 60); do
            online=$(aws ssm describe-instance-information \
              --filters "Key=InstanceIds,Values=${INSTANCE_IDS// /,}" \
              --query 'length(InstanceInformationList[?PingStatus==`Online`])' --output text)
            echo "online: $online"
            [ "$online" -ge 2 ] && exit 0
            sleep 10
          done
          exit 1

      - name: Session Manager plugin and Ansible
        working-directory: ansible
        run: |
          curl -fsSLo /tmp/smp.deb "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/ubuntu_64bit/session-manager-plugin.deb"
          sudo dpkg -i /tmp/smp.deb
          pip install -r requirements.txt
          ansible-galaxy collection install -r collections/requirements.yml -p ./collections

      - name: Inventory
        working-directory: ansible
        run: ansible-inventory -i inventories/demo --graph

      - name: Configure
        working-directory: ansible
        run: ansible-playbook -i inventories/demo playbooks/web.yml

      - name: Verify
        working-directory: ansible
        run: ansible-playbook -i inventories/demo playbooks/verify.yml
```

`.github/workflows/destroy.yml`:

```yaml
name: destroy
on:
  workflow_dispatch:
permissions:
  contents: read
  id-token: write
concurrency:
  group: infra-aws-demo
  cancel-in-progress: false
jobs:
  destroy:
    runs-on: ubuntu-24.04
    environment: demo
    steps:
      - uses: actions/checkout@v5
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ${{ vars.AWS_REGION }}
      - uses: hashicorp/setup-terraform@v3
      - working-directory: terraform
        run: |
          terraform init -backend-config="bucket=${{ vars.TF_STATE_BUCKET }}" -backend-config="region=${{ vars.AWS_REGION }}"
          terraform destroy -auto-approve -var region=${{ vars.AWS_REGION }}
```

### 4. What success looks like

```
Run ansible-inventory -i inventories/demo --graph
@all:
  |--@ungrouped:
  |--@aws_ec2:
  |  |--i-0a1b2c3d4e5f60001
  |  |--i-0a1b2c3d4e5f60002
  |--@role_web:
  |  |--i-0a1b2c3d4e5f60001
  |  |--i-0a1b2c3d4e5f60002

Run ansible-playbook -i inventories/demo playbooks/web.yml
TASK [Instance is reachable through SSM] ***********************************************
ok: [i-0a1b2c3d4e5f60001]
ok: [i-0a1b2c3d4e5f60002]
...
PLAY RECAP *****************************************************************************
i-0a1b2c3d4e5f60001 : ok=6  changed=3  unreachable=0  failed=0 ...
i-0a1b2c3d4e5f60002 : ok=6  changed=3  unreachable=0  failed=0 ...
```

Push a second commit that only changes the page text: Terraform shows `No changes`, Ansible shows one `changed` per instance.

---

## Hands-on lab

1. 💰 Set a budget alert. Do the bootstrap (state bucket, OIDC provider, role, GitHub environment and variables).
2. Create the repo with the files above; open a PR: `plan` runs (Terraform plan + lint).
3. Merge: `deploy` creates, configures and verifies. Read every step's log.
4. Change the page text, push, and confirm the second run: Terraform `No changes`, Ansible small `changed`.
5. In the AWS console, open **Systems Manager → Session Manager → Session history**: you'll see Ansible's sessions (audit).
6. Scale: change `instance_count` to 3, push. The new instance appears in `role_web` automatically and gets configured.
7. **Run the `destroy` workflow.** Check the EC2 console: instances terminated; S3 transfer bucket gone. (The state bucket and OIDC role stay for next time; delete them when you're finished with the course.)

**Done when:** a commit to `main` produces configured, verified instances with no keys, no open ports and no stored AWS secrets, and the destroy workflow removes them.

---

## Where you'll use it

- **Any AWS shop**: this is the modern, secure Terraform + Ansible pipeline.
- **Interviews**: walk through OIDC trust, tags → dynamic inventory, SSM connection, state locking, approval gates and destroy.
- **Variations**: Azure (OIDC + `azure_rm` inventory + Azure Arc/SSH), GCP (Workload Identity Federation + `gcp_compute` + IAP tunnelling) follow the same shape.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | Trust policy `sub`/`aud` doesn't match the repo/branch/environment | Fix the conditions; check the token's `sub` claim format |
| Terraform `Error acquiring the state lock` | Another run holds the lock (or a crashed run left it) | Concurrency group; `terraform force-unlock` only if sure |
| SSM agents never come `Online` | No outbound internet (no public IP/NAT), missing instance profile/policy | Default VPC public subnet or NAT/VPC endpoints for SSM; check the role |
| `aws_ssm` connection: `SessionManagerPlugin is not found` | Plugin not installed on the runner | Install `session-manager-plugin` |
| `failed to transfer file ... S3` | Bucket name/permissions missing for the instance role or the runner role | `ansible_aws_ssm_bucket_name`, bucket policy on both sides |
| Inventory empty | Wrong region, tags, or instances still `pending` | Check `AWS_REGION`, filters; wait for `running` |
| `Failed to import ... boto3` | Not in the venv | `pip install boto3 botocore` |
| Costs continue | Destroy not run, or failed | Run destroy; check the console; budget alerts |

---

## Quick recap

- **OIDC** gives CI short-lived AWS credentials: no stored keys. Terraform state in **S3 with native locking**.
- Terraform creates instances with **no inbound ports**, an **SSM instance profile**, an S3 transfer bucket, and **tags** (`default_tags` + `role`).
- Ansible: **`aws_ec2`** inventory (tags → `role_web`, hosts by instance ID), **`amazon.aws.aws_ssm`** connection (IAM-authenticated, audited), `wait_for_connection`, configure, verify.
- Pipeline: plan on PR, apply + configure + verify on `main` (environment protection), destroy on demand; concurrency to serialise.

### Interview questions

1. **Q: How does your pipeline authenticate to AWS without storing access keys?**
   A: GitHub Actions requests an OIDC token; AWS trusts GitHub's OIDC provider and lets the workflow assume a role whose trust policy matches the repository (and branch/environment), returning short-lived credentials.
2. **Q: How does Ansible connect to instances that have no SSH port open?**
   A: Through AWS Systems Manager with the `aws_ssm` connection plugin: the SSM agent on the instance keeps an outbound channel; Ansible starts sessions via the API (IAM-authorised) and transfers module files through an S3 bucket.
3. **Q: How does Ansible know which instances Terraform created?**
   A: Terraform tags them (`env`, `role`); the `aws_ec2` dynamic inventory filters on those tags and creates groups from them.

### Scenario question (answer out loud first, then compare)

> *"Take this demo to production for 300 instances in private subnets across two regions, with a strict security team. What changes?"*

**Model answer:**
1. **Network**: private subnets with **VPC endpoints** for SSM, SSM messages, EC2 messages and S3 (no internet needed); or NAT if packages need the internet (better: an internal package mirror).
2. **Identity**: separate OIDC roles per environment with least-privilege policies scoped by tags/resource ARNs; trust limited to `environment:production`; approvals on the production environment.
3. **Terraform**: modules, remote state per environment/region, plan on PR with review, apply on approval (Terraform notes topics 43–60).
4. **Ansible**: inventories per environment and region (two `aws_ec2.yml` or one with both regions), roles from the internal collection, EE image in ECR as the job container, `serial` rollouts with health checks, drift checks on a schedule.
5. **Scale**: run the two regions as parallel jobs; SSM session limits and throughput considered (or SSH via EC2 Instance Connect Endpoint as an alternative).
6. **Audit**: CloudTrail + Session Manager logging to S3/CloudWatch; GitHub/AWX job history.

**Likely follow-ups:**
- *Why not just open SSH from the runners?* More attack surface and key management; SSM gives IAM auth and audit with zero inbound ports.
- *How do new autoscaled instances get configured?* Baked AMI + user data, or an EventBridge rule triggering an Ansible job (EDA/AWX) for new instances.
