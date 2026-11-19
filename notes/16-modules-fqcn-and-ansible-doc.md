# 16 · Modules, FQCNs and ansible-doc

> **Level:** ★ foundation
>
> **You will learn:** what a module is from the user's side, how **FQCNs** (Fully Qualified Collection Names: `namespace.collection.module`) work, what's in **`ansible.builtin`**, why short names like `copy:` are ⚠️ discouraged, how Ansible resolves and **redirects** names, how to read a module's documentation offline with **`ansible-doc`** (options, required fields, choices, defaults, **attributes** such as check-mode support, examples, **return values**), and how to choose between several modules that seem to do the same job.
>
> **You need:** [15 · Inside ansible-playbook](15-inside-ansible-playbook.md), [06 · Linux recap II](06-linux-recap-admin-tasks.md).
>
> **Part 2 · Playbooks** · Next: [17 · Essential modules I](17-modules-packages-services-users.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Modules live in **collections**. A module's full name says which collection it comes from: `ansible.builtin.copy` = namespace `ansible`, collection `builtin`, module `copy`. Always write the full name. Read any module's manual offline with `ansible-doc`.

**Picture it.** An FQCN is like a full postal address: "copy" alone is "John"; `ansible.builtin.copy` is "John Smith, 12 Oak Street, Leeds". There may be several Johns.

**The one thing to know.**

```bash
ansible-doc ansible.builtin.user          # the full manual: options, examples, return values
ansible-doc -s ansible.builtin.user       # a short task snippet with every option
ansible-doc -l ansible.builtin            # list modules in a collection
ansible-doc -l | grep -i firewall         # search all installed modules
```

**Try it now:** run `ansible-doc -s ansible.builtin.copy` and find the option that makes a backup before overwriting a file.

**You've got it if you can:** look up any module, find its required options, and tell whether it supports check mode.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### A module, from your side

A module is a **unit of work with a contract**:

- **Inputs**: named **options** (arguments), each with a type, maybe a default, maybe a fixed list of allowed values (**choices**), maybe required.
- **Behaviour**: it checks the current state and changes only what's needed (idempotent), and says whether it can run in **check mode** (dry run).
- **Outputs**: a JSON result with `changed`, `failed`, and documented **return values**.

### FQCN anatomy

```
 community.general.timezone
 └───┬───┘ └──┬──┘ └──┬───┘
 namespace collection module
```

| Part | Who owns it | Examples |
|---|---|---|
| **Namespace** | An organisation or person on Galaxy | `ansible`, `community`, `amazon`, `kubernetes`, `cisco`, `redhat` |
| **Collection** | A package of content in that namespace | `builtin`, `posix`, `general`, `aws`, `core`, `ios` |
| **Module** | One module in it | `copy`, `firewalld`, `timezone`, `ec2_instance` |

### What's in ansible.builtin

The ~70 modules that ship **inside ansible-core**, always available. The ones you'll use most:

| Area | Modules |
|---|---|
| Commands | `command`, `shell`, `raw`, `script`, `expect` |
| Files | `file`, `copy`, `template`, `lineinfile`, `blockinfile`, `replace`, `stat`, `fetch`, `slurp`, `find`, `get_url`, `unarchive`, `tempfile`, `assemble` |
| Packages | `package`, `dnf`, `dnf5`, `apt`, `pip`, `rpm_key`, `yum_repository`, `deb822_repository`, `apt_repository`, `package_facts` |
| Services/system | `systemd_service`, `service`, `service_facts`, `user`, `group`, `cron`, `hostname`, `reboot`, `getent`, `known_hosts`, `mount_facts` |
| Network/web | `uri`, `get_url`, `wait_for`, `wait_for_connection` |
| Control flow / logic (run on the control node) | `debug`, `set_fact`, `assert`, `fail`, `pause`, `meta`, `include_tasks`, `import_tasks`, `include_role`, `import_role`, `include_vars`, `add_host`, `group_by`, `set_stats` |
| Facts | `setup`, `gather_facts` |
| Version control | `git` |

Everything else (firewalld, SELinux booleans, Docker, AWS, Kubernetes, PostgreSQL, Windows) is in **other collections** you install (topic 37).

### ⚠️ Short names

These all still work:

```yaml
- copy:                    # short name
    src: a
    dest: /b
```

But they're **discouraged**:

| Problem | Example |
|---|---|
| **Ambiguous** | Two installed collections could both have a module called `docker_container` or `user` |
| **Surprising resolution** | A `collections:` keyword at play level, or a role's collection list, changes which one you get |
| **Harder to read** | Is `firewalld` from `ansible.posix` or `community.general`? |
| **Linters complain** | `ansible-lint` rule `fqcn[action-core]` / `fqcn[action]` |

How a short name is resolved: Ansible looks in the collections listed by the play/role's `collections:` keyword (if any), then **`ansible.builtin`**, then the legacy library paths. A short name that isn't in those fails with `couldn't resolve module/action`.

**Rule: always write the FQCN.** These notes do.

### Redirects and renamed modules

Collections can **redirect** an old name to a new one (in their `meta/runtime.yml`). Examples you'll meet:

| You write | You actually get | Why |
|---|---|---|
| `ansible.builtin.yum` | `ansible.builtin.dnf` | yum removed (ansible-core 2.17) |
| `ansible.builtin.systemd` | `ansible.builtin.systemd_service` | Renamed; old name kept as alias |
| `community.general.docker_container` | `community.docker.docker_container` | Docker modules moved collections |

Redirects may print a deprecation warning. When they do, switch to the new name.

### ansible-doc

The offline manual for every installed module and plugin. Its pages have the same sections every time:

| Section | Tells you |
|---|---|
| Header | FQCN + the file path it was loaded from (which collection, which version) |
| Description | What it does, platform notes |
| **OPTIONS** | Every option: description, `type`, `default`, `choices`, `aliases`, whether it is **required** (printed in red; `-s` marks it `(required)`; `-j` shows `"required": true`) |
| **ATTRIBUTES** | `check_mode` support (full / partial / none), `diff_mode`, `platform` (posix, windows), `action` group |
| NOTES / SEE ALSO | Caveats, related modules |
| **EXAMPLES** | Copy-pasteable tasks |
| **RETURN VALUES** | Keys in the result JSON and when they're returned |

`ansible-doc` works for other plugin types with `-t`: `ansible-doc -t connection ssh`, `-t inventory amazon.aws.aws_ec2`, `-t lookup file`, `-t filter ansible.builtin.combine`, `-t callback ansible.posix.profile_tasks`.

---

## Why it exists

- **FQCNs** exist because Ansible content is now split into hundreds of independently released collections. Full names make it unambiguous which code runs, and let two collections use the same short name safely.
- **ansible-doc** exists because the documentation must match the **installed** version. The website shows the latest; your venv might have an older collection with different options. `ansible-doc` reads the docs embedded in the exact module file Ansible will run.

---

## Analogy

**A big hardware shop.**

- **Collections** are the shop's departments ("Plumbing", "Electrical", "Garden"). `ansible.builtin` is the **front counter** with the basics everyone needs.
- **FQCN** is the department + aisle + item code. Saying "the 10 mm one" (short name) works if there's only one; with five departments selling a "10 mm one", you get whichever the clerk finds first.
- **ansible-doc** is the **spec sheet on the shelf** next to the item you'll actually take home, not a catalogue from next year.
- **ATTRIBUTES** are the little icons on the box: "safe for dry run", "works outdoors (posix only)".

---

## How it works inside

What Ansible does with `ansible.posix.firewalld:` in a task:

```
 task: ansible.posix.firewalld: {...}
   │
   ├─ 1. split FQCN → namespace "ansible", collection "posix", module "firewalld"
   ├─ 2. search collection paths (ansible --version "ansible collection location"):
   │       ~/.ansible/collections/ansible_collections/ansible/posix/
   │       /usr/share/ansible/collections/ansible_collections/ansible/posix/
   │       (also ./collections/ next to the playbook)
   ├─ 3. read meta/runtime.yml in that collection → any redirect / deprecation / tombstone?
   │       redirect → repeat with the new name
   │       tombstone (removed) → error "This module has been removed"
   ├─ 4. found plugins/modules/firewalld.py → check for an action plugin with the same name
   └─ 5. build AnsiballZ from firewalld.py + its module_utils (topic 13)
```

`ansible-doc` uses the **same** search, so `ansible-doc ansible.posix.firewalld` shows the docs of exactly the file in step 4.

---

## Code / commands

### The full manual

```bash
ansible-doc ansible.builtin.user
```
```
> MODULE ansible.builtin.user (/home/you/.venvs/ansible/lib/python3.13/site-packages/ansible/modules/user.py)

  Manage user accounts and user attributes. For Windows targets, use the ansible.windows.win_user module instead.

OPTIONS (red indicates it is required):

- append    If `true', add the user to the groups specified in `groups'.
            If `false', user will only be added to the groups specified in `groups', removing them from all other groups.
        default: false
        type: bool

- create_home   Unless set to `false', a home directory will be made for the user when the account is created or if the home directory does not exist.
        aliases: [createhome]
        default: true
        type: bool
...
- name      Name of the user to create, remove or modify.
        aliases: [user]
        type: str
...
- state     Whether the account should exist or not, taking action if the state is different from what is stated.
        choices: [absent, present]
        default: present
        type: str

ATTRIBUTES:

        check_mode:
          description: Can run in check_mode and return changed status prediction without modifying target...
          support: full
        diff_mode:
          description: Will return details on what has changed (or possibly needs changing in check_mode), when in diff mode
          support: none
        platform:
          description: Target OS/families that can be operated against
          platforms: posix

EXAMPLES:

- name: Add the user 'johnd' with a specific uid and a primary group of 'admin'
  ansible.builtin.user:
    name: johnd
    comment: John Doe
    uid: 1040
    group: admin
...

RETURN VALUES:
- append    Whether or not to append the user to groups.
        returned: When state is `present' and the user exists
        type: bool
...
```

How to read it:

- The header path shows the module came from **ansible-core** in your venv.
- `name` is printed in **red** in a colour terminal: it is **required**. (In `-s` output it says `(required)`.) Everything else is optional.
- `aliases: [user]`: you may write `user:` instead of `name:`. (Prefer the main name.)
- `choices: [absent, present]` + `default: present`: only these values are allowed; if you omit `state`, it's `present`.
- `ATTRIBUTES → check_mode: support: full`: safe and accurate in `--check` runs. `diff_mode: none`: `--diff` shows nothing for this module.
- `RETURN VALUES`: what you can read after `register` (topic 26).

Press `q` to leave the pager. Add `| cat` or `-j` (JSON) to print without a pager.

### The snippet: every option in task form

```bash
ansible-doc -s ansible.builtin.copy
```
```
- name: Copy files to remote locations
  ansible.builtin.copy:
      attributes:            # The attributes the resulting filesystem object should have...
      backup:                # Create a backup file including the timestamp information so you can get the original file back...
      checksum:              # SHA1 checksum of the file being transferred...
      content:               # When used instead of `src', sets the contents of a file directly to the specified value...
      decrypt:               # This option controls the autodecryption of source files using vault.
      dest:                  # (required) Remote absolute path where the file should be copied to...
      directory_mode:        # Set the access permissions of newly created directories...
      follow:                # This flag indicates that filesystem links in the destination, if they exist, should be followed.
      force:                 # Influence whether the remote file must always be replaced...
      group:                 # Name of the group that should own the filesystem object...
      mode:                  # The permissions of the destination file or directory...
      owner:                 # Name of the user that should own the filesystem object...
      remote_src:            # Influence whether `src' needs to be transferred or already is present remotely...
      setype:                # The type part of the SELinux filesystem object context...
      src:                   # Local path to a file to copy to the remote server...
      validate:              # The validation command to run before copying the updated file into the final destination...
```

- One line per option with the start of its description. `(required)` is marked. Perfect for writing a new task: paste, delete what you don't need.

### Find modules

```bash
ansible-doc -l ansible.builtin | wc -l          # how many builtin modules (about 70)
ansible-doc -l | grep -iE 'firewall|ufw'        # across all installed collections
```
```
ansible.posix.firewalld        Manage arbitrary ports/services with firewalld
ansible.posix.firewalld_info   Gather information about firewalld
community.general.ufw          Manage firewall with UFW
```

- Only **installed** collections are searched. If you only have ansible-core, you'll see only `ansible.builtin` (and anything you added).

### Check an attribute quickly (JSON + a filter)

```bash
ansible-doc -j ansible.builtin.command | python3 -c \
 'import json,sys; d=json.load(sys.stdin)["ansible.builtin.command"]["doc"]["attributes"]; print({k:v.get("support") for k,v in d.items() if isinstance(v,dict)})'
```
```
{'check_mode': 'partial', 'diff_mode': 'none', 'platform': None, 'raw': 'full'}
```

- `command` has only **partial** check-mode support: in `--check` it's **skipped** unless you use `creates`/`removes` (topics 20, 27).

### Other plugin types

```bash
ansible-doc -t connection -l                         # all connection plugins
ansible-doc -t callback ansible.posix.profile_tasks
ansible-doc -t lookup ansible.builtin.file
ansible-doc -t keyword -l | head                     # playbook keywords (hosts, become, when, ...)
ansible-doc -t keyword become
```

- `-t keyword` documents **playbook keywords** themselves: where `become`, `when`, `serial` are allowed (play, block, role, task).

### Choosing between modules

When several modules seem to fit, prefer, in order:

1. **`ansible.builtin`** if it does the job.
2. A **specific** module over a generic one when you need specific features: `ansible.builtin.dnf` (has `enablerepo`, `security`) vs `ansible.builtin.package` (portable, basic options only).
3. **Red Hat Certified / vendor-maintained** collections (`amazon.aws`, `kubernetes.core`, `ansible.posix`) over community ones for production.
4. `community.*` modules: fine and widely used, but check their maintenance status and pin versions.
5. Only then `command`/`shell` (with idempotency guards, topic 27).

---

## Hands-on lab

1. Read `ansible-doc ansible.builtin.file`. Answer:
   - Which `state` values exist? (`absent`, `directory`, `file`, `hard`, `link`, `touch`)
   - What does `recurse` do, and with which `state` is it allowed?
   - Does it support check mode and diff mode?
2. `ansible-doc -s ansible.builtin.lineinfile`: find the options for "insert after a matching line" and "make a backup".
3. Find the module to manage SELinux booleans: `ansible-doc -l | grep -i seboolean`. Which collection is it in? Is it installed?
4. Prove resolution: in a scratch playbook, use the short name `copy:` and run with `-vvv 2>&1 | grep 'Using module file'`. Then use `ansible.builtin.copy`. Same file.
5. Use a removed name and read the result:

   ```yaml
   - hosts: web1
     gather_facts: false
     tasks:
       - ansible.builtin.yum:
           name: tree
           state: present
         become: true
   ```

   Run with `-vvv 2>&1 | grep 'Using module file'`. You'll see `dnf.py`: the redirect at work.

6. List every playbook keyword usable on a task: `ansible-doc -t keyword -l` and pick five you haven't met yet. You'll meet them all by topic 34.

**Done when:** you can write a correct task for a module you've never used, using only `ansible-doc`.

---

## Where you'll use it

- **Writing every task**: `ansible-doc -s` is faster than any web search and matches your installed version.
- **Reviewing code**: spotting short names, deprecated modules and options.
- **Upgrades**: after updating a collection, `ansible-doc` shows new/removed options.
- **Check mode planning**: `ATTRIBUTES` tells you which tasks will be accurate in `--check` (topic 20).
- **Air-gapped environments**: no internet, but `ansible-doc` still works.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `couldn't resolve module/action 'community.general.ufw'` | Collection not installed in any collection path | `ansible-galaxy collection install community.general`; check `ansible --version` collection location |
| `couldn't resolve module/action 'firewalld'` | Short name not in `ansible.builtin` | Use the FQCN `ansible.posix.firewalld` |
| `Unsupported parameters for (ansible.builtin.copy) module: dst` | Option name typo | `ansible-doc -s ansible.builtin.copy` |
| `value of state must be one of: absent, present, got: installed` | Not in `choices` | Use an allowed value (for `user`, `present`) |
| `missing required arguments: dest` | Required option omitted | Look for `(required)` in `ansible-doc -s` |
| `[DEPRECATION WARNING]: ... has been deprecated ... will be removed in version X` | Using an old module/option/redirect | Switch to the replacement named in the warning |
| `This module has been removed` | Tombstoned name | Use the replacement in its message |
| `ansible-doc` shows options your playbook says don't exist | Docs and run use **different** installs (two venvs, two collection paths) | Compare the header path in `ansible-doc` with `Using module file` in `-vvv` |

---

## Quick recap

- FQCN = `namespace.collection.module`. Always use it. `ansible.builtin` = the ~70 modules inside ansible-core.
- Short names resolve via `collections:` then `ansible.builtin`, and are ⚠️ discouraged (ambiguity, lint errors).
- Collections can **redirect** old names (`yum` → `dnf`, `systemd` → `systemd_service`) and **tombstone** removed ones.
- `ansible-doc MODULE` (full manual), `-s` (snippet), `-l` (list), `-t TYPE` (other plugins/keywords), `-j` (JSON).
- Read **OPTIONS** (required, choices, defaults), **ATTRIBUTES** (check/diff mode), **EXAMPLES**, **RETURN VALUES**.

### Interview questions

1. **Q: What is an FQCN and why use it?**
   A: The Fully Qualified Collection Name, `namespace.collection.module`, e.g. `ansible.builtin.copy`. It removes ambiguity between collections, makes the source of each module explicit, and is required by linting standards.
2. **Q: How do you find out what options a module accepts?**
   A: `ansible-doc <fqcn>` for the full documentation of the installed version, or `ansible-doc -s <fqcn>` for a task snippet.
3. **Q: How can you tell whether a module supports check mode?**
   A: Its `ATTRIBUTES` section in `ansible-doc` lists `check_mode` support as full, partial or none.

### Scenario question (answer out loud first, then compare)

> *"After upgrading collections in CI, a playbook that used `docker_container:` (short name) starts failing with `couldn't resolve module/action 'docker_container'`, but works on an engineer's laptop. Explain and fix."*

**Model answer:**
1. The short name used to resolve through an old redirect (or through `community.general` being installed with the legacy Docker modules). In CI the new collection set doesn't provide it under that name; on the laptop an older `community.general`/`community.docker` still does.
2. Confirm by comparing `ansible-galaxy collection list` and `ansible --version` collection paths in both places, and `-vvv | grep 'Using module file'` on the laptop.
3. Fix the playbook: use the FQCN `community.docker.docker_container` and add `community.docker` to `collections/requirements.yml` with a pinned version.
4. Fix the process: pin all collections, run `ansible-lint` (which flags short names) in CI, and install the same `requirements.yml` on laptops.

**Likely follow-ups:**
- *Where are collections searched?* `collections_path` (default `~/.ansible/collections`, `/usr/share/ansible/collections`), plus `./collections` next to the playbook.
- *What's a tombstone?* An entry in a collection's `meta/runtime.yml` saying a module was removed, with a pointer to its replacement.
