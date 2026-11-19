# 42 · Dynamic Inventory

> **Level:** ★★ intermediate
>
> **You will learn:** why static host lists break in the cloud, how **inventory plugins** build the inventory at run time from a source of truth (AWS EC2, Azure, GCP, VMware, Docker, Kubernetes, NetBox…), the plugin config file (`plugin:`, filters, **`keyed_groups`**, **`groups`**, **`compose`**, `hostnames`), the **`constructed`** plugin that builds groups from any variables, **caching**, mixing static and dynamic sources in one inventory directory, and ⚠️ legacy inventory **scripts**. You'll discover your lab containers dynamically from Docker, and see the full AWS EC2 setup.
>
> **You need:** [41 · External secrets](41-external-secrets.md), [10 · Inventory basics](10-inventory-basics.md), [24 · group_vars](24-group-vars-and-host-vars.md), [39 · Project layout](39-project-layout.md), [22 · Jinja2](22-jinja2-basics.md).
>
> **Part 6 · Inventory at scale and the cloud** · Next: [43 · Ansible and Terraform together](43-ansible-and-terraform.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** In the cloud, servers come and go (autoscaling, rebuilds). A hand-written host list is wrong within days. A **dynamic inventory** asks the platform **at the start of every run** "which servers exist?", and builds **groups from tags** (`role=web` → group `role_web`).

**Picture it.** Instead of a printed staff list, the receptionist checks the live badge system every morning: whoever badged in today is on the list, grouped by department.

**The one thing to know.** `inventories/prod/aws_ec2.yml`:

```yaml
plugin: amazon.aws.aws_ec2
regions: [eu-west-1]
filters:
  instance-state-name: running
  tag:env: prod
keyed_groups:
  - key: tags.role          # tag role=web → group "role_web"
    prefix: role
compose:
  ansible_host: private_ip_address
```

```bash
ansible-inventory -i inventories/prod/aws_ec2.yml --graph
ansible-playbook -i inventories/prod playbooks/site.yml      # hosts: role_web
```

**Try it now:** the Hands-on lab discovers your Docker lab containers dynamically, no cloud account needed.

**You've got it if you can:** explain `keyed_groups` and `compose`, and why the file name must end in `aws_ec2.yml`.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Inventory plugins

An **inventory plugin** (topic 38) produces hosts, groups and variables. You've already used `ini` and `yaml`. Dynamic ones query a source:

| Plugin (FQCN) | Source | Collection |
|---|---|---|
| `amazon.aws.aws_ec2` | EC2 instances | `amazon.aws` |
| `amazon.aws.aws_rds` | RDS databases | `amazon.aws` |
| `azure.azcollection.azure_rm` | Azure VMs | `azure.azcollection` |
| `google.cloud.gcp_compute` | GCP instances | `google.cloud` |
| `community.vmware.vmware_vm_inventory` / `vmware.vmware.*` | vSphere VMs | VMware collections |
| `openstack.cloud.openstack` | OpenStack | `openstack.cloud` |
| `community.docker.docker_containers` | Docker containers | `community.docker` |
| `kubernetes.core.k8s` | Pods/services | `kubernetes.core` |
| `netbox.netbox.nb_inventory` | NetBox CMDB/IPAM | `netbox.netbox` |
| `ansible.builtin.constructed` | **Existing** inventory variables | ansible-core |
| `ansible.builtin.script` | ⚠️ Legacy executable scripts | ansible-core |
| `ansible.builtin.auto` | Reads any YAML config with a `plugin:` key and loads that plugin | ansible-core |

### The config file

A dynamic inventory is a **YAML file** that names a plugin and configures it. Most cloud plugins require the file name to end in a specific **suffix**, so the `auto` plugin can recognise it:

| Plugin | File name must end in |
|---|---|
| `aws_ec2` | `aws_ec2.yml` / `aws_ec2.yaml` |
| `azure_rm` | `azure_rm.yml` |
| `gcp_compute` | `gcp.yml` / `gcp_compute.yml` |
| `docker_containers` | `docker.yml` / `docker.yaml` |
| `constructed` | any `.yml` (common: `constructed.yml`) |

### Common options (shared by most "constructable" plugins)

| Option | Does | Example |
|---|---|---|
| filters / `include_filters` / `exclude_filters` | Which resources to include (plugin-specific) | `tag:env: prod`, `instance-state-name: running` |
| **`keyed_groups`** | Create groups **from a variable's value** | `key: tags.role`, `prefix: role` → `role_web`, `role_db` |
| **`groups`** | Create groups from **Jinja2 conditions** | `big: instance_type.startswith('m5.')` |
| **`compose`** | Create/override **host variables** with expressions | `ansible_host: private_ip_address` |
| `hostnames` | Which attribute names the host | `tag:Name`, `private-dns-name`, `instance-id` |
| `strict` | Fail on bad expressions instead of skipping silently | `true` while developing |
| `cache`, `cache_plugin`, `cache_timeout`, `cache_connection` | Cache results between runs | `jsonfile`, `3600` |
| `leading_separator` | Whether keyed group names start with `_` when there's no prefix | `false` |

`keyed_groups` example in detail:

```yaml
keyed_groups:
  - key: tags.role         # value of tag "role" for each instance: web / db
    prefix: role           # group name = prefix + separator + value
    separator: "_"         # → role_web, role_db
  - key: placement.availability_zone
    prefix: az             # → az_eu_west_1a
  - key: tags.env
    parent_group: envs     # put the new groups under a parent group "envs"
```

Group names are sanitised (dashes become underscores).

### constructed: groups from anything

`ansible.builtin.constructed` doesn't fetch hosts; it **adds groups and variables** to the inventory that earlier sources built, using `keyed_groups`, `groups` and `compose`. It can use host variables and (with a fact cache) **facts**:

```yaml
# inventories/lab/99-constructed.yml
plugin: ansible.builtin.constructed
strict: false
keyed_groups:
  - key: ansible_facts['os_family'] | default('unknown')     # needs a fact cache
    prefix: os
groups:
  canary: inventory_hostname == 'web1'
```

Sources in an inventory directory are loaded in **alphabetical order**: name the constructed file so it comes **last** (`99-constructed.yml`).

### Caching

Cloud APIs are slow and rate-limited. With `cache: true`, the plugin stores its result (e.g. in `~/.ansible/inventory_cache`) for `cache_timeout` seconds. `ansible-inventory --flush-cache` or `ansible-playbook --flush-cache` forces a refresh. Trade-off: a cached inventory can miss servers created in the last few minutes.

### Static + dynamic in one inventory directory

```
inventories/prod/
├── 01-aws_ec2.yml          dynamic: all EC2 instances, groups from tags
├── 02-static.yml           static: on-prem bastion, legacy hosts
├── 99-constructed.yml      extra groups from variables of both
├── group_vars/
│   ├── all/{main.yml,vault.yml}
│   ├── role_web.yml        ← group created by keyed_groups
│   └── role_db.yml
└── host_vars/
```

`group_vars/role_web.yml` works for dynamically created groups exactly like for static ones.

### ⚠️ Inventory scripts

Before plugins, dynamic inventory was an **executable script** that printed JSON for `--list` (all groups + `_meta.hostvars`) and `--host <name>`. They still work (the `script` inventory plugin runs them), and you'll meet old ones like `ec2.py`. New code should use plugins: they support caching, `keyed_groups`, `compose`, and documentation via `ansible-doc -t inventory`.

---

## Why it exists

| Static inventory problem | Dynamic inventory answer |
|---|---|
| New autoscaled servers aren't in the file | Discovered at the start of each run |
| Terminated servers stay in the file (UNREACHABLE noise) | Gone from the API → gone from the inventory |
| IPs change when servers are rebuilt | `compose: ansible_host: private_ip_address` from live data |
| Someone must keep groups up to date | Groups come from **tags** set when the server was created (by Terraform, topic 43) |
| Thousands of hosts | No file to maintain |

The source of truth becomes the platform (or a CMDB), not a text file.

---

## Analogy

**A taxi company's dispatch screen.** The old way: a whiteboard listing which cars are on shift, wiped and rewritten by hand. The new way: every car's GPS reports in; the screen shows live which cars exist, where they are (`ansible_host`), and colours them by type ("estate", "electric") from the car's own registration data (`keyed_groups` from tags).

---

## How it works inside

```
 ansible-playbook -i inventories/prod playbooks/site.yml
   │
   ├─ inventory dir: load sources alphabetically
   │    01-aws_ec2.yml → auto plugin sees "plugin: amazon.aws.aws_ec2" → loads it
   │       cache valid? → use cached hosts
   │       else: boto3 → DescribeInstances (with filters, each region) → instances
   │       for each instance:
   │          host name   ← hostnames order (tag:Name, dns name...)
   │          host vars   ← every instance attribute (tags, private_ip_address, ...)
   │          compose     → ansible_host = private_ip_address
   │          keyed_groups→ add to role_web, az_eu_west_1a ...
   │          groups      → add to conditional groups
   │    02-static.yml → ini/yaml plugin
   │    99-constructed.yml → more groups/vars from everything above
   ├─ vars plugins: group_vars/role_web.yml etc.
   └─ continue as usual (topic 15)
```

The plugin runs on the **control node** with **its** credentials (AWS profile, instance role, OIDC role in CI). It needs read-only API permissions (e.g. `ec2:DescribeInstances`).

---

## Code / commands

### Lab: discover the Docker lab containers

No cloud needed. The `community.docker.docker_containers` plugin lists containers from the Docker daemon.

```bash
cd ~/infra
ansible-galaxy collection install community.docker -p ./collections
uv pip install docker requests          # Python SDK used by the plugin
mkdir -p inventories/docker
```

`inventories/docker/lab.docker.yml`:

```yaml
plugin: community.docker.docker_containers
docker_host: unix:///var/run/docker.sock
connection_type: docker-api           # reach containers through the Docker API (no SSH)
verbose_output: true                  # expose docker_* inspect data as host vars
strict: false
filters:
  - include: "'com.docker.compose.project' in docker_config.Labels and docker_config.Labels['com.docker.compose.project'] == 'ansible-lab'"
  - exclude: true
keyed_groups:
  - key: docker_config.Labels['com.docker.compose.service']
    prefix: svc
  - key: docker_config.Image | regex_replace('[:/.-]', '_')
    prefix: image
groups:
  web: docker_config.Labels['com.docker.compose.service'] in ['web1', 'web2']
compose:
  lab_ip: docker_networksettings.Networks['ansible-lab'].IPAddress
```

- `filters`: keep only containers from the `ansible-lab` Compose project (the Vault container from topic 41 is ignored), exclude everything else.
- `keyed_groups`: one group per Compose service (`svc_web1`) and per image (`image_ansible_lab_alma10`).
- `groups`: a conditional `web` group.
- `compose`: a new host variable `lab_ip` from Docker's network data.
- `connection_type: docker-api`: Ansible uses the `community.docker.docker_api` connection: no SSH at all.

(Plugin options evolve between collection versions. If an option is rejected, check `ansible-doc -t inventory community.docker.docker_containers`.)

```bash
ansible-inventory -i inventories/docker --graph
```
```
@all:
  |--@ungrouped:
  |--@svc_db1:
  |  |--ansible-lab-db1-1
  |--@svc_lb1:
  |  |--ansible-lab-lb1-1
  |--@svc_web1:
  |  |--ansible-lab-web1-1
  |--@svc_web2:
  |  |--ansible-lab-web2-1
  |--@image_ansible_lab_alma10:
  |  |--ansible-lab-db1-1
  |  |--ansible-lab-web1-1
  |--@image_ansible_lab_ubuntu2404:
  |  |--ansible-lab-lb1-1
  |  |--ansible-lab-web2-1
  |--@web:
  |  |--ansible-lab-web1-1
  |  |--ansible-lab-web2-1
```

```bash
ansible-inventory -i inventories/docker --host ansible-lab-web1-1 | grep -E '"lab_ip"|"ansible_connection"'
ansible -i inventories/docker web -m ansible.builtin.ping
```
```
    "ansible_connection": "community.docker.docker_api",
    "lab_ip": "172.30.0.11",
ansible-lab-web1-1 | SUCCESS => { "changed": false, "ping": "pong" }
ansible-lab-web2-1 | SUCCESS => { "changed": false, "ping": "pong" }
```

Scale it: `cd ~/ansible-lab/nodes && docker compose --profile bare up -d bare1` and re-run `--graph`: `svc_bare1` appears with no file edited.

### Lab: constructed groups from facts

Enable a fact cache in `~/infra/ansible.cfg` (topic 23):

```ini
[defaults]
fact_caching = ansible.builtin.jsonfile
fact_caching_connection = ~/.ansible/facts_cache
fact_caching_timeout = 86400
```

Fill the cache once: `ansible -i inventories/lab all -m ansible.builtin.setup >/dev/null`.

`inventories/lab/99-constructed.yml`:

```yaml
plugin: ansible.builtin.constructed
strict: false
use_vars_plugins: true
keyed_groups:
  - key: ansible_facts['distribution'] | default('unknown') | lower
    prefix: distro
groups:
  canary: inventory_hostname == 'web1'
  big_memory: (ansible_facts['memtotal_mb'] | default(0)) > 8000
```

```bash
ansible-inventory -i inventories/lab --graph distro_almalinux
ansible-inventory -i inventories/lab --graph canary
```

- `use_vars_plugins: true`: let constructed see `group_vars`/`host_vars` too.
- Facts come from the cache, so the groups exist **before** any host is contacted.

### Production: AWS EC2 (💰 needs an AWS account)

`inventories/prod/01-aws_ec2.yml`:

```yaml
plugin: amazon.aws.aws_ec2
regions:
  - eu-west-1
filters:
  instance-state-name: running
  tag:env: prod
hostnames:
  - tag:Name
  - private-dns-name
keyed_groups:
  - key: tags.role
    prefix: role
  - key: placement.availability_zone
    prefix: az
  - key: tags.os
    prefix: os
groups:
  canary: tags.get('canary', 'false') == 'true'
compose:
  ansible_host: private_ip_address
  ansible_user: "'ec2-user' if tags.get('os') == 'al2023' else 'ubuntu'"
cache: true
cache_plugin: ansible.builtin.jsonfile
cache_connection: ~/.ansible/inventory_cache
cache_timeout: 600
strict: true
```

- `compose: ansible_user: "'ec2-user' if ... else 'ubuntu'"`: note the **quotes inside quotes**: the compose value is a Jinja2 expression, so a literal string needs its own quotes.
- `ansible_host: private_ip_address`: connect over the private network (through a bastion or from a runner inside the VPC).
- Needs `amazon.aws` + `boto3`/`botocore` on the control node and credentials allowed to `ec2:DescribeInstances`.

```bash
ansible-inventory -i inventories/prod --graph role_web
```
```
@role_web:
  |--web-prod-01
  |--web-prod-02
  |--web-prod-03
```

---

## Hands-on lab

1. Set up the Docker dynamic inventory and get `--graph` and `ping` working.
2. Start and stop `bare1`; re-run `--graph` each time. Note how the inventory follows reality.
3. Add a `keyed_groups` entry by the container's health/status (`docker_state.Status`), see a group `status_running`.
4. Create the `constructed` file in the lab inventory using the fact cache. Use the new `canary` group in a play: `hosts: canary`.
5. Add `group_vars/svc_web1.yml` in `inventories/docker/` with a variable and confirm `ansible-inventory --host ansible-lab-web1-1` shows it.
6. Turn on `cache: true` for the Docker inventory, add a container, and see it missing until you use `--flush-cache`.
7. Optional 💰: create one EC2 instance tagged `env=lab, role=web` (Terraform topic 31 or the console), configure `aws_ec2.yml` with your region and filters, and list it. Destroy the instance afterwards.

**Done when:** you can create groups from tags/labels/facts and use them in `group_vars` and plays.

---

## Where you'll use it

- **Every cloud environment**: AWS/Azure/GCP inventories filtered by environment tags, grouped by role tags.
- **Autoscaling groups**: new instances get configured on the next run (or by a pull/AWX schedule, topics 55–56).
- **After Terraform**: Terraform sets the tags, Ansible groups by them (topic 43).
- **CMDB-driven shops**: NetBox/ServiceNow as source of truth.
- **AWX/AAP**: "inventory sources" are exactly these plugins, synced on a schedule (topic 56).

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `[WARNING]: Unable to parse ... as an inventory source` / plugin not used | File name doesn't end in the plugin's suffix (`aws_ec2.yml`, `docker.yml`) | Rename it; or enable the plugin in `[inventory] enable_plugins` |
| `Failed to import the required Python library (boto3)` / `docker` | Library missing on the control node | `uv pip install boto3 botocore` / `docker` |
| `Unable to locate credentials` / `AccessDenied` | No AWS credentials, or missing `ec2:DescribeInstances` | Fix the profile/role/OIDC and IAM policy |
| Groups missing, no error | `keyed_groups` key undefined for those hosts and `strict: false` | `strict: true` while developing; check tag names (case-sensitive) |
| New server not in the inventory | Cache still valid | `--flush-cache`; shorter `cache_timeout` |
| `compose` value is literally the attribute name | Missing inner quotes for a literal string | `"'ec2-user'"`; plain attribute names without quotes |
| constructed groups empty | Facts not cached / constructed file loaded **before** the source | Fill the cache; name it to sort last (`99-...`) |
| Hosts named by instance IDs | `hostnames` not set or tag missing | Set `hostnames` order with a fallback |
| Hosts unreachable though listed | `ansible_host` is a private IP, runner outside the VPC | Bastion `ProxyJump`, runner in the VPC, or SSM connection |
| Old `ec2.py` script breaks with new Python | Legacy script | Migrate to the `aws_ec2` plugin |

---

## Quick recap

- **Dynamic inventory** = an **inventory plugin** config file (`plugin: ...`) that queries the platform at the start of each run.
- Shape the result with **filters**, **`keyed_groups`** (groups from values like tags), **`groups`** (Jinja2 conditions), **`compose`** (host variables like `ansible_host`), `hostnames`.
- File-name suffixes matter (`aws_ec2.yml`, `docker.yml`). Credentials and Python libraries live on the **control node**.
- **`constructed`** adds groups/vars from any source (and cached facts); load it last.
- **Cache** for speed (`--flush-cache` to refresh). Mix static and dynamic sources in one inventory directory with shared `group_vars`. ⚠️ Scripts are legacy.

### Interview questions

1. **Q: What is a dynamic inventory?**
   A: An inventory generated at run time by a plugin that queries a source such as a cloud API or CMDB, so hosts and groups always reflect what exists, typically grouped by tags.
2. **Q: What do `keyed_groups` and `compose` do?**
   A: `keyed_groups` creates groups from variable values (e.g. tag `role=web` → `role_web`); `compose` sets host variables from expressions (e.g. `ansible_host: private_ip_address`).
3. **Q: How do you use group variables with a dynamic inventory?**
   A: Put `group_vars/<generated_group>.yml` next to the inventory config; generated groups behave like static groups.

### Scenario question (answer out loud first, then compare)

> *"Your AWS fleet autoscales between 10 and 80 web servers. New instances must be configured within minutes, and your playbook runs from a CI runner outside the VPC. How do you set up the inventory and connectivity?"*

**Model answer:**
1. Instances get tags at launch from the launch template (`env=prod`, `role=web`), set by Terraform.
2. `aws_ec2.yml` with filters on `env=prod` + running, `keyed_groups` on `tags.role`, `compose: ansible_host: private_ip_address`, short cache (or none) so new instances appear quickly.
3. Connectivity: either run the runner **inside** the VPC (self-hosted runner / AWX execution node), go through a **bastion** (`ProxyJump` in `group_vars/all`), or use the **SSM** connection plugin (no SSH ports open at all).
4. Timeliness: bake most configuration into the AMI (Packer + Ansible) so new instances are almost ready; then either a scheduled run every few minutes, an AWX job triggered by an EventBridge "instance launched" event (Event-Driven Ansible), or `ansible-pull` from user data (topic 55).
5. Credentials: OIDC from CI to a read-only role for the inventory + SSM, no stored keys.

**Likely follow-ups:**
- *What if the API is slow with 80 instances in many regions?* Limit regions, use filters server-side, enable caching with a short timeout.
- *How do you avoid configuring instances that are still booting?* `wait_for_connection` at the start of the play, and filter on instance state.
