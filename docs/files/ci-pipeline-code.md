# CI pipeline code in this role

**What:** the files that make up the CI pipeline in this repository, with what each part does and why.
**Why:** user direction 2026-10-12: the pipeline (Jenkins + Molecule on rootless Podman) gates merges into `staging`
and `main`; this page is the reference for reading, changing or copying it to another role.
**Related (in the `ansible-cis` workspace, not in this repo):** `docs/local-laptop/` (laptop setup, overview, results)
and `docs/Proxmox-Environment/` (lab server, agent host).

| File | Does |
|---|---|
| `Jenkinsfile` | The pipeline: which agent, two stages, reports |
| `molecule/default/molecule.yml` | The test scenario: three containers, driver, role settings, step order |
| `molecule/default/requirements.yml` | Ansible collection Molecule needs to drive Podman |
| `molecule/default/prepare.yml` | Puts Chromium on el10 before the role runs |
| `molecule/default/converge.yml` | Runs the role from this checkout |
| `molecule/default/verify.yml` | Read-only compliance check, one result per CIS rule |
| `molecule/default/templates/junit.xml.j2` | Turns the results into a JUnit report for Jenkins' Tests tab |
| `molecule/default/vars/expected-policies.yml` | What "compliant" means: 107 rules with their expected values |
| `.gitignore` | `reports/` (build output, never committed) |

What the agent must provide (set up by `docs/Proxmox-Environment/agent-host-setup.md`): Jenkins label `molecule`, `/opt/lint` (ansible-lint
26.9.0, yamllint 1.38.0), `/opt/molecule` (ansible-core 2.16.19, molecule 26.9.0, molecule-plugins[podman] 26.9.28),
the `containers.podman` collection, rootless Podman for the `jenkins` user.

---

## Jenkinsfile

```groovy
// Trusted pipeline for chrome_cis: static analysis, then molecule (create -> prepare -> converge -> idempotence
// -> check -> verify -> destroy) on Rocky 8/9/10 containers. GitHub Branch Source reports the result as a commit status;
// the GitHub ruleset on staging and main requires it.
pipeline {
  // Dedicated agent VM with the tools installed: /opt/lint (ansible-lint, yamllint), /opt/molecule (ansible-core 2.16,
  // molecule, podman driver) and rootless podman for the test containers.
  agent { label 'molecule' }

  // A push builds with the first choice (all). "Build with Parameters" picks one OS for a faster run.
  parameters {
    choice(name: 'OS', choices: ['all', 'el8', 'el9', 'el10'], description: 'OS containers to test')
  }

  options {
    timestamps()
    timeout(time: 60, unit: 'MINUTES')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '100', artifactNumToKeepStr: '100'))
  }

  environment {
    MOLECULE_REPORT_DIR = "${WORKSPACE}/reports"
    // Unique container names per build (molecule.yml uses it); only [a-z0-9-] for container names.
    MOLECULE_RUN_ID = "${env.BUILD_TAG.toLowerCase().replaceAll('[^a-z0-9-]', '-')}"
    // Plain console log, no colour codes
    ANSIBLE_FORCE_COLOR = '0'
    PY_COLORS = '0'
  }

  stages {
    stage('Static analysis') {
      steps {
        sh '''
          mkdir -p reports
          . /opt/lint/bin/activate
          yamllint -f parsable .
          ansible-lint --sarif-file reports/ansible-lint.sarif
        '''
      }
    }

    stage('Compliance test') {
      steps {
        sh '''
          . /opt/molecule/bin/activate
          ansible --version | head -1
          # The first build of a new branch has no parameters yet: OS is empty there, so empty = all
          if [ "${OS:-all}" = all ]; then
            molecule test
          else
            molecule test --platform-name "chrome-cis-$OS-$MOLECULE_RUN_ID"
          fi
        '''
      }
    }
  }

  post {
    always {
      // molecule test destroys its containers; this covers a build killed by the timeout or an abort.
      sh '. /opt/molecule/bin/activate && molecule destroy || true'
      junit allowEmptyResults: true, testResults: 'reports/cis-verify-*.xml'
      archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true, fingerprint: true
    }
  }
}
```

