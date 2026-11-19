# 22 · Jinja2 Basics: Expressions, Filters, Tests

> **Level:** ★ foundation
>
> **You will learn:** what **Jinja2** is and where Ansible uses it, the three delimiters (`{{ }}`, `{% %}`, `{# #}`), expressions (maths, comparisons, `and`/`or`/`not`, `in`, inline `if ... else`, string joining with `~`), the **filters** you'll use daily (`default`, `default(omit)`, `mandatory`, `bool`, `int`, `lower`, `replace`, `join`, `length`, `unique`, `sort`, `to_nice_yaml`, `regex_replace`, `quote`, `password_hash`…), the **tests** (`is defined`, `is none`, `is string`, `is mapping`, `is match`, `is version`…), the special **`omit`** value, and the ansible-core 2.19+ rules (no nested `{{ }}`, how to build a variable name dynamically). You'll use a one-line ad hoc **playground** to try any expression instantly.
>
> **You need:** [21 · Variables basics](21-variables-basics.md).
>
> **Part 3 · Variables, facts and logic** · Next: [23 · Facts and magic variables](23-facts-and-magic-variables.md) · [Back to INDEX](../INDEX.md)

---

## ⚡ Core: the 10-minute version

**The idea.** Everything inside `{{ }}` is a **Jinja2 expression**, evaluated on the control node. Besides plain variables, you can do maths, comparisons, and transform values with **filters** (`value | filter`), or ask questions with **tests** (`value is test`).

**Picture it.** Filters are kitchen tools in a line: `"  Shop  " | trim | lower` = take the value, peel it, chop it, serve `shop`.

**The one thing to know.**

```yaml
"{{ app_port | default(8080) }}"          # use 8080 if app_port isn't defined
"{{ db_password | mandatory }}"           # fail clearly if it isn't defined
"{{ app_users | length }}"                # how many items
"{{ packages | join(', ') }}"             # list → "git, tree, curl"
"{{ 'yes' if app_debug | bool else 'no' }}"
"{{ app_version is version('2.0', '>=') }}"
```

**Try it now** (the playground; no playbook needed):

```bash
ansible localhost -m ansible.builtin.debug -a "msg={{ [3,1,2,3] | unique | sort }}"
ansible localhost -m ansible.builtin.debug -a "msg={{ 'Hello World' | lower | replace(' ', '-') }}"
```

**You've got it if you can:** give an optional variable a default, make a required one fail with a clear message, and turn a list into a comma-separated string.

> 📖 **That's the core.** On a first pass, do the mini-lab and move on. Everything below is the **full reference**.

---

## What it is

**Jinja2** is a templating language for Python (the same one Flask web apps use). Ansible uses it for:

- every `{{ }}` in playbooks, variables and inventories,
- `when:` conditions (which are expressions without the braces, topic 26),
- template **files** (`.j2`) rendered by the `template` module (topic 29),
- `loop:` values, `changed_when`, `failed_when`, and more.

All of it runs **on the control node**, per host, before modules are sent.

### Delimiters

| Delimiter | Purpose | Where you'll see it |
|---|---|---|
| `{{ ... }}` | **Expression**: compute a value and insert it | Everywhere |
| `{% ... %}` | **Statement**: `if`, `for`, `set` | Mostly in template files (topic 29) |
| `{# ... #}` | Comment (not printed) | Template files |

### Expressions

| Kind | Examples | Result |
|---|---|---|
| Literals | `42`, `'text'`, `[1, 2]`, `{'a': 1}`, `true`, `none` | |
| Maths | `workers * 2`, `mem_mb // 1024`, `7 % 3` | `//` = whole-number division, `%` = remainder |
| Comparison | `port == 80`, `count > 3`, `name != 'db'` | boolean |
| Logic | `a and b`, `a or b`, `not a` | boolean |
| Membership | `'web' in group_names`, `'x' not in list` | boolean |
| String join | `'web' ~ '-' ~ env` | `~` converts both sides to strings and joins them |
| Inline if | `'prod' if env == 'production' else 'non-prod'` | |
| Attribute / index | `user.name`, `user['name']`, `list[0]` | (topic 21) |
| Python string methods | `name.upper()`, `path.split('/')`, `line.startswith('#')` | Work, but prefer filters for readability |

### Filters: `value | filter(arguments)`

Filters transform a value. You can chain them. The most useful ones:

**Defaults and required values**

