# 38 · Plugins and Writing Your Own Module

> **Level:** ★★★ advanced
>
> **You will learn:** the **plugin types** that make up Ansible (modules, action, connection, become, inventory, vars, lookup, filter, test, callback, cache, strategy…) and what each one does, where to put **local** plugins (`library/`, `filter_plugins/`… next to a playbook or role) versus **collection** plugins, how to **write an idempotent module in Python** with `AnsibleModule` (argument spec, check mode, diff, `exit_json`/`fail_json`, atomic writes, `DOCUMENTATION`/`EXAMPLES`/`RETURN`), how to write a **custom filter**, how to test them (`ansible-doc`, ad hoc runs, `ansible-test sanity`), and when writing a module is (and isn't) the right answer.
>
> **You need:** [37 · Collections](37-collections-and-galaxy.md) (you created `mycompany.platform`), [13 · Inside a module run](13-inside-a-module-run.md) (AnsiballZ, JSON), [27 · Idempotency](27-idempotency-in-depth.md), basic Python (functions, dicts, files).
>
> **Part 4 · Organizing and reusing code** · Next: [39 · Project layout for production](39-project-layout.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Everything in Ansible that isn't the core engine is a **plugin**: how it connects (connection plugins), what it prints (callbacks), where hosts come from (inventory plugins), the Jinja2 filters you use, and the modules that do the work. When no module fits and your `shell` task needs three lines of `changed_when` logic, **write your own module**: a small Python program that checks state, changes it only if needed, and prints JSON.

**Picture it.** Ansible is a power drill; plugins are the bits and attachments. You can buy them (collections) or machine your own for a special screw (a custom module).

**The one thing to know.** The skeleton of every module:

```python
from ansible.module_utils.basic import AnsibleModule

def main():
    module = AnsibleModule(
        argument_spec=dict(name=dict(type="str", required=True),
                           state=dict(type="str", choices=["present", "absent"], default="present")),
        supports_check_mode=True,
    )
    current = read_current_state(module.params["name"])        # 1. look
    wanted = module.params["state"] == "present"
    changed = current != wanted                                 # 2. compare
    if changed and not module.check_mode:
        apply_change(module.params["name"], wanted)             # 3. act only if needed
    module.exit_json(changed=changed)                           # 4. report JSON

if __name__ == "__main__":
    main()
```

**Try it now:** list the plugin types and count your filters: `ansible-doc -t filter -l | wc -l`.

**You've got it if you can:** name six plugin types and the four steps of an idempotent module.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

### Plugin types

| Type | Runs on | Job | Examples | Local folder |
|---|---|---|---|---|
| **module** | Target | Do the work, return JSON | `ansible.builtin.copy` | `library/` |
| **module_utils** | Target (shipped with modules) | Shared Python code for modules | `ansible.module_utils.basic` | `module_utils/` |
| **action** | Control node | Pre-/post-process a module (copy, template, debug...) | `ansible.builtin.template` | `action_plugins/` |
| **connection** | Control node | Reach the host | `ssh`, `local`, `community.docker.docker`, `winrm` | `connection_plugins/` |
| **become** | Control node | Privilege escalation commands | `sudo`, `su`, `doas` | `become_plugins/` |
| **shell** | Control node | Remote shell syntax | `sh`, `powershell` | `shell_plugins/` |
| **inventory** | Control node | Produce hosts/groups | `ini`, `yaml`, `amazon.aws.aws_ec2`, `constructed` | `inventory_plugins/` |
| **vars** | Control node | Produce variables for hosts/groups | `host_group_vars` (group_vars/host_vars) | `vars_plugins/` |
| **lookup** | Control node | Fetch data for expressions | `file`, `env`, `password` | `lookup_plugins/` |
| **filter** | Control node | Transform data in Jinja2 | `combine`, `to_nice_yaml` | `filter_plugins/` |
| **test** | Control node | Yes/no checks in Jinja2 | `version`, `match` | `test_plugins/` |
| **callback** | Control node | React to events (print, log, notify) | `default`, `json`, `profile_tasks` | `callback_plugins/` |
| **cache** | Control node | Store facts | `jsonfile`, `redis` | `cache_plugins/` |
| **strategy** | Control node | Task scheduling | `linear`, `free`, `host_pinned` | `strategy_plugins/` |
| **cliconf / httpapi / netconf / terminal** | Control node | Network device platforms | `cisco.ios.ios` | (network) |

List any type: `ansible-doc -t <type> -l`. Read one: `ansible-doc -t <type> <name>`.

### Where plugins can live

| Location | Scope | Name in playbooks |
|---|---|---|
| Next to the **playbook**: `library/`, `filter_plugins/`, `lookup_plugins/`, ... | That playbook directory | Short name (`kv_setting`) |
| Inside a **role**: `roles/x/library/`, `roles/x/filter_plugins/`, ... | When the role is used | Short name |
| Paths in `ansible.cfg` (`library`, `filter_plugins`...) / env vars (`ANSIBLE_LIBRARY`...) | Global | Short name |
| Inside a **collection**: `plugins/modules/`, `plugins/filter/`, ... | Anywhere it's installed | **FQCN** (`mycompany.platform.kv_setting`) |

Local folders are fine for experiments. For anything shared, use a **collection**: FQCNs, versioning, docs and testing come with it.

### When to write a module

| Write a module when... | Don't, when... |
|---|---|
| No existing module manages this thing (an internal app's config API, a proprietary CLI) | A module already exists (search first, topic 16) |
| Your `shell` tasks need complex `creates`/`changed_when`/`failed_when` logic | One `command` with `creates` does the job |
| You need check mode and diff support for it | It's a one-off |
| Several teams need the same logic | A role with existing modules can express it |

### Anatomy of a module

| Part | Purpose |
|---|---|
| `DOCUMENTATION` (YAML string) | Options, types, defaults → `ansible-doc`, and checked by `ansible-test sanity` |
| `EXAMPLES` | Copy-paste tasks |
| `RETURN` | Documented return values |
| `AnsibleModule(argument_spec=..., supports_check_mode=True, ...)` | Parses and validates arguments (types, required, choices, `required_if`, `mutually_exclusive`), handles check mode |
| Read current state | Idempotency starts here |
| Compute desired state & diff | `changed` = they differ |
| Apply only if changed **and** `not module.check_mode` | Check mode support |
| `module.exit_json(changed=..., **data)` | Success JSON |
| `module.fail_json(msg=..., **data)` | Failure JSON |
| Helpers: `module.run_command([...])`, `module.atomic_move()`, `module.get_bin_path()`, `module.tmpdir`, `module.warn()` | Safe, consistent behaviour |

Never `print()` other output: stdout must be **only** the JSON (topic 13). Since ansible-core 2.21, returning a non-zero `rc` **without** `failed` is deprecated (it will no longer imply failure): always set failure explicitly with `fail_json`.

---

## Why it exists

Ansible's plugin architecture is why one small core can manage Linux, Windows, network switches, clouds and Kubernetes: each new platform is just more plugins. For you, it means you're never stuck: if something is missing, you can add it in the same structure everyone else uses, with docs, check mode and tests, instead of a growing pile of `shell` tasks.

---

## Analogy

**A modular kitchen appliance** (a stand mixer). The motor (ansible-core) stays the same; attachments do different jobs: the pasta roller (modules), the speed dial (strategy), the bowl (connection), the timer display (callback). Third parties sell attachments (collections). If you need a special one, you build it to the **standard attachment socket** (`AnsibleModule`, plugin base classes) and it fits.

---

## How it works inside

A custom module goes through exactly the path from topic 13:

```
 task: mycompany.platform.kv_setting: {path: /opt/app/app.conf, key: workers, value: "8"}
   │
 control node
   ├─ collection loader finds plugins/modules/kv_setting.py
   ├─ AnsiballZ: zip kv_setting.py + ansible.module_utils.basic (+ anything it imports)
   └─ ship + run on the target with python3
 target
   ├─ AnsibleModule parses ANSIBLE_MODULE_ARGS, validates types → fail early on bad input
   ├─ your code: read file → compare → (not check mode) write atomically
   └─ exit_json → {"changed": true, "previous": "4", "diff": {...}}
```

A filter plugin, by contrast, is plain Python imported **on the control node** by Jinja2. It never travels to targets.

---

## Code / commands

We'll add a module and a filter to the `mycompany.platform` collection from topic 37:

```
~/ansible-lab/collections/ansible_collections/mycompany/platform/
├── galaxy.yml
├── meta/runtime.yml
└── plugins/
    ├── modules/
    │   └── kv_setting.py
    └── filter/
        └── platform.py
```

### The module: `kv_setting`

Manages one `key = value` line in a simple config file: what `lineinfile` does, but without regexes for users to get wrong, and with a useful return value.

`plugins/modules/kv_setting.py`:

```python
#!/usr/bin/python
# -*- coding: utf-8 -*-
# Copyright: (c) 2026, You
# GNU General Public License v3.0+ (see https://www.gnu.org/licenses/gpl-3.0.txt)
from __future__ import annotations

DOCUMENTATION = r"""
---
module: kv_setting
short_description: Ensure one C(key = value) setting in a simple config file
description:
  - Adds, updates or removes a single C(key = value) line.
  - Creates the file if it does not exist and O(state=present).
version_added: "1.0.0"
options:
  path:
    description: Config file to manage.
    type: path
    required: true
  key:
    description: Setting name.
    type: str
    required: true
  value:
    description: Setting value. Required when O(state=present).
    type: str
  state:
    description: Whether the setting must exist.
    type: str
    choices: [present, absent]
    default: present
attributes:
  check_mode:
    support: full
  diff_mode:
    support: full
author:
  - You (@you)
"""

EXAMPLES = r"""
- name: The app uses 8 workers
  mycompany.platform.kv_setting:
    path: /opt/app/app.conf
    key: workers
    value: "8"

- name: debug setting is gone
  mycompany.platform.kv_setting:
    path: /opt/app/app.conf
    key: debug
    state: absent
"""

RETURN = r"""
previous:
  description: The value before the change, or null if the key was not set.
  returned: always
  type: str
  sample: "4"
"""

import os
import re
import tempfile

from ansible.module_utils.basic import AnsibleModule


def read_lines(path):
    if not os.path.exists(path):
        return []
    with open(path, encoding="utf-8") as f:
        return f.read().splitlines()


def main():
    module = AnsibleModule(
        argument_spec=dict(
            path=dict(type="path", required=True),
            key=dict(type="str", required=True, no_log=False),
            value=dict(type="str"),
            state=dict(type="str", choices=["present", "absent"], default="present"),
        ),
        required_if=[("state", "present", ["value"])],
        supports_check_mode=True,
    )
    path = module.params["path"]
    key = module.params["key"]
    value = module.params["value"]
    state = module.params["state"]

    # 1. Look: find the current line for this key
    lines = read_lines(path)
    pattern = re.compile(r"^\s*" + re.escape(key) + r"\s*=\s*(.*?)\s*$")
    index, previous = None, None
    for i, line in enumerate(lines):
        match = pattern.match(line)
        if match:
            index, previous = i, match.group(1)

    # 2. Compare: build the desired content
    new_lines = list(lines)
    if state == "present":
        wanted = f"{key} = {value}"
        if index is None:
            new_lines.append(wanted)
        elif lines[index] != wanted:
            new_lines[index] = wanted
    elif index is not None:
        del new_lines[index]

    changed = new_lines != lines
    before = "\n".join(lines) + ("\n" if lines else "")
    after = "\n".join(new_lines) + ("\n" if new_lines else "")
    result = dict(
        changed=changed,
        previous=previous,
        diff=dict(before=before, after=after, before_header=path, after_header=path),
    )

    # 3. Act: only if needed, never in check mode, atomically
    if changed and not module.check_mode:
        directory = os.path.dirname(path) or "."
        if not os.path.isdir(directory):
            module.fail_json(msg=f"Directory {directory} does not exist", **result)
        fd, tmp = tempfile.mkstemp(dir=module.tmpdir)
        with os.fdopen(fd, "w", encoding="utf-8") as f:
            f.write(after)
        module.atomic_move(tmp, path)

    # 4. Report
    module.exit_json(**result)


if __name__ == "__main__":
    main()
```

Line by line, the important parts:

- `from __future__ import annotations` + the GPL header: conventions for Ansible content (`ansible-test sanity` checks them for collections).
- `DOCUMENTATION`: the same YAML format `ansible-doc` shows; `O(...)`/`C(...)` are documentation markup for option names and code.
- `argument_spec` + `required_if`: AnsibleModule rejects bad input before your code runs (`value` is required when `state=present`).
- `no_log=False` on `key`: tells the sanity checker this option isn't a secret despite its name.
- `supports_check_mode=True` + `if changed and not module.check_mode`: honest prediction, no changes in `--check`.
- `diff=...`: Ansible prints it in `--diff` mode.
- `tempfile` in `module.tmpdir` + `module.atomic_move`: write a temp file, then rename it over the target, so a crash never leaves a half-written config. (`atomic_move` keeps the existing file's permissions and SELinux context.)
- **Known limitations of this teaching module** (fix them before using it for real):
  - if a key appears **more than once**, only the last occurrence is updated or removed; a real module should update all of them, or fail with a clear message;
  - when it **creates** a new file, it sets no `mode`/`owner`/`group`. Add `add_file_common_args=True` to `AnsibleModule(...)` and call `module.set_fs_attributes_if_different(module.load_file_common_arguments(module.params), changed)`, the way `ansible.builtin.lineinfile` does;
  - there are no tests: add `ansible-test units` for the parser and `ansible-test integration` for check/diff mode.
- `exit_json(**result)`: the JSON report.

### The filter: `to_env_lines`

`plugins/filter/platform.py`:

```python
# GNU General Public License v3.0+ (see https://www.gnu.org/licenses/gpl-3.0.txt)
from __future__ import annotations

DOCUMENTATION = r"""
name: to_env_lines
short_description: Render a dict as KEY=value lines
description: Turns C({'port': 80}) into C(PORT=80) lines, sorted by key.
options:
  _input:
    description: Dictionary to render.
    type: dict
    required: true
  prefix:
    description: Prefix for every key.
    type: str
    default: ""
"""


def to_env_lines(data, prefix=""):
    return "".join(f"{prefix}{str(k).upper()}={v}\n" for k, v in sorted(data.items()))


class FilterModule:
    def filters(self):
        return {"to_env_lines": to_env_lines}
```

- `class FilterModule` with a `filters()` method returning `{name: function}`: that's all Jinja2 needs.
- The `DOCUMENTATION` block lets `ansible-doc -t filter mycompany.platform.to_env_lines` work (filters can also be documented in a separate `.yml` file next to the plugin).

### Use them

`~/ansible-lab/custom.yml`:

```yaml
---
- name: Custom module and filter
  hosts: web
  become: true
  vars:
    app_env:
      port: 8080
      log_level: info
  tasks:
    - name: App uses 8 workers
      mycompany.platform.kv_setting:
        path: /opt/app/app.conf
        key: workers
        value: "8"
      register: workers

    - name: Show the previous value
      ansible.builtin.debug:
        msg: "workers was {{ workers.previous }}"

    - name: Env file from a dict (custom filter)
      ansible.builtin.copy:
        dest: /opt/app/app.env
        content: "{{ app_env | mycompany.platform.to_env_lines(prefix='APP_') }}"
        mode: "0644"
```

```bash
ansible-doc mycompany.platform.kv_setting | head -20
ansible-playbook custom.yml --check --diff -l web1
ansible-playbook custom.yml
ansible-playbook custom.yml                    # changed=0
```
```
TASK [App uses 8 workers] **************************************************************
--- before: /opt/app/app.conf
+++ after: /opt/app/app.conf
@@ -1,4 +1,4 @@
 # app.conf: managed by Ansible
 listen = 8080
-workers = 4
+workers = 8
 log_level = info

changed: [web1]

TASK [Show the previous value] *********************************************************
ok: [web1] => { "msg": "workers was 4" }
```

(If `files.yml` from topic 18 manages `/opt/app/app.conf` with `copy`, the two will fight: one sets 4, the other 8. One owner per file! For this lab, run `custom.yml` and leave `files.yml` alone, or point the module at another file.)

### Test without a playbook

```bash
ansible web1 -b -m mycompany.platform.kv_setting -a "path=/tmp/test.conf key=a value=1"
ansible web1 -b -m mycompany.platform.kv_setting -a "path=/tmp/test.conf key=a value=1"     # SUCCESS, changed false
ansible web1 -b -m mycompany.platform.kv_setting -a "path=/tmp/test.conf key=a"             # FAILED: state is present but all of the following are missing: value
ansible localhost -m ansible.builtin.debug -a "msg={{ {'a': 1} | mycompany.platform.to_env_lines }}"
```

### Sanity tests for the collection

```bash
cd ~/ansible-lab/collections/ansible_collections/mycompany/platform
git init -q && git add -A                 # ansible-test only looks at files known to git
ansible-test sanity --docker default -v   # runs pylint, validate-modules, docs checks... in a container
```

- `ansible-test` must run **from inside** `ansible_collections/<ns>/<name>/`.
- `--docker default` uses the official test container (needs Docker), so you don't have to install every linter.
- Unit tests go in `tests/unit/`, integration tests (that actually run the module) in `tests/integration/targets/<name>/` (`ansible-test integration`).

### Local plugins without a collection (quick experiments)

```
~/ansible-lab/
├── library/kv_setting.py          → usable as  kv_setting:  in playbooks here
└── filter_plugins/platform.py     → usable as  | to_env_lines
```

No FQCN, no versioning: fine for prototyping, move to a collection when it works.

---

## Hands-on lab

1. Create the module and filter in `mycompany.platform`. Run `ansible-doc mycompany.platform.kv_setting` and fix any YAML errors it reports.
2. Run the ad hoc tests above: create, re-run (no change), missing `value` (validation error), `state=absent`.
3. Run `custom.yml` with `--check --diff`, then for real, then again (`changed=0`).
4. Add a `backup` option to the module: when `true` and changed, call `module.backup_local(path)` before writing and return `backup_file`. Document it in `DOCUMENTATION` and `RETURN`.
5. Break it on purpose: add a `print("hello")` before `exit_json` and run. Read the `MODULE FAILURE` (stdout not pure JSON), then remove it.
6. Run `ansible-test sanity --docker default` (if Docker is available in your WSL2) and fix one finding.
7. Write a **test plugin** `is_port` (returns true for ints 1–65535) in `plugins/test/platform.py` with `class TestModule: def tests(self): return {"is_port": ...}`, and use it: `when: app_port is mycompany.platform.is_port`.

**Done when:** your module is idempotent, supports check and diff mode, and is documented by `ansible-doc`.

---

## Where you'll use it

- **Internal systems**: modules for your company's config service, license server, deployment API, so playbooks stay declarative.
- **Wrapping awkward CLIs**: replace fragile `shell` + `changed_when` chains with a tested module.
- **Data helpers**: filters for company-specific formats (env files, naming conventions).
- **Callbacks**: send run summaries to Slack/Teams or a log system; enforce output formats in CI.
- **Inventory/vars plugins**: hosts and variables from a CMDB (topic 42).
- **Interviews (senior)**: "Have you written a custom module?" → explain argument spec, check mode, idempotency, sanity tests.

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `couldn't resolve module/action 'mycompany.platform.kv_setting'` | Collection not in a searched path, or wrong folder name | `ansible_collections/mycompany/platform/plugins/modules/`; check `collections_path` |
| `MODULE FAILURE ... JSON` errors | Something printed to stdout besides the JSON | Remove prints; use `module.warn()`/`module.log()` |
| `Unsupported parameters for ... module: x` | Option not in `argument_spec` | Add it, or fix the playbook |
| Module always reports `changed` | Comparison bug (e.g. trailing whitespace, missing newline) | Compare normalised content; test the re-run case |
| Module changes things in `--check` | Missing `not module.check_mode` guard | Guard every write |
| `ansible-doc` fails: `ERROR! module kv_setting missing documentation` / YAML error | Broken `DOCUMENTATION` YAML | Fix indentation; it's YAML in a string |
| `No filter named 'to_env_lines'` | Wrong folder (`plugins/filter/`) or missing `FilterModule` class | Fix structure; use the FQCN |
| `ansible-test`: "must be run from within a collection" | Wrong directory | `cd .../ansible_collections/<ns>/<name>` |
| Works with ansible-core X, not Y | Used private/internal APIs | Stick to `ansible.module_utils.basic` public API |

---

## Quick recap

- Ansible = core + **plugins**: modules (target), and action, connection, become, inventory, vars, lookup, filter, test, callback, cache, strategy plugins (control node).
- Local plugin folders (`library/`, `filter_plugins/`...) for experiments; **collections** (`plugins/modules/`, `plugins/filter/`...) with FQCNs for anything shared.
- A module: `DOCUMENTATION`/`EXAMPLES`/`RETURN`, `AnsibleModule(argument_spec, supports_check_mode)`, **look → compare → act only if needed (not in check mode) → `exit_json`/`fail_json`**, atomic writes, diffs, JSON-only stdout.
- A filter: `class FilterModule: def filters(self): return {...}`.
- Test with `ansible-doc`, ad hoc runs (twice!), `--check --diff`, and `ansible-test sanity`.

### Interview questions

1. **Q: What are Ansible plugins? Name a few types.**
   A: Code that extends Ansible's core: connection (how to reach hosts), inventory (where hosts come from), lookup and filter (data in templates), callback (output/events), become (privilege escalation), cache, strategy, and modules (the units of work run on targets).
2. **Q: How do you make a custom module idempotent and check-mode safe?**
   A: Read the current state, compute the desired state, set `changed` only when they differ, perform changes only if `changed and not module.check_mode`, and declare `supports_check_mode=True`. Return a diff if useful.
3. **Q: Where do you put a custom module so a whole team can use it?**
   A: In a collection (`plugins/modules/`), versioned and published to a (private) hub or Git, and referenced by FQCN.

### Scenario question (answer out loud first, then compare)

> *"Your playbooks call an internal REST API with `uri` tasks: GET the current config, compare with Jinja2, PUT if different, with complex `changed_when` and `failed_when`. It's slow, hard to read and has broken idempotency twice. What would you do?"*

**Model answer:**
1. Write a custom module (in the team collection) that wraps the API: options for the desired settings, `supports_check_mode=True`.
2. Inside: GET current config, normalise both sides (types, ordering, defaults), compute a diff, PUT only when different and not in check mode, return `changed`, `diff` and the final object. Use `ansible.module_utils.urls.fetch_url`/`open_url` for HTTP with proper TLS handling and timeouts; mark tokens with `no_log=True`.
3. Document it (`DOCUMENTATION`, `RETURN`), add sanity and integration tests (against a test API or mock), version and publish it.
4. Replace the `uri` chains with one declarative task per setting; check mode now gives a real plan.

**Likely follow-ups:**
- *Why not an action plugin?* The API is reachable from the managed host or `localhost`; a normal module run with `delegate_to: localhost` is simpler. Action plugins are for logic that must run on the control node around a module.
- *How do you protect the API token?* `no_log=True` option in the spec (masked as a secret since 2.22), pass it from Vault or a secret lookup.
