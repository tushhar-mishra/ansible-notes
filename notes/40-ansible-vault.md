# 40 · Ansible Vault

> **Level:** ★★ intermediate
>
> **You will learn:** how to keep secrets (passwords, tokens, private keys) **encrypted in Git** with **Ansible Vault**: encrypting whole files (`create`, `edit`, `view`, `encrypt`, `decrypt`, `rekey`) and single values (`encrypt_string`), giving Ansible the password (`--ask-vault-pass`, password files, **password scripts**), **vault IDs** for different environments, the **`vars.yml` + `vault.yml`** pattern, encrypted files for `copy`/`template`, **`no_log`** and ansible-core 2.22's automatic **secret masking**, readable `git diff` for vaulted files, and Vault's limits (why big teams add a secret manager, topic 41).
>
> **You need:** [39 · Project layout](39-project-layout.md) (`inventories/<env>/group_vars/all/`), [24 · group_vars](24-group-vars-and-host-vars.md), [21 · Variables](21-variables-basics.md).
>
> **Part 5 · Secrets** · Next: [41 · External secret managers](41-external-secrets.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Secrets must not sit in Git as plain text. **Ansible Vault** encrypts a YAML file (or one value) with a password. The encrypted file **is** committed; Ansible decrypts it in memory when you run a playbook and give it the password.

**Picture it.** A locked document box inside the shared filing cabinet. Everyone can see the box is there and who changed it; only people with the key can read it.

**The one thing to know.**

```bash
ansible-vault create inventories/lab/group_vars/all/vault.yml     # opens your editor; saved encrypted
ansible-vault view   inventories/lab/group_vars/all/vault.yml
ansible-vault edit   inventories/lab/group_vars/all/vault.yml
ansible-playbook -i inventories/lab playbooks/site.yml --ask-vault-pass
```

The pattern:

```yaml
# group_vars/all/main.yml      (plain, readable, greppable)
db_password: "{{ vault_db_password }}"

# group_vars/all/vault.yml     (encrypted)
vault_db_password: "S3cr3t-value"
```

**Try it now:** create a vault file with one secret and run `cat` on it: you'll see `$ANSIBLE_VAULT;1.1;AES256` followed by hex.

**You've got it if you can:** encrypt a secret, use it in a playbook, and keep it out of the output.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### What gets encrypted

| What | How | Result |
|---|---|---|
| A whole **variables file** | `ansible-vault create/encrypt file.yml` | Whole file is ciphertext |
| A **single value** inside a normal YAML file | `ansible-vault encrypt_string` | `!vault \|` block inline |
| **Any file** (certificate, key, binary) used by `copy`/`template`/`unarchive` | `ansible-vault encrypt files/tls.key` | Decrypted automatically when copied |

Encryption: **AES-256** (Advanced Encryption Standard) with a key derived from your password (PBKDF2 with a salt), plus an HMAC (integrity check) so tampering is detected.

File header:

```
$ANSIBLE_VAULT;1.1;AES256                 ← format 1.1, no vault ID
$ANSIBLE_VAULT;1.2;AES256;prod            ← format 1.2, vault ID "prod"
```

### The `ansible-vault` commands

| Command | Does |
|---|---|
| `create FILE` | New encrypted file, opens `$EDITOR` |
| `edit FILE` | Decrypt to a temp file, open editor, re-encrypt on save |
| `view FILE` | Print decrypted content (pager) |
| `encrypt FILE...` | Encrypt existing plain files in place |
| `decrypt FILE...` | Decrypt in place (⚠️ don't commit the result) |
| `rekey FILE...` | Change the password (or vault ID) |
| `encrypt_string --name VAR 'value'` | Print an encrypted variable to paste into YAML |

### Giving Ansible the password

| Method | Example | Use |
|---|---|---|
| Prompt | `--ask-vault-pass` (or `-J`) | Interactive runs |
| Password file | `--vault-password-file ~/.vault_pass_lab` (mode `0600`, outside the repo) | Your laptop |
| Password **script** (executable) | `--vault-password-file ./scripts/vault-pass.sh` → prints the password | Fetch from a password manager / keyring / CI secret |
| Config | `vault_password_file = ~/.vault_pass_lab` or `vault_identity_list = dev@~/.vault_dev, prod@prompt` in `ansible.cfg` | Convenience (not in Git for real passwords) |
| Environment | `ANSIBLE_VAULT_PASSWORD_FILE=/run/secrets/vault_pass` | CI pipelines |

A password script just prints the password on stdout. If its name ends in `-client` (e.g. `vault-client.py`), Ansible passes `--vault-id <label>` to it so one script can serve several IDs.

### Vault IDs: different passwords per environment

```bash
ansible-vault create --vault-id dev@prompt  inventories/dev/group_vars/all/vault.yml
ansible-vault create --vault-id prod@prompt inventories/prod/group_vars/all/vault.yml

ansible-playbook -i inventories/prod playbooks/site.yml --vault-id prod@~/.vault_pass_prod
ansible-playbook -i inventories/dev  playbooks/site.yml --vault-id dev@prompt
```

- `label@source`: label = name of the ID (written into the file header), source = `prompt`, a file, or a script.
- You can pass several `--vault-id` options; Ansible tries the matching label first, then the others.
- Benefit: developers can hold the **dev** password without being able to read **prod** secrets.

### The vars + vault pattern

```
inventories/prod/group_vars/all/
├── main.yml     db_password: "{{ vault_db_password }}"       ← plain: everyone sees WHICH secrets exist
└── vault.yml    vault_db_password: "..."                     ← encrypted
```

Why the indirection with the `vault_` prefix:

- `grep -r db_password` still finds where it's used and defined, without decrypting.
- Reviewers see in `main.yml` diffs that a secret is referenced, even though they can't read `vault.yml`.
- The secret's **name** is in plain text, its **value** isn't.

### encrypt_string: one value inline

```bash
ansible-vault encrypt_string --vault-id lab@prompt --name 'vault_api_token' 'tok_123456'
```
```yaml
vault_api_token: !vault |
          $ANSIBLE_VAULT;1.2;AES256;lab
          62313365396662343061393464336163383764373764613633653634306231386433626436623361
          ...
```

Paste the output into any YAML vars file. Good for one or two secrets in an otherwise plain file; for many secrets, a vaulted file is easier to manage (`encrypt_string` values can't be viewed with `ansible-vault view`; use `ansible localhost -m debug -a var=vault_api_token -e @file.yml --ask-vault-pass`).

### Vaulted files for copy and template

```yaml
- name: TLS private key
  ansible.builtin.copy:
    src: tls/app.key           # files/tls/app.key, encrypted with ansible-vault encrypt
    dest: /etc/pki/tls/private/app.key
    owner: root
    mode: "0600"
  no_log: true
```

`copy` decrypts vaulted source files automatically (`decrypt: true` is the default). The `unvault` lookup reads a vaulted file's content into a variable.

### Keeping secrets out of output: `no_log` and masking

| Tool | Effect |
|---|---|
| `no_log: true` (task/play) | The task's arguments and result are replaced by `"the output has been hidden due to the fact that 'no_log: true' was specified for this result"` in output and logs |
| ansible-core **2.22 secret masking** | Decrypted vault values, `no_log` option values and prompted passwords are registered as **secrets** and replaced with **`$REDACTED$`** anywhere they appear (output, `log_path`, callbacks), even in `debug` |
| `diff: false` | No diff output for that task (diffs show file contents) |
| `loop_control: label` | Don't print whole items (which may contain secrets) |

Use `no_log: true` on every task that **handles** a secret anyway: masking catches known secret values, but `no_log` also hides derived values (a hash, a connection string that contains the password).

### Readable `git diff` for vaulted files

```bash
# .gitattributes (committed)
*vault.yml diff=ansible-vault

# once per developer
git config diff.ansible-vault.textconv "ansible-vault view"
git config diff.ansible-vault.cachetextconv false
```

Now `git diff` shows decrypted changes (for people who have the password).

### Vault's limits

| Limit | Consequence | Answer |
|---|---|---|
| One password per vault ID, shared by everyone who needs it | Rotating it means telling everyone | Vault IDs per env; password in a password manager / CI secret |
| No per-secret access control or audit log | Can't tell who read which secret | A secret manager (HashiCorp Vault, AWS Secrets Manager...) (topic 41) |
| Old ciphertexts stay in **Git history** | If the vault password leaks, **all past** secrets are exposed; `rekey` doesn't help history | Rotate the secrets themselves, not just the vault password |
| Secrets are static values in files | No automatic rotation, no short-lived credentials | Secret manager with dynamic secrets |

---

## Why it exists

Automation needs secrets: database passwords, API tokens, TLS keys, the become password. Putting them in plain text in Git means everyone with repo access (and every CI log, fork and laptop backup) has them forever. Keeping them **out** of Git (on someone's laptop) breaks reproducibility and CI. Vault is the simplest middle ground: secrets **versioned with the code**, but encrypted.

---

## Analogy

**A safe-deposit box in a bank.** The bank (Git) stores the box (vaulted file), tracks when it was moved or replaced, and lets anyone see that box #42 exists. Only key holders (vault password) can open it. If a key is copied, every box ever locked with that key is compromised, including the old boxes in the archive (Git history): you must change what's **inside** (rotate the secrets), not just the lock.

---

## How it works inside

```
 ansible-playbook ... --vault-id prod@~/.vault_pass_prod
   │
   ├─ load vault secrets: label "prod" → password from the file
   ├─ while loading vars files:
   │     file starts with $ANSIBLE_VAULT;1.2;AES256;prod
   │       → derive key from password + salt (PBKDF2) → check HMAC → decrypt AES-256 → YAML
   │       (wrong password → "Decryption failed (no vault secrets were found that could decrypt)")
   ├─ values marked as secrets (2.22+ masking) → $REDACTED$ in any output
   └─ decrypted data exists only in memory on the control node;
      modules receive rendered values (so the TARGET sees the secret in its arguments,
      which is why temp files are private: umask 77, topic 13)
```

---

## Code / commands

### 1. Create the vault for the lab environment

```bash
cd ~/infra
printf 'lab-vault-password-123\n' > ~/.vault_pass_lab && chmod 600 ~/.vault_pass_lab
ansible-vault create --vault-id lab@~/.vault_pass_lab inventories/lab/group_vars/all/vault.yml
```

In the editor, write:

```yaml
---
vault_app_db_password: "Db-Pass-For-Lab-42"
vault_app_api_token: "tok_lab_0123456789"
```

Save and quit. Look at it:

```bash
head -3 inventories/lab/group_vars/all/vault.yml
```
```
$ANSIBLE_VAULT;1.2;AES256;lab
33616536373039316633386261353637656263333937353431653532376436633565636232316537
6438646536303461356461663737373139656162386534640a356463336634663136356237323138
```

Reference the secrets in `inventories/lab/group_vars/all/main.yml`:

```yaml
app_db_password: "{{ vault_app_db_password }}"
app_api_token: "{{ vault_app_api_token }}"
```

Tell `ansible.cfg` where the lab password is (fine for a lab; real passwords stay out of the repo):

```ini
[defaults]
vault_identity_list = lab@~/.vault_pass_lab
```

### 2. Use the secrets

`playbooks/secrets-demo.yml`:

```yaml
---
- name: Use vaulted secrets
  hosts: web
  become: true
  tasks:
    - name: App credentials file
      ansible.builtin.copy:
        dest: /opt/app/credentials.env
        content: |
          DB_PASSWORD={{ app_db_password }}
          API_TOKEN={{ app_api_token }}
        owner: root
        mode: "0600"
      no_log: true

    - name: What does debug show? (masking test)
      ansible.builtin.debug:
        msg: "token is {{ app_api_token }}"
```

```bash
ansible-playbook -i inventories/lab playbooks/secrets-demo.yml --diff
```
```
TASK [App credentials file] ************************************************************
changed: [web1]
changed: [web2]

TASK [What does debug show? (masking test)] ********************************************
ok: [web1] => {
    "msg": "token is $REDACTED$"
}
```

- `no_log: true` hid the task's details (and its diff, even with `--diff`).
- The debug task shows `$REDACTED$`: ansible-core 2.22 masks decrypted vault values automatically. (Older versions would have printed the token: always use `no_log` on secret-handling tasks.)
- On web1, `sudo cat /opt/app/credentials.env` shows the real values: the **target** gets the secret, as intended.

Without the password:

```bash
ANSIBLE_VAULT_IDENTITY_LIST= ansible-playbook -i inventories/lab playbooks/secrets-demo.yml
```
```
ERROR! Attempting to decrypt but no vault secrets found
```

### 3. Inline secret with encrypt_string

```bash
ansible-vault encrypt_string --vault-id lab@~/.vault_pass_lab --name vault_smtp_password 'Mail-Pass-7'
```

Paste the result into `inventories/lab/group_vars/all/main.yml` (or another vars file) and check it:

```bash
ansible -i inventories/lab web1 -m ansible.builtin.debug -a "var=vault_smtp_password"
```
```
web1 | SUCCESS => {
    "vault_smtp_password": "$REDACTED$"
}
```

(To actually *see* a value while debugging: `ansible-vault view` for files, or temporarily `msg="{{ x | length }}"` to check it's set without printing it.)

### 4. A second environment with its own password

```bash
printf 'prod-vault-password-different\n' > ~/.vault_pass_prod && chmod 600 ~/.vault_pass_prod
cp -r inventories/lab inventories/prodlike
ansible-vault rekey --vault-id lab@~/.vault_pass_lab --new-vault-id prod@~/.vault_pass_prod \
  inventories/prodlike/group_vars/all/vault.yml
head -1 inventories/prodlike/group_vars/all/vault.yml          # $ANSIBLE_VAULT;1.2;AES256;prod
```

Now the `lab` password can't open `prodlike`'s vault. Run it with `--vault-id prod@~/.vault_pass_prod`. (Remove `inventories/prodlike` after the lab.)

### 5. A password script (CI style)

`scripts/vault-pass.sh`:

```bash
#!/usr/bin/env bash
# Print the vault password from an environment variable (set by CI from its secret store).
set -euo pipefail
: "${ANSIBLE_VAULT_PASSWORD:?ANSIBLE_VAULT_PASSWORD is not set}"
printf '%s\n' "$ANSIBLE_VAULT_PASSWORD"
```

```bash
chmod +x scripts/vault-pass.sh
ANSIBLE_VAULT_PASSWORD="$(cat ~/.vault_pass_lab)" \
  ansible-playbook -i inventories/lab playbooks/secrets-demo.yml --vault-id lab@scripts/vault-pass.sh
```

The script is safe to commit (it contains no secret); the password comes from the CI's secret store at run time (topics 53–54).

### 6. Readable diffs

```bash
echo '*vault.yml diff=ansible-vault' >> .gitattributes
git config diff.ansible-vault.textconv "ansible-vault view"
ansible-vault edit inventories/lab/group_vars/all/vault.yml     # change a value
git diff inventories/lab/group_vars/all/vault.yml               # shows decrypted lines
```

---

## Hands-on lab

1. Create the lab vault, the `main.yml` references, and run `secrets-demo.yml`. Confirm the masking and the file on web1.
2. Remove `no_log: true` from the copy task and run with `--diff -v`. What's shown? (2.22 masks the known secret values; derived values wouldn't be.) Put `no_log` back.
3. `encrypt_string` a value and use it.
4. Create `prodlike` with a different vault ID via `rekey`; show that the lab password can't decrypt it.
5. Write the password script and run with `ANSIBLE_VAULT_PASSWORD` set; then unset it and read the script's error.
6. Encrypt a file (`echo "fake key" > roles/nginx_web/files/demo.key && ansible-vault encrypt --vault-id lab@~/.vault_pass_lab roles/nginx_web/files/demo.key`), copy it to web1 with `mode: "0600"`, and check it arrived decrypted.
7. Set up `git diff` textconv and see a vault change as plain text.
8. Think: the lab password leaks. What do you do? (Rotate **every secret** in the vault, then `rekey`; history still contains the old ciphertexts, which are now useless because the secrets changed.)

**Done when:** no secret in your repo is readable without the vault password, and none appears in run output.

---

## Where you'll use it

- **group_vars per environment**: DB passwords, API tokens, `ansible_become_password`, SMTP credentials.
- **TLS keys and certificates**, SSH private keys for service accounts, license files.
- **CI/CD**: vault password from the CI secret store via a password script or env var (topics 53–54).
- **AWX/AAP**: "Vault" credential type supplies the password to job templates (topic 56).
- **Small teams**: often all the secret management they need; larger orgs pair it with a secret manager.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Attempting to decrypt but no vault secrets found` | No password given | `--ask-vault-pass`, `--vault-id`, `vault_identity_list` |
| `Decryption failed (no vault secrets were found that could decrypt)` | Wrong password or wrong vault ID | Check the header's label and the password source |
| `input is not vault encrypted data` | `ansible-vault view/decrypt` on a plain file | It wasn't encrypted (or was decrypted and committed!) |
| Secret visible in CI logs | Task without `no_log`, `-vvv`, or a derived value | `no_log: true`, avoid high verbosity in CI, 2.22 masking |
| Vault file committed **decrypted** | `ansible-vault decrypt` then `git add` | Pre-commit hook that rejects unencrypted `*vault.yml`; rotate the leaked secrets |
| `git diff` of vault files is unreadable hex | No textconv | `.gitattributes` + `git config diff.ansible-vault.textconv` |
| Password file ignored / permission warnings | File in the repo or world-readable | Keep it outside the repo, mode `0600` |
| Password script error: `Permission denied` | Not executable | `chmod +x` |
| Everyone needs prod's password | One ID for all environments | Vault IDs per environment, prod only in CI/AAP |

---

## Quick recap

- **Vault** = AES-256 encryption of vars files, single values (`encrypt_string`) or any file; ciphertext is committed to Git.
- Commands: `create`, `edit`, `view`, `encrypt`, `decrypt`, `rekey`, `encrypt_string`. Password via prompt, file (`0600`, outside the repo), **script**, config or env var.
- **Vault IDs** (`label@source`) give each environment its own password.
- Pattern: `main.yml` references `vault_*` variables defined in an encrypted `vault.yml`.
- Hide secrets: `no_log: true`, `diff: false`, `loop_control.label`; 2.22 masks vault values as `$REDACTED$`.
- Limits: shared password, no audit, Git history keeps old ciphertexts → if the password leaks, **rotate the secrets**.

### Interview questions

1. **Q: How do you store secrets in an Ansible repository?**
   A: Encrypt them with Ansible Vault (vaulted vars files or `encrypt_string`), reference them through plain variables, provide the vault password at run time (prompt, file, script, CI secret), and hide them in output with `no_log`. For larger setups, fetch them from a secret manager instead.
2. **Q: What are vault IDs for?**
   A: To use different vault passwords for different sets of secrets (typically per environment), so access to one doesn't grant access to the other.
3. **Q: What does `no_log: true` do?**
   A: It hides a task's arguments and results from output and logs, so secrets handled by that task aren't exposed.

### Scenario question (answer out loud first, then compare)

> *"A developer laptop with the production vault password file was stolen. The repo is on GitHub (private). What do you do, in order?"*

**Model answer:**
1. **Assume every secret ever encrypted with that password is compromised**, including old versions in Git history.
2. **Rotate the secrets themselves**: database passwords, API tokens, keys (in the systems that issue them), prioritising internet-facing and high-privilege ones. Use Ansible to push new values where it manages them.
3. **New vault password**: `ansible-vault rekey` all prod vault files with a new password stored only in the CI secret store / AAP credential, not on laptops.
4. Revoke the laptop's other access (SSH keys, Git tokens, VPN).
5. Check logs for misuse (DB logins, API audit logs) during the exposure window.
6. Prevent: prod vault password only in CI/AAP (developers get the dev ID), password scripts reading from a password manager instead of plain files, and consider a secret manager with per-user access and audit (topic 41).

**Likely follow-ups:**
- *Why isn't rekey enough?* Old ciphertexts in Git history can still be decrypted with the old password.
- *How would a secret manager change this?* Secrets aren't in Git at all; access is per identity and audited; dynamic secrets expire on their own.