| Filter | Example | Result |
|---|---|---|
| `default(x)` | `app_port \| default(8080)` | `8080` if `app_port` is **undefined** |
| `default(x, true)` | `app_name \| default('shop', true)` | Also used when the value is **empty/false** (`''`, `0`, `none`) |
| `default(omit)` | `mode: "{{ file_mode \| default(omit) }}"` | If undefined, the option is **left out** of the module call entirely |
| `mandatory` | `db_password \| mandatory` | Fails with "Mandatory variable 'db_password' not defined" |
| `mandatory('msg')` | `db_password \| mandatory('Set db_password in vault')` | Your own message (newer ansible-core) |

**Type conversion**

| Filter | Example | Result |
|---|---|---|
| `int`, `float` | `'8080' \| int` | `8080` |
| `string` | `8080 \| string` | `'8080'` |
| `bool` | `'yes' \| bool`, `'false' \| bool` | `True`, `False` (accepts yes/no/true/false/1/0/on/off). ⚠️ Unrecognised values are deprecated and become an error in 2.23 |
| `type_debug` | `x \| type_debug` | Python type name (debugging) |

**Strings**

| Filter | Example | Result |
|---|---|---|
| `lower`, `upper`, `capitalize`, `title` | `'Shop App' \| lower` | `shop app` |
| `replace(a, b)` | `'a-b-c' \| replace('-', '_')` | `a_b_c` |
| `trim` | `'  x  ' \| trim` | `x` |
| `regex_replace(pat, rep)` | `'v1.2.3' \| regex_replace('^v', '')` | `1.2.3` |
| `regex_search(pat)` | `'release-42' \| regex_search('[0-9]+')` | `42` |
| `quote` | `filename \| quote` | Safe for the **shell**: `'my file.txt'`. Use it in every `shell` command built from variables |
| `basename`, `dirname` | `'/opt/app/a.conf' \| basename` | `a.conf` |
| `b64encode`, `b64decode` | `'hi' \| b64encode` | `aGk=` |
| `hash('sha256')` | `'x' \| hash('sha256')` | hex digest |
| `password_hash('sha512')` | `'pw' \| password_hash('sha512')` | A crypt hash for `/etc/shadow` (topic 17) |

**Lists**

| Filter | Example | Result |
|---|---|---|
| `length` | `[1,2,3] \| length` | `3` (also strings and dicts) |
| `join(sep)` | `['a','b'] \| join(',')` | `a,b` |
| `first`, `last` | `[4,5,6] \| first` | `4` |
| `unique`, `sort`, `reverse` | `[3,1,3] \| unique \| sort` | `[1, 3]` |
| `min`, `max`, `sum` | `[2,9,4] \| max` | `9` |
| `union`, `intersect`, `difference` | `[1,2] \| union([2,3])` | `[1, 2, 3]` |
| `flatten` | `[[1],[2,[3]]] \| flatten` | `[1, 2, 3]` |
| `select`, `reject`, `map`, `selectattr` | (topic 32) | |

**Data formats**

| Filter | Use |
|---|---|
| `to_json`, `to_nice_json` | Write a variable as JSON (e.g. into a config file) |
| `to_yaml`, `to_nice_yaml` | Write as YAML |
| `from_json`, `from_yaml` | Parse a string (e.g. command output) into data |
| `dict2items`, `items2dict`, `combine` | Dictionary tools (topics 28, 32) |

### Tests: `value is test`

Tests answer yes/no questions. Used mostly in `when:` (topic 26):

| Test | Example | True when |
|---|---|---|
| `defined` / `undefined` | `app_port is defined` | The variable exists |
| `none` | `x is none` | Value is null |
| `string`, `number`, `integer`, `float`, `boolean` | `x is string` | Type checks |
| `mapping` | `x is mapping` | It's a dictionary |
| `sequence`, `iterable` | `x is sequence` | List-like (⚠️ strings are sequences too) |
| `match`, `search`, `regex` | `name is match('web[0-9]+')` | Regex at the start (`match`) or anywhere (`search`) |
| `version` | `pkg_version is version('2.4', '>=')` | Version comparison (handles `1.10 > 1.9` correctly, unlike string comparison) |
| `subset`, `superset`, `contains` | `['a'] is subset(['a','b'])` | Set relations |
| `succeeded`, `failed`, `changed`, `skipped` | `result is changed` | For registered results (topic 26) |
| `file`, `directory`, `exists`, `link` | `'/etc/x' is file` | ⚠️ Checks the **control node**, not the managed host |

### `omit`: leaving an option out

`omit` is a special value: if a module option's value renders to `omit`, Ansible **removes that option** from the call, as if you'd never written it. Combine with `default`:

```yaml
- name: Create user, shell only if specified
  ansible.builtin.user:
    name: "{{ item.name }}"
    shell: "{{ item.shell | default(omit) }}"     # no 'shell' key → module default
```

