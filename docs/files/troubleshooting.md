# Troubleshooting: chrome_cis

Every error hit while building or testing the role: exact output, cause, fix, prevent. IDs `T-Cn`.

## T-C1. google-chrome RPM: "key 1 import failed" during install

**When:** discovery step 2, `sudo dnf install -y https://dl.google.com/linux/direct/google-chrome-stable_current_x86_64.rpm`
on RHEL 8, 9 and 10 (2026-10-07). The install itself finished (`Complete!`).

```text
  Running scriptlet: google-chrome-stable-155.0.8059.39-1.x86_64                                       6/6
error: can't create transaction lock on /var/lib/rpm/.rpm.lock (Resource temporarily unavailable)
error: /tmp/google.sig.Uzg8ZC: key 1 import failed.
```

RHEL 10 shows the same with `/usr/lib/sysimage/rpm/.rpm.lock` (the rpm database moved there).
Afterwards `rpm -q gpg-pubkey --qf '%{VERSION}-%{RELEASE} %{SUMMARY}\n' | grep -i google` prints nothing.

**Cause:** the RPM's post-install script runs `rpm --import` for Google's signing key while `dnf` still holds the rpm
database lock, so the import fails. The package came from a URL (a local file to dnf), which dnf does not
signature-check by default, so nothing stopped the install.

**Fix (manual):** `sudo rpm --import https://dl.google.com/linux/linux_signing_key.pub`. The daily cron
`/etc/cron.daily/google-chrome` also retries the import.