| Block | Does | Why |
|---|---|---|
| `agent { label 'molecule' }` | Runs the whole build on an agent with that label | Builds never run on the controller; any agent with the label and the tools can take the build |
| `parameters { choice OS }` | `all` (default), `el8`, `el9`, `el10` | A push tests all hosts; *Build with Parameters* tests one host for a faster run while fixing something |
| `timestamps()` | Time on every console line | Shows which step is slow |
| `timeout(60 min)` | Kills a hung build | A stuck `dnf` or container must not block the single executor forever |
| `disableConcurrentBuilds()` | One build per branch at a time | Two builds of one branch would fight over the same workspace |
| `buildDiscarder(100)` | Keeps the last 100 builds and their artifacts | History for evidence without filling the disk |
| `MOLECULE_REPORT_DIR` | Tells `verify.yml` where to write reports | Reports land in the workspace, where Jenkins collects them |
| `MOLECULE_RUN_ID` | Build tag in lower case, only `[a-z0-9-]` | Unique container names per build (`molecule.yml` uses it), valid for Podman |
| `ANSIBLE_FORCE_COLOR`, `PY_COLORS` = 0 | No colour codes | Readable console log |
| Stage *Static analysis* | Activates `/opt/lint`, runs `yamllint` and `ansible-lint` (writes a SARIF report) | Cheap gate first: seconds, no containers |
| Stage *Compliance test* | Activates `/opt/molecule`, prints the ansible version, runs `molecule test` for all hosts or one | The real test; the version line in the log records which ansible-core tested this commit |
| `${OS:-all}` | Empty `OS` counts as `all` | The first build of a new branch or pull request has no parameters yet (T-C10) |
| `post { always { ... } }` | `molecule destroy`, then JUnit and artifacts | Runs after success, failure, abort or timeout: containers are removed and evidence is kept either way |
| `junit ... cis-verify-*.xml` | Fills the Tests tab, one test per rule per host | A failure names the rule ID |
| `archiveArtifacts reports/**` | Keeps JUnit files, each host's policy file, the lint report, fingerprinted | Audit trail per build |

---

## molecule/default/molecule.yml

```yaml
---
# CI compliance test, one browser case per host:
#   el8   Chromium from EPEL 8 (frozen at 133): gated policies inside their range are written
#   el9   Google Chrome 143 (an old RPM Google still serves): a different set of gated policies
#   el10  newest Chrome + newest Chromium: two policy folders, no gated policy
# Rootless podman on the agent VM runs the containers.
# Containers prove the policy files only; the browser reading them (chrome://policy) stays a VM test.
dependency:
  name: galaxy
  options:
    requirements-file: molecule/default/requirements.yml
driver:
  name: podman
platforms:
  # Names carry the CI run id so two branches building at once do not share containers.
  - name: "chrome-cis-el8-${MOLECULE_RUN_ID:-local}"
    hostname: el8
    image: docker.io/rockylinux/rockylinux:8
    pre_build_image: true
    groups:
      - chromium
  - name: "chrome-cis-el9-${MOLECULE_RUN_ID:-local}"
    hostname: el9
    image: docker.io/rockylinux/rockylinux:9
    pre_build_image: true
    groups:
      - chrome_old
  - name: "chrome-cis-el10-${MOLECULE_RUN_ID:-local}"
    hostname: el10
    image: docker.io/rockylinux/rockylinux:10
    pre_build_image: true
    groups:
      - chrome_and_chromium
provisioner:
  name: ansible
  inventory:
    group_vars:
      all:
        chrome_cis_install: true
        chrome_cis_level_2: true
        # 2.3.5 site value, so its version gate is tested too
        chrome_cis_third_party_storage_blocked_origins: ["https://example.com"]
      chromium:
        chrome_cis_install_browser: chromium
      chrome_old:
        # Oldest Chrome RPM on Google's server on 2026-10-09; Google drops RPMs about a year after release
        chrome_cis_version: "143.0.7499.40"
verifier:
  name: ansible
scenario:
  test_sequence:
    - dependency
    - destroy
    - syntax
    - create
    - prepare
    - converge
    - idempotence
    - check
    - verify
    - destroy
```

