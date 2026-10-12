# Implementation suggestion: browser discovery

Branch: `discovery-logic`. The role finds Google Chrome and Chromium on each host and hardens every one it finds.
`chrome_cis_browser` is removed; `chrome_cis_install_browser` is only used when `chrome_cis_install: true`.

## Code

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

## Result

| Installed | Policy file written to |
|---|---|
| nothing (install off) | none: "nothing to harden" |
| Chrome (any channel) | `/etc/opt/chrome/policies/managed/` |
| Chromium | `/etc/chromium/policies/managed/` |
| Chrome + Chromium | both, in one run |

## Testing