**Prevent (role):** opt-in install imports the key first (`ansible.builtin.rpm_key`), then adds the repo
(`ansible.builtin.yum_repository`, same content as Google's file), then installs from the repo with `dnf`, so the
package is verified and the post-install import has nothing to do.

## T-C2. RHEL 8: "Could not detect a supported package manager from the following list: ['rpm']"

**When:** first test-project run on rhel8 (2026-10-08), task `PRELIM | AUDIT | Gather installed packages`. Screenshot:
`~/chrome_cis_test/docs/evidence/image.png`.

```text
[WARNING]: Requested package manager rpm was not usable by this module: Found executable at /bin/rpm. Failed to
import the required Python library (rpm) on rhel8's Python /usr/bin/python3.12. ...
[ERROR]: Task failed: Module failed: Could not detect a supported package manager from the following list: ['rpm'],
or the required Python library is not installed. Check warnings for details.
fatal: [rhel8]: FAILED! => {"changed": false, "msg": "Could not detect a supported package manager from the following
list: ['rpm'], or the required Python library is not installed. Check warnings for details."}
```

**Cause:** the run used a newer ansible-core (the `[ERROR]: Task failed: ... Origin:` format is 2.19+), not the
2.16 venv. ansible-core 2.17+ no longer supports Python 3.6 on targets, so on RHEL 8 it picks `/usr/bin/python3.12`.
The `rpm` (and `dnf`) Python bindings exist only for RHEL 8's system Python 3.6 (`/usr/libexec/platform-python`), so
`package_facts` (and later `dnf`) can't work. Same root cause as the control-node decision D15 in `mongodb8_cis`.

**Fix:** run from the 2.16 venv: `source ~/.ansible-2.16.1-env/bin/activate`, check `ansible --version` shows
`core 2.16.x`, rerun.

**Prevent:** always activate the 2.16 venv before running the test project (`docs/control-node-setup.md`); prelim
only asserts `>= 2.16.1`, so a newer core passes the check and fails here on RHEL 8.

## T-C5. Chromium install: "Failed to download metadata for repo 'epel'"

**When:** 2026-10-09, container test, clean Rocky 9, `chrome_cis_browser: chromium`, `chrome_cis_install: true`, first
run. The EPEL key and `epel-release` were installed; the Chromium install failed.

```text
fatal: [ce9]: FAILED! => {"changed": false, "msg": "Failed to download metadata for repo 'epel': Yum repo downloading error: Downloading error(s): repodata/af4e6d44470ba0ca54e240b7fc075ec1d55485d1ece9eb931cbde10acd5371e8-filelists.xml.xz - Cannot download, all mirrors were already tried without success; repodata/99a82c6df15885ab18e74f861779eb555ded72a8dc116d0d91d99f0f166ffc90-updateinfo.xml.bz2 - Cannot download, all mirrors were already tried without success", "rc": 1, "results": []}
```

**Cause:** EPEL's mirrors did not serve two metadata files at that moment (outside the role). Rocky 8 in the same run
worked.

**Fix:** run the playbook again: the rerun installed Chromium and wrote the policy file (`changed=3`).

**Prevent:** nothing in the role for now (a one-off mirror problem). If it repeats, add `retries`/`until` to the
Chromium install task.

## T-C6. Version gate: "'<=' not supported between instances of 'int' and 'AnsibleUnsafeText'"

```
fatal: [gate9]: FAILED! => {"msg": "The conditional check '67 <= chrome_cis_browser_major <= 127' failed. The error
was: Unexpected templating type error occurred on (...): '<=' not supported between instances of 'int' and
'AnsibleUnsafeText'. ..."}
```

**Cause:** `chrome_cis_browser_major` is set by `set_fact` from a template, so it is stored as text ("128"), not a
number; Jinja cannot compare a number with text.

**Fix:** convert in each gate: `67 <= chrome_cis_browser_major | int <= 127` (D17). Found in the container gate test
before any VM run.

**Prevent:** compare facts built from templates with `| int` (or `is version(...)`).

## T-C7. Molecule converge: "the role 'chrome_cis' was not found"

**When:** 2026-10-12, first manual `molecule test` in the CI image (`ci/Dockerfile`: molecule 26.9.0,
ansible-core 2.16.19), branch `ci/molecule`. Create worked on Rocky 8/9/10; converge failed.

```text
TASK [Converge | Run the role from this checkout] ******************************
ERROR! the role 'chrome_cis' was not found in /home/bunny/ansible-cis/chrome_cis/molecule/default/roles:/tmp/.ansible/roles:/usr/share/ansible/roles:/etc/ansible/roles:/home/bunny/ansible-cis/chrome_cis/molecule/default
```

**Cause:** `converge.yml` included the role by its folder name (`MOLECULE_PROJECT_DIRECTORY | basename`), which
needs the role's parent folder on the roles path. Molecule 26 does not add it (the searched paths above don't include
`/home/bunny/ansible-cis`).

**Fix:** `include_role` with the full path: `name: "{{ lookup('ansible.builtin.env', 'MOLECULE_PROJECT_DIRECTORY') }}"`.
Works for any folder name (the Jenkins workspace is not called `chrome_cis`).

**Prevent:** in molecule scenarios, include the role under test by path, not by name.

## T-C8. Molecule verify: 21 rules fail with "expected 1, found "1""

**When:** 2026-10-12, manual `molecule test` in the CI image (ansible-core 2.16.19). Converge, idempotence and check
passed on Rocky 8/9/10; verify failed with 21 failures per host, all rules with a numeric value.

```text
<testcase classname="chrome-cis-el9-local.level2" name="1.8 SafeSitesFilterBehavior">
  <failure message="expected 1, found &#34;1&#34;"/>
```

**Cause:** a bug in `verify.yml`, not in the role: the written file had numbers (`"SafeSitesFilterBehavior": 1`).
The actual value was passed through a task-level `vars:` entry (`actual: "{{ ... | default('ABSENT') }}"`), and on
ansible-core 2.16 it came back as the string `"1"`; booleans, strings and lists were not affected.

**Fix:** read the value inline in the `set_fact` expression (no intermediate var).

**Prevent:** in verify code, compare values inside one expression; keep the strict type check (Chrome rejects `"1"`
where it expects an integer).

## T-C9. Molecule verify: final assert fails although every rule passed

