// Trusted pipeline for chrome_cis:
//   branch push:   static analysis, then molecule on Rocky 8/9/10 containers (el8 Chromium, el9 Chrome 143,
//                  el10 Chrome + Chromium)
//   pull request:  the PR merged with its target, tested on a real RHEL 9 VM only (Terraform from the golden image);
//                  lint and containers already ran on the branch build, which keeps running while the PR is open
// GitHub Branch Source reports both results as commit statuses; the GitHub ruleset on staging and main requires both.
pipeline {
  // Dedicated agent VM with the tools installed: /opt/lint (ansible-lint, yamllint), /opt/molecule (ansible-core 2.16,
  // molecule, podman driver), rootless podman for the test containers, Terraform and an SSH key for the laptop's libvirt.
  agent { label 'molecule' }

  options {
    timestamps()
    timeout(time: 90, unit: 'MINUTES')
    disableConcurrentBuilds()
    buildDiscarder(logRotator(numToKeepStr: '100', artifactNumToKeepStr: '100'))
  }

  environment {
    MOLECULE_REPORT_DIR = "${WORKSPACE}/reports"
    // Unique container and VM names per build (molecule.yml uses it); only [a-z0-9-] for container names.
    MOLECULE_RUN_ID = "${env.BUILD_TAG.toLowerCase().replaceAll('[^a-z0-9-]', '-')}"
    // Plain console log, no colour codes
    ANSIBLE_FORCE_COLOR = '0'
    PY_COLORS = '0'
  }

  stages {
    stage('Static analysis') {
      when { not { changeRequest() } }
      steps {
        sh '''
          mkdir -p reports
          . /opt/lint/bin/activate
          yamllint -f parsable .
          ansible-lint --sarif-file reports/ansible-lint.sarif
        '''
      }
    }

    stage('Compliance test (containers)') {
      when { not { changeRequest() } }
      steps {
        sh '''
          . /opt/molecule/bin/activate
          ansible --version | head -1
          molecule test
        '''
      }
    }

    stage('Compliance test (VM)') {
      when { changeRequest() }
      steps {
        // The VM registers with Red Hat through cloud-init to install the browser's dependencies.
        withCredentials([string(credentialsId: 'rhsm-org', variable: 'TF_VAR_rhsm_org'),
                         string(credentialsId: 'rhsm-activation-key', variable: 'TF_VAR_rhsm_activation_key')]) {
          sh '''
            . /opt/molecule/bin/activate
            ansible --version | head -1
            terraform version | head -1
            molecule test -s vm
          '''
        }
      }
    }
  }

  post {
    always {
      // molecule test removes its containers or VM; this covers a build killed by the timeout or an abort.
      script {
        if (env.CHANGE_ID) {
          sh '. /opt/molecule/bin/activate && molecule destroy -s vm || true'
        } else {
          sh '. /opt/molecule/bin/activate && molecule destroy || true'
        }
      }
      junit allowEmptyResults: true, testResults: 'reports/cis-verify-*.xml'
      archiveArtifacts artifacts: 'reports/**', allowEmptyArchive: true, fingerprint: true
    }
  }
}
