# 54 · Ansible in Jenkins

> **Level:** ★★ intermediate
>
> **You will learn:** how to run Ansible from a **Jenkins declarative pipeline**: plain `sh 'ansible-playbook ...'` vs the **Ansible plugin**'s `ansiblePlaybook` step, running inside your **execution environment** image as a Docker agent, Jenkins **credentials** for the SSH key (`sshagent` / `withCredentials(sshUserPrivateKey)`), the vault password and known_hosts, **parameters** (environment, tags, check-only), a **plan → approve → apply** flow with `input`, preventing concurrent production runs (`lock`, `disableConcurrentBuilds`), archiving plan logs and JUnit results, and multibranch behaviour for PRs. You'll run `~/infra` from the Jenkins you built in your Jenkins notes.
>
> **You need:** [53 · GitHub Actions](53-ansible-in-github-actions.md) (same pipeline ideas), [52 · EE image](52-execution-environments.md), and your Jenkins notes: [14 · Jenkins in Docker](../../Jenkins/notes/14-lab-jenkins-in-docker.md), [23 · credentials model](../../Jenkins/notes/23-credentials-model.md), [24 · using credentials safely](../../Jenkins/notes/24-using-credentials-safely.md), [27 · Docker-out-of-Docker](../../Jenkins/notes/27-lab-dood-setup.md), [31 · declarative pipeline](../../Jenkins/notes/31-declarative-pipeline-anatomy.md), [37 · parameters](../../Jenkins/notes/37-options-parameters-triggers.md), [38 · input/lock](../../Jenkins/notes/38-when-input-milestone-lock.md), [40 · Docker agents](../../Jenkins/notes/40-per-stage-and-docker-agents.md).
>
> **Part 9 · CI/CD** · Next: [55 · ansible-pull and drift detection](55-ansible-pull-and-drift.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Same pipeline as topic 53, written as a **Jenkinsfile**: lint → plan (check/diff) → **human approval** → apply → smoke test. Jenkins **credentials** provide the SSH key and vault password only inside the steps that need them. The job runs in your **EE image**, so Jenkins agents don't need Ansible installed.

**Picture it.** A factory foreman (Jenkins) with a locked key cabinet (credentials store): he hands the key to a worker only for the job that needs it, takes it back afterwards, and won't open the production door until a manager signs the form (`input`).

**The one thing to know.**

```groovy
stage('Apply') {
  steps {
    sshagent(credentials: ['ansible-ssh']) {
      withCredentials([string(credentialsId: 'ansible-vault-lab', variable: 'ANSIBLE_VAULT_PASSWORD')]) {
        sh 'ansible-playbook -i inventories/lab playbooks/site.yml --diff --vault-id lab@scripts/vault-pass.sh'
      }
    }
  }
}
```

**Try it now:** in your Jenkins, create an "SSH Username with private key" credential `ansible-ssh` with `~/.ssh/ansible_key`.

**You've got it if you can:** explain how the SSH key reaches `ansible-playbook` without being written into the workspace.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### `sh` vs the Ansible plugin

| | `sh 'ansible-playbook ...'` | Ansible plugin `ansiblePlaybook(...)` |
|---|---|---|
| Needs | Ansible available on the agent (or EE image) | The **Ansible** Jenkins plugin + an "Ansible installation" configured, or a path |
| Credentials | Wrap with `sshagent` / `withCredentials` | Parameters `credentialsId`, `vaultCredentialsId` |
| Flexibility | Anything the CLI can do; same command as laptops | Covers common options (`inventory`, `extras`, `tags`, `limit`, `colorized`) |
| Recommendation | ✅ Default: transparent, portable, no extra plugin | Fine if your organisation standardises on it |

```groovy
// Plugin form, for reference
ansiblePlaybook(
  playbook: 'playbooks/site.yml',
  inventory: 'inventories/lab',
  credentialsId: 'ansible-ssh',
  vaultCredentialsId: 'ansible-vault-file',
  extras: '--diff',
  colorized: true
)
```

### Where it runs

| Agent | How | Notes |
|---|---|---|
| A static agent with Ansible installed | `agent { label 'ansible' }` | Must keep ansible-core/collections in sync |
| **Docker agent from the EE image** | `agent { docker { image 'infra-ee:1.0'; args '...' } }` | ✅ Pinned toolchain; needs Docker on the agent ([Jenkins 40](../../Jenkins/notes/40-per-stage-and-docker-agents.md)) |
| Kubernetes pod agent | `agent { kubernetes { yaml '...' } }` with the EE image | For Jenkins on Kubernetes ([Jenkins 53–54](../../Jenkins/notes/53-kubernetes-and-cloud-agents.md)) |

The agent must be able to **reach the managed hosts** (network placement matters, as with self-hosted runners).

### Credentials

| What | Jenkins credential type | Used with |
|---|---|---|
| SSH private key for the automation user | **SSH Username with private key** (`ansible-ssh`) | `sshagent(['ansible-ssh'])` (SSH Agent plugin) or `withCredentials([sshUserPrivateKey(credentialsId: 'ansible-ssh', keyFileVariable: 'SSH_KEY_FILE', usernameVariable: 'SSH_USER')])` |
| Vault password | **Secret text** (`ansible-vault-lab`) or **Secret file** | `withCredentials([string(credentialsId: ..., variable: 'ANSIBLE_VAULT_PASSWORD')])` + the password script (topic 40), or `file(...)` + `--vault-password-file $VAULT_FILE` |
| Known hosts | **Secret file** (`ansible-known-hosts`) or a file in the repo you trust | `withCredentials([file(credentialsId: ..., variable: 'KNOWN_HOSTS_FILE')])` + `ANSIBLE_SSH_ARGS` or copy into `~/.ssh` |
| Cloud | IAM role on the agent / OIDC from Jenkins (plugin) / short-lived keys | |

Rules from your [Jenkins topic 24](../../Jenkins/notes/24-using-credentials-safely.md) apply: use **single quotes** in `sh` so Groovy doesn't interpolate secrets into the command string, keep `withCredentials` blocks small, never `echo` secrets.

### Pipeline features you'll use

| Feature | Use |
|---|---|
| `parameters { choice(name: 'ENV', ...), string(name: 'TAGS'), booleanParam(name: 'CHECK_ONLY') }` | Choose environment, tags, dry run |
| `input message: 'Apply to prod?', submitter: 'infra-leads'` | Human approval between plan and apply |
| `lock(resource: "ansible-${params.ENV}")` (Lockable Resources plugin) | No two applies to the same environment at once |
| `options { disableConcurrentBuilds(); timeout(time: 60, unit: 'MINUTES'); ansiColor('xterm') }` | One build at a time per job; a time limit; colours (AnsiColor plugin) |
| `when { branch 'main' }` / `changeRequest()` | Apply only from `main`; plan only for PRs (multibranch) |
| `archiveArtifacts 'plan.log'` | Keep the plan for review/audit |
| `junit 'junit/*.xml'` with the `community.general.junit` callback | Show task failures as test results |
| `post { failure { ... } }` | Notify (Slack/email, [Jenkins topic 71](../../Jenkins/notes/71-notifications.md)) |

---

## Why it exists

Many companies already run Jenkins for builds and deployments; adding Ansible to the same pipelines gives one place for approvals, credentials, logs and audit. The same principles as GitHub Actions apply: no laptops in production, pinned toolchain, credentials released only to the steps that need them, a plan before apply, approval for production, one apply at a time.

---

## Analogy

**A bank vault procedure.** The pipeline is the procedure manual. Credentials are in the vault (Jenkins credentials store); the teller (step) gets the key only inside the vault room (`withCredentials` block) and hands it back when leaving. Large withdrawals (production) need a manager's signature (`input` with `submitter`). Only one teller in the vault room at a time (`lock`).

---

## How it works inside

```
 Jenkins controller
   ├─ reads Jenkinsfile from the repo (multibranch: per branch/PR)
   ├─ schedules the pipeline on an agent; docker agent → starts infra-ee:1.0 container
   ├─ stage Lint     : sh 'ansible-lint'  (inside the EE)
   ├─ stage Plan     : sshagent(ansible-ssh) → SSH_AUTH_SOCK in the container
   │                   withCredentials(vault) → env var, masked in the log as ****
   │                   sh 'ansible-playbook ... --check --diff | tee plan.log'
   ├─ stage Approve  : input (waits; only 'infra-leads' can click Proceed)
   ├─ stage Apply    : lock("ansible-prod") { same credentials; ansible-playbook ... }
   ├─ stage Smoke    : curl / uri playbook
   └─ post           : archive plan.log, junit, notify
```

---

## Code / commands

### Jenkinsfile

`~/infra/Jenkinsfile`:

```groovy
pipeline {
  agent {
    docker {
      image 'infra-ee:1.0'
      args '--network ansible-lab -u root:root'   // reach the lab nodes; root avoids "no user for uid" in ssh
    }
  }

  parameters {
    choice(name: 'ENV', choices: ['lab', 'prod'], description: 'Inventory to use')
    string(name: 'TAGS', defaultValue: 'all', description: 'Ansible tags')
    booleanParam(name: 'CHECK_ONLY', defaultValue: true, description: 'Only run --check --diff')
  }

  options {
    disableConcurrentBuilds()
    timeout(time: 60, unit: 'MINUTES')
    ansiColor('xterm')
    buildDiscarder(logRotator(numToKeepStr: '30'))
  }

  environment {
    ANSIBLE_FORCE_COLOR = '1'
    ANSIBLE_CALLBACKS_ENABLED = 'ansible.posix.profile_tasks,community.general.junit'
    JUNIT_OUTPUT_DIR = 'junit'
  }

  stages {
    stage('Lint') {
      steps {
        sh 'yamllint . && ansible-lint'
        sh 'ansible-playbook -i inventories/${ENV} playbooks/site.yml --syntax-check'
      }
    }

    stage('Plan') {
      steps {
        withCredentials([file(credentialsId: 'ansible-known-hosts', variable: 'KH')]) {
          sh 'mkdir -p ~/.ssh && cp "$KH" ~/.ssh/known_hosts && chmod 600 ~/.ssh/known_hosts'
        }
        sshagent(credentials: ['ansible-ssh']) {
          withCredentials([string(credentialsId: "ansible-vault-${params.ENV}", variable: 'ANSIBLE_VAULT_PASSWORD')]) {
            sh '''
              set -o pipefail
              ansible-playbook -i inventories/${ENV} playbooks/site.yml \
                --check --diff --tags "${TAGS}" \
                --vault-id ${ENV}@scripts/vault-pass.sh | tee plan.log
            '''
          }
        }
      }
      post {
        always { archiveArtifacts artifacts: 'plan.log', allowEmptyArchive: true }
      }
    }

    stage('Approve') {
      when {
        allOf {
          expression { !params.CHECK_ONLY }
          expression { params.ENV == 'prod' }
        }
      }
      steps {
        input message: "Apply to ${params.ENV}? Review plan.log first.", submitter: 'infra-leads'
      }
    }

    stage('Apply') {
      when {
        allOf {
          expression { !params.CHECK_ONLY }
          branch 'main'
        }
      }
      steps {
        lock(resource: "ansible-${params.ENV}") {
          sshagent(credentials: ['ansible-ssh']) {
            withCredentials([string(credentialsId: "ansible-vault-${params.ENV}", variable: 'ANSIBLE_VAULT_PASSWORD')]) {
              sh 'ansible-playbook -i inventories/${ENV} playbooks/site.yml --diff --tags "${TAGS}" --vault-id ${ENV}@scripts/vault-pass.sh'
            }
          }
        }
      }
    }

    stage('Smoke test') {
      when { expression { !params.CHECK_ONLY } }
      steps {
        sh 'curl -fsS --retry 5 --retry-delay 2 http://172.30.0.31/'
      }
    }
  }

  post {
    always {
      junit testResults: 'junit/*.xml', allowEmptyResults: true
    }
    failure {
      echo "Ansible run failed for ${params.ENV}: see plan.log / console"
    }
  }
}
```

Explained:

- **Docker agent** with the EE image; `--network ansible-lab` lets the container reach the lab nodes directly; `-u root:root` avoids SSH refusing to run for a UID that has no `/etc/passwd` entry in the image (a common Docker-agent issue). In production, build the EE with a matching user instead of using root.
- `sh '...'` with **single quotes**: `${ENV}` and `${TAGS}` are expanded by the **shell** from the environment (Jenkins exports parameters as env vars), not by Groovy, so secrets never end up in the Groovy string.
- Known hosts come from a **Secret file** credential.
- `sshagent` starts an agent, adds the key, sets `SSH_AUTH_SOCK` inside the block, and removes it afterwards.
- The vault password is a **Secret text** credential per environment (`ansible-vault-lab`, `ansible-vault-prod`), read by `scripts/vault-pass.sh` (topic 40). Jenkins masks it as `****` if it ever appears in the log.
- `Approve` only for production applies; `submitter` restricts who can approve.
- `Apply` only from `main` (multibranch) and inside a `lock` per environment.
- `community.general.junit` callback writes one XML per playbook run into `junit/`; Jenkins shows failed tasks as failed tests.

### Lab setup in your Jenkins

1. Start Jenkins from your [Jenkins topic 14](../../Jenkins/notes/14-lab-jenkins-in-docker.md) with Docker access ([topic 27](../../Jenkins/notes/27-lab-dood-setup.md)), and plugins: Pipeline, Docker Pipeline, SSH Agent, Credentials Binding, Lockable Resources, AnsiColor, JUnit (most are in the suggested set).
2. Make sure the EE image `infra-ee:1.0` exists on the Docker host Jenkins uses (topic 52).
3. Credentials (Manage Jenkins → Credentials → Global):
   - `ansible-ssh`: SSH Username with private key, username `ansible`, key = contents of `~/.ssh/ansible_key`.
   - `ansible-vault-lab`: Secret text = the lab vault password.
   - `ansible-known-hosts`: Secret file = output of `ssh-keyscan -t ed25519 172.30.0.11 172.30.0.12 172.30.0.21 172.30.0.31` (verified, topic 53).
4. Create a **Multibranch Pipeline** (or Pipeline from SCM) pointing at your `infra` repo ([Jenkins topic 43](../../Jenkins/notes/43-multibranch-pipelines.md)).
5. Build with parameters: `ENV=lab`, `CHECK_ONLY=true` → read `plan.log` in the build's artifacts. Then `CHECK_ONLY=false` on `main`.

Sample console excerpt:

```
[Pipeline] sshagent
[ssh-agent] Using credentials ansible (ansible-ssh)
[ssh-agent] Looking for ssh-agent implementation...
$ ssh-agent
SSH_AUTH_SOCK=/tmp/ssh-XXXXXX/agent.123
Identity added: /var/jenkins_home/workspace/infra_main@tmp/private_key_123.key (ansible-lab)
[Pipeline] withCredentials
Masking supported pattern matches of $ANSIBLE_VAULT_PASSWORD
+ ansible-playbook -i inventories/lab playbooks/site.yml --check --diff --tags all --vault-id lab@scripts/vault-pass.sh
PLAY [Web tier] ****************************************************************
...
PLAY RECAP *********************************************************************
lb1    : ok=6   changed=0 ...
web1   : ok=14  changed=0 ...
[Pipeline] archiveArtifacts
Archiving artifacts
```

### Running in a separate stage container with Molecule (optional)

```groovy
stage('Molecule') {
  agent {
    docker { image 'infra-ee:1.0'; args '-v /var/run/docker.sock:/var/run/docker.sock -u root:root' }
  }
  steps {
    dir('roles/nginx_web') { sh 'molecule test' }
  }
}
```

(Molecule needs Docker access to start its test containers, hence the socket mount: the same "Docker group = root" trade-off as in your Docker notes.)

---

## Hands-on lab

1. Set up the credentials and the job as above; run with `CHECK_ONLY=true`, open `plan.log` from the artifacts.
2. Change a page text in `inventories/lab/host_vars/web1.yml`, push, and run the plan again: the diff shows the change.
3. Run with `CHECK_ONLY=false` on `main` and verify with curl.
4. Add `prod` as an inventory copy of `lab` (for practice), run with `ENV=prod`, `CHECK_ONLY=false`, and approve at the `input` step (as a member of `infra-leads`, or remove `submitter` for the lab).
5. Start two builds at once: `disableConcurrentBuilds` queues the second; then remove it and see `lock` serialising only the Apply stage.
6. Break a task on purpose and look at the JUnit test report in Jenkins.
7. Search the console log for the vault password value: it should appear nowhere (masked or not printed).

**Done when:** Jenkins plans, waits for approval, applies, smoke-tests and archives the plan, with no secret written to the workspace or log.

---

## Where you'll use it

- **Jenkins shops**: deploy pipelines that build an artifact, then run Ansible to deploy it (topic 49's playbook in a Jenkins stage).
- **Parameterised ops jobs**: patching, restarts, certificate renewals with `ENV`/`LIMIT`/`TAGS` parameters and approvals.
- **Shared libraries** ([Jenkins topics 55–58](../../Jenkins/notes/55-shared-libraries-concepts.md)): wrap the Ansible stages in a `runAnsible(env:, playbook:)` step reused by many repos.
- **Migration**: the same playbooks run unchanged in GitHub Actions, Jenkins or AWX.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `No user exists for uid 1000` / `ssh: ... getpwuid` | Docker agent runs as Jenkins' UID, which doesn't exist in the image | Run as root (lab) or build the EE with a matching user / nss_wrapper |
| `Host key verification failed` | known_hosts not installed in the container | Copy the secret file to `~/.ssh/known_hosts` in the same container/stage |
| `Permission denied (publickey)` | `sshagent` not wrapping the `sh`, wrong credential, or key file path forced in inventory | Wrap the step; check the credential; remove `ansible_ssh_private_key_file` from shared inventory |
| `UNREACHABLE` from the Docker agent | Container not on a network that reaches the hosts | `--network` / agent placement |
| Groovy "string interpolation of sensitive variables" warning | Double-quoted `sh "...${SECRET}..."` | Single quotes; let the shell expand env vars |
| `input` step never appears | `when` conditions false (CHECK_ONLY, branch) | Check parameters and branch |
| Two applies at once | No `lock` / `disableConcurrentBuilds` | Add them |
| JUnit report empty | Callback not enabled or wrong dir | `ANSIBLE_CALLBACKS_ENABLED=community.general.junit`, `JUNIT_OUTPUT_DIR` |
| No colours / garbage escape codes | AnsiColor plugin missing | Install it and use `ansiColor('xterm')` |

---

## Quick recap

- Prefer `sh 'ansible-playbook ...'` (portable) over the plugin; run inside the **EE image** as a Docker/Kubernetes agent.
- Credentials: **SSH Username with private key** + `sshagent`, vault password as **Secret text** read by a password script, known_hosts as **Secret file**; single-quoted `sh`.
- Flow: Lint → **Plan** (`--check --diff`, archive `plan.log`) → **Approve** (`input` with `submitter`) → **Apply** (`lock`, main only) → Smoke → JUnit/notifications.
- Parameters for `ENV`, `TAGS`, `CHECK_ONLY`; `disableConcurrentBuilds`, `timeout`, `ansiColor`.
- The agent's network placement decides what it can reach.

### Interview questions

1. **Q: How do you give a Jenkins pipeline the SSH key Ansible needs?**
   A: Store it as an "SSH Username with private key" credential and wrap the `ansible-playbook` step in `sshagent(['id'])` (or `withCredentials(sshUserPrivateKey(...))`), so the key lives in an agent only for that step.
2. **Q: How do you add a manual approval before applying to production?**
   A: A stage with the `input` step (optionally restricted with `submitter`), after a plan stage that runs `--check --diff` and archives the output for review.
3. **Q: How do you prevent concurrent production runs in Jenkins?**
   A: `lock(resource: 'ansible-prod')` from Lockable Resources around the apply, and/or `disableConcurrentBuilds()` on the job.

### Scenario question (answer out loud first, then compare)

> *"Ops wants a Jenkins job 'Patch servers' with parameters for environment and host limit, which patches in batches, needs approval for prod, and posts a summary to Slack. Sketch it."*

**Model answer:**
1. **Parameters**: `ENV` (choice), `LIMIT` (string, default `all`), `BATCH` (string, default `10%`), `REBOOT` (boolean).
2. **Stages**: Lint → Plan (`--check --diff -l "$LIMIT"`, archive) → Approve (only for `prod`, `submitter: 'ops-leads'`) → Apply inside `lock("patch-${ENV}")` running `playbooks/ops/patch.yml -e serial_batch=$BATCH -e allow_reboot=$REBOOT -l "$LIMIT"`.
3. **Playbook**: `serial: "{{ serial_batch }}"`, `max_fail_percentage`, LB drain/enable per host, `dnf`/`apt` upgrades with `async`, reboot when needed, health checks (topics 45–49).
4. **Credentials**: `sshagent` with the per-environment key, vault password secret text.
5. **Post**: JUnit from the junit callback; Slack message with the PLAY RECAP excerpt and links to `plan.log`; on failure, list hosts left drained.

**Likely follow-ups:**
- *How do you stop someone running it against all of prod by accident?* Default `LIMIT` to a canary group, require approval, and add an `assert` in the playbook that refuses `all` in prod unless an explicit `CONFIRM_ALL=true` is passed.
- *Where do the logs go for audits?* Jenkins build history (retained), archived `plan.log`, and Ansible's `log_path` shipped to central logging.
