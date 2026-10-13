# Test results and evidence index

Screenshots live in [evidence-images/](evidence-images/), named `NN-<os>-<scenario>-<result>.png`.

## Discovery

| # | File | Host | What it shows | Result |
|---|------|------|---------------|--------|
| 01 | [01-rhel8-discovery-readonly-ok.png](evidence-images/01-rhel8-discovery-readonly-ok.png) | rhel8 (.50) | step 1 read-only baseline ([platform-notes D-1](platform-notes.md#d-1-step-1-read-only-baseline-2026-10-07)) | RHEL 8.10, glibc 2.28, graphical, Enforcing, no Chrome/repo |
| 02 | [02-rhel9-discovery-readonly-ok.png](evidence-images/02-rhel9-discovery-readonly-ok.png) | rhel9 (.30) | same | RHEL 9.7, glibc 2.34, graphical, Enforcing, no Chrome/repo |
| 03 | [03-rhel10-discovery-readonly-ok.png](evidence-images/03-rhel10-discovery-readonly-ok.png) | rhel10 (.40) | same | RHEL 10.2, glibc 2.39, graphical, Enforcing, no Chrome/repo |
| 04 | [04-rhel8-discovery-policy-page-ok.png](evidence-images/04-rhel8-discovery-policy-page-ok.png) | rhel8 | `chrome://policy` with the test file ([platform-notes D-3](platform-notes.md#result-c-chromepolicy-2026-10-07)) | control policy OK; 9 suspected rules Error |
| 05 | [05-rhel9-discovery-policy-page-ok.png](evidence-images/05-rhel9-discovery-policy-page-ok.png) | rhel9 | same | same |
| 06 | [06-rhel10-discovery-policy-page-ok.png](evidence-images/06-rhel10-discovery-policy-page-ok.png) | rhel10 | same | same |
| — | no screenshot (101 rows) | rhel8, rhel9, rhel10 | full test: [evidence-files/cis-applicable-101.json](evidence-files/cis-applicable-101.json) in `managed/`, `chrome://policy` sorted by Status ([platform-notes D-4](platform-notes.md#d-4-step-4-full-test-of-all-applicable-policies-2026-10-07)) | **all 101 OK** on all 3; `ProxyMode` labelled Deprecated |

## CI pipeline (Jenkins + Molecule, 2026-10-12)

Branch `version-policy-rule`, job `chrome_cis`, agent `kvm-agent-01`. A push builds the branch on its own; opening the
pull request builds the merge of the branch with `staging` (`pr-merge`, required by the ruleset). Details:
`ansible-cis/docs/local-laptop/ci-overview-laptop.md`.

| # | File | Step | What it shows | Result |
|---|---|---|---|---|
| 07 | [07-ci-branch-build2-success.png](../evidence-images/07-ci-branch-build2-success.png) | 1. Push, branch build | Jenkins `version-policy-rule` #2 console end, 22:23:40: Molecule recap `failed=0`, JUnit recorded, artifacts archived, "GitHub has been notified" | `Finished: SUCCESS` (324 tests, 0 failures) |
| 08 | [08-ci-pr2-required-check-pending.png](../evidence-images/08-ci-pr2-required-check-pending.png) | 2. PR opened | PR #2 into `staging`: `jenkins/branch` green, `jenkins/pr-merge` pending and **Required**, merge button disabled; commit `88e3fd8` red (build #1, T-C10) | Gate blocks the untested merge |
| 09 | [09-ci-pr2-merge-build-success.png](../evidence-images/09-ci-pr2-merge-build-success.png) | 3. PR build | Jenkins "Version policy rule (#2)" #1 console end, 22:43:44: the merge result tested, "GitHub has been notified" | `Finished: SUCCESS` |
| 10 | [10-ci-pr2-checks-green-merge-enabled.png](../evidence-images/10-ci-pr2-checks-green-merge-enabled.png) | 4. Ready to merge | PR #2: "All checks have passed", `branch` and `pr-merge` green, merge button enabled | Gate opens only when green |
