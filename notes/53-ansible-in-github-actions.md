# 53 · Ansible in GitHub Actions

> **Level:** ★★ intermediate
>
> **You will learn:** how to run Ansible from **GitHub Actions** safely: the pipeline shape (**lint + Molecule + check mode on pull requests**, apply to staging on merge, **approved** apply to production), where the job runs (GitHub-hosted vs **self-hosted runners** inside your network), installing pinned tools with caching (or running in your **execution environment** image), giving the job its **SSH key**, **known_hosts** and **vault password** from secrets, **environments** with required reviewers, **concurrency** so two applies never overlap, **OIDC** for cloud access, and writing a readable summary. You'll run your `~/infra` playbooks from a workflow on a self-hosted runner in WSL2.
>
> **You need:** [52 · Execution environments](52-execution-environments.md), [50 · Linting](50-linting.md), [51 · Molecule](51-testing-with-molecule.md), [40 · Vault password scripts](40-ansible-vault.md), [05 · known_hosts](05-linux-recap-ssh.md), and your GitHub Actions notes: [06 · workflows/jobs/steps](../../GitHub-Actions-Basics-To-Production/notes/06-workflows-jobs-steps.md), [17 · secrets](../../GitHub-Actions-Basics-To-Production/notes/17-secrets-and-variables.md), [28 · environments](../../GitHub-Actions-Basics-To-Production/notes/28-environments.md), [29 · concurrency](../../GitHub-Actions-Basics-To-Production/notes/29-concurrency.md), [36 · OIDC](../../GitHub-Actions-Basics-To-Production/notes/36-oidc-cloud-authentication.md), [38 · self-hosted runners](../../GitHub-Actions-Basics-To-Production/notes/38-self-hosted-runners-deep-dive.md).
>
> **Part 9 · CI/CD** · Next: [54 · Ansible in Jenkins](54-ansible-in-jenkins.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Nobody runs production Ansible from a laptop. A **pipeline** runs it: checks on every pull request, applies after merge, and production only after a human approves. The pipeline gets its credentials (SSH key, vault password, cloud role) from **GitHub secrets / OIDC**, never from a person's machine.

**Picture it.** A factory line: every change goes through inspection stations (lint, tests, dry run) before it reaches the customer (production), and the last gate needs a supervisor's signature (environment approval).

**The one thing to know.**

```
 pull request ─▶ lint ─▶ molecule ─▶ check --diff on staging ─▶ (review)
 merge to main ─▶ apply staging ─▶ smoke test ─▶ [approval] ─▶ apply prod ─▶ smoke test
```

```yaml
- name: Load the SSH key into an agent
  run: |
    eval "$(ssh-agent -s)"
    echo "SSH_AUTH_SOCK=$SSH_AUTH_SOCK" >> "$GITHUB_ENV"
    echo "$SSH_KEY" | ssh-add -
  env:
    SSH_KEY: ${{ secrets.ANSIBLE_SSH_KEY }}
```

**Try it now:** list the secrets your pipeline needs (SSH key, known_hosts, vault password) and where each would come from.

**You've got it if you can:** sketch the workflow's jobs, which events trigger them, and which ones need approval.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### The pipeline

| Event | Job | Purpose | Touches servers? |
|---|---|---|---|
| Pull request | `lint` | yamllint + ansible-lint (production profile) | No |
| Pull request | `molecule` (matrix per role) | Role tests in containers | No (test containers only) |
| Pull request | `plan-staging` | `--check --diff` against staging; summary on the PR | Staging-only credential (check mode is **not** read-only, see below) |
| Push to `main` | `apply-staging` | Apply to staging + smoke test | Yes (staging) |
| After staging, **approval** | `apply-prod` | Apply to production + smoke test | Yes (prod) |
| Schedule (nightly) | `drift` | `--check --diff` on prod; alert if `changed > 0` (topic 55) | Read-only |

### Where the job runs

| Runner | Reaches private servers? | Notes |
|---|---|---|
| **GitHub-hosted** (`ubuntu-24.04`) | Only public endpoints, or through a bastion/VPN you expose | Fine for lint/Molecule; for applies only if hosts/bastion are reachable from the internet (and allow-listed) |
| **Self-hosted** inside your network (VM, Kubernetes ARC) | ✅ Yes | ✅ Usual choice for applies to private infra. Harden it: ephemeral runners, only for this repo, no forks from outside |
| Cloud: GitHub-hosted + **SSM connection** / EC2 Instance Connect | ✅ Via the cloud API, no inbound SSH | Good for AWS with OIDC |

### Secrets the job needs

| Secret | Store as | How the job uses it |
|---|---|---|
| SSH private key for the automation user | Environment secret `ANSIBLE_SSH_KEY` (per environment) | Load into `ssh-agent` |
| Known host keys | Environment **variable** `KNOWN_HOSTS` (public data, but must be trusted) | Write to `~/.ssh/known_hosts` |
| Vault password | Environment secret `ANSIBLE_VAULT_PASSWORD` | Password script reads it (topic 40) |
| Cloud access | **OIDC** role (no stored secret) | `aws-actions/configure-aws-credentials` with `role-to-assume` |
| Become password (if needed) | Secret, passed as an extra var file | Prefer NOPASSWD for the automation user |

Use **environment** secrets (`staging`, `production`) so the production key is only available to jobs that passed the environment's protection rules (required reviewers, branch restriction).

### Installing the toolchain

| Option | How | Pros/cons |
|---|---|---|
| pip/uv in the job | `setup-python` (3.13) + `pip install -r requirements.txt` + `ansible-galaxy collection install -r ...`, with `actions/cache` | Simple; a minute or two per run without cache |
| **EE image as the job container** | `container: { image: registry.example.com/platform/infra-ee:1.0 }` | ✅ Exactly the toolchain you tested (topic 52); needs registry credentials |
| ansible-navigator | `ansible-navigator run ... --eei ...` | Same as above, with artifacts |

### Safety settings

| Setting | Why |
|---|---|
| `concurrency: { group: ansible-prod, cancel-in-progress: false }` | Two applies to the same environment never run at the same time; a new one waits (don't cancel an apply halfway) |
| `permissions:` minimal (`contents: read`, `id-token: write` only where OIDC is used) | Least privilege for `GITHUB_TOKEN` |
| Pin third-party actions by **commit SHA** | Supply-chain safety ([GHA topic 37](../../GitHub-Actions-Basics-To-Production/notes/37-security-hardening.md)). The examples below use readable `@v5` tags for teaching; in a real repo write `uses: actions/checkout@<40-char SHA> # v5.x.y` and let Dependabot update the SHAs. |
| No `-vvv` in CI; `no_log` on secret tasks | Logs are kept and widely readable |
| PRs from forks never get secrets | GitHub's default; don't use `pull_request_target` to work around it |
| `ANSIBLE_FORCE_COLOR=1` | Coloured logs (optional) |
| `ANSIBLE_HOST_KEY_CHECKING` left **True** | Use the trusted `KNOWN_HOSTS` instead |

---

## Why it exists

Running Ansible from laptops means: nobody knows who ran what, credentials live on laptops, different tool versions behave differently, and changes reach production without review. A pipeline gives **review** (PRs with a check-mode plan), **repeatability** (same toolchain), **controlled credentials** (secrets/OIDC scoped to environments), **approval gates**, **serialisation** (concurrency), and an **audit trail** (who approved, which commit, full logs).

---

## Analogy

**Air traffic control for changes.** Pilots (engineers) file a flight plan (PR with a check-mode diff). The tower checks it (lint, tests). Take-off to the test airfield (staging) is automatic; take-off to the main airport (production) needs clearance from the tower supervisor (environment approval). Only one plane uses the runway at a time (concurrency). Every flight is recorded (logs).

---

## How it works inside

```
 GitHub event (pull_request / push / workflow_dispatch / schedule)
   │
   ├─ runner picks the job (hosted or self-hosted)
   ├─ checkout repo
   ├─ toolchain: setup-python 3.13 + cached pip/collections   (or job container = EE image)
   ├─ credentials:
   │     ssh-agent ← secrets.ANSIBLE_SSH_KEY
   │     ~/.ssh/known_hosts ← vars.KNOWN_HOSTS
   │     ANSIBLE_VAULT_PASSWORD ← secrets (read by scripts/vault-pass.sh)
   │     (OIDC → short-lived cloud credentials)
   ├─ ansible-playbook -i inventories/<env> playbooks/site.yml [--check --diff]
   │     exit code 0 → job green; 2/3/4 → job red
   └─ summary → $GITHUB_STEP_SUMMARY (PLAY RECAP, changed hosts)
```

---

## Code / commands

### The workflow

`~/infra/.github/workflows/ansible.yml`:

```yaml
name: ansible

on:
  pull_request:
    paths: ["inventories/**", "playbooks/**", "roles/**", "collections/requirements.yml", "requirements.txt", "ansible.cfg"]
  push:
    branches: [main]
  workflow_dispatch:
  schedule:
    - cron: "17 3 * * *"          # nightly drift check

permissions:
  contents: read

env:
  ANSIBLE_FORCE_COLOR: "1"
  PY_VERSION: "3.13"

jobs:
  lint:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ env.PY_VERSION }}
          cache: pip
      - run: pip install -r requirements.txt
      - name: Collections (cached)
        uses: actions/cache@v4
        with:
          path: collections/ansible_collections
          key: collections-${{ hashFiles('collections/requirements.yml') }}
      - run: ansible-galaxy collection install -r collections/requirements.yml -p ./collections
      - run: yamllint .
      - run: ansible-lint
      - run: ansible-playbook -i inventories/lab playbooks/site.yml --syntax-check

  plan-staging:
    if: github.event_name == 'pull_request'
    needs: lint
    runs-on: [self-hosted, linux, ansible-lab]   # a runner that can reach the servers
    environment: staging-plan                     # read-only credentials
    steps:
      - uses: actions/checkout@v5
      - uses: ./.github/actions/ansible-setup     # composite action below
        with:
          ssh-key: ${{ secrets.ANSIBLE_SSH_KEY }}
          known-hosts: ${{ vars.KNOWN_HOSTS }}
      - name: Check mode with diff
        env:
          ANSIBLE_VAULT_PASSWORD: ${{ secrets.ANSIBLE_VAULT_PASSWORD }}
        run: |
          set -o pipefail
          ansible-playbook -i inventories/lab playbooks/site.yml --check --diff \
            --vault-id lab@scripts/vault-pass.sh | tee plan.log
      - name: Summary
        if: always()
        run: |
          {
            echo '### Ansible plan (staging)'
            echo '```'
            sed -n '/PLAY RECAP/,$p' plan.log
            echo '```'
          } >> "$GITHUB_STEP_SUMMARY"

  apply-staging:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    needs: lint
    runs-on: [self-hosted, linux, ansible-lab]
    environment: staging
    concurrency: { group: ansible-staging, cancel-in-progress: false }
    steps:
      - uses: actions/checkout@v5
      - uses: ./.github/actions/ansible-setup
        with:
          ssh-key: ${{ secrets.ANSIBLE_SSH_KEY }}
          known-hosts: ${{ vars.KNOWN_HOSTS }}
      - name: Apply
        env:
          ANSIBLE_VAULT_PASSWORD: ${{ secrets.ANSIBLE_VAULT_PASSWORD }}
        run: ansible-playbook -i inventories/lab playbooks/site.yml --diff --vault-id lab@scripts/vault-pass.sh
      - name: Smoke test
        run: curl -fsS --retry 5 --retry-delay 2 http://172.30.0.31/

  apply-prod:
    needs: apply-staging
    runs-on: [self-hosted, linux, ansible-prod]
    environment: production                       # required reviewers + only from main
    concurrency: { group: ansible-prod, cancel-in-progress: false }
    steps:
      - uses: actions/checkout@v5
      - uses: ./.github/actions/ansible-setup
        with:
          ssh-key: ${{ secrets.ANSIBLE_SSH_KEY }}
          known-hosts: ${{ vars.KNOWN_HOSTS }}
      - name: Apply to production
        env:
          ANSIBLE_VAULT_PASSWORD: ${{ secrets.ANSIBLE_VAULT_PASSWORD }}
        run: ansible-playbook -i inventories/prod playbooks/site.yml --diff --vault-id prod@scripts/vault-pass.sh
```

Explained:

- `on.pull_request.paths`: only run for Ansible-related changes.
- `lint` runs on a GitHub-hosted runner (no access to servers needed).
- `plan-staging`, `apply-*` run on **self-hosted** runners with labels that identify where they can reach (`ansible-lab`, `ansible-prod`).
- `environment:` ties each job to a GitHub **environment**: its secrets/variables, and its protection rules (production: required reviewers, deployment branch = `main`).
- `concurrency` with `cancel-in-progress: false`: applies queue instead of overlapping or being killed halfway.
- `set -o pipefail` + `tee`: keep the log **and** the real exit code.
- The schedule trigger runs `lint` only as written; topic 55 adds a drift job.

### The composite action that prepares Ansible

`~/infra/.github/actions/ansible-setup/action.yml`:

```yaml
name: ansible-setup
description: Toolchain + SSH agent + known_hosts for Ansible jobs
inputs:
  ssh-key: { required: true, description: Private key for the automation user }
  known-hosts: { required: true, description: Trusted host keys }
runs:
  using: composite
  steps:
    - uses: actions/setup-python@v5
      with: { python-version: "3.13", cache: pip }
    - shell: bash
      run: pip install -r requirements.txt
    - shell: bash
      run: ansible-galaxy collection install -r collections/requirements.yml -p ./collections
    - name: SSH agent and known_hosts
      shell: bash
      env:
        SSH_KEY: ${{ inputs.ssh-key }}
        KNOWN_HOSTS: ${{ inputs.known-hosts }}
      run: |
        mkdir -p ~/.ssh && chmod 700 ~/.ssh
        printf '%s\n' "$KNOWN_HOSTS" > ~/.ssh/known_hosts
        chmod 600 ~/.ssh/known_hosts
        eval "$(ssh-agent -s)"
        echo "SSH_AUTH_SOCK=$SSH_AUTH_SOCK" >> "$GITHUB_ENV"
        echo "SSH_AGENT_PID=$SSH_AGENT_PID" >> "$GITHUB_ENV"
        printf '%s\n' "$SSH_KEY" | ssh-add - >/dev/null
```

- Composite actions avoid repeating these steps in every job ([GHA topic 32](../../GitHub-Actions-Basics-To-Production/notes/32-custom-actions-and-composite.md)).
- The key goes into the **agent** (memory), not to a file on disk.
- Writing `SSH_AUTH_SOCK` to `$GITHUB_ENV` makes later steps use the agent.

For this to work in CI, the shared inventory must not force a key **file** path. Topic 39 put `ansible_ssh_private_key_file: ~/.ssh/ansible_key` in `inventories/lab/group_vars/all/main.yml`: remove that line now. On your laptop, either keep the key in your personal `~/.ssh/config` (topic 09) or load it into your agent (`ssh-add ~/.ssh/ansible_key`). Then laptops and CI both authenticate the same way: through the agent.

### Running in the EE instead

```yaml
  apply-staging:
    runs-on: [self-hosted, linux, ansible-lab]
    container:
      image: registry.example.com/platform/infra-ee:1.0
      credentials:
        username: ${{ secrets.REGISTRY_USER }}
        password: ${{ secrets.REGISTRY_TOKEN }}
      options: --network host
    steps:
      - uses: actions/checkout@v5
      # SSH agent + known_hosts steps as above (no pip/galaxy: they're in the image)
      - run: ansible-playbook -i inventories/lab playbooks/site.yml --diff --vault-id lab@scripts/vault-pass.sh
```

### AWS with OIDC (no stored cloud keys)

```yaml
    permissions:
      contents: read
      id-token: write
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/ansible-prod
          aws-region: eu-west-1
      - run: ansible-playbook -i inventories/prod playbooks/site.yml   # aws_ec2 inventory + secretsmanager lookups use these credentials
```

### Getting KNOWN_HOSTS safely

From a trusted machine (you already verified the fingerprints, topic 05):

```bash
ssh-keyscan -t ed25519 172.30.0.11 172.30.0.12 172.30.0.21 172.30.0.31 > /tmp/kh
ssh-keygen -lf /tmp/kh          # compare with fingerprints from the servers themselves
```

Store the content of `/tmp/kh` as the environment variable `KNOWN_HOSTS`. (In the lab, containers get new host keys on every `reset.sh`, so update it after resets.)

---

## Hands-on lab

1. Push `~/infra` to a **private** GitHub repo. Make sure `.gitignore` excludes installed collections, vault password files and `credentials/`.
2. Add the `lint` job only and get it green on a pull request.
3. Register a **self-hosted runner** in your WSL2 Ubuntu with labels `ansible-lab` ([GHA topic 38](../../GitHub-Actions-Basics-To-Production/notes/38-self-hosted-runners-deep-dive.md)). It can reach `172.30.0.x`.
4. Create environments `staging-plan` and `staging` with secrets `ANSIBLE_SSH_KEY` (content of `~/.ssh/ansible_key`), `ANSIBLE_VAULT_PASSWORD` (the lab vault password) and variable `KNOWN_HOSTS`.
5. Add the composite action and the `plan-staging` job; open a PR changing a page text; read the step summary.
6. Merge and watch `apply-staging` run and the smoke test pass.
7. Create a `production` environment with yourself as **required reviewer** and point `apply-prod` at the lab too (labels `ansible-lab`, inventory `lab`) to practise the approval flow.
8. Push two commits quickly and observe `concurrency` queueing the second apply.

**Done when:** a merged PR changes the lab servers through the pipeline, and production-style applies wait for your approval.

---

## Where you'll use it

- **GitHub-centred teams**: this workflow is the standard shape for infra repos.
- **With Terraform**: a `terraform plan/apply` job before the Ansible jobs (topic 43; Terraform notes topic 57).
- **Role/collection repos**: lint + Molecule matrix on PRs, release on tags (publish to the private hub).
- **Scheduled drift detection** (topic 55).

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Host key verification failed` | `KNOWN_HOSTS` missing/stale (servers rebuilt) | Update the variable from verified fingerprints |
| `Permission denied (publickey)` | Agent not used (key file path forced in inventory), wrong key in the environment | Remove `ansible_ssh_private_key_file` from shared inventory; check the secret |
| `UNREACHABLE ... timed out` on GitHub-hosted runners | Private IPs not reachable from GitHub | Self-hosted runner in the network, bastion, or SSM |
| `Attempting to decrypt but no vault secrets found` | Secret not passed to the step / wrong env name | `env: ANSIBLE_VAULT_PASSWORD: ${{ secrets... }}` on the step |
| Secrets empty in PR jobs | PR from a fork (no secrets by design), or environment not set | Run plans only for internal branches; set `environment:` |
| Two applies overlapped | No `concurrency` | `concurrency` group per environment |
| Job green but the play failed | Output piped without `pipefail` | `set -o pipefail` |
| Slow jobs | Installing tools every run | Cache pip and collections, or use the EE image |
| `ansible.cfg` ignored on the runner | Checkout path world-writable (rare) or wrong working dir | Run from the repo root |

---

## Quick recap

- Pipeline: **PR** → lint + Molecule + **check/diff** plan; **merge** → apply staging + smoke test; **approval** → apply prod; **nightly** drift check.
- Applies to private infra run on **self-hosted runners** (or via SSM/OIDC); lint/tests on GitHub-hosted.
- Credentials: SSH key in an **agent** from environment secrets, **known_hosts** from a trusted variable, vault password via a **password script** reading an env var, cloud via **OIDC**.
- Guardrails: **environments** with required reviewers, **concurrency** without cancelling, minimal `permissions`, pinned actions, no `-vvv`.
- Toolchain: cached pip/collections, or the **EE image** as the job container.

### Interview questions

1. **Q: How do you run Ansible from GitHub Actions against private servers?**
   A: On a self-hosted runner inside the network (or through a bastion/SSM), with the SSH key loaded from an environment secret into ssh-agent, known_hosts from a trusted variable, and the vault password from a secret via a password script.
2. **Q: How do you add an approval step before production?**
   A: Use a GitHub environment (`production`) with required reviewers and branch restrictions on the apply job; its secrets are only released after approval.
3. **Q: How do you prevent two pipeline runs from changing the same environment simultaneously?**
   A: A `concurrency` group per environment with `cancel-in-progress: false`, so runs queue.

### Scenario question (answer out loud first, then compare)

> *"Security asks: 'If someone opens a malicious PR, can it steal the production SSH key or run against production?' Walk through your GitHub Actions design."*

**Model answer:**
1. **Forks** get no secrets; PR workflows use `pull_request` (not `pull_request_target`), so a fork PR can lint but can't access environment secrets or self-hosted runners (restrict self-hosted runners to private repos / specific workflows).
2. **Production secrets** live only in the `production` environment, which requires reviewer approval and allows deployments only from `main`. A PR branch can't use it.
3. **Plans** on PRs use a separate `staging-plan` environment with a **staging-only** key; they never touch prod.
   - ⚠️ A "read-only" key for Ansible plans is largely an illusion: `--check` still logs in over SSH, usually needs `become` (root), and **really executes** every task marked `check_mode: false` (topic 20). A malicious or careless PR can add such a task. So: give PR plans a credential that can only reach staging, audit every `check_mode: false` task in review (`CODEOWNERS`), and keep prod credentials in an environment that only `main` can use.
4. **Self-hosted runners**: ephemeral, dedicated to this repo, labels per environment; prod runner only accepts jobs from the protected environment.
5. **Code review**: CODEOWNERS require infra team review for workflow files and inventories; workflow changes from PRs don't run with prod secrets until merged.
6. Pinned actions by SHA, minimal `GITHUB_TOKEN` permissions, OIDC roles scoped to repo + branch + environment.

**Likely follow-ups:**
- *Could a merged malicious change still reach prod?* Only after review + merge + environment approval; that's what the reviews are for, plus check-mode plans in the PR.
- *Why not store the SSH key as a repo secret?* Repo secrets are available to any workflow job; environment secrets add approval and branch gates.