### ansible-core 2.19+ templating rules

ansible-core 2.19 rewrote templating. Rules that break old code:

| Rule | ❌ Old style | ✅ Now |
|---|---|---|
| **No nested `{{ }}`** inside an expression | `"{{ foo_{{ env }} }}"`, `when: "{{ x }} == 1"` | `"{{ lookup('ansible.builtin.vars', 'foo_' ~ env) }}"`, `when: x == 1` |
| Conditionals must be **boolean** | `when: my_string` (a non-empty string) | `when: my_string \| length > 0` or `\| bool` (topic 26) |
| Only **trusted** strings are templated | Templates inside data from a module result or a fetched file were rendered | They're treated as plain text (safer) |
| Native types kept | | `"{{ 8080 }}"` → int |

Building a variable name from other variables is the most common real case:

```yaml
vars:
  port_dev: 8080
  port_prod: 80
  env: prod
tasks:
  - ansible.builtin.debug:
      msg: "{{ lookup('ansible.builtin.vars', 'port_' ~ env) }}"     # 80
  - ansible.builtin.debug:
      msg: "{{ vars['port_' ~ env] }}"     # also works today; ⚠️ the `vars` cache is deprecated (removal planned for 2.24)
```

(Better design: a dictionary `ports: {dev: 8080, prod: 80}` and `ports[env]`.)

---

## Why it exists

Plain YAML can only hold fixed values. Real playbooks need small bits of logic: "use 8080 unless told otherwise", "the URL is `https://` + hostname + port", "how many web servers are there?", "fail if the password wasn't provided". Jinja2 gives you a **small, safe expression language** for that, without turning YAML into a full programming language.

---

## Analogy

**A spreadsheet formula bar.** A cell can hold a plain value (`8080`) or a formula (`=IF(A1="prod", 80, 8080)`). Jinja2 expressions are Ansible's formulas: they compute a value from other cells (variables), with functions (filters) like `UPPER()`, `JOIN()` or `LEN()`, and checks (tests) like `ISBLANK()`. And like a spreadsheet, the formula is computed **when it's needed** (lazy), on **your** computer (the control node).

---

## How it works inside

```
 task arg:   msg: "{{ app_users | map(attribute='name') | join(', ') | upper }}"
                │
 1. Ansible sees a trusted string containing {{ }}  → hand it to Jinja2
 2. Jinja2 parses the expression into a small tree:
        upper( join( map(app_users, attribute='name'), ', ' ) )
 3. look up app_users in THIS host's variables (lazy: only now)
 4. call each filter in order (filters are Python functions from ansible.builtin,
    ansible.utils, community.general...)
 5. result "ALICE, BOB" → the module receives a plain string
```

Filters and tests are **plugins** (`ansible-doc -t filter -l`, `ansible-doc -t test -l`). Collections add more (`ansible.utils.ipaddr`, `community.general.json_query`, topic 32).

---

## Code / commands

### The playground

Try any expression without writing a playbook. Variables can come from `-e`:

```bash
ansible localhost -m ansible.builtin.debug -a "msg={{ 17 // 5 }} {{ 17 % 5 }}"
ansible localhost -m ansible.builtin.debug -a "msg={{ ['b','a','b'] | unique | sort | join('-') }}"
ansible localhost -m ansible.builtin.debug -a "msg={{ undefined_thing | default('fallback') }}"
ansible localhost -m ansible.builtin.debug -a "msg={{ '1.10' is version('1.9', '>') }}"
ansible localhost -m ansible.builtin.debug -a "msg={{ '1.10' > '1.9' }}"
ansible localhost -m ansible.builtin.debug -a "msg={{ env | upper }}" -e env=staging
```
```
localhost | SUCCESS => { "msg": "3 2" }
localhost | SUCCESS => { "msg": "a-b" }
localhost | SUCCESS => { "msg": "fallback" }
localhost | SUCCESS => { "msg": true }
localhost | SUCCESS => { "msg": false }
localhost | SUCCESS => { "msg": "STAGING" }
```

- `'1.10' is version('1.9', '>')` → `true`; plain string comparison `'1.10' > '1.9'` → `false` (character by character, `1` < `9`). **Always compare versions with the `version` test.**

Print a variable as JSON for an inline structure:

```bash
ansible localhost -m ansible.builtin.debug -a 'msg={{ {"a": 1, "b": [1,2]} | to_nice_yaml }}'
```

### A playbook of everyday patterns

`~/ansible-lab/jinja.yml`:

```yaml
---
- name: Jinja2 everyday patterns
  hosts: localhost
  gather_facts: false
  vars:
    app_name: "  Shop-API  "
    app_version: "v2.4.1"
    app_users:
      - { name: alice, shell: /bin/bash }
      - { name: bob }
    packages: [git, tree, git, curl]
    env: staging

  tasks:
    - name: Clean a name
      ansible.builtin.debug:
        msg: "{{ app_name | trim | lower | replace('-', '_') }}"          # shop_api

    - name: Version without the v, and a comparison
      ansible.builtin.debug:
        msg: "{{ app_version | regex_replace('^v', '') }} is new: {{ app_version | regex_replace('^v', '') is version('2.0', '>=') }}"

    - name: Unique sorted packages as a sentence
      ansible.builtin.debug:
        msg: "Installing {{ packages | unique | sort | join(', ') }} ({{ packages | unique | length }} packages)"

    - name: Optional value with default and omit
      ansible.builtin.debug:
        msg: "{{ item.name }} uses {{ item.shell | default('the system default shell') }}"
      loop: "{{ app_users }}"

    - name: Inline if
      ansible.builtin.debug:
        msg: "{{ 'big' if (app_users | length) > 1 else 'small' }} team"

    - name: Required value fails clearly
      ansible.builtin.debug:
        msg: "{{ db_password | mandatory }}"
      ignore_errors: true

    - name: Safe shell quoting
      ansible.builtin.debug:
        msg: "rm -f {{ 'my file; rm -rf /' | quote }}"
```

```bash
ansible-playbook jinja.yml
```
```
TASK [Clean a name] ********************************************************************
ok: [localhost] => { "msg": "shop_api" }

TASK [Version without the v, and a comparison] *****************************************
ok: [localhost] => { "msg": "2.4.1 is new: True" }

TASK [Unique sorted packages as a sentence] ********************************************
ok: [localhost] => { "msg": "Installing curl, git, tree (3 packages)" }

TASK [Optional value with default and omit] ********************************************
ok: [localhost] => (item={'name': 'alice', 'shell': '/bin/bash'}) => { "msg": "alice uses /bin/bash" }
ok: [localhost] => (item={'name': 'bob'}) => { "msg": "bob uses the system default shell" }

TASK [Inline if] ***********************************************************************
ok: [localhost] => { "msg": "big team" }

TASK [Required value fails clearly] ****************************************************
fatal: [localhost]: FAILED! => {"msg": "Mandatory variable 'db_password' not defined."}
...ignoring

TASK [Safe shell quoting] **************************************************************
ok: [localhost] => { "msg": "rm -f 'my file; rm -rf /'" }
```

- `loop: "{{ app_users }}"` with `item`: a preview of loops (topic 28): the task runs once per list element.
- `...ignoring`: the failure was ignored because of `ignore_errors: true` (topic 31), so the play continued.
- `quote` wrapped the dangerous string in single quotes, so a shell treats it as **one filename**, not as a second command. This is how you prevent **shell injection** when variables go into `shell:` commands.

### The nested-braces error (2.19+)

```yaml
- ansible.builtin.debug:
    msg: "{{ port_{{ env }} }}"
```
```
fatal: [localhost]: FAILED! => {"msg": "Syntax error in template: unexpected '{'..."}
```

Fix: `"{{ lookup('ansible.builtin.vars', 'port_' ~ env) }}"`, or redesign as a dictionary.

### List all filters and tests you have

```bash
ansible-doc -t filter -l | wc -l
ansible-doc -t filter ansible.builtin.regex_replace
ansible-doc -t test -l | grep version
```

---

## Hands-on lab

1. Use the playground to compute: the number of seconds in a day; `'web01'` → `'WEB01'`; the max of `[12, 7, 30]`; whether `'2.10.0'` is newer than `'2.9.5'`.
2. Run `jinja.yml`. Then change `env` to `production` and add a task printing `"{{ 'prod' if env == 'production' else 'non-prod' }}"`.
3. **default vs default(x, true)**: set `app_title: ""` and compare `app_title | default('Untitled')` (prints empty) with `app_title | default('Untitled', true)` (prints Untitled).
4. **omit**: write a `file` task where `mode: "{{ wanted_mode | default(omit) }}"`. Run once without `wanted_mode`, once with `-e wanted_mode=0700`, and check the directory's mode on web1 each time.
5. **Shell injection**: on web1 (a lab container!), run a `shell` task `echo {{ user_input }}` with `-e 'user_input=hi; touch /tmp/pwned'`. Check that `/tmp/pwned` was created. Then use `{{ user_input | quote }}`, delete the file, run again: no file. Never build shell commands from unquoted variables.
6. Break a template with nested braces, read the error, fix it with `lookup('ansible.builtin.vars', ...)`.

