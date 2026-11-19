# 59 · Error Catalogue

> **Level:** ★★ intermediate
>
> **You will learn:** the errors you will actually meet with Ansible, grouped by where they come from (SSH and host keys, become/sudo, Python on targets, YAML and parsing, modules and collections, variables and templating, conditionals and loops, inventory and config, Vault, files and permissions, packages and services, SELinux, Windows and network devices, timeouts and async, Galaxy, lint/Molecule/EE), each with **what it means**, **the fix**, and **the topic** that explains it in depth. Use it as a lookup page during incidents.
>
> **You need:** [58 · Debugging Ansible](58-debugging-ansible.md) (the method). Everything else is linked per row.
>
> **Part 12 · Production operations** · Next: [60 · Security best practices](60-security.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Ansible errors look scary but follow patterns. Find the message (or the key phrase) in the tables below, read the cause, apply the fix.

**Picture it.** A car's dashboard manual: "orange oil-can light = low oil pressure: stop and check the level". Each light has one page.

**The one thing to know.** Look at **three** things in every error: the **status** (`UNREACHABLE!` vs `FAILED!` vs `ERROR!` before any play), the **host(s)** it happened on, and the **`msg`** / `stderr` text. Then search this page for a key phrase from `msg`.

**Try it now:** take the last error you saw in this course and find its row.

**You've got it if you can:** given an error, name its category and the first command you'd run to confirm the cause.

> 📖 **That's the core.** Everything below is the **catalogue**.

---

## What it is

A lookup reference. Each table: **message (key phrase)** → **meaning** → **fix** → **topic**.

### 1. Connection, SSH and host keys

| Message | Meaning | Fix | Topic |
|---|---|---|---|
| `UNREACHABLE! ... Failed to connect to the host via ssh: ssh: connect to host X port 22: Connection timed out` | Network path blocked (firewall, security group, wrong IP, host down) | `nc -vz X 22`; security groups; `ansible_host` | 05, 11 |
| `... Connection refused` | Host reachable, sshd not listening on that port | Start sshd; `ansible_port` | 05 |
| `... No route to host` | Routing/ARP problem, or container/VM down | Check the host and the network | 09 |
| `... Could not resolve hostname X` | DNS for the inventory name | Set `ansible_host` to an IP/FQDN; fix DNS | 10 |
| `... Permission denied (publickey)` / `(publickey,password)` | Server rejected all keys offered | Right user (`ansible_user`), right key/agent, `authorized_keys` content and modes | 05, 11 |
| `Too many authentication failures` | Agent offered too many keys | `IdentitiesOnly=yes` + the right key | 05 |
| `Host key verification failed.` | Unknown host key and no TTY to ask (CI) | Pre-load verified `known_hosts`, or `accept-new` for fresh hosts | 05, 53 |
| `WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` | Host key differs from `known_hosts`: rebuilt host **or attack** | Verify the rebuild; `ssh-keygen -R host` | 05 |
| `Load key "...": bad permissions` / `UNPROTECTED PRIVATE KEY FILE!` | Private key file too open (e.g. on `/mnt/c`) | `chmod 600`; keep keys in the Linux home | 05 |
| `Data could not be sent to remote host ... Make sure this host can be reached over ssh` | SSH dropped during transfer (network, MTU, `MaxSessions`) | `-vvvv`; check bastion limits; retry | 48 |
| `Timeout (12s) waiting for privilege escalation prompt` | sudo prompted unexpectedly or hung (shown as unreachable since 2.19) | NOPASSWD or `-K`; remove `requiretty`/lecture; fix slow LDAP | 19 |
| `ControlPath too long` | Socket path exceeds the OS limit | `control_path_dir = /tmp/.ansible-cp` | 48 |
| `mux_client_request_session: session request failed: Session open refused by peer` | sshd `MaxSessions` reached on a shared master | Raise `MaxSessions`, lower forks | 48 |
| `sftp transfer mechanism failed ... trying scp` | SFTP subsystem disabled | Enable `Subsystem sftp` or `transfer_method = smart/scp` | 13 |

### 2. Privilege escalation (become)

| Message | Meaning | Fix | Topic |
|---|---|---|---|
| `Missing sudo password` | sudo needs a password; none given (`-n` failed) | `-K`, vaulted `ansible_become_password`, or NOPASSWD rule | 19 |
| `Incorrect sudo password` | Wrong password | Fix the vaulted value | 19 |
| `X is not in the sudoers file. This incident will be reported.` | No sudo rule for the login user | Add `/etc/sudoers.d/<user>` | 06, 19 |
| `Sorry, user X is not allowed to execute '/bin/sh -c ...'` | Rule restricted to specific commands; Ansible runs `/bin/sh` | Give the automation user a full rule; protect the key instead | 19 |
| `sudo: a terminal is required to read the password` / `sorry, you must have a tty to run sudo` | `requiretty`, or pipelining with a password prompt | Remove `requiretty`; NOPASSWD for automation | 19, 48 |
| `Failed to set permissions on the temporary files Ansible needs to create when becoming an unprivileged user` | `become_user` ≠ root without ACL support | Install `acl`, or enable pipelining | 19 |
| `Missing sudo password` on `[host -> localhost]` | Play-level `become` applied to a localhost task | `become: false` on delegated/local tasks | 19, 46 |
| `su: Authentication failure` | `become_method: su` with the wrong/no root password | Use sudo, or provide the password via vault | 19 |

### 3. Python on the target

| Message | Meaning | Fix | Topic |
|---|---|---|---|
| `/usr/bin/python3: not found` / `rc: 127` / `MODULE FAILURE` | No Python on the target | `raw` bootstrap with `gather_facts: false`; cloud-init | 07 |
| `[WARNING]: No python interpreters found for host` | Discovery found nothing | Same | 07 |
| `[WARNING]: Platform linux on host X is using the discovered Python interpreter at ...` | Informational | Pin `ansible_python_interpreter` or `interpreter_python = auto_silent` | 07 |
| `ansible-core requires a minimum of Python version 3.9` / `SyntaxError` on the target | Target Python too old for this ansible-core | Upgrade the OS, or an older ansible-core/EE for those hosts | 07, 63 |
| `Could not import the dnf python module using /x/python` | Interpreter lacks system bindings | Use the OS `/usr/bin/python3` | 07 |
| `Aborting, target uses selinux but python bindings (libselinux-python) aren't installed!` | Missing SELinux bindings | Install `python3-libselinux` | 07 |
| `MODULE FAILURE` + `module_stdout` with text before the JSON | Shell startup files print to stdout | Make `.bashrc` quiet for non-interactive shells | 13 |
| `Failed to create temporary directory` | Home not writable, disk full, `noexec` | Free space; `remote_tmp` elsewhere | 13 |

### 4. YAML and parsing (fail before any play runs)

| Message | Meaning | Fix | Topic |
|---|---|---|---|
| `Syntax Error while loading YAML. found character '\t'` | Tab in indentation | Spaces only | 04 |
| `mapping values are not allowed in this context` | Bad indentation or unquoted `: ` | Fix alignment; quote | 04 |
| `found unhashable key` / `We could be wrong, but this one looks like it might be an issue with missing quotes` | Value starts with `{{` unquoted | `"{{ var }}"` | 04 |
| `did not find expected key` / `expected <block end>` | Misaligned list item | Align | 04 |
| `'x' is not a valid attribute for a Task` (or Play) | Argument at the wrong level, or a typo in a keyword | Indent under the module; check spelling | 04, 14 |
| `conflicting action statements: X, Y` | Two modules in one task | One module per task; missing `- ` | 04 |
| `no module/action detected in task` | Module missing or mis-indented | Fix | 14 |
| `'include' is not a valid attribute` / removed feature | Bare `include:` (removed 2.16) | `import_tasks`/`include_tasks`/`import_playbook` | 34 |
| `You cannot use loops on 'import_tasks' statements` | Loop on a static import | `include_tasks` | 34 |
| `the role 'x' was not found in ...` | Role path | `roles_path`, run from repo root, FQCN for collection roles | 35, 39 |
| `Could not find or access 'x.yml'` | Imported file missing / wrong relative path | Fix path (relative to the playbook/including file) | 34, 39 |

### 5. Modules and collections

| Message | Meaning | Fix | Topic |
|---|---|---|---|
| `couldn't resolve module/action 'x'` | Typo, short name not in builtin, collection not installed | FQCN; install the collection; check `collections_path` | 16, 37 |
| `Unsupported parameters for (x) module: y` | Option name wrong | `ansible-doc -s x` | 16 |
| `missing required arguments: y` | Required option missing | Add it | 16 |
| `value of x must be one of: a, b, got: c` | Not in `choices` | Allowed value | 16 |
| `parameters are mutually exclusive: a|b` | Both given | Pick one | 16 |
| `This module has been removed` | Tombstoned | Use the replacement in the message | 16 |
| `[DEPRECATION WARNING]: ... will be removed in version X` | Deprecated module/option/feature | Migrate before upgrading | 63 |
| `Failed to import the required Python library (X) on <host>'s Python` | Library missing on the target (or control node for local modules) | Install it where the module runs | 07, 44 |
| `Collection x does not support Ansible version 2.22.0` | `requires_ansible` mismatch | Compatible collection version | 37 |
| `ERROR! Unexpected Exception ... ` | Plugin/module crash | `ANSIBLE_DISPLAY_TRACEBACK=always`, `-vvv`; report a bug | 58 |

### 6. Variables and templating

| Message | Meaning | Fix | Topic |
|---|---|---|---|
| `The task includes an option with an undefined variable. 'x' is undefined` | Not defined for this host | Define it; `default()`; check groups/scope | 21, 25 |
| `'dict object' has no attribute 'y'` | Key missing (typo, skipped task's result, missing facts) | Check with `debug: var=`; `is defined`; `is not skipped` | 21, 26 |
| `'list object' has no element N` | Index out of range | Check length | 21 |
| `template error while templating string: unexpected '{'` | Nested `{{ }}` (2.19+) | Build names with `~` and `lookup('ansible.builtin.vars', ...)` | 22 |
| `No filter named 'x'` / `No test named 'x'` | Collection filter not installed or short name | Install; FQCN (`community.general.json_query`) | 22, 32 |
| `You need to install "jmespath"` / `netaddr` | Library for a filter missing on the **control node** | `uv pip install jmespath netaddr` | 32 |
| `Mandatory variable 'x' not defined.` | Required variable missing (your `mandatory` filter) | Provide it | 22 |
| `AnsibleUndefinedVariable` in a template | Template uses an undefined var for this host | Same as above | 29 |
| Value is a string, expected int/bool | `-e key=value` or INI inventory | JSON `-e`, `| int`, `| bool`, YAML group_vars | 21 |
| A dict lost keys | Variables replace, not merge | `combine`; flat variables | 25, 32 |
| Value "ignored" | A higher-precedence source sets it | Walk the precedence ladder | 25 |

### 7. Conditionals, loops, handlers, blocks

| Message | Meaning | Fix | Topic |
|---|---|---|---|
| `Conditional result (...) was derived from value of type 'str' ... Conditionals must have a boolean result` | Non-boolean `when`/`failed_when`/`until` (2.19+) | `| bool`, `| length > 0`, comparisons, `is` tests | 26 |
| `error while evaluating conditional ... is undefined` | Undefined var in `when` | `is defined and ...`, `default(false)` | 26 |
| `Invalid data passed to 'loop', it requires a list, got this instead: {...}` | Dict or range object | `dict2items`, `| list` | 28 |
| `'item' is undefined` in included tasks | Loop var clash | `loop_control: loop_var` | 28, 34 |
| `The requested handler 'x' was not found` | Handler name mismatch | Exact name or `listen` | 30 |
| Handler didn't run | Notifying task wasn't `changed`, or host failed before handler phase | Expected / `force_handlers`, `flush_handlers` | 30 |
| `async task did not complete within the requested time` | `async` too short | Raise it | 47 |
| `FAILED - RETRYING: ... (N retries left)` then failure | `until` condition never true | Check the registered result's shape | 28 |
| Validation failed in a role: `Validation of arguments failed: ...` | Role `argument_specs` rejected inputs | Fix types/required options | 36 |

### 8. Inventory and configuration

| Message | Meaning | Fix | Topic |
|---|---|---|---|
| `[WARNING]: provided hosts list is empty, only localhost is available` | No inventory parsed | `-i`, `inventory =` in ansible.cfg | 10 |
| `Unable to parse X as an inventory source` | Wrong path or syntax, or plugin not matched (file name suffix) | Check path; `ansible-inventory --graph` for details; suffix like `aws_ec2.yml` | 10, 42 |
| `Could not match supplied host pattern, ignoring: X` | Pattern typo / not in this inventory | `--list-hosts` | 10, 12 |
| `Invalid characters were found in group names` | Dashes/dots in group names | Underscores | 10 |
| `Ansible is being run in a world writable directory ... ignoring it as an ansible.cfg source` | WSL `/mnt/c` or `0777` dir | Move the project to `~`; `ANSIBLE_CONFIG` | 11 |
| `config file = None` | No ansible.cfg found where you are | Run from the repo root | 11, 39 |
| `Unable to locate credentials` (aws_ec2) | No AWS credentials on the control node | Profile/role/OIDC | 42 |

### 9. Vault and secrets

| Message | Meaning | Fix | Topic |
|---|---|---|---|
| `Attempting to decrypt but no vault secrets found` | No vault password provided | `--ask-vault-pass`, `--vault-id`, `vault_identity_list` | 40 |
| `Decryption failed (no vault secrets were found that could decrypt)` | Wrong password or vault ID | Check label and password source | 40 |
| `input is not vault encrypted data` | Plain file passed to `view`/`decrypt` | It isn't encrypted (or was committed decrypted!) | 40 |
| `$REDACTED$` where you expected a value | 2.22 secret masking | Expected for secrets; check with `| length` instead of printing | 40, 41 |
| Vault/secret-manager `403 permission denied` | Policy doesn't allow the path | Fix policy / KV v2 path | 41 |

### 10. Files, permissions, packages, services, SELinux

| Message | Meaning | Fix | Topic |
|---|---|---|---|
| `Could not find or access 'x' ... Searched in: ...` (copy/template src) | Source not in `files/`/`templates/` | Move it; read the search paths | 18, 29 |
| `failed to validate` | Your `validate` command rejected the new file (good!) | Fix content | 18, 29 |
| `Destination ... not writable` / `Permission denied` | No `become`, or path owned by another user | `become: true` / right owner | 18, 19 |
| `Could not replace file ... Device or resource busy` | Bind-mounted file (containers) | `unsafe_writes: true` (containers only) | 18 |
| `No package matching 'x' is available` (apt) / `No package x available` (dnf) | Name differs per distro, stale cache, repo missing | Per-family names; `update_cache`; add repo | 17 |
| `Could not get lock /var/lib/dpkg/lock-frontend` | Another apt process (unattended-upgrades) | `lock_timeout`, retry with `until` | 17 |
| `This command has to be run under the root user` | Package op without become | `become: true` | 12 |
| `Could not find the requested service X: host` | Service name differs (sshd/ssh, crond/cron) | Per-family variables | 06, 24 |
| `Unable to start service X: Job for X.service failed` | The service itself fails (bad config) | `journalctl -u X` on the host | 17 |
| `AVC denial` / app gets 403 after copying files (RHEL) | Wrong SELinux context | `setype`, `restorecon`, `sefcontext`; [Linux 073](../../linux-mastery-rhel/notes/073-selinux-file-contexts.md) | 06, 18 |

### 11. Windows, network devices, cloud, Kubernetes

| Message | Meaning | Fix | Topic |
|---|---|---|---|
| `winrm or requests is not installed` / `pypsrp` missing | Library missing on the control node | Install it | 44 |
| `the specified credentials were rejected by the server` / `401` (WinRM) | Auth method/user format | Kerberos `user@REALM`, right transport | 44 |
| `ansible_network_os must be specified` | Network connection without platform | Set it | 44 |
| `unable to open shell` | Wrong network OS / prompt not recognised | Fix `ansible_network_os`; `-vvvv` | 44 |
| `Failed to import the required Python library (kubernetes)` / `boto3` | Library missing on the control node | Install it in Ansible's venv / EE | 44, 52 |
| `Unauthorized` (Kubernetes) | Wrong kubeconfig/context | `kubeconfig`, `context` options | 44 |

### 12. Galaxy, lint, Molecule, execution environments

| Message | Meaning | Fix | Topic |
|---|---|---|---|
| `Failed to resolve the requested dependencies map` | Conflicting collection version ranges | Align pins | 37 |
| `Galaxy server ... does not support the v3 API` | Old server (2.20+ requires v3) | Upgrade server | 37 |
| `HTTP Error 401` from Automation Hub | Token missing/expired | Refresh token, env var | 37 |
| `fqcn[action-core]`, `no-changed-when`, `risky-file-permissions` ... (ansible-lint) | Best-practice violations | Topic 50's rule table | 50 |
| `Idempotence test failed because of the following tasks` (Molecule) | Second converge changed something | Fix the task | 27, 51 |
| `Failed to find driver docker` (Molecule) | Driver plugin missing | `molecule-plugins[docker]` | 51 |
| `System has not been booted with systemd` | Container without systemd as PID 1 | Topic 09 settings | 09, 51 |
| `UNREACHABLE` only inside the EE / navigator | Container network can't reach targets | `--net=host` (labs) / proper networking | 52 |
| `No user exists for uid` (Jenkins Docker agent) | Agent UID missing in the image | Run as root (lab) or add the user to the EE | 54 |

---

## Why it exists

During an incident you don't want to re-learn Ansible; you want the matching line. Grouping errors by layer also reinforces the debugging method (topic 58): the category tells you which tool to reach for next.

---

## Analogy

**A phrasebook for a foreign country.** You don't read it cover to cover; when someone says a sentence you don't understand, you look it up, and next to it is what it means and what to answer.

---

## How it works inside

The categories map onto the execution path you learned:

```
 parse (4) ─▶ inventory/config (8) ─▶ variables/templating (6, 7) ─▶ connection (1) ─▶ become (2)
   ─▶ Python on target (3) ─▶ module (5, 10, 11) ─▶ results/handlers/async (7) ─▶ tooling (12)
```

An error in an earlier stage hides everything after it; fix in order.

---

## Code / commands

Commands that confirm the most common causes quickly:

```bash
nc -vz HOST 22                                         # network/port
ssh -v -i KEY USER@HOST true                           # SSH auth outside Ansible
ansible HOST -m ansible.builtin.ping -vvvv             # Ansible's own SSH path
ansible HOST -m ansible.builtin.raw -a 'python3 -V'    # Python present?
ansible HOST -b -m ansible.builtin.command -a whoami   # become works?
ansible-inventory --host HOST                          # what variables it gets
ansible-config dump --only-changed                     # which config is active
ansible-playbook PB --syntax-check                     # parse errors
ansible-doc FQCN -s                                    # correct options
ansible-galaxy collection list                         # which collection versions, which paths
```

---

## Hands-on lab

1. Reproduce **one error from each section 1–7** in the lab (most are one-line changes: wrong user, removed sudo rule, `bare1` without Python, a tab, a short module name of a missing collection, an undefined variable, a string conditional). For each, find its row here and apply the fix.
2. Add the three errors you hit most often in your own work to a team runbook, with the confirm-command from the Code section.

**Done when:** you can map any error you see to its row and its confirming command.

---

## Where you'll use it

- **On-call** when a pipeline or AWX job fails.
- **Reviews** of failed runs in CI.
- **Training** new team members.

---

## When it breaks

If an error isn't here: classify it by stage (diagram above), run the confirm commands, use `-vvv` and the debugger (topic 58), and add the new row to your team's copy of this catalogue.

---

## Quick recap

- Read **status**, **hosts** and **`msg`/`stderr`**; find the key phrase here.
- Categories follow the execution path: parse → inventory/config → variables → connection → become → Python → module → results → tooling.
- Fix the earliest stage first; confirm with the one-line commands; add a guard (assert/lint/test) afterwards.

### Interview questions

1. **Q: What's the difference between `UNREACHABLE` and `FAILED`?**
   A: UNREACHABLE: Ansible couldn't connect/authenticate/escalate, so the module didn't run. FAILED: the module ran and reported failure (or a template/data error for that task).
2. **Q: A task fails with "Missing sudo password". What are your options?**
   A: Provide the become password (`-K` or a vaulted `ansible_become_password`), or configure a NOPASSWD sudo rule for the automation user.
3. **Q: What does "Conditionals must have a boolean result" mean?**
   A: Since ansible-core 2.19, `when`/`failed_when`/`until` expressions must evaluate to true/false; convert strings/lists with `| bool`, `| length > 0` or explicit comparisons.

### Scenario question (answer out loud first, then compare)

> *"After upgrading CI from ansible-core 2.16 to 2.22, 40 playbooks fail with a mix of errors: 'Conditionals must have a boolean result', 'template error ... unexpected {', 'No filter named ipaddr', and some RHEL 8 hosts 'MODULE FAILURE'. Plan the clean-up."*

**Model answer:**
1. **Categorise**: conditionals and nested templates (2.19 templating changes, topics 22/26); `ipaddr` (filter moved to `ansible.utils` + needs `netaddr`, topic 32); RHEL 8 (target Python 3.6 unsupported since 2.17, topic 07).
2. **Short term**: pin CI back only for the RHEL 8 inventory (separate EE/toolchain), keep 2.22 for the rest.
3. **Fix code**: run `ansible-lint` (it flags many of these), grep for `{{.*{{` and bare `when:` variables; convert to boolean expressions and `lookup('ansible.builtin.vars', ...)`; switch to `ansible.utils.ipaddr`, add `ansible.utils` + `netaddr` to requirements/EE.
4. Temporary `ALLOW_BROKEN_CONDITIONALS` only to unblock while fixing, then remove.
5. **Prevent**: test upgrades in a branch with Molecule and check mode against staging before switching CI (topic 63).

**Likely follow-ups:**
- *How do you find all affected playbooks quickly?* Run `ansible-playbook --syntax-check` and `ansible-lint` across the repo in CI with the new version; check-mode runs against staging catch runtime errors.
- *What about RHEL 8 long term?* Upgrade those hosts; the old toolchain is a temporary bridge.
