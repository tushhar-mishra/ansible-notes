# 44 · Beyond Linux Servers

> **Level:** ★★ intermediate
>
> **You will learn:** how Ansible manages things that aren't Linux servers over SSH: **Windows** (WinRM, PSRP or SSH; PowerShell modules from `ansible.windows`; `runas`), **network devices** (`network_cli`, `httpapi`, `netconf`; `ansible_network_os`; resource modules with `merged`/`replaced`/`overridden`/`deleted`/`gathered`), **cloud and SaaS APIs** (modules running on the control node with `connection: local`), **containers** (`community.docker`), and **Kubernetes** (`kubernetes.core.k8s`, `helm`). You'll deploy to a kind Kubernetes cluster and run a container with Ansible.
>
> **You need:** [43 · Ansible and Terraform](43-ansible-and-terraform.md), [11 · Connection plugins](11-connection-settings-and-ansible-cfg.md), [07 · Targets without Python](07-python-on-managed-nodes.md), your [Kubernetes notes, topic 11 (kind)](../../CKA-Certification-Course-2025/notes/11-lab-setup-kind-and-minikube.md).
>
> **Part 6 · Inventory at scale and the cloud** · Next: [45 · Strategies, forks and serial](45-strategies-forks-serial.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** The same playbook language works for any target. What changes is **how Ansible reaches it** (the connection plugin) and **where the module runs**:

| Target | Connection | Module runs on | Modules from |
|---|---|---|---|
| Linux | `ssh` | the target (Python) | `ansible.builtin`, ... |
| Windows | `winrm` / `psrp` / `ssh` | the target (**PowerShell**) | `ansible.windows`, `community.windows` |
| Switch/router/firewall | `network_cli` / `httpapi` / `netconf` | the **control node** (talks to the device) | `cisco.ios`, `arista.eos`, `junipernetworks.junos`... |
| Cloud/SaaS API | `local` | the **control node** | `amazon.aws`, `azure.azcollection`, ... |
| Containers | `local` (manage via the daemon) or `community.docker.docker` (exec into) | control node / container | `community.docker` |
| Kubernetes | `local` | the **control node** (API calls) | `kubernetes.core` |

**Picture it.** One manager, many kinds of workers: some speak English (Linux/Python), some French (Windows/PowerShell), some only accept forms through a hatch (network devices, APIs). The instructions (playbook) look the same; the delivery method differs.

**The one thing to know.**

```yaml
- name: Kubernetes from Ansible
  hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: Namespace exists
      kubernetes.core.k8s:
        api_version: v1
        kind: Namespace
        name: shop
        state: present
```

**Try it now:** the Hands-on lab creates a kind cluster and deploys an app to it with Ansible.

**You've got it if you can:** say for each target type where the module code actually runs and which credentials it uses.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Windows

| Item | Detail |
|---|---|
| Connections | **WinRM** (`ansible_connection: winrm`, HTTP 5985 / HTTPS 5986), **PSRP** (`psrp`, PowerShell Remoting Protocol, faster), or **OpenSSH for Windows** (`ssh` + `ansible_shell_type: powershell`) |
| Authentication (WinRM/PSRP) | **Kerberos** (domain accounts, recommended), NTLM, CredSSP, certificates. Basic auth only over HTTPS and only for labs |
| Requirements on Windows | PowerShell **5.1**, or PowerShell **7** (supported since ansible-core 2.21), .NET Framework 4.0+ |
| Modules | `ansible.windows.win_ping`, `win_feature`, `win_package`, `win_service`, `win_copy`, `win_template`, `win_user`, `win_updates`, `win_regedit`, `win_shell`, `win_powershell`… (`community.windows` for more) |
| Privilege | Usually log in as an admin; `become_method: runas` to act as another user (e.g. for network access with the user's credentials) |
| No Python needed | Modules are PowerShell; the control node still needs Python (`pywinrm` or `pypsrp` libraries) |

```ini
[windows]
win1 ansible_host=10.0.5.20

[windows:vars]
ansible_connection=psrp
ansible_psrp_auth=kerberos
ansible_user=svc_ansible@CORP.EXAMPLE.COM
ansible_psrp_cert_validation=validate
```

```yaml
- hosts: windows
  tasks:
    - name: IIS is installed
      ansible.windows.win_feature:
        name: Web-Server
        include_management_tools: true
        state: present
    - name: Patches applied (security only)
      ansible.windows.win_updates:
        category_names: [SecurityUpdates]
        reboot: true
```

### Network devices

Network devices can't run Python modules. Instead, the module runs **on the control node** and the connection plugin talks to the device:

| Connection | How | Devices |
|---|---|---|
| `ansible.netcommon.network_cli` | SSH to the device CLI, send commands, parse output | Cisco IOS/NX-OS, Arista EOS, Juniper, VyOS... |
| `ansible.netcommon.httpapi` | HTTPS API on the device | Arista eAPI, Cisco NX-API, Fortinet, Palo Alto (PAN-OS)... |
| `ansible.netcommon.netconf` | NETCONF (XML over SSH) | Juniper Junos, IOS-XR... |

Key variables:

```yaml
# group_vars/ios.yml
ansible_connection: ansible.netcommon.network_cli
ansible_network_os: cisco.ios.ios            # which "cliconf/terminal" plugins to use
ansible_user: netops
ansible_password: "{{ vault_netops_password }}"
ansible_become: true
ansible_become_method: enable                 # "enable" mode on Cisco
```

Module families per platform:

| Kind | Example | Use |
|---|---|---|
| `*_command` | `cisco.ios.ios_command` | Run show commands (read-only), with `wait_for` |
| `*_config` | `cisco.ios.ios_config` | Push config lines (`lines`, `parents`), `backup: true` |
| `*_facts` | `cisco.ios.ios_facts` | Gather device facts + resource facts |
| **Resource modules** | `cisco.ios.ios_interfaces`, `ios_vlans`, `ios_bgp_global`, `arista.eos.eos_vlans` | Declarative, structured config with **`state`** |

Resource module `state` values:

| State | Meaning |
|---|---|
| `merged` | Add/update what you give; leave other config alone |
| `replaced` | Make each listed object exactly as given (per object) |
| `overridden` | Make the **whole** resource match the list (removes unlisted objects) |
| `deleted` | Remove the listed (or all) objects |
| `gathered` | Read current config as structured data |
| `rendered` | Produce the CLI commands offline, without a device |
| `parsed` | Turn saved CLI text into structured data |

```yaml
- hosts: ios_switches
  gather_facts: false
  tasks:
    - name: VLANs are exactly these
      cisco.ios.ios_vlans:
        config:
          - { vlan_id: 10, name: users }
          - { vlan_id: 20, name: servers }
        state: overridden
```

### Cloud and SaaS APIs

Cloud modules call HTTPS APIs **from the control node**:

```yaml
- name: S3 bucket for backups
  hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: Bucket exists with versioning
      amazon.aws.s3_bucket:
        name: mycompany-backups-eu
        region: eu-west-1
        versioning: true
        state: present
```

- `hosts: localhost` + `connection: local`: no SSH; the module runs in your Ansible venv.
- Credentials and Python libraries (`boto3`) are on the **control node**.
- Remember topic 02: long-lived infrastructure belongs in **Terraform**. Use cloud modules for orchestration around configuration (snapshots before patching, DNS changes during cut-overs, registering instances), or quick one-offs.

### Containers (`community.docker`)

| Module | Does |
|---|---|
| `community.docker.docker_image` | Pull/build/push images |
| `community.docker.docker_container` | Run/maintain a container (idempotent: recreated only if config differs) |
| `community.docker.docker_compose_v2` | Manage a Compose project (`docker compose up/down`) |
| `community.docker.docker_network`, `docker_volume`, `docker_login` | Networks, volumes, registries |
| Connection `community.docker.docker` / `docker_api` | Run normal modules **inside** a container (topic 11) |

These modules talk to a Docker daemon: locally (`connection: local`) or on a remote host (run the task on that host over SSH; it needs the Docker SDK for Python there).

### Kubernetes (`kubernetes.core`)

| Module | Does |
|---|---|
| `kubernetes.core.k8s` | Create/update/delete any object from inline YAML, a file, or a **template** (`template:` renders Jinja2 first) |
| `kubernetes.core.k8s_info` | Read objects (with label selectors) |
| `kubernetes.core.helm` / `helm_repository` | Install/upgrade Helm charts |
| `kubernetes.core.k8s_scale`, `k8s_rollback`, `k8s_exec`, `k8s_log` | Day-2 operations |
| `wait: true` + `wait_condition` | Wait until a Deployment is available, a Job completes... |

They use your **kubeconfig** on the control node (`~/.kube/config` or `kubeconfig:`), and need the `kubernetes` Python library. Ansible + Kubernetes is common for **bootstrapping clusters and platform add-ons**; for app delivery, GitOps tools (Argo CD, Flux) are usually preferred ([Jenkins topic 72](../../Jenkins/notes/72-jenkins-and-gitops.md)).

---

## Why it exists

Real infrastructure is mixed: Linux and Windows servers, switches and firewalls, cloud services, container hosts, Kubernetes clusters. Using one automation language for all of it means one set of skills, one CI pipeline, one way to handle secrets and inventories, and orchestration **across** them ("drain the server in the load balancer, patch it, update the firewall rule, update the DNS record").

---

## Analogy

**A universal remote control.** One remote (Ansible), one set of buttons (playbook syntax), many devices: the TV speaks infrared (SSH/Python), the sound bar Bluetooth (WinRM/PowerShell), the smart lights an app API (httpapi/REST). The remote's codes library (collections) knows how to talk to each.

---

## How it works inside

```
 LINUX:    control node ──SSH──▶ host runs Python module ──JSON──▶ back
 WINDOWS:  control node ──WinRM/PSRP/SSH──▶ host runs PowerShell module ──JSON──▶ back
 NETWORK:  control node runs the module ──network_cli (SSH CLI) / httpapi (HTTPS)──▶ device
                (module builds commands, connection plugin sends them, parses replies)
 API/K8s:  control node runs the module ──HTTPS API──▶ cloud / Kubernetes API server
```

That's why, for network/API/Kubernetes work, the **control node** needs the libraries, network access and credentials: it does the talking.

---

## Code / commands

### Lab 1: Kubernetes with kind

Prerequisites: `kind` and `kubectl` from your [Kubernetes topic 11](../../CKA-Certification-Course-2025/notes/11-lab-setup-kind-and-minikube.md).

```bash
kind create cluster --name ansible-lab
kubectl get nodes
cd ~/infra
ansible-galaxy collection install kubernetes.core -p ./collections
uv pip install kubernetes
```

`roles/k8s_app/templates/deployment.yml.j2`:

```jinja
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ k8s_app_name }}
  namespace: {{ k8s_app_namespace }}
  labels: { app: {{ k8s_app_name }} }
spec:
  replicas: {{ k8s_app_replicas }}
  selector:
    matchLabels: { app: {{ k8s_app_name }} }
  template:
    metadata:
      labels: { app: {{ k8s_app_name }} }
    spec:
      containers:
        - name: web
          image: {{ k8s_app_image }}
          ports: [{ containerPort: 80 }]
          resources:
            requests: { cpu: 50m, memory: 32Mi }
            limits: { memory: 64Mi }
```

`playbooks/k8s.yml`:

```yaml
---
- name: Deploy to Kubernetes with Ansible
  hosts: localhost
  connection: local
  gather_facts: false
  vars:
    k8s_app_name: shop
    k8s_app_namespace: shop
    k8s_app_replicas: 2
    k8s_app_image: nginx:1.27
  tasks:
    - name: Namespace exists
      kubernetes.core.k8s:
        api_version: v1
        kind: Namespace
        name: "{{ k8s_app_namespace }}"
        state: present

    - name: Deployment is applied and available
      kubernetes.core.k8s:
        state: present
        template: "{{ playbook_dir }}/../roles/k8s_app/templates/deployment.yml.j2"
        wait: true
        wait_condition:
          type: Available
          status: "True"
        wait_timeout: 120

    - name: Service exists
      kubernetes.core.k8s:
        state: present
        definition:
          apiVersion: v1
          kind: Service
          metadata: { name: "{{ k8s_app_name }}", namespace: "{{ k8s_app_namespace }}" }
          spec:
            selector: { app: "{{ k8s_app_name }}" }
            ports: [{ port: 80, targetPort: 80 }]

    - name: Read the pods back
      kubernetes.core.k8s_info:
        kind: Pod
        namespace: "{{ k8s_app_namespace }}"
        label_selectors: ["app={{ k8s_app_name }}"]
      register: pods

    - name: Pod names and phases
      ansible.builtin.debug:
        msg: "{{ pods.resources | map(attribute='metadata.name') | zip(pods.resources | map(attribute='status.phase')) | list }}"
```

```bash
ansible-playbook playbooks/k8s.yml
ansible-playbook playbooks/k8s.yml          # changed=0: the API objects already match
kubectl -n shop get deploy,svc,pods
```

- `k8s` is idempotent: by default it GETs the live object, diffs it with the desired one and sends a **patch** (`merge_type`, strategic-merge by default) only for the differences. Real **server-side apply** happens only when you set `apply: true` with `server_side_apply: { field_manager: ansible }`, which is the better choice when other controllers also manage fields of the same object.
- Change `k8s_app_replicas: 3`, re-run: one `changed` and a third pod.

### Lab 2: a container with community.docker

```yaml
# playbooks/docker.yml
- name: Run Redis on the local Docker host
  hosts: localhost
  connection: local
  gather_facts: false
  tasks:
    - name: Redis container is running
      community.docker.docker_container:
        name: lab-redis
        image: redis:7.4
        state: started
        restart_policy: unless-stopped
        published_ports: ["127.0.0.1:6379:6379"]
```

```bash
ansible-playbook playbooks/docker.yml        # changed
ansible-playbook playbooks/docker.yml        # ok: same config, not recreated
docker exec lab-redis redis-cli ping         # PONG
docker rm -f lab-redis
```

### Reference: Windows and network examples (need real devices/VMs)

Windows (a Windows Server evaluation VM with WinRM over HTTPS enabled):

```bash
uv pip install pypsrp            # or pywinrm
ansible win1 -i inventories/win -m ansible.windows.win_ping
```
```
win1 | SUCCESS => { "changed": false, "ping": "pong" }
```

Network (a lab device, e.g. an Arista cEOS or Cisco virtual router in containerlab/EVE-NG):

```bash
ansible-galaxy collection install arista.eos ansible.netcommon
ansible-playbook -i inventories/net playbooks/vlans.yml --check --diff
```

Resource modules support `--check` (and `state: rendered` to see the commands without a device at all).

---

## Hands-on lab

1. Create the kind cluster and run `k8s.yml` twice (`changed=0` the second time). Scale with a variable change.
2. Delete a pod by hand (`kubectl -n shop delete pod <one>`): the Deployment recreates it; Ansible reports `ok` (the Deployment object didn't change). Now `kubectl -n shop scale deploy shop --replicas=5` by hand and re-run Ansible: it puts replicas back to the declared value (drift repair).
3. Run the Redis container playbook twice; then change the image tag and re-run: the container is **recreated** (config differs).
4. Use `state: rendered` on any network resource module (install e.g. `cisco.ios`) to see generated commands without a device:

   ```yaml
   - hosts: localhost
     gather_facts: false
     tasks:
       - cisco.ios.ios_vlans:
           config: [{ vlan_id: 10, name: users }]
           state: rendered
         register: r
       - ansible.builtin.debug: { var: r.rendered }
   ```

5. Clean up: `kind delete cluster --name ansible-lab`.

**Done when:** you can explain where each module ran in these labs, and why Kubernetes/Docker/network work doesn't need SSH to the target.

---

## Where you'll use it

- **Windows fleets**: patching, IIS, AD-joined servers, software installs, in the same pipelines as Linux.
- **Network automation**: VLANs, ACLs, config backups, compliance checks across hundreds of switches.
- **Cloud orchestration**: snapshots before patching, DNS cut-overs, temporary resources in workflows.
- **Container hosts**: running infrastructure containers on VMs (monitoring agents, reverse proxies).
- **Kubernetes**: bootstrapping clusters (kubeadm/Kubespray), installing operators and Helm charts, platform configuration.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| Windows: `winrm or requests is not installed` / `pypsrp` missing | Python library missing on the control node | `uv pip install pywinrm` / `pypsrp` |
| Windows: `the specified credentials were rejected by the server` / `401` | Wrong auth method, user, or domain format | Kerberos with `user@REALM`, check `ansible_*_auth`; verify with `Test-WSMan` |
| Windows: `certificate verify failed` | Self-signed WinRM cert | Trust the CA, or `..._cert_validation=ignore` (labs only) |
| Network: `unable to open shell` | Wrong `ansible_network_os`, or device prompts not recognised | Set the right `ansible_network_os`; check `-vvvv` |
| Network: config pushed every run | `*_config` with lines not matching the device's own format | Resource modules, or match the exact running-config syntax; `diff_against` |
| Network: `ansible_network_os must be specified` | Missing variable | Set it in `group_vars` |
| Kubernetes: `Failed to import the required Python library (kubernetes)` | Missing on the control node | `uv pip install kubernetes` |
| Kubernetes: `Unauthorized` / wrong cluster | kubeconfig/context | `kubeconfig:` and `context:` options; `kubectl config current-context` |
| Kubernetes: `wait` times out | Pods not becoming ready (image pull, resources) | `kubectl describe` / `kubernetes.core.k8s_info`; fix the manifest |
| Docker: `Error connecting: ... docker.sock permission denied` | User not allowed to use Docker | Docker group (= root, [Docker topic 04](../../Docker/notes/04-installing-docker.md)) or run where allowed |
| Cloud module tries SSH to `localhost` | Missing `connection: local` (or implicit localhost not used) | `hosts: localhost` + `connection: local` |

---

## Quick recap

- Same playbooks, different **connections**: `ssh` (Linux), `winrm`/`psrp`/`ssh`+PowerShell (Windows), `network_cli`/`httpapi`/`netconf` (network), `local` (APIs, Docker, Kubernetes).
- **Windows**: PowerShell modules (`ansible.windows`), Kerberos auth, `runas`; libraries `pywinrm`/`pypsrp` on the control node.
- **Network**: modules run on the control node; `ansible_network_os`; **resource modules** with `merged`/`replaced`/`overridden`/`deleted`/`gathered`/`rendered`/`parsed`.
- **Cloud/Kubernetes/Docker**: `hosts: localhost`, `connection: local`; credentials, kubeconfig and libraries on the control node; `kubernetes.core.k8s` with `template:` and `wait:`.
- Infrastructure lifecycle still belongs in Terraform; Ansible orchestrates and configures.

### Interview questions

1. **Q: How does Ansible manage Windows hosts?**
   A: Over WinRM, PSRP or OpenSSH, running PowerShell modules from `ansible.windows`/`community.windows` on the host; authentication typically via Kerberos; `runas` for privilege changes. No Python is needed on Windows.
2. **Q: Why do network modules run on the control node?**
   A: Network devices can't run Python modules, so the module runs locally and the `network_cli`/`httpapi`/`netconf` connection plugin sends commands or API calls to the device and parses the replies.
3. **Q: What's the difference between `merged` and `overridden` in network resource modules?**
   A: `merged` adds/updates only what you specify and leaves the rest; `overridden` makes the whole resource match your list, removing anything not listed.

### Scenario question (answer out loud first, then compare)

> *"A maintenance window requires, in order: take a web server out of the F5/HAProxy pool, snapshot its EBS volume, patch and reboot it (Linux), update a Windows reporting server, then put the web server back. How would you structure this in Ansible?"*

**Model answer:**
1. One playbook with several plays (or one play with delegation), run with `serial: 1` for the web tier:
   - **pre_tasks** on each web host: disable it in the load balancer via its API/module, `delegate_to: localhost` (or the LB host) (topic 46);
   - **snapshot**: `amazon.aws.ec2_snapshot` delegated to localhost, using the instance's volume ID from facts/inventory;
   - **patch + reboot**: `dnf`/`apt` + `ansible.builtin.reboot`, then a health check (`uri` + `until`);
   - **post_tasks**: re-enable in the LB, verify traffic.
2. A separate play for `hosts: windows_reporting` with `ansible.windows.win_updates` (reboot allowed).
3. Error handling: `block`/`rescue` per web host that leaves it out of the pool and fails the run if health checks fail; `max_fail_percentage: 0` so one failure stops the rollout (topic 31).
4. Credentials: LB API token, AWS role and Windows Kerberos credentials from the secret manager/AWX credentials.

**Likely follow-ups:**
- *Where does each module run?* LB and AWS modules on the control node; patching on the Linux hosts via SSH; Windows updates on the Windows host via WinRM/PSRP.
- *Why `serial: 1`?* So capacity is never reduced by more than one server, and a bad patch stops after the first host.
