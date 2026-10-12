# Test cases: browser discovery and version gates

What: prove on a real VM that the role writes each removed policy only on browser versions inside its
`supported_on` range (D17), and that the browser accepts it (`chrome://policy` status OK).
Why: the container test (D17) proves which policies the role writes; only a real browser shows OK or Error.

Branch: `version-policy-rule`. VM: rhel9 (192.168.20.30), unless noted. Evidence goes to the test repo
(`~/chrome_cis_test`): logs in `runs/`, screenshots in `docs/evidence/NN-rhel9-vg-<browser><version>-<result>.png`.

## The 8 gated policies

| Rule | Policy | Versions |
|---|---|---|
| 1.10 | `CertificateTransparencyEnforcementDisabledForLegacyCas` | 67-127 |
| 2.3.4 | `DefaultThirdPartyStoragePartitioningSetting` | 113-145 |
| 2.3.5 | `ThirdPartyStoragePartitioningBlockedForOrigins` | 113-145 |
| 2.3.6 | `ExtensionManifestV2Availability` | 110-138 |
| 2.7.1 | `CloudPrintProxyEnabled` | 17-144 |
| 2.9.1 | `FirstPartySetsEnabled` | 113-152 |
| 2.23 | `EnforceLocalAnchorConstraintsEnabled` | 112-127 |
| 2.29 | `InsecureHashesInTLSHandshakesEnabled` | 114-123 |

## 0. Prepare (once)

1. Proxmox: snapshot rhel9 as **`pre-gates`** (Chrome 143 + Chromium 154, as it is now). Rollback = restore it.
2. Test project:
   ```bash
   cd ~/chrome_cis_test
   source ~/.ansible-2.16.1-env/bin/activate
   sed -i 's/^    version: .*/    version: version-policy-rule/' requirements.yml
   ansible-galaxy role install -r requirements.yml --force
   ```
3. `sysconfig/group_vars/rhel-os.yml`: delete the `chrome_cis_rule_<N/A>: false` lines (they would block the gates).
4. `sysconfig/host_vars/rhel9.yml`:
   ```yaml
   ---
   chrome_cis_install: false
   chrome_cis_third_party_storage_blocked_origins: ["https://example.com"]   # so 2.3.5 can be written
   ```
5. Helper on the workstation: shows the installed browsers and the gated policies in each policy file.
   ```bash
   gated() { ssh -i ~/.ssh/id_ed25519_cis frqadmin@192.168.20.30 'rpm -q google-chrome-stable chromium; for f in /etc/opt/chrome/policies/managed/chrome_cis.json /etc/chromium/policies/managed/chrome_cis.json; do [ -f $f ] && echo "== $f" && sudo grep -oE "\"(CertificateTransparencyEnforcementDisabledForLegacyCas|DefaultThirdPartyStoragePartitioningSetting|ThirdPartyStoragePartitioningBlockedForOrigins|ExtensionManifestV2Availability|CloudPrintProxyEnabled|FirstPartySetsEnabled|EnforceLocalAnchorConstraintsEnabled|InsecureHashesInTLSHandshakesEnabled)\"" $f; done; true'; }
   ```

## Every case

1. Swap the browser (commands in the case) on rhel9.
2. Remove the policy folder of a browser you uninstalled, so an old file does not confuse the check:
   `sudo rm -rf /etc/opt/chrome/policies` (Chrome removed) or `sudo rm -rf /etc/chromium/policies` (Chromium removed).
3. Run and check (workstation):
   ```bash
   ansible-playbook playbooks/site.yml --limit rhel9 | tee runs/rhel9-vg-<case>.log    # failed=0
   gated
   ```
4. On the rhel9 desktop: an older browser cannot open a newer profile (it starts, then stops). Before opening a
   downgraded browser, move its profile aside:
   `mv ~/.config/google-chrome ~/.config/google-chrome.bak-$(date +%s)` or
   `mv ~/.config/chromium ~/.config/chromium.bak-$(date +%s)`.