**Done when:** you can write any small transformation in the playground first, then paste it into a playbook.

---

## Where you'll use it

- **Defaults** for every optional setting (`| default(...)`), **required** settings (`| mandatory`).
- **Building values**: URLs, paths, connection strings from several variables.
- **Turning data into config**: `to_nice_yaml`, `to_json`, `join` inside templates.
- **Version checks** before upgrades: `is version`.
- **Safe shell commands**: `| quote`.
- **Conditions** in `when:` (topic 26).

---

## When it breaks

| Symptom | Cause | Fix |
|---|---|---|
| `template error ... unexpected '{'` | Nested `{{ }}` inside `{{ }}` (or inside `when:`) | Use `~` to build strings and `lookup('ansible.builtin.vars', ...)`; plain expressions in `when` |
| `No filter named 'json_query'` / `'ipaddr'` | Filter from a collection that isn't installed, or short name | Install `community.general` / `ansible.utils`; use the FQCN `community.general.json_query` |
| `'x' is undefined` inside a filter chain | Variable missing | `| default(...)` or define it |
| Version comparison wrong (`1.10 < 1.9`) | String comparison | `is version('1.9', '>')` |
| `'"false"' bool` style deprecation warning | `bool` filter got an unrecognised value | Pass real booleans or recognised strings |
| `can only concatenate str (not "int") to str` | `+` mixing types | Use `~`, or convert with `| string` / `| int` |
| An option you defaulted to `omit` still appears | `omit` used inside a string (`"x-{{ omit }}"`) or in a nested structure | `omit` only works as the **entire** value of a module option |
| `is file` says false though the file exists on the server | File tests run on the **control node** | Use the `stat` module + `register` (topic 26) |
| Injection risk in `shell` tasks | Variables inserted raw | `| quote`, or avoid `shell` |

---

## Quick recap

- Jinja2 runs **on the control node**: `{{ expression }}`, `{% statement %}`, `{# comment #}`.
- Expressions: maths (`//`, `%`), comparisons, `and/or/not`, `in`, `~` to join strings, inline `x if c else y`.
- Filters: `default`, `default(x, true)`, `default(omit)`, `mandatory`, `int`/`bool`/`string`, `lower`/`replace`/`regex_replace`/`quote`, `length`/`join`/`unique`/`sort`, `to_nice_yaml`/`from_json`.
- Tests: `is defined`, `is none`, `is mapping`, `is match`, `is version(...)`, `is changed`... File tests check the **control node**.
- 2.19+: no nested `{{ }}`, boolean conditionals, only trusted templates. Build names with `lookup('ansible.builtin.vars', 'x_' ~ y)`.

### Interview questions

1. **Q: How do you give a variable a default value?**
   A: `{{ var | default('value') }}`. Add `true` as the second argument to also replace empty/false values, and use `default(omit)` to leave a module option out entirely.
2. **Q: How do you compare software versions in Ansible?**
   A: With the `version` test: `my_version is version('2.4', '>=')`, which compares version components properly instead of as strings.
3. **Q: How do you avoid shell injection when a variable goes into a `shell` command?**
   A: Use the `quote` filter on every variable (`{{ path | quote }}`), or better, use a module instead of `shell`.

### Scenario question (answer out loud first, then compare)

> *"After upgrading to ansible-core 2.19+, an old playbook fails with template errors on lines like `msg: \"{{ config_{{ env }}.port }}\"` and `when: \"{{ enable_feature }}\"`. How do you fix them properly?"*

**Model answer:**
1. Both use **nested templating**, which 2.19 removed (it was ambiguous and a security risk).
2. `config_{{ env }}`: build the name inside one expression: `{{ lookup('ansible.builtin.vars', 'config_' ~ env).port }}`. Better: restructure to one dictionary keyed by environment (`config[env].port`), or put per-environment values in per-environment `group_vars` so there's just `config.port`.
3. `when: "{{ enable_feature }}"`: `when` is already an expression, so write `when: enable_feature`. And it must be a real boolean: if it can be a string like `"yes"`, use `when: enable_feature | bool`.
4. Find all occurrences: `ansible-lint` flags many of these; `grep -rn '{{[^}]*{{' .` catches nested braces. Fix, then run the playbook in check mode on staging before upgrading production CI.

**Likely follow-ups:**
- *Is there a temporary switch for broken conditionals?* `ALLOW_BROKEN_CONDITIONALS` can downgrade the error to a warning while migrating; don't leave it on.
- *Why was multi-pass templating removed?* It could render untrusted data as templates (injection) and made results unpredictable.
