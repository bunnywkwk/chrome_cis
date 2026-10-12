# Test cases: chrome_cis

Six cases in two rounds. RHEL 8, 9 and 10 behave the same for Chrome (platform-notes D-1), so each VM runs a
different case and one playbook run tests three at once. Shared settings stay in `group_vars/rhel-os.yml`; the
case per VM is set in `sysconfig/host_vars/<host>.yml`. Roll all three VMs back to `pre-chrome` before each round.

Evidence: `docs/evidence-images/NN-<os>-<case>-<result>.png`.

## Round 1: the role installs the browser

| ID | VM | host_vars | Expected |
|----|----|-----------|----------|
| TC-1 | rhel8 | `chrome_cis_browser: chrome`, `chrome_cis_install: true`, `chrome_cis_version: ""` | newest `google-chrome-stable` installed; policy file `/etc/opt/chrome/policies/managed/chrome_cis.json` |
| TC-2 | rhel9 | `chrome_cis_browser: chrome`, `chrome_cis_install: true`, `chrome_cis_version: "143.0.7499.40"` | `rpm -q google-chrome-stable` = `143.0.7499.40-1`; policies applied |
| TC-3 | rhel10 | `chrome_cis_browser: chromium`, `chrome_cis_install: true` | EPEL enabled by the role, `chromium` installed; policy file `/etc/chromium/policies/managed/chrome_cis.json` |

## Round 2: browser already installed, or missing

Before the run, on the VM:

```bash
# rhel8: Chrome 144 by hand
sudo rpm --import https://dl.google.com/linux/linux_signing_key.pub
sudo dnf install -y https://dl.google.com/linux/chrome/rpm/stable/x86_64/google-chrome-stable-144.0.7559.132-1.x86_64.rpm

# rhel9: Chromium by hand
sudo dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm
sudo dnf install -y chromium
```

| ID | VM | host_vars | Expected |
|----|----|-----------|----------|
| TC-4 | rhel8 | `chrome_cis_browser: chrome`, `chrome_cis_install: false` | hardened; report `browser: {"google-chrome-stable": "144.0.7559.132"}`; version unchanged |
| TC-5 | rhel9 | `chrome_cis_browser: chromium`, `chrome_cis_install: false` | hardened; file in `/etc/chromium/policies/managed/` |
| TC-6 | rhel10 | `chrome_cis_browser: chrome`, `chrome_cis_install: false` | nothing installed: "No google-chrome-stable package ... nothing to harden", `failed=0` |

## Every case

1. `ansible-playbook playbooks/site.yml`: `failed=0`.
2. Rerun: `changed=0`.
3. In the browser: `chrome://policy` shows the CIS policies with status OK (N/A policies show "Error" unless switched
   off) and `chrome://management` shows "managed by your organization".

| ID | Run | Rerun | chrome://policy | Evidence |
|----|-----|-------|-----------------|----------|
| TC-1 | | | | |
| TC-2 | | | | |
| TC-3 | | | | |
| TC-4 | | | | |
| TC-5 | | | | |
| TC-6 | | | n/a | |
