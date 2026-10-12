# Session handoff: chrome_cis

**Read this first when you continue on another machine or in a new Claude session.** It says where the work stopped,
how we think about it, how Claude should work with you, and what to do next. Last update: **2026-10-13**.

New Claude session: say *"read `chrome_cis/docs/files/session-handoff.md` and `chrome_cis/docs/files/CLAUDE.md` first"*.

## 1. Where we are (2026-10-13)

Released role on **`main`** (PR #1, Lockdown layout). Mentor suggestions are built on feature branches, not merged
yet (CI will test them first):

| Branch | From | Holds | Status |
|--------|------|-------|--------|
| `main` | | the role, Lockdown layout, D1-D15 | stable |
| `staging` | | experiments (pipeline later) | = `main` |
| `discovery-logic` | `main` | **D16** browser discovery: no `chrome_cis_browser`; hardens every browser found | VM-tested on rhel9 (both browsers, one run) |
| `version-policy-rule` | `discovery-logic` | **D17** version gates for removed policies; Google Update 2.1.1/2.1.2 as tasks | container-tested; **VM test next** |

Standup summary of both: [../implementation-suggestion.md](../implementation-suggestion.md).

| Phase | Status |
|-------|--------|
| Benchmark, matrix, decisions D1-D17 | Done |
| Lockdown build, 120 rule tasks (118 + 2.1.1, 2.1.2) | Done; lint passes |
| D16 discovery | Done; rhel9 Chrome + Chromium in one run: `failed=0`, both files written |
| D17 version gates | Done; container gate test (10 versions) matches the ranges; VM: Chrome 143 shows 4 OK, Chrome 155 / Chromium 154 all Error |
| VM test of the gates | **Next:** [test-cases-version-gates.md](test-cases-version-gates.md) VG-1..VG-9 |
| README, compliance record, merge to `main` | After the VM test and CI |

## 2. Structure (branch `version-policy-rule`)

```
tasks/main.yml              prelim (always) -> section_1..5 (switches) -> post (always)
tasks/prelim.yml            ansible/OS checks -> install (opt-in, imports install.yml) -> package_facts
                            -> Discover installed browsers (chrome_found, chromium_found, browser_major)
                            -> stop if none found and install off -> start the policy set ({})
tasks/install.yml           chrome_cis_install_browser: chrome (Google key, repo, package / exact RPM) or chromium (EPEL)
tasks/section_<n>/cis_*.yml one task per CIS rule, benchmark order; 8 removed policies carry a version gate
tasks/post.yml              write chrome_cis.json into every policy folder found (loop), short report
vars/main.yml               chrome_cis_policy_dirs = folders of the browsers found
```

- **Discovery (D16):** `PRELIM | AUDIT | Discover installed browsers` after `package_facts`: any `google-chrome-*`
  package = Chrome, `chromium` = Chromium. Found = hardened (agreed, no per-browser switch). Install runs before
  discovery so one discovery also finds a browser installed in the same run. Install handles one browser per run.
- **Version gates (D17):** `chrome_cis_browser_major` = lowest major version of the browsers found. 8 rules get
  one `when` line, e.g. `113 <= chrome_cis_browser_major | int <= 145   # supported_on chrome.*:113-145`. Ranges
  from Chromium's policy definitions (`evidence-files/chromium-supported-on-2026-10-13.txt`); Chrome and Chromium
  share version numbers. `| int` is required (T-C6).
- **N/A rules (16):** 8 version-gated (`true`, tag `version_gated`): 1.10, 2.3.4, 2.3.5, 2.3.6, 2.7.1, 2.9.1, 2.23,
  2.29. 8 always off (`false`, tag `not_applicable`): Windows only 1.19, 2.8.2, 2.10.1, 2.30, 3.6, 3.13; Google
  Update 2.1.1, 2.1.2.
- **Every run starts the policy set empty** (prelim), so the file matches the settings; a `--tags` run writes only
  the selected rules (use it for focused tests only).
- **Site rules:** run only when the site value is set (`is not none` / `| length > 0`).

## 3. Test project `~/chrome_cis_test` (`github.com/bunnywkwk/chrome_cis_test`)

- `requirements.yml` → `version: <branch>`; reinstall with `ansible-galaxy role install -r requirements.yml --force`.
- `sysconfig/group_vars/rhel-os.yml`: site settings. `chrome_cis_browser` is now `chrome_cis_install_browser`.
  Delete the old N/A `false` lines before testing the gates.
- `sysconfig/host_vars/rhel9.yml`: `chrome_cis_install: false` (+ the 2.3.5 value for the gate tests).
- Evidence: `runs/*.log`, `docs/evidence/NN-...png`, `docs/na-policy-test.md` (all 16 N/A Error on Chrome 155 /
  Chromium 154).
- rhel9 now: Chrome **143.0.7499.40** + Chromium 154 (Chrome 155 profile backed up as
  `~/.config/google-chrome.155-backup`).

## 4. Next steps

1. Snapshot rhel9 `pre-gates`, run VG-1..VG-9 ([test-cases-version-gates.md](test-cases-version-gates.md)), fill the
   results table, fill "Testing" in `implementation-suggestion.md`.
2. CI pipeline (user), then merge `version-policy-rule` (contains `discovery-logic`) into `staging` / `main`.
3. README (CLAUDE.md "End-user experience"), compliance record, `test-results.md` index.

## 5. The mindset (why the role looks like this)

| Principle | In this role |
|-----------|--------------|
| **The benchmark is the source of truth** | only the 118 CIS rules; never extra Chrome policies; CIS IDs never renumbered |
| **Evidence, not assumption** | every "works on RHEL" / "N/A" claim is backed by Google's policy template for Chrome 155 (Ctrl+F search text per rule) **and** a VM test. Don't add new sources mid-way |
| **The benchmark is written for Windows** | PDF printed p. 8–9 (viewer p. 9–10): Group Policy / registry. On RHEL the same **policy names** go into a JSON file in `/etc/opt/chrome/policies/managed/` |
| **One task per rule (Lockdown)** | each CIS rule is its own task with name, toggle, level and tags; it adds its policy, `post.yml` writes one `chrome_cis.json` (D3 revised 2026-10-09) |
| **Full CIS by default, exceptions in `group_vars`** (2026-10-08) | L1 on, L2 off; the 12 risky rules are **on** with a `# WARNING:` saying what breaks; the site turns off what it can't accept (`chrome_cis_rule_4_7: false`), so `group_vars` = the compliance record's exception list. Applies to future roles too (CLAUDE.md principle 3). `mongodb8_cis` keeps the old "risky = false" |
| **Site values** | rules where CIS lets the organization decide (Manual): `null`/`[]` = report only; set = written (D6) |
| **All 118 accounted for** | N/A rules stay in the list with a reason and appear in every report |
| **Versions** | install can pin an exact version (D15); policies removed from Chrome are written only on versions that still have them (D17, ranges from Chromium's policy definitions) |
| **Don't over-engineer** | simple inputs only (one value or a flat list); no browser restart; no SELinux tasks |

## 6. How Claude works with you (keep this the same on every machine)

| You prefer | So Claude |
|------------|-----------|
| Learning by typing | you hand-type `tasks/` (and the computed vars); Claude gives code in a **guide or in chat**, block by block, each with a one-line **what** and **why**; never writes your task files unless you ask. If you say "I'll do it", Claude gives code + steps, no files |
| Plain words | simple English, tables, short answers; you have a Python background, so Python comparisons help |
| Snippet questions | when you paste a snippet, explain only that syntax, 3–8 lines with a tiny example |
| Honest answers | Claude says when something isn't verified, when it was wrong, and pushes back with a reason (e.g. risky defaults), then follows your decision |
| Simple options | one value or a flat list per option; complex per-item inputs → report-only + a documented hand fix |
| Reverts | when you say "revert", Claude reverts fully, no arguing |
| New change ideas | Claude explains the design and gives code first; edits files only after you say go (you create a branch per change) |
| Review | after you type a file, Claude reviews it against the tested code, fixes or lists typos, runs `yamllint` + `ansible-lint` |
| Evidence | you drop screenshots, say "check it"; Claude reads each image, checks the command is right, renames `NN-<os>-<scenario>-<result>.png`, indexes it in `test-results.md` |
| Docs | Claude writes all docs; every doc has what / why / evidence; decisions changed → marked "Revised <date>"; **no emojis** (Pass / Fail / N/A / Yes / No) |
| Standup | short entry per day in `docs/standup.md` |
| Safety | Claude never commits or pushes unless asked, names every commit it makes, and **never adds a `Co-Authored-By: Claude` line** (user decision 2026-10-09); never hardens your workstation (commands for it are given to you to run); never commits CIS PDFs/spreadsheets |
| Errors | every error gets a troubleshooting entry with the exact output, cause, fix, prevent |

## 7. `defaults/main.yml` and `vars/main.yml`

**`defaults/main.yml`:** install (`chrome_cis_install`, `chrome_cis_install_browser`, channel, version), levels,
sections, one `chrome_cis_rule_<id>` per rule (12 risky ones with `# WARNING:`, on), site values under their rule,
then "Removed from Chrome" (8 gated, `true`, range in the comment) and "Not applicable on RHEL" (8, `false`).
**`vars/main.yml`:** internal values only (supported OS, repo and EPEL URLs, policy file name,
`chrome_cis_policy_dirs`).

## 8. Facts to remember

| Fact | Where proven |
|------|--------------|
| Chrome 155.0.8059.39 on RHEL 8.10 / 9.7 / 10.2: same behaviour, no OS branches | platform-notes D-1..D-4 |
| The RPM doesn't create `policies/managed/`; the role does | D-2 |
| Installing by hand doesn't import Google's key (rpm lock) → role imports it first | T-C1 |
| "Chrome Enterprise" on Linux = the same RPM + policies; all channels in the same repo, one build each | platform-notes W-2 |
| All 101 applicable policies OK in `chrome://policy`; `ProxyMode` "Deprecated" but works | D-4 |
| 4.7 DoH "secure" without `DnsOverHttpsTemplates` breaks all browsing | user-view.md, 2026-10-08 |
| Some policies load only at Chrome start; the role never restarts Chrome | D11 |

**Still open (small):** desktop session type (X11/Wayland) for 4.1.1/4.12; beta/canary read the same policy folder
(expected); a policy Chrome removes after 2026-10-13 needs its range added (D17); README (batch 6); maybe make 1.9 a fixed rule instead of a site
value (only one CIS value); maybe add a DoH site variable next to 4.7 (CIS note) later.

## 9. Environment

| Item | Value |
|------|-------|
| VMs (Proxmox) | rhel8 `192.168.20.50`, rhel9 `192.168.20.30`, rhel10 `192.168.20.40`, "Server with GUI"; SSH user `frqadmin`, key `~/.ssh/id_ed25519_cis` |
| Snapshot | **`pre-chrome`** on all 3; revert before each clean test |
| Control node | ansible-core 2.16 venv `~/.ansible-2.16.1-env` ([control-node-setup](../../docs/control-node-setup.md)); **activate it first** (T-C2); no collections needed |
| Lint | `yamllint . && ansible-lint --offline` inside `chrome_cis/` (uv tools) |
| Benchmark files | `cis-pdf/`, `cis-spreadsheet/` at the repo root; **never commit them** |
| Standup notes | [docs/standup.md](../../docs/standup.md) |
| Container test | Rocky 8/9/10 with podman + `containers.podman` (test only); recipe below |

### Optional: Claude's container test

```bash
ansible-galaxy collection install containers.podman -p ~/chrome-cis-ctest/collections
for v in 8 9 10; do podman run -d --name cis$v docker.io/rockylinux/rockylinux:$v sleep infinity; done
cd ~/chrome-cis-ctest
cat > inv.ini <<'EOF'
[containers]
cis8 ansible_python_interpreter=/usr/libexec/platform-python
cis9 ansible_python_interpreter=/usr/bin/python3
cis10 ansible_python_interpreter=/usr/bin/python3
[containers:vars]
ansible_connection=containers.podman.podman
ansible_become=false
EOF
printf -- '---\n- name: Test chrome_cis\n  hosts: all\n  roles:\n    - role: chrome_cis\n' > site.yml
ANSIBLE_ROLES_PATH=~/ansible-cis ANSIBLE_COLLECTIONS_PATH=./collections \
  ansible-playbook -i inv.ini site.yml -e '{"chrome_cis_install": true}'      # then rerun: changed=0
podman rm -f cis8 cis9 cis10                                                   # clean up
```
