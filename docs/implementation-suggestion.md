# Implementation suggestions

## 1. Browser discovery

Branch: `discovery-logic`. The role finds Google Chrome and Chromium on each host and hardens every one it finds.
`chrome_cis_browser` is removed; `chrome_cis_install_browser` is only used when `chrome_cis_install: true`.

### Code

`tasks/prelim.yml`: optional install first, then one discovery.

```yaml
- name: "PRELIM | Install a browser (opt-in)"
  when: chrome_cis_install
  ansible.builtin.import_tasks:
    file: install.yml

- name: "PRELIM | AUDIT | Gather installed packages"
  ansible.builtin.package_facts:
    manager: rpm

- name: "PRELIM | AUDIT | Discover installed browsers"
  ansible.builtin.set_fact:
    chrome_cis_chrome_found: "{{ ansible_facts['packages'] | select('match', 'google-chrome-') | list | length > 0 }}"
    chrome_cis_chromium_found: "{{ 'chromium' in ansible_facts['packages'] }}"
```

`vars/main.yml`: policy folder of each browser found.

```yaml
chrome_cis_policy_dirs: >-
  {{ (['/etc/opt/chrome/policies/managed'] if chrome_cis_chrome_found else [])
   + (['/etc/chromium/policies/managed'] if chrome_cis_chromium_found else []) }}
```

`tasks/post.yml`: the same policy file into every folder found.

```yaml
- name: "POST | PATCH | Write {{ chrome_cis_policy_file }}"
  ansible.builtin.copy:
    dest: "{{ item }}/{{ chrome_cis_policy_file }}"
    content: "{{ chrome_cis_policies | to_nice_json }}\n"
    owner: root
    group: root
    mode: "0644"
  loop: "{{ chrome_cis_policy_dirs }}"
```

### Result

| Installed | Policy file written to |
|---|---|
| nothing (install off) | none: "nothing to harden" |
| Chrome (any channel) | `/etc/opt/chrome/policies/managed/` |
| Chromium | `/etc/chromium/policies/managed/` |
| Chrome + Chromium | both, in one run |

### Testing

## 2. Version gates for N/A policies

Branch: `version-policy-rule`. A CIS policy that Chrome removed is written only on browser versions that still
support it. Ranges come from `supported_on` in Chromium's policy definitions
(`evidence-files/chromium-supported-on-2026-10-13.txt`); Chrome and Chromium share version numbers.

16 N/A rules: **8 version-gated** (removed, but existed on Linux), **8 always off** (no Linux version has them).

| Group | Rules | Default |
|---|---|---|
| Version-gated | 1.10, 2.3.4, 2.3.5, 2.3.6, 2.7.1, 2.9.1, 2.23, 2.29 | `true`; the version decides |
| Windows only | 1.19, 2.8.2, 2.10.1, 2.30, 3.6, 3.13 | `false` |
| Google Update (not a Chrome policy) | 2.1.1, 2.1.2 | `false` |

### Code

`tasks/prelim.yml`: the discovery task also reads the version (lower one when both browsers are installed).

```yaml
    chrome_cis_browser_major: >-
      {{ ansible_facts['packages'] | dict2items | selectattr('key', 'match', 'google-chrome-|chromium$')
         | map(attribute='value.0.version') | map('regex_replace', '[.].*', '') | map('int') | min | default(0, true) }}
```

Each gated rule: one extra `when` line with its range.

```yaml
- name: "2.3.5 | PATCH | Ensure 'Block third-party storage partitioning for these origins' Is Configured"
  when:
    - chrome_cis_level_1
    - chrome_cis_rule_2_3_5
    - 113 <= chrome_cis_browser_major | int <= 145   # supported_on chrome.*:113-145
```

### Result

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

| Browser | Gated policies written |
|---|---|
| Chromium 128 / 133 | 2.3.4, 2.3.5, 2.3.6, 2.7.1, 2.9.1 |
| Chrome 143 | 2.3.4, 2.3.5, 2.7.1, 2.9.1 |
| Chrome 146 | 2.9.1 |
| Chrome 155 / Chromium 154 | none |

### Testing