| Block | Does | Why |
|---|---|---|
| `dependency: galaxy` | Installs the collections in `requirements.yml` before the test | A local run on a fresh machine gets `containers.podman` too |
| `driver: podman` | Molecule creates the containers with Podman | Rootless, ships with RHEL, no daemon |
| `platforms` | Three Rocky Linux containers, names with `MOLECULE_RUN_ID` | One container per RHEL major version; `-local` when run by hand |
| `pre_build_image: true` | Uses the image as it is | Nothing to build; the role installs what it needs |
| `groups` | `chromium`, `chrome_old`, `chrome_and_chromium` | One browser case per host, set by the group variables below |
| `group_vars: all` | Install on, Level 1 + 2, 2.3.5 site value set | Tests every rule, the install path, and the 2.3.5 version gate |
| `chromium` | `chrome_cis_install_browser: chromium` | el8 gets Chromium 133 from EPEL 8, inside several gate ranges |
| `chrome_old` | `chrome_cis_version: 143.0.7499.40` | el9 gets a real old Chrome build: a different set of gated policies |
| `test_sequence` | dependency, destroy, syntax, create, prepare, converge, idempotence, check, verify, destroy | Starts clean, applies, proves a second run changes nothing and check mode works, verifies, cleans up |

---

## molecule/default/requirements.yml

```yaml
---
# Collection molecule's podman driver needs to drive the containers (the role itself uses ansible.builtin only).
# Same pin as on the agent VM (ansible-cis docs/Proxmox-Environment/agent-host-setup.md); the driver needs >= 1.8.1.
collections:
  - name: containers.podman
    version: ">=1.8.1,<2.0.0"
```

The podman driver's create and destroy playbooks use modules from `containers.podman`; the role itself needs no
collection.

---

## molecule/default/prepare.yml

```yaml
---
# el10 gets Chromium before the role runs; the role then installs Chrome and must harden both.
- name: Prepare | Chromium next to Chrome
  hosts: chrome_and_chromium
  gather_facts: true
  tasks:
    - name: Prepare | Install Chromium with the role's own install tasks
      ansible.builtin.include_role:
        name: "{{ lookup('ansible.builtin.env', 'MOLECULE_PROJECT_DIRECTORY') }}"
        tasks_from: install.yml
      vars:
        chrome_cis_install_browser: chromium
```

Runs once after *create*, on el10 only. It installs Chromium with the role's own `install.yml`, so the role then
installs Chrome on top and must find and harden both browsers (two policy folders). Prepare is not repeated in the
idempotence run.

---

## molecule/default/converge.yml

```yaml
---
- name: Converge | Apply chrome_cis
  hosts: all
  gather_facts: true
  tasks:
    - name: Converge | Run the role from this checkout
      ansible.builtin.include_role:
        # Full path, not a name: molecule 26 does not put the role's parent folder on the roles path
        name: "{{ lookup('ansible.builtin.env', 'MOLECULE_PROJECT_DIRECTORY') }}"
```

Applies the role from the checked-out repository. The full path is used because Molecule 26 does not put the role's
parent folder on the roles path (T-C7). Facts are gathered because the role needs the OS version.

---

## molecule/default/verify.yml