**When:** 2026-10-12, rerun after T-C8, `molecule test -p chrome-cis-el9-local`. The JUnit report had 0 failures
(104 rules + extra-policy check), but `verify: Executed: Failed`.

```text
default > verify: Executed: Failed (Ansible return code was 2, command was: ansible-playbook --inventory /tmp/.ansible/tmp/molecule.x8jt.default/inventory --skip-tags molecule-notest,notest /home/bunny/ansible-cis/chrome_cis/molecule/default/verify.yml)
```

Reproduced in the CI image:

```text
localhost | FAILED! => {
    "msg": "Incorrect type for fail_msg or msg, expected a string or list and got <class 'dict'>"
}
```

**Cause:** the last `assert` in `verify.yml` had a dict as `fail_msg`; `assert` accepts only a string or a list and
checks the type before evaluating `that`, so it failed even with every condition true.

**Fix:** `fail_msg` as a list of two strings (failed rules, unexpected policies).

**Prevent:** `fail_msg`/`success_msg` are always a string or a list of strings.

## T-C10. Jenkins first build of a branch: "Instances missing from the 'platform' section of molecule.yml"

**When:** 2026-10-12, Jenkins job `chrome-cis`, build #1 of `version-policy-rule` (first build on the agent
`kvm-agent-01`), stage Compliance test:

```
22:03:12  + '[' '' = all ']'
22:03:12  + molecule test --platform-name chrome-cis--jenkins-chrome-cis-version-policy-rule-1
...
22:03:13  ERROR    Instances missing from the 'platform' section of molecule.yml.
...
ERROR: script returned exit code 4
```

**Cause:** the first build of a new branch runs before Jenkins has read the Jenkinsfile's `parameters` block, so
`OS` is empty, not `all`. The script took the "one OS" path with an empty OS: platform name `chrome-cis--...`, which
does not exist.

**Fix:** `Jenkinsfile` treats an empty `OS` as `all`: `if [ "${OS:-all}" = all ]`.

**Prevent:** give every Jenkins parameter a shell default (`${PARAM:-default}`); the first build of each new branch
or PR never has parameters.

## T-C11. CI idempotence: "The read operation timed out" from dl.google.com

**Seen:** 2026-10-13, laptop Jenkins, branch build `version-policy-rule` #4. Converge passed on all three containers;
the second run (idempotence) failed on el10 and el9 while contacting Google, el8 (Chromium, EPEL) passed with
`changed=0`:

```
22:58:16  fatal: [chrome-cis-el10-jenkins-chrome-cis-version-policy-rule-4]: FAILED! => {"changed": false, "msg": "failed to fetch key at https://dl.google.com/linux/linux_signing_key.pub , error was: Connection failure: The read operation timed out"}
22:58:55  fatal: [chrome-cis-el9-jenkins-chrome-cis-version-policy-rule-4]: FAILED! => {"changed": false, "msg": "Failure downloading https://dl.google.com/linux/chrome/rpm/stable/x86_64/google-chrome-stable-143.0.7499.40-1.x86_64.rpm, The read operation timed out"}
...
22:59:18  default > idempotence: Executed: Failed (Ansible return code was 2, ...)
```

**Cause:** network timeout to `dl.google.com`, not the role: no task reported `changed`. The install tasks contact
Google on every run, also when Chrome is already installed: `rpm_key` with a URL downloads the key to compare it, and
the `dnf` module downloads an RPM given by URL (the pinned Chrome 143 on el9) before it can see it is installed.

**Fix:** rebuild the branch.

**Prevent (done 2026-10-13, test only, role unchanged):** `molecule/default/prepare.yml` installs the browsers once with
the role's install tasks, one host at a time (`serial: 1`); `molecule.yml` sets `chrome_cis_install: false`, so
converge, idempotence and check only harden and need no network. The install path keeps its full test (install +
idempotence) in the VM scenario on pull requests. The air-gapped lab bakes the browsers into the test images instead.
