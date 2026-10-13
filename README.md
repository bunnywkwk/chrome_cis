# chrome_cis

Ansible role that hardens **Google Chrome** and **Chromium** on **RHEL 8, 9 and 10** to the
**CIS Google Chrome Benchmark v3.0.0**, in the [ansible-lockdown](https://github.com/ansible-lockdown) style.
Every task is one CIS recommendation. The benchmark is written for Windows; on Linux Chrome reads the same settings
(policies) from a file, so the role writes one policy file for each browser it finds:

| Browser | Policy file |
|---------|-------------|
| Google Chrome (any channel) | `/etc/opt/chrome/policies/managed/chrome_cis.json` |
| Chromium | `/etc/chromium/policies/managed/chrome_cis.json` |

Platforms: RHEL and compatible (Rocky, Alma, Oracle), x86_64, with a desktop.

## Requirements

- ansible-core **2.16.x** on the control node: the last version that can manage RHEL 8.
- No collections needed.
- Chrome or Chromium installed, or `chrome_cis_install: true` to install one.

## Quick start

```bash
cat > site.yml <<'EOF'
- hosts: workstations
  become: true
  roles:
    - chrome_cis
EOF
ansible-playbook -i inventory site.yml --check --diff   # preview, change nothing
ansible-playbook -i inventory site.yml                  # apply; a second run reports changed=0
```

Then open `chrome://policy` and press **Reload policies** (or restart the browser). The role never closes the
browser, so users don't lose their work.

## What runs by default

| Setting | Default | Meaning |
|---------|---------|---------|
| `chrome_cis_level_1` | `true` | CIS Level 1: safe for most organizations |
| `chrome_cis_level_2` | `false` | CIS Level 2: stricter. Set **both** to `true` for Level 2 |
| `chrome_cis_section1` .. `chrome_cis_section5` | `true` | the 5 benchmark sections |
| `chrome_cis_rule_<id>` | `true` | one switch per CIS rule, e.g. `chrome_cis_rule_4_7` for 4.7 |
| site values | `null` / `[]` | settings your organization decides: the rule is skipped until you set one |

**The full benchmark is on, including rules that change what users can do.** Switch off what your organization does
not accept; that list is also your compliance exception list. Every setting, with a one-line explanation, is in
[`defaults/main.yml`](defaults/main.yml).

## Your settings (`group_vars`)

Never edit the tasks. Write switches unquoted (`true` / `false`). Example `group_vars/workstations.yml`:

```yaml
# Install (only when the browser is missing)
chrome_cis_install: true
chrome_cis_install_browser: chrome     # chrome or chromium
chrome_cis_version: ""                 # "" = newest, or an exact version, e.g. "143.0.7499.40"

# Level 2 as well
chrome_cis_level_2: true

# Exceptions: rules this organization does not apply, and why
chrome_cis_rule_4_7: false             # no DNS-over-HTTPS server
chrome_cis_rule_4_4: false             # web meetings need the microphone
chrome_cis_rule_4_5: false             # web meetings need the camera

# Site values
chrome_cis_safe_browsing_protection_level: 1
chrome_cis_extension_allowlist: ["<extension id>"]
```

## Rules that change what users can do

On by default. Turn one off with `chrome_cis_rule_<id>: false` when users need what it blocks.

| Rule | What users notice |
|------|-------------------|
| 2.2.3 | USB devices and security keys stop working in web apps |
| 2.3.3 | **every** extension is removed, except `chrome_cis_extension_allowlist` |
| 2.4.1 | sites using Basic/Digest login stop working |
| 2.5.1 | extensions can't talk to local programs (password managers), except `chrome_cis_native_messaging_allowlist` |
| 2.12 | no "proceed anyway" on certificate errors: internal sites with bad certificates are unreachable |
| 2.18 | certificates from local CAs fail when the OCSP/CRL server is down |
| 3.1.1 | cookies deleted when the browser closes: users log in again every day |
| 4.1.1 | no screen sharing in web meetings (also 4.12) |
| 4.3 | no file upload or "save as" dialogs |
| 4.4 | no microphone for websites |
| 4.5 | no camera for websites |
| 4.7 | DNS over HTTPS only: **every site fails** without a DoH server |

## Site values

The rule is skipped until you set its value.

| Rule | Setting | Example |
|------|---------|---------|
| 1.2.2 | `chrome_cis_safe_browsing_protection_level` | `1` standard, `2` enhanced (sends more data to Google) |
| 1.9 | `chrome_cis_variations` | `0` (the CIS value) |
| 2.3.3 | `chrome_cis_extension_allowlist` | `["<extension id>"]` |
| 2.3.5 | `chrome_cis_third_party_storage_blocked_origins` | `["https://example.com"]` |
| 2.5.1 | `chrome_cis_native_messaging_allowlist` | `["<host name>"]` |
| 2.6.1 | `chrome_cis_password_manager_enabled` | `true` or `false` |
| 2.8.1 | `chrome_cis_remote_access_connections` | `false` (CIS), `true` if Chrome Remote Desktop is approved |
| 2.8.3 | `chrome_cis_remote_access_client_domains` | `["example.com"]` |
| 2.17 | `chrome_cis_proxy_mode` | `direct` or `system` |
| 2.27 | `chrome_cis_http_allowlist` | `["intranet.example.com"]` |
| 4.2.3 / 4.2.4 | `chrome_cis_clipboard_allowed_urls` / `_blocked_urls` | `["https://app.example.com"]` / `["[*.]example.org"]` |
| 4.2.7 / 4.2.8 | `chrome_cis_window_management_allowed_urls` / `_blocked_urls` | same format (Level 2) |

## Rules that depend on the browser version

Google removed these settings from newer Chrome and Chromium versions. The role applies each one only when the
installed version is in its range and skips it otherwise. If your browsers are always outside the range, you can
switch the rule off in `group_vars` to show it as an exception; search the switch name in `defaults/main.yml`.

| Rule | Applied on versions | Switch it off |
|------|---------------------|---------------|
| 1.10 | 67-127 | `chrome_cis_rule_1_10: false` |
| 2.3.4 | 113-145 | `chrome_cis_rule_2_3_4: false` |
| 2.3.5 | 113-145 | `chrome_cis_rule_2_3_5: false` |
| 2.3.6 | 110-138 | `chrome_cis_rule_2_3_6: false` |
| 2.7.1 | 17-144 | `chrome_cis_rule_2_7_1: false` |
| 2.9.1 | 113-152 | `chrome_cis_rule_2_9_1: false` |
| 2.23 | 112-127 | `chrome_cis_rule_2_23: false` |
| 2.29 | 114-123 | `chrome_cis_rule_2_29: false` |

## Run and check

| Goal | How |
|------|-----|
| Preview, change nothing | `--check --diff` |
| Apply | run the playbook without tags |
| Skip a rule | its switch, `chrome_cis_rule_<id>: false` |

Use switches, not `--tags`: every run rebuilds the policy file from your settings, so a run limited by tags writes
only the selected rules.

To check: open `chrome://policy` → **Reload policies** → every CIS policy shows Status **OK**. The run also ends with
a short report: browsers found, policy folders, number of policies, levels.

## License

MIT