```yaml
---
# Read-only compliance check: one JUnit test case per CIS rule, plus the policy files and browser versions as evidence.
- name: Verify | CIS Google Chrome Benchmark v3.0.0 policy values
  hosts: all
  gather_facts: false
  vars_files:
    - vars/expected-policies.yml
  vars:
    policy_file_name: chrome_cis.json
    report_dir: >-
      {{ lookup('ansible.builtin.env', 'MOLECULE_REPORT_DIR')
         | default(lookup('ansible.builtin.env', 'MOLECULE_EPHEMERAL_DIRECTORY'), true) }}
  tasks:
    - name: Verify | Read installed packages
      ansible.builtin.package_facts:
        manager: rpm

    # Any Chrome channel counts as Chrome; chromium$ skips chromium-common and chromium-headless
    - name: Verify | Find the installed browsers and their policy folders
      ansible.builtin.set_fact:
        verify_browsers: >-
          {{ ansible_facts['packages'] | dict2items | selectattr('key', 'match', 'google-chrome-|chromium$')
             | map(attribute='value.0') | items2dict(key_name='name', value_name='version') }}
        verify_folders: >-
          {{ (['/etc/opt/chrome/policies/managed'] if ansible_facts['packages'] | select('match', 'google-chrome-') | list else [])
           + (['/etc/chromium/policies/managed'] if 'chromium' in ansible_facts['packages'] else []) }}

    - name: Verify | Lowest browser major version (the one the gates use)
      ansible.builtin.set_fact:
        verify_major: "{{ verify_browsers.values() | map('regex_replace', '[.].*', '') | map('int') | min }}"

    - name: Verify | The browsers this host should have
      ansible.builtin.assert:
        that:
          - verify_folders | length == (2 if 'chrome_and_chromium' in group_names else 1)
        fail_msg: "Found {{ verify_browsers }}; host groups {{ group_names }}."

    - name: Verify | Stat the policy files
      ansible.builtin.stat:
        path: "{{ item }}/{{ policy_file_name }}"
      loop: "{{ verify_folders }}"
      register: verify_stats

    - name: Verify | Policy files exist, root-owned, 0644
      ansible.builtin.assert:
        that:
          - item.stat.exists
          - item.stat.pw_name == 'root'
          - item.stat.gr_name == 'root'
          - item.stat.mode == '0644'
        fail_msg: "{{ item.item }}/{{ policy_file_name }} is missing or has the wrong owner/mode."
      loop: "{{ verify_stats.results }}"
      loop_control:
        label: "{{ item.item }}"

    - name: Verify | Read the policy files
      ansible.builtin.slurp:
        src: "{{ item }}/{{ policy_file_name }}"
      loop: "{{ verify_folders }}"
      register: verify_raw

    - name: Verify | Parse the policy files
      ansible.builtin.set_fact:
        verify_contents: "{{ verify_raw.results | map(attribute='content') | map('b64decode') | list }}"
        verify_policies: "{{ verify_raw.results[0].content | b64decode | from_json }}"

    - name: Verify | Same file in every policy folder
      ansible.builtin.assert:
        that:
          - verify_contents | unique | length == 1
        fail_msg: "The policy files in {{ verify_folders | join(' and ') }} differ."

    - name: Verify | Decide per rule whether its policy belongs on this browser version
      ansible.builtin.set_fact:
        verify_expected: >-
          {{ verify_expected | default([])
             + [item | combine({'applies': not (item.off_on_rhel | default(false))
                                           and (item.versions is not defined
                                                or item.versions[0] <= verify_major | int <= item.versions[1])})] }}
      loop: "{{ chrome_cis_expected }}"
      loop_control:
        label: "{{ item.rule }} {{ item.policy }}"

    # Read the value inline: a task var would turn numbers into strings on ansible-core 2.16 (1 != "1").
    - name: Verify | Compare every rule with its benchmark value (ABSENT when it does not apply)
      ansible.builtin.set_fact:
        verify_results: >-
          {{ verify_results | default([])
             + [item | combine({'expected': item.value if item.applies else 'ABSENT',
                                'actual': verify_policies[item.policy] | default('ABSENT'),
                                'pass': (verify_policies[item.policy] | default('ABSENT'))
                                        == (item.value if item.applies else 'ABSENT')})] }}
      loop: "{{ verify_expected }}"
      loop_control:
        label: "{{ item.rule }} {{ item.policy }}"

    - name: Verify | Find policies the benchmark does not ask for
      ansible.builtin.set_fact:
        verify_unexpected: "{{ verify_policies.keys() | difference(chrome_cis_expected | map(attribute='policy')) }}"

    - name: Verify | Save the evidence on the controller
      delegate_to: localhost
      block:
        - name: Verify | Save the evidence on the controller | Report folder
          ansible.builtin.file:
            path: "{{ report_dir }}"
            state: directory
            mode: "0755"

        - name: Verify | Save the evidence on the controller | JUnit report
          ansible.builtin.template:
            src: junit.xml.j2
            dest: "{{ report_dir }}/cis-verify-{{ inventory_hostname }}.xml"
            mode: "0644"

        - name: Verify | Save the evidence on the controller | Policy file as written
          ansible.builtin.copy:
            content: "{{ verify_contents[0] }}"
            dest: "{{ report_dir }}/chrome_cis-{{ inventory_hostname }}.json"
            mode: "0644"

    - name: Verify | All rules compliant, no unexpected policies
      ansible.builtin.assert:
        that:
          - verify_results | rejectattr('pass') | list | length == 0
          - verify_unexpected | length == 0
        fail_msg:
          - "Failed rules: {{ verify_results | rejectattr('pass') | map(attribute='rule') | join(', ') }}"
          - "Unexpected policies: {{ verify_unexpected | join(', ') }}"
        success_msg: >-
          {{ verify_results | length }} rules compliant on {{ inventory_hostname }}
          ({{ verify_browsers.items() | map('join', ' ') | join(', ') }}; gates use version {{ verify_major }}).
```

