# 41 · External Secret Managers

> **Level:** ★★★ advanced
>
> **You will learn:** why larger teams keep secrets in a **secret manager** instead of (or as well as) Ansible Vault, how Ansible reads them with **lookup plugins** (HashiCorp Vault / OpenBao via `community.hashi_vault`, AWS Secrets Manager and SSM Parameter Store via `amazon.aws`, Azure Key Vault, CyberArk, 1Password, Bitwarden), how the **control node authenticates** to the secret manager (tokens, AppRole, cloud IAM roles, CI **OIDC**), how to keep looked-up secrets out of logs (`no_log`, the 2.22 `register_secret` filter), how to avoid evaluating lookups hundreds of times, how to pass secrets to targets safely (not on command lines), and **dynamic secrets**. You'll run a local HashiCorp Vault in Docker and read a secret from a playbook.
>
> **You need:** [40 · Ansible Vault](40-ansible-vault.md), [32 · Lookups](32-advanced-jinja2-and-lookups.md) (they run on the control node), [37 · Collections](37-collections-and-galaxy.md).
>
> **Part 5 · Secrets** · Next: [42 · Dynamic inventory](42-dynamic-inventory.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** A **secret manager** is a server whose only job is storing secrets, controlling **who** may read **which** secret, **logging** every access, and **rotating** secrets. Ansible doesn't store the secret at all: at run time, a **lookup** on the control node asks the secret manager for it.

**Picture it.** Ansible Vault is a locked box you carry around (in Git). A secret manager is a **bank teller**: you show ID each time, they check your permission, hand over only what you're allowed, and write it in the ledger.

**The one thing to know.**

```yaml
# group_vars/all/main.yml
app_db_password: "{{ lookup('community.hashi_vault.vault_kv2_get', 'shop/db', engine_mount_point='secret').secret.password }}"
app_api_token:   "{{ lookup('amazon.aws.secretsmanager_secret', 'prod/shop/api-token') }}"
```

```yaml
- name: Use it, quietly
  ansible.builtin.template: { src: app.env.j2, dest: /opt/app/app.env, mode: "0600" }
  no_log: true
```

The control node needs: network access to the secret manager, the client library (`hvac` for HashiCorp Vault, `boto3` for AWS), and **credentials** to authenticate (from CI, a token, or a cloud role).

**Try it now:** follow the Hands-on lab: start Vault in Docker, store a secret, read it with Ansible.

**You've got it if you can:** explain who authenticates to the secret manager (the control node, not the targets) and how you keep the value out of logs.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Ansible Vault vs a secret manager

| | Ansible Vault (topic 40) | Secret manager |
|---|---|---|
| Where secrets live | Encrypted in **Git** | On a **server** (not in Git) |
| Access control | Whoever has the vault password (per vault ID) | **Per identity, per secret path** (policies) |
| Audit | None | Every read logged (who, when, what) |
| Rotation | Edit file, re-run, tell everyone the password | Built in; often **automatic** |
| Dynamic / short-lived secrets | No | ✅ (e.g. DB credentials valid for 1 hour) |
| Works offline / no infrastructure | ✅ | ❌ Needs the service |
| Cost/complexity | Zero | A service to run/pay for |

Many teams use **both**: Vault files for a few bootstrap values, a secret manager for everything else.

### The common secret managers and their lookups

| Secret manager | Collection | Lookup (FQCN) | Python lib on control node |
|---|---|---|---|
| **HashiCorp Vault** / **OpenBao** (the open-source fork, API compatible) | `community.hashi_vault` | `community.hashi_vault.vault_kv2_get` (KV v2), `vault_kv1_get`, `vault_read` (any path, e.g. dynamic DB creds) | `hvac` |
| **AWS Secrets Manager** | `amazon.aws` | `amazon.aws.secretsmanager_secret` | `boto3` |
| **AWS SSM Parameter Store** | `amazon.aws` | `amazon.aws.ssm_parameter` | `boto3` |
| **Azure Key Vault** | `azure.azcollection` | `azure.azcollection.azure_keyvault_secret` | `azure-*` SDK |
| **Google Secret Manager** | `google.cloud` | `google.cloud.gcp_secret_manager` | `google-auth`, `requests` |
| **CyberArk** Conjur / CCP | `cyberark.conjur`, `cyberark.pas` | `cyberark.conjur.conjur_variable` | |
| **1Password**, **Bitwarden** | `community.general` | `community.general.onepassword`, `community.general.bitwarden` | their CLIs |
| AWX / AAP | built in | **Credential plugins** inject secrets into jobs without lookups in your code (topic 56) | |

(Names of AWS lookups changed over the years: older code uses `amazon.aws.aws_secret` / `amazon.aws.aws_ssm`, which were renamed. Use `ansible-doc -t lookup -l | grep amazon` to see what your installed version has.)

### Who authenticates, and how

Lookups run on the **control node**, so the **control node** (your laptop, the CI runner, the AWX execution node) must prove its identity to the secret manager:

| Method | How | Good for |
|---|---|---|
| Token in an env var | `VAULT_TOKEN`, `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` | Labs, quick tests. ⚠️ Long-lived tokens are a risk |
| HashiCorp Vault **AppRole** | role_id + secret_id → short-lived token | Servers/CI without cloud identity |
| **Cloud IAM role** | EC2 instance profile, EKS IRSA/Pod Identity, Azure Managed Identity | Control nodes running in the cloud |
| **OIDC from CI** | GitHub Actions / GitLab / Jenkins get a signed identity token; AWS STS or Vault's JWT auth exchange it for short-lived credentials | ✅ CI pipelines: **no stored secret at all** ([GitHub Actions topic 36](../../GitHub-Actions-Basics-To-Production/notes/36-oidc-cloud-authentication.md)) |
| AAP credential plugin | Controller holds the connection to the secret manager | AAP users |

The first credential always has to come from **somewhere** (the "secret zero" problem). OIDC and cloud roles solve it best: the identity comes from the platform, not from a stored secret.

### Keeping secrets out of logs

ansible-core 2.22 automatically masks **vault** values, **`no_log`** module options and **prompted** passwords. Values that come from a **lookup** are **not** automatically known to be secret. So:

| Tool | Use |
|---|---|
| `no_log: true` on every task that uses the secret | Hides arguments and results |
| `"{{ lookup(...) \| register_secret }}"` (2.22+) | Marks the value as a secret so it's masked as `$REDACTED$` everywhere |
| Don't `debug` secrets; don't run CI at `-vvv` | Verbose output shows module arguments |
| `diff: false` on templates containing secrets | Diffs show file contents |

### Evaluate once, not per task

Lookups are **lazy** (topic 32): a variable defined with a lookup is fetched **every time** it's used, **per host**. 50 hosts × 3 tasks = 150 calls to the secret manager (slow, and noisy in its audit log, and may hit rate limits). Fetch once:

```yaml
- name: Fetch secrets once for the whole play
  ansible.builtin.set_fact:
    _app_secrets: "{{ lookup('community.hashi_vault.vault_kv2_get', 'shop/app', engine_mount_point='secret').secret | register_secret }}"
  run_once: true
  delegate_to: localhost
  no_log: true
```

- `run_once: true`: run once for the play (not per host); with `set_fact`, the fact is applied to **all** hosts in the play.
- Later tasks use `_app_secrets.db_password`.

### Passing secrets to targets safely

| ❌ Avoid | Why | ✅ Instead |
|---|---|---|
| `command: mysql -p{{ pw }} ...` | Command lines are visible in `ps` to every local user, and in shell history/audit logs | Module options (`community.mysql.*` with `login_password`), env vars via `environment:`, or a `0600` config file |
| Writing secrets into world-readable files | Any local user can read them | `mode: "0600"`, correct owner |
| Leaving secrets in `/tmp` | Survive and leak | Write to the final place; remove temp files in `always:` |
| Logging command output that contains secrets | Logs are long-lived | `no_log`, quiet commands |

```yaml
- name: Run a migration with the password in the environment, not the command line
  ansible.builtin.command: /opt/app/bin/migrate
  environment:
    DB_PASSWORD: "{{ _app_secrets.db_password }}"
  no_log: true
  changed_when: false
```

### Dynamic secrets

Some secret managers can **create** credentials on demand: HashiCorp Vault's database engine returns a brand-new DB user/password valid for, say, 1 hour, and deletes it afterwards. With `community.hashi_vault.vault_read` on `database/creds/<role>`, each Ansible run uses **fresh, short-lived** credentials: nothing long-lived to leak.

---

## Why it exists

As teams and fleets grow, a shared vault password doesn't scale: you can't revoke one person, you can't see who read what, rotation is manual, and compliance (PCI, SOC 2, ISO 27001) demands access control and audit trails. Central secret managers provide those, and Ansible's lookup plugins let playbooks use them without changing how tasks are written.

---

## Analogy

**A hotel's key card system** vs **a bunch of physical keys**. Physical keys (vault password) open everything they fit, forever, and you can't tell who used them. Key cards (secret manager) are issued per person, open only certain doors, expire at checkout (dynamic secrets), and every swipe is logged. The front desk (the control node) must show its staff badge (IAM role / OIDC) to the card system before it can print a card for a guest (the target).

---

## How it works inside

```
 ansible-playbook (on CI runner)               secret manager (HashiCorp Vault / AWS SM)
 ──────────────────────────────                ─────────────────────────────────────────
 1. CI gives the runner an OIDC token
 2. lookup plugin authenticates ─────────────▶ verify token/role → issue short-lived credentials
 3. template a task arg that uses the secret
    → lookup('...vault_kv2_get', 'shop/db') ─▶ check policy: may this identity read shop/db?
                                    ◀───────── return {password: ...}  + write audit log entry
 4. value inserted into the module args (control node, in memory)
 5. module shipped to the target over SSH (private temp dir, umask 77)
    → target writes /opt/app/app.env (0600)
 6. no_log / register_secret → "$REDACTED$" in output
```

The **targets never talk to the secret manager** in this design. (Alternatively, apps on the targets can fetch secrets themselves at runtime with an agent; then Ansible only configures that agent.)

---

## Code / commands

### Lab: HashiCorp Vault in Docker

**1. Start a dev-mode Vault** (in-memory, unsealed, root token `root`; **never** use dev mode for real):

```bash
docker run -d --name vault-dev -p 8200:8200 \
  -e VAULT_DEV_ROOT_TOKEN_ID=root hashicorp/vault:1.20
export VAULT_ADDR=http://127.0.0.1:8200
export VAULT_TOKEN=root
```

(HashiCorp Vault is under the BSL licence since 2023. **OpenBao** (`openbao/openbao` image) is the open-source fork with the same API; the Ansible collection works with both.)

**2. Store secrets** (KV version 2 engine, mounted at `secret/` in dev mode):

```bash
docker exec -e VAULT_ADDR -e VAULT_TOKEN vault-dev vault kv put secret/shop/app \
  db_password='Db-From-Vault-99' api_token='tok_from_vault_42'
docker exec -e VAULT_ADDR -e VAULT_TOKEN vault-dev vault kv get secret/shop/app
```
```
== Secret Path ==
secret/data/shop/app
...
======= Data =======
Key            Value
---            -----
api_token      tok_from_vault_42
db_password    Db-From-Vault-99
```

- `secret/data/shop/app`: KV v2 stores data under `data/`; the Ansible lookup handles that for you.

**3. Install the collection and library:**

```bash
cd ~/infra
uv pip install hvac
ansible-galaxy collection install community.hashi_vault -p ./collections
```

**4. Read it from a playbook** (`playbooks/vault-lookup.yml`):

```yaml
---
- name: Secrets from HashiCorp Vault
  hosts: web
  become: true
  tasks:
    - name: Fetch the app secrets once
      ansible.builtin.set_fact:
        _shop: "{{ lookup('community.hashi_vault.vault_kv2_get', 'shop/app', engine_mount_point='secret').secret | register_secret }}"
      run_once: true
      delegate_to: localhost
      become: false
      no_log: true

    - name: Credentials file on the web servers
      ansible.builtin.copy:
        dest: /opt/app/credentials.env
        content: |
          DB_PASSWORD={{ _shop.db_password }}
          API_TOKEN={{ _shop.api_token }}
        owner: root
        mode: "0600"
      no_log: true

    - name: Proof the value is masked
      ansible.builtin.debug:
        msg: "token = {{ _shop.api_token }}, length = {{ _shop.api_token | length }}"
```

- The lookup reads `VAULT_ADDR` and `VAULT_TOKEN` from the environment (the collection also accepts `url=`, `token=`, `auth_method=` options, or `ANSIBLE_HASHI_VAULT_*` variables).
- `.secret`: `vault_kv2_get` returns a dict with `secret` (the data), `metadata`, `raw`.
- `delegate_to: localhost` + `become: false`: the lookup runs on the control node anyway; this keeps the `set_fact` task from trying sudo on your machine (topic 19's trap).

```bash
ansible-playbook -i inventories/lab playbooks/vault-lookup.yml
```
```
TASK [Fetch the app secrets once] ******************************************************
ok: [web1 -> localhost]

TASK [Credentials file on the web servers] *********************************************
changed: [web1]
changed: [web2]

TASK [Proof the value is masked] *******************************************************
ok: [web1] => {
    "msg": "token = $REDACTED$, length = 17"
}
```

- Fetched **once** (`run_once`), used on both hosts.
- `$REDACTED$` thanks to `register_secret`. Remove `| register_secret` and run again: the token would be printed. That's the difference between lookup values and vault values.

**5. Rotate and re-run:**

```bash
docker exec -e VAULT_ADDR -e VAULT_TOKEN vault-dev vault kv put secret/shop/app \
  db_password='Rotated-Db-100' api_token='tok_rotated_43'
ansible-playbook -i inventories/lab playbooks/vault-lookup.yml        # credentials file changes
```

No change to the repo: rotation is entirely in the secret manager.

**6. Least privilege instead of the root token** (policy + token for "read shop only"):

```bash
cat > /tmp/shop-read.hcl <<'EOF'
path "secret/data/shop/*" { capabilities = ["read"] }
EOF
docker cp /tmp/shop-read.hcl vault-dev:/tmp/shop-read.hcl
docker exec -e VAULT_ADDR -e VAULT_TOKEN vault-dev vault policy write shop-read /tmp/shop-read.hcl
export VAULT_TOKEN=$(docker exec -e VAULT_ADDR -e VAULT_TOKEN=root vault-dev \
  vault token create -policy=shop-read -ttl=1h -field=token)
ansible-playbook -i inventories/lab playbooks/vault-lookup.yml        # still works
docker exec -e VAULT_ADDR -e VAULT_TOKEN vault-dev vault kv get secret/other   # permission denied
```

The token can read `shop/*` only, and expires in 1 hour.

**7. Clean up:** `docker rm -f vault-dev; unset VAULT_TOKEN VAULT_ADDR`.

### AWS equivalents (for reference, 💰 tiny cost)

```yaml
app_api_token: "{{ lookup('amazon.aws.secretsmanager_secret', 'prod/shop/api-token', region='eu-west-1') }}"
app_db_host:   "{{ lookup('amazon.aws.ssm_parameter', '/prod/shop/db_host', region='eu-west-1') }}"
```

Credentials come from the standard AWS chain: env vars, `~/.aws` profile, instance role, or OIDC-assumed role in CI. The IAM policy should allow `secretsmanager:GetSecretValue` / `ssm:GetParameter` on **only** those ARNs.

---

## Hands-on lab

1. Do steps 1–5 above. Confirm `/opt/app/credentials.env` on web1 has the rotated values after step 5.
2. Remove `run_once`/`set_fact` and use the lookup directly in the `copy` task. Watch the Vault audit effect: count requests by enabling an audit device (`vault audit enable file file_path=/tmp/audit.log` in the container) and `docker exec vault-dev wc -l /tmp/audit.log` before and after.
3. Remove `| register_secret` and see the token printed; put it back.
4. Do step 6 (least-privilege token) and try to read a path outside the policy.
5. Write a migration-style task that passes the DB password via `environment:` instead of the command line. On web1, while a `sleep 30` version runs, check `ps aux | grep -i pass` shows nothing.
6. Optional: run OpenBao instead (`openbao/openbao` image, `BAO_DEV_ROOT_TOKEN_ID=root`, `bao` CLI) and confirm the same playbook works.

**Done when:** your playbook reads secrets from a secret manager with a least-privilege, short-lived credential, and nothing secret appears in output.

---

## Where you'll use it

- **Enterprise environments**: HashiCorp Vault/OpenBao, CyberArk or cloud secret managers are often mandated.
- **Cloud**: AWS Secrets Manager / SSM, Azure Key Vault, GCP Secret Manager with IAM roles or OIDC from CI.
- **CI/CD**: GitHub Actions/Jenkins authenticate via OIDC, Ansible reads secrets at run time; no long-lived secret stored in CI (topics 53–54).
- **AAP**: credential plugins inject secrets into jobs (topic 56).
- **Dynamic database credentials** for automation tasks.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Failed to import the required Python library (hvac)` / `boto3` | Library missing in Ansible's venv on the control node | `uv pip install hvac` / `boto3` |
| `couldn't resolve ... lookup ... vault_kv2_get` | Collection not installed | `ansible-galaxy collection install community.hashi_vault` |
| `permission denied` / `403` from Vault | Token/role lacks a policy for that path | Fix the policy; check the KV v2 path (`secret/data/...` in policies) |
| `invalid token` / expired | Short-lived token expired mid-run | Longer TTL for the job, or a renewable auth method (AppRole/JWT) |
| `Unable to locate credentials` (AWS) | No credentials in the chain on the control node | Profile, env vars, instance role, or OIDC role in CI |
| Run is slow; secret manager rate-limits | Lookup evaluated per task per host | `set_fact` once with `run_once` |
| Secret printed in logs | Lookup values aren't auto-masked; no `no_log` | `no_log: true`, `\| register_secret`, low verbosity in CI |
| Password visible in `ps` on targets | Passed on a command line | `environment:`, module options, files `0600` |
| Works locally, fails in CI | CI runner can't reach the secret manager (network) | Allow egress/VPN; run the runner inside the network |
| `SSL: CERTIFICATE_VERIFY_FAILED` | Internal CA not trusted on the control node | Add the CA bundle (`ca_cert` option / `REQUESTS_CA_BUNDLE`) |

---

## Quick recap

- **Secret manager** = central service with per-identity access, audit, rotation and dynamic secrets; Ansible reads it through **lookups on the control node**.
- Lookups: `community.hashi_vault.vault_kv2_get`/`vault_read`, `amazon.aws.secretsmanager_secret`, `amazon.aws.ssm_parameter`, Azure/GCP/CyberArk/1Password/Bitwarden equivalents; client libraries (`hvac`, `boto3`) on the control node.
- Authenticate the **control node** with short-lived credentials: AppRole, cloud IAM roles, **OIDC from CI**; avoid long-lived tokens.
- Lookup values aren't auto-masked: use **`no_log`** and **`register_secret`** (2.22). Fetch once with `set_fact` + `run_once`.
- Give secrets to targets via module options, `environment:` or `0600` files, never command lines.

### Interview questions

1. **Q: How do you use HashiCorp Vault (or AWS Secrets Manager) secrets in Ansible?**
   A: With a lookup plugin from the relevant collection (`community.hashi_vault.vault_kv2_get`, `amazon.aws.secretsmanager_secret`) evaluated on the control node, which authenticates with a token, role or OIDC credential. Hide the values with `no_log`.
2. **Q: Ansible Vault vs a secret manager?**
   A: Ansible Vault encrypts secrets in the repo with a shared password; a secret manager stores them centrally with per-identity access control, auditing, rotation and dynamic credentials. Larger teams prefer secret managers.
3. **Q: Why is putting a password in a `command` task risky even with `no_log`?**
   A: `no_log` hides Ansible's output, but the command line is still visible on the target to other users via `ps` and may land in audit logs. Use environment variables, module options or protected files.

### Scenario question (answer out loud first, then compare)

> *"Design secret handling for Ansible running from GitHub Actions against AWS EC2 servers, with no long-lived secrets stored in GitHub."*

**Model answer:**
1. **Identity**: GitHub Actions requests an **OIDC** token; an AWS IAM role trusts GitHub's OIDC provider, restricted to this repo and the `main` branch/production environment. The job assumes it with `aws-actions/configure-aws-credentials` and gets short-lived credentials.
2. **Secrets**: application secrets live in **AWS Secrets Manager**; the role's policy allows `GetSecretValue` only on `prod/shop/*`.
3. **SSH**: no stored private key either: use **EC2 Instance Connect** or **SSM Session Manager** as the connection (`amazon.aws.aws_ssm` connection plugin), or SSH certificates signed per run.
4. **Ansible**: lookups `amazon.aws.secretsmanager_secret` in a `set_fact` with `run_once`, `register_secret` + `no_log`; tasks write secrets into `0600` files or pass them as module options.
5. **Audit**: CloudTrail logs every `GetSecretValue` and role assumption; GitHub environments require approval for prod.

**Likely follow-ups:**
- *Where's the "secret zero"?* There isn't a stored one: trust comes from GitHub's signed OIDC token and the IAM trust policy.
- *How do you rotate?* Secrets Manager rotation (Lambda or managed rotation for RDS); the next Ansible run picks up the new value.