5. `chrome://policy` → **Reload policies** → screenshot. Expected: the gated policies of the case show **OK**; no
   other policy shows Error.

Shortcuts for the swaps (on rhel9):

```bash
G=https://dl.google.com/linux/chrome/rpm/stable/x86_64
K=https://kojipkgs.fedoraproject.org/packages/chromium
chrome() { sudo dnf remove -y google-chrome-stable; sudo dnf install -y $G/google-chrome-stable-$1-1.x86_64.rpm; }
chromium() { sudo dnf remove -y chromium chromium-common; sudo dnf install -y $K/$1/1.el9/x86_64/chromium-$1-1.el9.x86_64.rpm $K/$1/1.el9/x86_64/chromium-common-$1-1.el9.x86_64.rpm; }
```

Chrome RPMs: Google keeps about one year (oldest 143.0.7499.40 on 2026-10-09, W-5). Old Chromium for EL9: Fedora
Koji (the EPEL build system) still has 128.0.6613.119 and 149-155 (checked 2026-10-13). An old build may need old
libraries; if `dnf` reports missing dependencies, note it and skip the case.

## Cases

| ID | Browser(s) | Swap (on rhel9) | Gated policies expected | In the browser |
|---|---|---|---|---|
| VG-1 | Chrome 143 + Chromium 154 | none (state of `pre-gates`) | 2.3.4, 2.3.5, 2.7.1, 2.9.1 in both files (lower version counts) | Chrome: 4 OK. Chromium: the same 4 Error (expected) |
| VG-2 | Chrome 143 | `sudo dnf remove -y chromium chromium-common`; `sudo rm -rf /etc/chromium/policies` | 2.3.4, 2.3.5, 2.7.1, 2.9.1 | 4 OK, no Error |
| VG-3 | Chrome 145 | `chrome 145.0.7632.159` | 2.3.4, 2.3.5, 2.9.1 (2.7.1 gone: ends at 144) | 3 OK, no Error |
| VG-4 | Chrome 146 | `chrome 146.0.7680.177` | 2.9.1 (2.3.4, 2.3.5 gone: end at 145) | 1 OK, no Error |
| VG-5 | Chrome 155 | `chrome 155.0.8059.39` | none | no Error |
| VG-6 | Chromium 128 | `sudo dnf remove -y google-chrome-stable`; `sudo rm -rf /etc/opt/chrome/policies`; `chromium 128.0.6613.119` | 2.3.4, 2.3.5, 2.3.6, 2.7.1, 2.9.1 (1.10, 2.23 gone: end at 127) | 5 OK, no Error |
| VG-7 | Chromium 152 | `chromium 152.0.7977.82` | 2.9.1 | 1 OK, no Error |
| VG-8 | Chromium 153 | `chromium 153.0.8010.52` | none (2.9.1 ends at 152) | no Error |
| VG-9 | Chromium 133 on **rhel8** | EPEL 8 (frozen at 133): `sudo dnf install -y chromium` after the EPEL 8 key + `epel-release-latest-8` | 2.3.4, 2.3.5, 2.3.6, 2.7.1, 2.9.1 | 5 OK, no Error |

Every case: rerun → `changed=0`.

## Rollback

Restore the `pre-gates` snapshot (rhel9) and `pre-chrome` (rhel8). Without snapshots: `chrome 155.0.8059.39` and
`sudo dnf install -y chromium` (EPEL current), then a normal run.

## Results

| ID | Run | Rerun | Gated written (`gated`) | chrome://policy | Evidence |
|---|---|---|---|---|---|
| VG-1 | | | | | |
| VG-2 | | | | | |
| VG-3 | | | | | |
| VG-4 | | | | | |
| VG-5 | | | | | |
| VG-6 | | | | | |
| VG-7 | | | | | |
| VG-8 | | | | | |
| VG-9 | | | | | |
