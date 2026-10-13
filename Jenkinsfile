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
