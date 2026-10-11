# chrome_cis

Ansible role that hardens **Google Chrome** (or **Chromium**) on **RHEL 8, 9 and 10** to the
**CIS Google Chrome Benchmark v3.0.0**, in the [ansible-lockdown](https://github.com/ansible-lockdown) style.
Every task is one CIS recommendation. The benchmark is written for Windows Group Policy; on Linux Chrome reads the
same policy names from a JSON file, so the role writes one managed policy file:
`/etc/opt/chrome/policies/managed/chrome_cis.json` (Chromium: `/etc/chromium/policies/managed/`).
Platforms: RHEL and compatible (Rocky, Alma, Oracle), x86_64, with a desktop (Chrome is a desktop app).

## Requirements

- ansible-core **2.16.x** on the control node: the last version that can manage RHEL 8 (Python 3.6). With a newer
  ansible-core, RHEL 8 fails at "Gather installed packages" (docs/troubleshooting.md T-C2).
- No collections needed (only `ansible.builtin`).
- Chrome or Chromium installed, or `chrome_cis_install: true`:
  - Chrome: the role adds Google's signing key and repo, then installs `google-chrome-<channel>`.
  - Chromium: the role enables EPEL (`epel-release`), then installs `chromium`.

## Quick start

```bash
cat > site.yml <<'EOF'
- hosts: workstations
  become: true
  roles:
    - chrome_cis
EOF
ansible-playbook -i inventory site.yml --check --diff   # preview the policy file, change nothing
ansible-playbook -i inventory site.yml                  # write it; a second run reports changed=0
```

Then open `chrome://policy` in the browser and press **Reload policies** (or restart Chrome). The role never closes
Chrome: users would lose their work, so some settings apply at the next Chrome start.

## What runs by default

| Setting | Default | Meaning |
|---------|---------|---------|
| `chrome_cis_level_1` | `true` | CIS Level 1: safe for most organizations |
| `chrome_cis_level_2` | `false` | CIS Level 2: stricter, limits features. Set **both** to `true` for Level 2 |
| `chrome_cis_section1` .. `chrome_cis_section5` | `true` | the 5 benchmark sections |
| `chrome_cis_rule_<id>` | `true` | one switch per CIS rule, e.g. `chrome_cis_rule_4_7` for 4.7 |
| site values | `null` / `[]` | rules where CIS lets your organization decide: skipped until you set a value |

**The role applies the full benchmark, including the rules that change what users can do.** You list what your
organization does **not** accept in `group_vars`. That list is also your compliance exception list.

## Tailor it in `group_vars` (never edit the tasks)

Write switches unquoted (`true` / `false`). Example `group_vars/workstations.yml`:

```yaml
# Browser
chrome_cis_browser: chrome          # chrome or chromium
chrome_cis_install: true            # install it if missing
chrome_cis_channel: stable          # Chrome only: stable, beta, unstable, canary
chrome_cis_version: ""              # "" = newest; or an exact version, e.g. "143.0.7499.40"

# Level 2 as well
chrome_cis_level_2: true

# Exceptions: rules this organization does not apply (and why)
chrome_cis_rule_4_7: false          # no DNS-over-HTTPS server
chrome_cis_rule_4_4: false          # web meetings need the microphone
chrome_cis_rule_4_5: false          # web meetings need the camera

# Site values (CIS: "organization defined")
chrome_cis_safe_browsing_protection_level: 1        # 1.2.2
chrome_cis_http_allowlist: ["intranet.example.com"] # 2.27
chrome_cis_extension_allowlist: ["<extension id>"]  # 2.3.3: extensions still allowed
```

Every variable, with a one-line explanation, is in [`defaults/main.yml`](defaults/main.yml).

## Rules that change what users can do (on by default)

Turn one off with `chrome_cis_rule_<id>: false` when your users need what it blocks.

| Rule | What users notice | Switch |
|------|-------------------|--------|
| 2.2.3 | WebUSB blocked: some USB devices and security keys stop working in web apps | `chrome_cis_rule_2_2_3` |
| 2.3.3 | **every** extension removed, except `chrome_cis_extension_allowlist` | `chrome_cis_rule_2_3_3` |
| 2.4.1 | only NTLM/Negotiate login: sites using Basic/Digest login stop working | `chrome_cis_rule_2_4_1` |
| 2.5.1 | extensions can't talk to local programs (password managers, GNOME extensions), except `chrome_cis_native_messaging_allowlist` | `chrome_cis_rule_2_5_1` |
| 2.12 | no "proceed anyway" on certificate errors: internal sites with bad certificates become unreachable | `chrome_cis_rule_2_12` |
| 2.18 | certificates from local CAs fail if the OCSP/CRL server is down | `chrome_cis_rule_2_18` |
| 3.1.1 | cookies deleted when Chrome closes: users are logged out of every site | `chrome_cis_rule_3_1_1` |
| 4.1.1 (+ 4.12) | no screen sharing in web meetings | `chrome_cis_rule_4_1_1` |
| 4.3 | no file upload or "save as" dialogs | `chrome_cis_rule_4_3` |
| 4.4 | no microphone for websites | `chrome_cis_rule_4_4` |
| 4.5 | no camera for websites | `chrome_cis_rule_4_5` |
| 4.7 | DNS-over-HTTPS only: **every site fails** without a DoH server, and company DNS filtering is bypassed | `chrome_cis_rule_4_7` |

What each one looks like in the browser: [`docs/user-view.md`](docs/user-view.md).

## Site values: rules your organization decides

The rule is skipped (nothing written) until you set its value.

| Rule | Variable | Example |
|------|----------|---------|
| 1.2.2 | `chrome_cis_safe_browsing_protection_level` | `1` standard, `2` enhanced (sends more data to Google) |
| 1.9 | `chrome_cis_variations` | `0` (the CIS value) |
| 2.3.3 | `chrome_cis_extension_allowlist` | `["<extension id>"]` |
| 2.5.1 | `chrome_cis_native_messaging_allowlist` | `["<host name>"]` |
| 2.6.1 | `chrome_cis_password_manager_enabled` | `true` or `false` |
| 2.8.1 | `chrome_cis_remote_access_connections` | `false` (CIS), `true` if Chrome Remote Desktop is approved |
| 2.8.3 | `chrome_cis_remote_access_client_domains` | `["example.com"]` |
| 2.17 | `chrome_cis_proxy_mode` | `direct` or `system` |
| 2.27 | `chrome_cis_http_allowlist` | `["intranet.example.com"]` |
| 4.2.3 / 4.2.4 | `chrome_cis_clipboard_allowed_urls` / `_blocked_urls` | `["https://app.example.com"]` / `["[*.]example.org"]` |
| 4.2.7 / 4.2.8 | `chrome_cis_window_management_allowed_urls` / `_blocked_urls` | same format (Level 2) |

## Policies that show "Error" in `chrome://policy`

14 CIS rules use policies that **Chrome on Linux no longer knows**: Google removed them after CIS tested Chrome 120,
or they only exist on Windows. Chrome ignores them and lists them as **Error** ("Unknown policy") in
`chrome://policy`. They don't harm anything, but the page looks broken.

They are **on by default**, so you can see for yourself what your Chrome version supports. If `chrome://policy` shows
one as Error on your version, switch it off:

| Rule | Policy | Why it errors on Chrome 155 (Linux) | Switch off |
|------|--------|-------------------------------------|------------|
| 1.10 | `CertificateTransparencyEnforcementDisabledForLegacyCas` | removed from Chrome | `chrome_cis_rule_1_10: false` |
| 1.19 | `ThirdPartyBlockingEnabled` | removed (was Windows only) | `chrome_cis_rule_1_19: false` |
| 2.3.4 | `DefaultThirdPartyStoragePartitioningSetting` (L2) | removed from Chrome | `chrome_cis_rule_2_3_4: false` |
| 2.3.5 | `ThirdPartyStoragePartitioningBlockedForOrigins` | removed from Chrome | `chrome_cis_rule_2_3_5: false` |
| 2.3.6 | `ExtensionManifestV2Availability` (L2) | removed (Manifest V2 retired) | `chrome_cis_rule_2_3_6: false` |
| 2.7.1 | `CloudPrintProxyEnabled` | removed (Cloud Print retired) | `chrome_cis_rule_2_7_1: false` |
| 2.8.2 | `RemoteAccessHostAllowUiAccessForRemoteAssistance` | Windows only | `chrome_cis_rule_2_8_2: false` |
| 2.9.1 | `FirstPartySetsEnabled` | removed from Chrome | `chrome_cis_rule_2_9_1: false` |
| 2.10.1 | `CloudAPAuthEnabled` | Windows only | `chrome_cis_rule_2_10_1: false` |
| 2.23 | `EnforceLocalAnchorConstraintsEnabled` (L2) | removed from Chrome | `chrome_cis_rule_2_23: false` |
| 2.29 | `InsecureHashesInTLSHandshakesEnabled` | removed from Chrome | `chrome_cis_rule_2_29: false` |
| 2.30 | `RendererAppContainerEnabled` | Windows only | `chrome_cis_rule_2_30: false` |
| 3.6 | `ChromeCleanupReportingEnabled` | removed (was Windows only) | `chrome_cis_rule_3_6: false` |
| 3.13 | `SafeBrowsingForTrustedSourcesEnabled` | Windows only | `chrome_cis_rule_3_13: false` |

**How to check your own version:**
1. Run the role with these rules on (the default).
2. Open `chrome://policy`, press **Reload policies**, sort by **Status**.
3. Every policy marked **Error** is unknown to your Chrome version: add its `chrome_cis_rule_<id>: false` to
   `group_vars` and run the role again. The policy disappears from the file.

Older Chrome versions or Chromium may still support some of them; that's why they are switches and not removed.
Proof for each one (Google's policy list for Chrome 155 + VM test): [`docs/cis-requirements.md`](docs/cis-requirements.md),
"Evidence for the 16 not-applicable rules". Two more N/A rules (2.1.1, 2.1.2, Google Update) have no Linux policy at
all and are only listed there.

## Preview and run

| Goal | Command |
|------|---------|
| Preview the policy file, change nothing | `--check --diff` |
| Apply | run the playbook without tags |

Every run rebuilds the policy file from your settings, so a rule you switch off is removed from it. Choose rules with
their switches, not with `--tags`: a run limited by tags writes only the selected rules' policies.

## Check the result

```bash
cat /etc/opt/chrome/policies/managed/chrome_cis.json     # what the role wrote (Chromium: /etc/chromium/...)
```

In the browser: `chrome://policy` → **Reload policies** → every CIS policy is listed with Status **OK** (except the
N/A ones you kept on). The run ends with a short report: browser and version, policy file, number of policies, levels.

## Layout

`tasks/main.yml` → `prelim.yml` (checks, browser detection) → `install.yml` (opt-in) → `section_1..5/` (one task per
CIS rule; each adds its policy) → `post.yml` (writes the one policy file). Design decisions, platform facts and test
evidence: [`docs/`](docs/).

## License

MIT
