# 18 · Essential Modules II: Files and Content

> **Level:** ★ foundation
>
> **You will learn:** the file modules in depth: **`file`** (directories, links, permissions, delete), **`copy`** (from the control node, inline `content`, backups, `validate`, the trailing-slash rule for directories), **`lineinfile`** / **`blockinfile`** / **`replace`** (editing parts of files safely, and why the regex must match the new line too), **`stat`**, **`fetch`**, **`get_url`** (with checksums), **`unarchive`**, **`ansible.posix.synchronize`**, plus owner/group/mode, SELinux context and the "manage whole files, not lines" principle.
>
> **You need:** [17 · Essential modules I](17-modules-packages-services-users.md), [Linux 026 · Permissions](../../linux-mastery-rhel/notes/026-file-permissions-and-umask.md), [Linux 017 · Regular expressions](../../linux-mastery-rhel/notes/017-grep-and-regular-expressions.md).
>
> **Part 2 · Playbooks** · Next: [19 · Privilege escalation: become](19-become.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Most configuration is files. Ansible has a module for every way to put content on a server: create paths (`file`), send whole files (`copy`, later `template`), edit one line (`lineinfile`) or one block (`blockinfile`), and download (`get_url`) or unpack (`unarchive`) things.

**Picture it.** Decorating a flat: `file` builds rooms and sets the locks; `copy` brings in finished furniture; `lineinfile` is changing one picture on the wall; `blockinfile` is one labelled shelf you own; `get_url` is ordering online with a receipt you can check.

**The one thing to know.**

```yaml
- name: App directory exists
  ansible.builtin.file:
    path: /opt/app
    state: directory
    owner: root
    group: root
    mode: "0755"

- name: App config is in place (whole file)
  ansible.builtin.copy:
    src: app.conf              # from files/app.conf next to the playbook
    dest: /opt/app/app.conf
    mode: "0644"
    backup: true

- name: sshd allows only 3 auth tries (one line, validated)
  ansible.builtin.lineinfile:
    path: /etc/ssh/sshd_config
    regexp: '^#?MaxAuthTries'
    line: MaxAuthTries 3
    validate: /usr/sbin/sshd -t -f %s
```

**Try it now:** run the lab below, then run it again: `changed=0`. Then edit a file by hand and watch Ansible put it back.

**You've got it if you can:** explain why `regexp: '^#?MaxAuthTries'` is needed and what goes wrong without it.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Common options (all file modules)

Almost every file module accepts these, with the same meaning:

| Option | Meaning | Example |
|---|---|---|
| `owner`, `group` | Ownership | `nginx`, `root` |
| `mode` | Permissions: octal **string** or symbolic | `"0640"`, `"u=rw,g=r,o="`, `"u+x"` |
| `setype`, `seuser`, `serole`, `selevel` | SELinux context parts ([Linux 073](../../linux-mastery-rhel/notes/073-selinux-file-contexts.md)) | `setype: httpd_sys_content_t` |
| `attributes` | `chattr` attributes ([Linux 027](../../linux-mastery-rhel/notes/027-special-permissions-acls-and-attributes.md)) | `+i` |
| `backup` | Keep a timestamped copy before changing | `true` |
| `validate` | Command to check the new content **before** it replaces the real file; `%s` = temp file path | `/usr/sbin/visudo -cf %s` |

⚠️ If you omit `mode` when **creating** a file, the result depends on the remote **umask** and module defaults. Always set `mode` explicitly for anything security-related.

### `ansible.builtin.file`: paths, not content

| `state` | Result | Like |
|---|---|---|
| `directory` | Directory exists (parents created) | `mkdir -p` |
| `file` | Only **checks/sets attributes** of an existing file; fails if missing | `chmod`/`chown` |
| `touch` | File exists (created empty if missing); updates timestamps → **changed every run** unless you set `access_time: preserve` and `modification_time: preserve` | `touch` |
| `link` | Symbolic link `path` → `src` | `ln -sfn` |
| `hard` | Hard link | `ln` |
| `absent` | Gone, recursively for directories | `rm -rf` |

`recurse: true` (with `state: directory`) applies owner/group/mode to everything inside. ⚠️ Careful: it applies the **same mode to files and directories** unless you use symbolic modes like `u=rwX,g=rX,o=` (capital `X` = execute only on directories).

### `ansible.builtin.copy`: whole files

| Option | Meaning |
|---|---|
| `src` | Local file on the control node. Relative paths are searched in `files/` next to the playbook/role, then the playbook directory |
| `content` | Write this text instead (good for short files; use `template` for anything with variables, topic 29) |
| `dest` | Remote path. If it ends in `/`, it's a directory and the file keeps its name |
| `remote_src: true` | `src` is already **on the target** (copy remote → remote) |
| `force: false` | Only create if missing; never overwrite (for "seed" files users may edit) |
| `backup: true` | Save the old version as `dest.12345.2026-11-19@12:00:00~` |
| `validate` | Test before replacing (e.g. `nginx -t -c %s`, `visudo -cf %s`, `sshd -t -f %s`) |

**Copying directories** follows the rsync rule:

| `src` | Result on target with `dest: /opt/app/` |
|---|---|
| `site` (no slash) | `/opt/app/site/...` (the directory itself is copied) |
| `site/` (slash) | `/opt/app/...` (only its **contents**) |

`copy` handles directories file by file. For hundreds of files, use `ansible.posix.synchronize` (rsync) or an archive + `unarchive`.

**How `copy` decides "changed":** it compares the **SHA-1 checksum** of the local source with the remote file. Same checksum → no transfer, `ok`. That's why `copy` is idempotent and fast on re-runs.

### `ansible.builtin.lineinfile`: one line

Makes sure **one line** is present (or absent), optionally replacing the line a regex matches.

| Option | Meaning |
|---|---|
| `path` | File to edit |
| `line` | The exact line that must exist |
| `regexp` | Python regex: if it matches a line, **that line is replaced** with `line` (the **last** match is replaced) |
| `state: absent` | Remove lines matching `regexp` (or equal to `line`) |
| `insertafter` / `insertbefore` | Where to add `line` if no `regexp` matched: a regex, `EOF` or `BOF` |
| `create: true` | Create the file if missing |
| `backrefs: true` | Use regex groups (`\1`) in `line`; if the regex doesn't match, **do nothing** |

**The golden rule:** `regexp` must match **both** the old line **and** the new line.

```yaml
# ❌ Wrong: only matches the COMMENTED form
regexp: '^#MaxAuthTries'
line: MaxAuthTries 3
```

- Run 1: `#MaxAuthTries 6` matches → replaced by `MaxAuthTries 3`. Fine.
- Run 2: the exact line `MaxAuthTries 3` already exists → `ok`. Still fine.
- Someone edits it by hand to `MaxAuthTries 10`. Run 3: the regexp doesn't match `MaxAuthTries 10`, and the exact `line` isn't there, so Ansible **appends a second line**. Now the file has both `MaxAuthTries 10` and `MaxAuthTries 3`, and sshd uses the **first** one it reads.

```yaml
# ✅ Right: matches commented, uncommented, any value
regexp: '^#?\s*MaxAuthTries\s'
line: MaxAuthTries 3
```

Rule of thumb: anchor with `^`, allow an optional `#`, and match the **key**, not the value.

### `ansible.builtin.blockinfile`: one managed block

Inserts or updates a multi-line block between **markers**:

```
# BEGIN ANSIBLE MANAGED BLOCK lab-hosts
172.30.0.11 web1
172.30.0.12 web2
# END ANSIBLE MANAGED BLOCK lab-hosts
```

| Option | Meaning |
|---|---|
| `block` | The content (use `\|` in YAML) |
| `marker` | Marker template; include a **unique** name: `"# {mark} ANSIBLE MANAGED BLOCK lab-hosts"` |
| `insertafter` / `insertbefore`, `create`, `backup`, `validate` | As above |
| `state: absent` | Remove the block |

Two `blockinfile` tasks on the same file **must** use different markers, or they overwrite each other's block.

### `ansible.builtin.replace`: regex everywhere

Replaces **all** matches of a regex in a file (like `sed -i 's/a/b/g'`). Idempotent only if the replacement doesn't match the regex again.

### Information and transfer modules

| Module | Does | Notes |
|---|---|---|
| `ansible.builtin.stat` | Reports whether a path exists, type, mode, owner, checksum, SELinux context... | Read-only; combine with `register` + `when` (topic 26) |
| `ansible.builtin.fetch` | Copies a remote file **to the control node** | Saved as `dest/<inventory_hostname>/<full path>`; `flat: true` saves straight to `dest` |
| `ansible.builtin.slurp` | Reads a remote file into a variable (base64) | Small files only |
| `ansible.builtin.find` | Lists files by pattern/age/size | Like `find` ([Linux 020](../../linux-mastery-rhel/notes/020-finding-files.md)) |
| `ansible.builtin.get_url` | Downloads from a URL **on the target** | Set `checksum: sha256:<hash>` for integrity **and** idempotency |
| `ansible.builtin.unarchive` | Extracts `.tar.gz`, `.zip`... | Copies from the control node by default; `remote_src: true` for archives already on the host or a URL. Use `creates:` to skip when already extracted |
| `ansible.builtin.tempfile` | Creates a temp file/dir on the target | |
| `ansible.posix.synchronize` | rsync between control node and target | Needs `rsync` on both ends; great for large trees |

### Whole files vs lines: the principle

| Approach | Pros | Cons |
|---|---|---|
| **Whole file** (`copy`, `template`) | You know the **exact** final content; easy to review; no regex surprises | You own the entire file (vendor changes in new package versions won't merge in) |
| **Line/block** (`lineinfile`, `blockinfile`) | Touches only what you need | Fragile regexes; the rest of the file is whatever happens to be there |

Prefer **whole files** (with `template`, topic 29) for config you own. Use `lineinfile`/`blockinfile` for small changes to files the OS or another tool owns (`/etc/hosts`, `sshd_config` on images you don't control). Even better: many programs read **drop-in directories** (`/etc/ssh/sshd_config.d/`, `/etc/sudoers.d/`, `/etc/sysctl.d/`, `/etc/systemd/system/x.service.d/`): write a whole small file there.

---

## Why it exists

Editing files by hand or with `sed`/`echo >>` in scripts causes the classic failures: duplicate lines, half-written files, broken configs that take down a service, no backup, no record of what changed. These modules give you **atomic writes** (write to a temp file, then rename), **validation before replacing**, **backups**, **checksums** to avoid pointless rewrites, and **diffs** of what changed (topic 20).

---

## Analogy

**Editing a shared document.**

- `copy`/`template` = **replace the whole document** with the approved version from the archive. You always know exactly what it says.
- `lineinfile` = **find-and-replace one sentence**. Works if your search phrase finds the sentence both before and after you change it; otherwise you end up with the sentence twice.
- `blockinfile` = **a clearly labelled box** in the document ("Section maintained by IT, do not edit"), which you rewrite as a whole.
- `validate` = a **proof-reader** who checks the new version before it goes on the noticeboard.
- `backup` = keeping yesterday's copy in the drawer.

---

## How it works inside

How `copy` (with a local `src`) works, combining the action plugin and the module (topic 13):

```
 control node (action plugin)                          target
 ─────────────────────────────                         ──────
 1. find src: files/app.conf (role/playbook search path)
 2. compute SHA-1 of the local file
 3. run "stat" module remotely on dest ───────────────▶ stat /opt/app/app.conf → checksum
 4. checksums equal? → only fix owner/mode if needed → ok/changed (no transfer)
    different/missing? → transfer the file to a temp path ─▶ ~/.ansible/tmp/.../source
 5. run the "copy" module ────────────────────────────▶ a. if validate: run "cmd %s" on the temp file
                                                          → fails? stop, real file untouched
                                                       b. if backup: copy old dest aside
                                                       c. atomic move: rename temp → dest
                                                       d. set owner/group/mode/SELinux
 6. ◀────────── {"changed": true, "checksum": "...", "backup_file": "..."}
```

`lineinfile`/`blockinfile` work entirely on the target: read the file, compute the new content in memory, compare, and if different write a temp file, validate it, and rename it into place.

---

## Code / commands

### The lab playbook

Create `~/ansible-lab/files/app.conf`:

```ini
# app.conf: managed by Ansible
listen = 8080
workers = 4
log_level = info
```

`~/ansible-lab/files.yml`:

```yaml
---
- name: File modules on the web servers
  hosts: web
  become: true

  tasks:
    - name: App directories exist
      ansible.builtin.file:
        path: "{{ item }}"
        state: directory
        mode: "0755"
      loop:
        - /opt/app
        - /opt/app/releases
        - /opt/app/logs

    - name: App config is in place
      ansible.builtin.copy:
        src: app.conf
        dest: /opt/app/app.conf
        owner: root
        group: root
        mode: "0644"
        backup: true

    - name: current -> releases symlink exists
      ansible.builtin.file:
        src: /opt/app/releases
        dest: /opt/app/current
        state: link

    - name: sshd allows only 3 auth tries
      ansible.builtin.lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?\s*MaxAuthTries\s'
        line: MaxAuthTries 3
        validate: /usr/sbin/sshd -t -f %s

    - name: Lab hosts are in /etc/hosts
      ansible.builtin.blockinfile:
        path: /etc/hosts
        marker: "# {mark} ANSIBLE MANAGED BLOCK lab-hosts"
        block: |
          172.30.0.11 web1
          172.30.0.12 web2
          172.30.0.21 db1
          172.30.0.31 lb1
        unsafe_writes: true

    - name: Release archive is unpacked
      ansible.builtin.unarchive:
        src: release-1.0.tar.gz
        dest: /opt/app/releases/
        creates: /opt/app/releases/release-1.0/VERSION

    - name: Copy of os-release comes back to the control node
      ansible.builtin.fetch:
        src: /etc/os-release
        dest: fetched/
```

New things, explained:

- `path: "{{ item }}"` + `loop:` with a list: run the task once **per list item**, with `item` set to the current one. (A one-line preview; topic 28 covers loops properly.)
- `state: link`: `/opt/app/current` points to `/opt/app/releases`. A common deploy pattern: switch the link to a new release directory.
- `validate: /usr/sbin/sshd -t -f %s`: if the edited config is invalid, the real `sshd_config` is never touched. (We don't restart sshd here; restarts on change come with **handlers**, topic 30.)
- `blockinfile` on `/etc/hosts` with `unsafe_writes: true`: inside Docker, `/etc/hosts` is a **bind mount**, so it can't be replaced by renaming a temp file (atomic move). `unsafe_writes` lets the module fall back to writing in place. On real servers you don't need it.
- `unarchive` with `creates:`: skip if that file already exists, so the archive isn't re-extracted every run. `src` is a local archive in `files/`.
- `fetch` to `fetched/`: files arrive as `fetched/web1/etc/os-release` and `fetched/web2/etc/os-release`.

Make the release archive locally first:

```bash
cd ~/ansible-lab
mkdir -p /tmp/release-1.0 && echo "1.0" > /tmp/release-1.0/VERSION
tar -czf files/release-1.0.tar.gz -C /tmp release-1.0
```

### Run it

```bash
ansible-playbook files.yml
```
```
TASK [App directories exist] ***********************************************************
changed: [web1] => (item=/opt/app)
changed: [web2] => (item=/opt/app)
changed: [web1] => (item=/opt/app/releases)
changed: [web2] => (item=/opt/app/releases)
changed: [web1] => (item=/opt/app/logs)
changed: [web2] => (item=/opt/app/logs)

TASK [App config is in place] **********************************************************
changed: [web1]
changed: [web2]

TASK [current -> releases symlink exists] **********************************************
changed: [web1]
changed: [web2]

TASK [sshd allows only 3 auth tries] ***************************************************
changed: [web1]
changed: [web2]

TASK [Lab hosts are in /etc/hosts] *****************************************************
changed: [web1]
changed: [web2]

TASK [Release archive is unpacked] *****************************************************
changed: [web1]
changed: [web2]

TASK [Copy of os-release comes back to the control node] *******************************
changed: [web1]
changed: [web2]

PLAY RECAP *****************************************************************************
web1   : ok=8  changed=7  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
web2   : ok=8  changed=7  unreachable=0  failed=0  skipped=0  rescued=0  ignored=0
```

- `=> (item=/opt/app)`: one result line per loop item.
- `fetch` reports `changed` the first time (a new local file) and `ok` after that (same checksum).

Second run: `changed=0` everywhere.

### See validation stop a bad edit

Change the `line:` to `MaxAuthTries three` and run with `-l web1`:

```
TASK [sshd allows only 3 auth tries] ***************************************************
fatal: [web1]: FAILED! => {"changed": false, "exit_status": 255, "msg": "failed to validate", "stderr": "/home/ansible/.ansible/tmp/ansible-tmp-.../tmpq2x: line 133: Bad number 'three'.\r\n", ...}
```

- `failed to validate`: sshd rejected the temp copy. **`/etc/ssh/sshd_config` is unchanged.** Without `validate`, the next sshd restart would have failed and locked everyone out.

Put `3` back.

### Inspect a file with stat

```bash
ansible web1 -b -m ansible.builtin.stat -a "path=/opt/app/app.conf"
```
```
web1 | SUCCESS => {
    "changed": false,
    "stat": {
        "checksum": "4d1c6e0a5f2b7c...",
        "exists": true,
        "gr_name": "root",
        "isdir": false,
        "mode": "0644",
        "path": "/opt/app/app.conf",
        "pw_name": "root",
        "size": 72,
        ...
    }
}
```

- `stat.exists`, `stat.mode`, `stat.checksum`: the fields you'll test with `register` + `when` (topic 26).

### Download with a checksum

```yaml
- name: node_exporter tarball is downloaded and verified
  ansible.builtin.get_url:
    url: https://github.com/prometheus/node_exporter/releases/download/v1.10.2/node_exporter-1.10.2.linux-amd64.tar.gz
    dest: /tmp/node_exporter.tar.gz
    checksum: "sha256:<the sha256 from the release's sha256sums.txt>"
    mode: "0644"
```

- With `checksum`, `get_url` first checks whether `dest` already has that checksum → `ok`, no download. And a tampered or corrupted download **fails**. (Without `checksum` it re-downloads only if `dest` is missing, or always with `force: true`.)
- The URL/version is an example; use the vendor's current release and its published checksum.

---

## Hands-on lab

1. Create `files/app.conf`, the release archive, and `files.yml`. Run twice; second run `changed=0`.
2. **Drift**: `ssh ansible@172.30.0.11 'echo "debug = true" | sudo tee -a /opt/app/app.conf'`. Re-run: `copy` reports `changed` on web1 and keeps a backup. Find it: `ssh ansible@172.30.0.11 'ls /opt/app/'`.
3. **The regexp rule**: change the regexp to `'^#MaxAuthTries'` (commented form only). On web1 set the line by hand to `MaxAuthTries 10` (`sudo sed -i 's/^MaxAuthTries.*/MaxAuthTries 10/' /etc/ssh/sshd_config`). Run. Now `grep MaxAuthTries /etc/ssh/sshd_config` shows **two** lines. Fix the regexp back to `'^#?\s*MaxAuthTries\s'`, run again: one line.
4. **Validation**: try `MaxAuthTries three` as above and confirm the real file is untouched.
5. **Blocks**: change one IP in the block and re-run; only the block changes. Add a second `blockinfile` task with **the same marker** and different content, run, and watch the two tasks fight (each run changes). Give it its own marker.
6. **Fetch**: `tree ~/ansible-lab/fetched` and compare the two OS files.
7. **Clean up**: write a small play that removes `/opt/app` (`state: absent`) and the `/etc/hosts` block (`state: absent` with the same marker). Run it, then run `files.yml` again to restore.

**Done when:** you can predict, for each task, what makes it `changed` and what keeps it `ok`.

---

## Where you'll use it

- **App deploys**: release directories, a `current` symlink, config files, log directories with the right owner.
- **Hardening**: SSH settings, sudoers drop-ins, banners, `/etc/security/limits.d/`, all with `validate`.
- **Fleet data collection**: `fetch` logs or configs from 200 servers into one folder for review.
- **Installing software not in repos**: `get_url` with checksum + `unarchive`.
- **Small OS-owned file edits**: `/etc/hosts`, `/etc/environment`, one line in a vendor config.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `Could not find or access 'app.conf'` | Local `src` not in `files/` or the playbook dir | Put it in `files/`; check the search path in the error |
| Duplicate lines after `lineinfile` | `regexp` doesn't match the current line form | Match the key with an optional `#`: `'^#?\s*Key\s'` |
| `lineinfile` adds the line at the end instead of the right section | No `regexp` match and no `insertafter` | Add `insertafter: '^\[section\]'` (or manage the whole file) |
| `failed to validate` | The new content is invalid (good: the real file is safe) | Fix the content; read `stderr` |
| `Destination /etc/hosts ... Device or resource busy` / `Could not replace file` | Bind-mounted file (containers) can't be atomically replaced | `unsafe_writes: true` (containers only) |
| `touch` task `changed` every run | Timestamps update each time | `access_time: preserve`, `modification_time: preserve`, or use `copy` with `force: false` |
| Permissions wrong on new files | No `mode` given; umask decided | Always set `mode` (quoted) |
| `recurse` made all files executable | Same octal mode applied to files and dirs | Symbolic mode with capital `X`: `u=rwX,g=rX,o=` |
| `unarchive` runs every time | No `creates:` | Add `creates:` pointing to a file inside the archive |
| `get_url` re-downloads / no integrity check | No `checksum` | Add `checksum: sha256:...` |
| SELinux denials after copying web content on RHEL | Wrong context (e.g. `user_home_t` from `remote_src`) | `setype: httpd_sys_content_t` or `restorecon` (topic 6 table) |

---

## Quick recap

- `file` = paths and attributes (`directory`, `link`, `absent`, `touch`); `copy` = whole files from the control node (checksum-based, `backup`, `validate`, trailing-slash rule).
- `lineinfile` = one line; **regexp must match old and new forms**. `blockinfile` = one marked block; **unique marker per block**. `replace` = regex everywhere.
- `stat` (info), `fetch` (bring back), `get_url` (+ **checksum**), `unarchive` (+ **creates**), `synchronize` (rsync).
- Always set `mode` (quoted). Use `validate` for configs that can break services or logins.
- Prefer **whole files** and **drop-in directories** over line edits for anything you own.

### Interview questions

1. **Q: `copy` vs `template`?**
   A: `copy` sends a file as is (or inline `content`). `template` renders a Jinja2 template with variables on the control node first, then sends the result. Both are checksum-based and idempotent.
2. **Q: Why does `lineinfile` sometimes add duplicate lines?**
   A: When the `regexp` doesn't match the existing line (e.g. it only matches the commented form, or a specific value), the module treats the line as missing and adds a new one. Write a regexp that matches the key in all its forms.
3. **Q: What does `validate` do?**
   A: Runs a check command on a temporary copy of the new file (`%s`) and only replaces the real file if the command succeeds. Essential for sudoers, sshd, nginx and similar configs.

### Scenario question (answer out loud first, then compare)

> *"A teammate pushed an Ansible change that edited `sshd_config` with `lineinfile` on 300 servers. A handler restarted sshd, and now 40 servers refuse SSH. What likely happened, how do you recover, and how do you prevent it?"*

**Model answer:**
1. Likely: the edit produced an invalid or more restrictive config on some servers (different OS versions or prior manual edits), and with no `validate`, the restart applied a broken config. On those 40, sshd failed to start or rejects the automation user.
2. Recover: existing SSH sessions and ControlPersist masters may still work; otherwise use out-of-band access (cloud serial console, SSM, hypervisor console) to restore the backup (`backup: true` files) and restart sshd. Then re-run a corrected playbook.
3. Prevent:
   - `validate: /usr/sbin/sshd -t -f %s` on every sshd edit;
   - prefer a drop-in file in `/etc/ssh/sshd_config.d/` (whole file) over regex edits;
   - roll out with `serial` batches and `max_fail_percentage`, test SSH after the restart in each batch before continuing (topics 45, 49);
   - test on staging and with Molecule across all OS versions in the fleet.

**Likely follow-ups:**
- *Why didn't failing hosts stop the rollout?* Without batching, all 300 ran at once. With `serial: 10%` and `max_fail_percentage: 0`, the first failure would have stopped it.
- *Does `validate` catch "valid but locks you out" configs?* No (e.g. `AllowUsers` without the automation user). That needs a post-change connection test in each batch.
