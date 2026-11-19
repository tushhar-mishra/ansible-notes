# 60 · Security Best Practices

> **Level:** ★★★ advanced
>
> **You will learn:** the **threat model** of an Ansible setup (the control node and automation key are effectively root on every server), and how to defend each part: the **automation identity** (dedicated user, key restrictions, per-environment keys, **short-lived SSH certificates**), **control nodes and runners**, **secrets** (vault/secret managers, `no_log`, masking, no secrets on command lines or in logs), **supply chain** (pinned and verified collections, EEs, actions), **safe playbook code** (no shell injection, validation, guards against the wrong environment, safe file modes, untrusted templates), **access control** (reviews, approvals, RBAC), **audit** and **patching Ansible itself**. You'll switch the lab to SSH certificates that expire after one hour.
>
> **You need:** [59 · Error catalogue](59-error-catalogue.md), [19 · become security](19-become.md), [40](40-ansible-vault.md)/[41](41-external-secrets.md) (secrets), [37 · supply chain](37-collections-and-galaxy.md), [53](53-ansible-in-github-actions.md)/[56](56-awx-and-aap.md) (CI/AWX controls), [Linux 064–066 · crypto and SSH hardening](../../linux-mastery-rhel/notes/066-ssh-server-hardening.md).
>
> **Part 12 · Production operations** · Next: [61 · Scaling to many servers](61-scaling.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Whoever controls your Ansible **control node**, **automation key**, **repository** or **pipeline** controls **every server** it manages. Security means shrinking and guarding those four things: short-lived credentials, protected runners, reviewed code, pinned dependencies, secrets that never leak, and a full audit trail.

**Picture it.** The automation key is a **master key** to every room in the building. You don't make copies for everyone, you don't leave it on a desk, it should expire tonight, and the logbook records every use.

**The one thing to know.** The top ten:

| # | Practice |
|---|---|
| 1 | Dedicated automation user; key-only; restrict where it can log in from; **short-lived certificates** |
| 2 | Production credentials only in CI/AWX, never on laptops; separate per environment |
| 3 | Secrets in vault/secret manager; `no_log`; no `-vvv` in CI; nothing secret on command lines |
| 4 | Host key checking **on** |
| 5 | Pin and verify everything: ansible-core, collections, EEs, CI actions |
| 6 | Reviewed changes only: PRs, CODEOWNERS, check-mode plans, approvals for prod |
| 7 | Safe code: modules over `shell`, `quote` filter, `validate`, explicit `mode`, environment guards |
| 8 | Least privilege in AWX/CI: who can run what, where |
| 9 | Audit: job history, logs to a central system, Git history |
| 10 | Patch ansible-core, collections and the control nodes |

**Try it now:** `grep -rniE 'password|token|secret' ~/infra --include='*.yml' | grep -v vault_ | grep -v '{{'` → anything printed is a potential plaintext secret.

**You've got it if you can:** explain why `NOPASSWD: ALL` for the automation user is acceptable only when the key itself is well protected.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### The threat model

| Asset | Why it matters | Attacker gains |
|---|---|---|
| **Automation SSH key/cert** | Logs in as a user with sudo on every host (topic 19) | Root on the fleet |
| **Control node / CI runner / AWX** | Holds keys, vault passwords, cloud credentials; runs code from the repo | Everything above + cloud access |
| **Repository** | Defines what runs as root everywhere; may contain vaulted secrets | Arbitrary code as root on merge; offline vault cracking |
| **Vault password / secret-manager access** | Decrypts all secrets | Database, API, cloud credentials |
| **Collections, roles, EEs, CI actions** | Third-party code executed with your credentials | Supply-chain compromise |
| **Logs and job output** | May contain secrets or sensitive data | Credential leaks |

### 1. The automation identity

| Control | How |
|---|---|
| Dedicated, non-human account | `ansible`/`svc_ansible`, no password (`usermod -p '*'`), no interactive use |
| Key-only, modern keys | ed25519; `PasswordAuthentication no` |
| Restrict source addresses | `from="10.0.5.0/24"` in `authorized_keys` (via `authorized_key`'s `key_options`) + firewall/security groups allowing SSH only from runners/bastions |
| Separate per environment | Different user/key for prod vs staging; prod key only in the prod pipeline/AWX |
| **SSH certificates** (preferred at scale) | A CA signs the automation key with a **short validity** (e.g. 1 hour) and a principal (`ansible`); servers trust the CA (`TrustedUserCAKeys`). No `authorized_keys` to distribute or clean up; stolen certs expire |
| Rotation | Regular, automated (Ansible itself rotates `authorized_keys` or CA trust) |
| sudo | NOPASSWD is standard for automation (Ansible runs `/bin/sh` anyway, topic 19); compensate with the controls above. Where policy demands a password, vault it |
| Humans | Personal accounts via SSO/IdM, MFA on bastions; not the automation key |

### 2. Control nodes, runners and AWX

| Control | How |
|---|---|
| Purpose-built | Control nodes/runners only run automation; no browsing, no personal tools |
| Ephemeral CI runners | Fresh runner per job (GitHub ARC/ephemeral runners, Kubernetes agents) |
| Network segmentation | Runners in a management network; only they can reach SSH on servers |
| Patching | OS, Python, ansible-core, collections, AWX/AAP |
| Access | MFA, least privilege; audit logins |
| Secrets in memory | ssh-agent, password scripts, injected credentials; no key files left in workspaces |

### 3. Secrets

(Topics 40–41.)

| Do | Don't |
|---|---|
| Vault / secret manager / AWX credentials | Plain text in Git, even "temporarily" |
| `no_log: true` on secret-handling tasks; `register_secret` for looked-up values | Rely only on 2.22 masking for derived values |
| `diff: false` on templates with secrets | `--diff` on secret files in CI logs |
| Pass secrets via module options, `environment:`, `0600` files | Command-line arguments (`ps`, audit logs) |
| Short-lived / dynamic credentials | Long-lived shared passwords |
| Rotate secrets after any suspected exposure | Only rekey the vault (Git history keeps old ciphertexts) |
| Secret scanning in CI (e.g. gitleaks, trufflehog) and pre-commit | Hope reviewers notice |

### 4. Host identity

- Keep `host_key_checking = True`. Distribute verified `known_hosts` or use **SSH host certificates** (servers' host keys signed by a CA; clients trust the CA with `@cert-authority` in `known_hosts`).
- Never set `StrictHostKeyChecking=no` in production; it makes man-in-the-middle trivial and exposes become passwords and secrets.

### 5. Supply chain

| Layer | Control |
|---|---|
| ansible-core, ansible-lint, Python deps | Pinned in `requirements.txt` (optionally with `--require-hashes`) |
| Collections/roles | Pinned in `requirements.yml`; private hub mirror; `ansible-galaxy collection verify`; signatures (topic 37) |
| Execution environments | Built in CI from pinned files; image scanning; signed images; immutable tags |
| CI actions/plugins | Pin GitHub Actions by SHA; minimal Jenkins plugins, kept updated |
| Review | Read community collections/roles before adopting, especially anything running `shell` or downloading |

### 6. Safe playbook code

| Risk | Safe pattern |
|---|---|
| Shell injection | Prefer modules; if `shell`/`command` is unavoidable, `| quote` every variable (topic 22) |
| Broken configs locking you out | `validate:` for sudoers/sshd/nginx...; validate-then-restart handlers (topics 18, 30) |
| Wrong environment | `assert` that the inventory's environment matches what was requested; separate inventories and credentials (topics 10, 31) |
| World-readable secrets/keys | Explicit `mode` on every file (`risky-file-permissions` lint rule) |
| Downloads without integrity | `get_url` with `checksum`; signed repos (`gpgcheck`) |
| Untrusted data rendered as templates | ansible-core 2.19+ doesn't template untrusted strings (module output, fetched files); don't work around it |
| Hidden failures | Avoid `ignore_errors`; precise `failed_when` (topic 31) |
| Blast radius | `serial` + `max_fail_percentage`, canaries, check mode first (topics 20, 45) |
| Leaving debug artefacts | Never commit `ANSIBLE_KEEP_REMOTE_FILES`, `debugger`, `-vvvv` in pipelines |

### 7. Access control and change control

- Protected `main` branch; required reviews; **CODEOWNERS** for inventories, roles and workflows.
- Production applies only from `main`, through CI environments with required approvers, or AWX templates with RBAC (topics 53, 56).
- Check-mode plans attached to PRs; changes applied in batches.
- Separate duties: people who approve prod changes aren't the only authors.

### 8. Audit

| Source | Gives |
|---|---|
| Git history + PRs | What changed, who wrote and approved it |
| CI/AWX job history | Who ran what, when, against which hosts, with which result (keep it, ship it to a SIEM) |
| `log_path` / callbacks | Ansible-side log (protect: may contain data) |
| Target: sudo logs, journald, auditd | What happened on the server (sudo shows only the wrapper; auditd rules on sensitive files help, [Linux 077](../../linux-mastery-rhel/notes/077-auditd.md)) |

### 9. Patch Ansible itself

ansible-core publishes security releases (patch versions) for supported branches; collections fix CVEs too. Track them, keep within the support window (topic 63), rebuild EEs regularly.

---

## Why it exists

Configuration management concentrates power: one compromised key or merged pull request can change every server at once. The same automation that makes you fast also makes an attacker fast. Treating the automation path as **critical infrastructure** (least privilege, short-lived credentials, reviews, pinning, audit) is what lets security teams approve automation in production at all.

---

## Analogy

**A bank's cash-in-transit service.** The armoured van (control node) is guarded and only used for this job. The crew's keys (credentials) are issued per shift and expire (certificates). Routes are planned and approved in advance (PRs and approvals). Suppliers of the van's parts are vetted (supply chain). Every stop is logged (audit). And the crew never shouts the safe combination across the street (secrets on command lines and logs).

---

## How it works inside

SSH user certificates (the lab below):

```
 CA key pair (kept offline/in a secret manager)
     │ signs
     ▼
 ansible_key.pub ──▶ ansible_key-cert.pub   (principal "ansible", valid 1 hour, serial, key ID)
                                                │ presented automatically by ssh/Ansible
 server: sshd_config TrustedUserCAKeys /etc/ssh/ca_user.pub ◀─┘
     sshd checks: signed by the trusted CA? principal matches the login user? not expired?
     → login allowed, no authorized_keys entry needed
```

---

## Code / commands

### Lab: short-lived SSH certificates for the automation user

**1. Create a user CA** (in the lab, a file; in production, keep the CA private key in a secret manager or HashiCorp Vault's SSH engine):

```bash
mkdir -p ~/lab-ca && cd ~/lab-ca
ssh-keygen -t ed25519 -f ca_user -N "" -C "lab user CA"
```

**2. Trust the CA on every node** (add to the `baseline` role, or a one-off play):

`playbooks/ssh-ca.yml`:

```yaml
---
- name: Trust the lab user CA for SSH logins
  hosts: all
  become: true
  tasks:
    - name: CA public key
      ansible.builtin.copy:
        src: "{{ lookup('ansible.builtin.env', 'HOME') }}/lab-ca/ca_user.pub"
        dest: /etc/ssh/ca_user.pub
        owner: root
        mode: "0644"

    - name: sshd trusts the CA
      ansible.builtin.copy:
        dest: /etc/ssh/sshd_config.d/30-user-ca.conf
        content: "TrustedUserCAKeys /etc/ssh/ca_user.pub\n"
        owner: root
        mode: "0644"
        validate: /usr/sbin/sshd -t -f %s
      notify: Restart sshd

  handlers:
    - name: Restart sshd
      ansible.builtin.systemd_service:
        name: "{{ 'sshd' if ansible_facts['os_family'] == 'RedHat' else 'ssh' }}"
        state: restarted
```

```bash
cd ~/infra
ansible-playbook -i inventories/lab playbooks/ssh-ca.yml
```

**3. Sign the automation key for one hour:**

```bash
ssh-keygen -s ~/lab-ca/ca_user -I "ansible-$(date +%Y%m%d%H%M)" -n ansible -V +1h ~/.ssh/ansible_key.pub
ssh-keygen -Lf ~/.ssh/ansible_key-cert.pub
```
```
/home/you/.ssh/ansible_key-cert.pub:
        Type: ssh-ed25519-cert-v01@openssh.com user certificate
        Public key: ED25519-CERT SHA256:Qm3k8T...
        Signing CA: ED25519 SHA256:Ca9...  (using ssh-ed25519)
        Key ID: "ansible-202611191400"
        Serial: 0
        Valid: from 2026-11-19T13:59:00 to 2026-11-19T14:59:00
        Principals:
                ansible
        ...
```

- `-n ansible`: the **principal** (allowed login name).
- `-V +1h`: valid for one hour.
- OpenSSH automatically presents `ansible_key-cert.pub` next to `ansible_key` (and Ansible uses OpenSSH).

**4. Remove the static key from one node and prove the certificate works:**

```bash
ssh ansible@172.30.0.11 'sudo truncate -s0 ~ansible/.ssh/authorized_keys'
ansible -i inventories/lab web1 -m ansible.builtin.ping                 # SUCCESS via the certificate
```

Wait until the certificate expires (or sign one with `-V -1h:-1m`, already expired) and try again: `Permission denied (publickey)`. A stolen key+cert is useless after its validity window.

**5. Restore** the static key on web1 (`./nodes/reset.sh` or re-add it with `authorized_key`) if you want to keep using it for the rest of the course.

In production: CI/AWX requests a fresh cert per job from the CA service (HashiCorp Vault SSH secrets engine, Teleport, Smallstep...), valid for the job's duration.

### Quick security audit of the repo

```bash
cd ~/infra
# 1. Plaintext secret candidates
grep -rniE '(password|passwd|token|secret|api_key)\s*[:=]' --include='*.yml' --include='*.yaml' . \
  | grep -vE 'vault_|\{\{|no_log|collections/ansible_collections' || echo "none found"
# 2. Unencrypted vault files
for f in $(git ls-files '*vault*.yml'); do head -1 "$f" | grep -q '^\$ANSIBLE_VAULT' || echo "NOT ENCRYPTED: $f"; done
# 3. Host key checking disabled anywhere?
grep -rn -iE 'host_key_checking\s*=\s*false|StrictHostKeyChecking=no' . --include='*.cfg' --include='*.yml' --include='*.ini'
# 4. Lint security-relevant rules
ansible-lint -t risky-file-permissions,no-free-form,command-instead-of-shell,risky-shell-pipe || true
```

Add a secret scanner to pre-commit/CI (e.g. gitleaks) for stronger coverage.

### Restrict the key to known sources (if not using certificates)

```yaml
- name: Automation key only from the management network
  ansible.posix.authorized_key:
    user: ansible
    key: "{{ lookup('ansible.builtin.file', 'files/ansible_key.pub') }}"
    key_options: 'from="172.30.0.1,10.0.5.0/24",no-agent-forwarding,no-X11-forwarding'
    exclusive: true
```

- `exclusive: true` removes any other keys (careful: test on one host first, topic 17).

### Environment guard in every production playbook

```yaml
- name: Refuse to run against the wrong environment
  hosts: all
  gather_facts: false
  tasks:
    - ansible.builtin.assert:
        that:
          - env_name is defined
          - target_env is defined
          - env_name == target_env
        fail_msg: "Inventory is '{{ env_name | default('?') }}' but you asked for '{{ target_env | default('?') }}'"
      run_once: true
      delegate_to: localhost
```

(`env_name` set in each inventory's `group_vars/all`; `target_env` passed by the pipeline with `-e`.)

---

## Hands-on lab

1. Do the SSH certificate lab: CA, trust, sign for 1 hour, remove `authorized_keys` on web1, prove access, then prove expiry.
2. Run the repo audit commands; fix anything found.
3. Add gitleaks (or another scanner) to `.pre-commit-config.yaml` and commit a fake `aws_secret_access_key` to see it blocked.
4. Add the environment guard play to `site.yml` with `env_name: lab` in `inventories/lab/group_vars/all/main.yml`; run with `-e target_env=prod` and see it refuse.
5. Review your CI workflow/Jenkinsfile against section 7: protected branch, environments, approvals, concurrency, pinned actions.
6. Write a one-page threat model for `~/infra`: assets, who can access each, and the control you rely on.

**Done when:** a stolen automation key from your lab would stop working within an hour, and your repo passes the audit.

---

## Where you'll use it

- **Security reviews / audits** of automation (SOC 2, ISO 27001, PCI DSS).
- **Designing CI/AWX** for production access.
- **Incident response** when a key, runner or repo is suspected compromised.
- **Interviews**: "How do you secure Ansible?" is a common senior question.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| Certificate login fails: `Permission denied (publickey)` | CA not trusted, principal ≠ login user, expired cert, clock skew | `ssh-keygen -Lf cert`; check `TrustedUserCAKeys`; NTP |
| `authorized_key exclusive` locked out automation | Automation key not in the list | Always include it; test one host first |
| Secrets found in CI logs | Missing `no_log`, verbose runs, diffs | `no_log`, `diff: false`, lower verbosity; rotate exposed secrets |
| Pipeline can deploy prod from a branch | Missing environment/branch protection | Environment rules, branch filters, CODEOWNERS |
| Third-party collection update broke/compromised runs | Unpinned versions | Pin, mirror, verify, review upgrades |
| Auditors can't tell who changed a server | Manual changes, shared accounts | All changes via pipeline/AWX; personal accounts for humans; drift reports |

---

## Quick recap

- The automation path (**key, control node/runner/AWX, repo, secrets, dependencies, logs**) is root on the fleet: protect it like production.
- Identity: dedicated user, key-only, `from=` limits, per-environment keys, **short-lived SSH certificates**; host key checking **on**.
- Secrets: vault/secret managers, `no_log`/`register_secret`, no command-line secrets, rotate on exposure, scan repos.
- Supply chain: pin + verify collections, EEs, actions; review community content.
- Code: modules over shell, `quote`, `validate`, explicit modes, environment guards, batches.
- Access/audit: PR reviews, approvals, RBAC, job history in a SIEM; patch Ansible and control nodes.

### Interview questions

1. **Q: How do you secure the SSH access Ansible uses?**
   A: A dedicated automation account with key-only auth, ideally short-lived SSH certificates from a CA; keys only in CI/AWX credential stores; restricted source addresses; per-environment credentials; host key verification enabled; regular rotation.
2. **Q: How do you keep secrets out of Ansible logs?**
   A: `no_log: true` on tasks handling secrets, `register_secret` for looked-up values (2.22), `diff: false` on secret files, no `-vvv` in CI, never pass secrets on command lines, and protect/rotate any logs that might contain them.
3. **Q: What supply-chain risks exist in Ansible and how do you mitigate them?**
   A: Third-party collections/roles, Python deps, EE base images and CI actions run with your credentials. Pin versions, mirror via a private hub, verify signatures/checksums, scan images, review code, and pin CI actions by SHA.

### Scenario question (answer out loud first, then compare)

> *"A security review finds: one shared automation key used for 3 years across all environments, stored on 12 laptops and in CI, with `NOPASSWD: ALL` everywhere. Propose a remediation plan that doesn't stop operations."*

**Model answer:**
1. **Immediately**: generate new per-environment keys; deploy them with Ansible alongside the old key (`authorized_key`), switch CI/AWX to the new keys, then remove the old key everywhere with `exclusive: true` (tested on a canary first). Revoke laptop copies by removing the old key from servers.
2. **Restrict**: `from=` options and firewall rules so automation SSH works only from CI runners/AWX execution nodes/bastion.
3. **Move to certificates**: stand up an SSH CA (e.g. HashiCorp Vault SSH engine); servers trust the CA via the baseline role; CI/AWX request short-lived certs per job; then remove static `authorized_keys`.
4. **People**: engineers use personal SSO accounts via a bastion with MFA; no human uses the automation identity.
5. **Process**: prod credentials only in the protected prod environment/AWX; approvals and RBAC; job logs to the SIEM; quarterly access review.
6. NOPASSWD stays (required in practice for Ansible), justified by the controls above.

**Likely follow-ups:**
- *How do you know you removed every copy of the old key?* You can't on laptops; that's why you rotate on the servers (the lock), not hunt the copies (the keys).
- *What if a server was unreachable during rotation?* It still trusts the old key; track unreachable hosts, isolate or fix them, re-run until zero remain.