| Task | Does | Why |
|---|---|---|
| Read installed packages; find browsers | Lists `google-chrome-*` and `chromium` with versions; derives the policy folders | The test discovers on its own instead of trusting the role's discovery |
| Lowest browser major version | E.g. 133 on el8 | The same rule the role uses to decide gated policies: the oldest browser on the host counts |
| The browsers this host should have | el10 must have two folders, the others one | Catches a failed Chromium prepare that would otherwise pass silently |
| Stat / assert policy files | Every folder has `chrome_cis.json`, root:root, 0644 | Wrong owner or mode would let a user change the policy |
| Read / parse; same file everywhere | All folders hold identical content | Chrome and Chromium must get the same policy |
| Decide per rule | `applies` = not `off_on_rhel` and inside `versions` (if given) | Turns the benchmark list into this host's expected set |
| Compare every rule | `expected` = value, or `ABSENT` when it does not apply; compared inline | Inline keeps numbers as numbers on ansible-core 2.16 (T-C8) |
| Policies the benchmark does not ask for | Keys in the file but not in the list | A stray policy fails the build |
| Save the evidence | JUnit report and the policy file into `reports/` on the agent | What Jenkins archives |
| Final assert | Fails with the failed rule IDs and stray policies | The build goes red and the message says why (list form, T-C9) |

---

## molecule/default/templates/junit.xml.j2

```xml
<?xml version="1.0" encoding="UTF-8"?>
{% set failed = verify_results | rejectattr('pass') | list %}
<testsuite name="CIS Google Chrome v3.0.0 | {{ inventory_hostname }}" tests="{{ verify_results | length + 1 }}" failures="{{ failed | length + (1 if verify_unexpected else 0) }}">
  <properties>
    <property name="browsers" value="{{ verify_browsers.items() | map('join', ' ') | join(', ') }}"/>
    <property name="gate_version" value="{{ verify_major }}"/>
    <property name="policy_folders" value="{{ verify_folders | join(', ') }}"/>
  </properties>
{% for r in verify_results %}
{%   if r.off_on_rhel | default(false) %}
{%     set note = ' (off on RHEL)' %}
{%   elif r.versions is defined %}
{%     set note = ' (versions ' ~ r.versions | join('-') ~ (': applied on ' if r.applies else ': not on ') ~ verify_major ~ ')' %}
{%   else %}
{%     set note = '' %}
{%   endif %}
  <testcase classname="{{ inventory_hostname }}.level{{ r.level }}" name="{{ r.rule }} {{ r.policy | e }}{{ note }}">
{% if not r.pass %}
    <failure message="expected {{ r.expected | to_json | e }}, found {{ r.actual | to_json | e }}"/>
{% endif %}
  </testcase>
{% endfor %}
  <testcase classname="{{ inventory_hostname }}.policy_file" name="no policies outside the benchmark">
{% if verify_unexpected %}
    <failure message="{{ verify_unexpected | join(', ') | e }}"/>
{% endif %}
  </testcase>
</testsuite>
```

One `testsuite` per host with the browsers, the gate version and the policy folders as properties; one `testcase` per
rule (`classname` = host and level, so the Tests tab groups by host, then level). The name says why a gated or off
rule was expected present or absent, e.g. `2.3.4 DefaultThirdPartyStoragePartitioningSetting (versions 113-145:
applied on 133)` or `1.19 ThirdPartyBlockingEnabled (off on RHEL)`. A failed rule carries
`expected <value>, found <value>`.

---

## molecule/default/vars/expected-policies.yml

Header and format (the full list has 107 entries):

```yaml
---
# Expected policy values for this scenario (Level 1 + Level 2; site value set only for 2.3.5, so its gate is tested).
# Source: CIS Google Chrome Benchmark v3.0.0, via docs/files/cis-requirements.md, not the role tasks.
# "Disabled" exception lists are expected as [] (role decision D7).
# versions: [first, last] = supported_on range in Chromium's policy definitions
#   (docs/evidence-files/chromium-supported-on-2026-10-13.txt); outside it the policy must be absent.
# off_on_rhel: Windows-only or Google Update setting; off by default on RHEL, so it must be absent.
chrome_cis_expected:
  - {rule: "1.1.1", level: 1, policy: AllowCrossOriginAuthPrompt, value: false}
  - {rule: "2.3.4", level: 2, policy: DefaultThirdPartyStoragePartitioningSetting, value: 2, versions: [113, 145]}
  - {rule: "2.8.2", level: 1, policy: RemoteAccessHostAllowUiAccessForRemoteAssistance, value: false, off_on_rhel: true}
```

| Field | Meaning |
|---|---|
| `rule`, `level` | CIS ID and profile level (1 or 2) |
| `policy`, `value` | Chrome policy name and the value the benchmark asks for (site exception lists as `[]`, design decision D7) |
| `versions: [first, last]` | Policy removed from Chrome: present only when the host's browser version is in the range, absent otherwise. Ranges from `docs/evidence-files/chromium-supported-on-2026-10-13.txt` |
| `off_on_rhel: true` | Windows-only (1.19, 2.8.2, 2.10.1, 2.30, 3.6, 3.13) or Google Update (2.1.1, 2.1.2): off by default on RHEL, must be absent |

Values come from `docs/files/cis-requirements.md` (the benchmark matrix), never from the role's tasks: a test copied
from the code would pass a wrong implementation. 2.3.5 carries this scenario's site value (`https://example.com`, set
in `molecule.yml`) so its version gate is tested.

---

## Run it by hand (on the agent, as jenkins)

```bash
git clone https://github.com/bunnywkwk/chrome_cis.git && cd chrome_cis && git switch <branch>
. /opt/molecule/bin/activate
molecule test                                   # all hosts
molecule test --platform-name chrome-cis-el9-local   # one host
molecule converge && molecule verify            # keep containers for debugging; molecule destroy when done
```

Reports land in `~/.cache/molecule/<repo>/default/` unless `MOLECULE_REPORT_DIR` is set.

## Copying to another role

| File | Reuse |
|---|---|
| `Jenkinsfile` | As is; change the container name prefix `chrome-cis-` in the one-host command |
| `molecule.yml` | Structure; platforms and group variables per role. A service role (MongoDB, Cassandra) needs systemd in the containers |
| `requirements.yml`, `converge.yml` | As is |
| `prepare.yml` | Only if the role needs something present before it runs |
| `verify.yml`, `junit.xml.j2`, `expected-*.yml` | Rewrite per role from its benchmark: what "compliant" means differs per target. Keep the shape (one result per rule, `verify_results` list) so the JUnit template fits |
