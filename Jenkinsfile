// Routers are only touched on the main branch, after manual approval.
def withNetCreds(Closure body) {
  withCredentials([
    usernamePassword(credentialsId: 'net-ssh', usernameVariable: 'NET_USER', passwordVariable: 'NET_PASS'),
    string(credentialsId: 'net-enable', variable: 'NET_ENABLE')
  ]) {
    body()
  }
}

pipeline {
  agent { label 'ansible-net' }

  options {
    timestamps()
    ansiColor('xterm')
    disableConcurrentBuilds()
    timeout(time: 30, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '30'))
  }

  parameters {
    string(name: 'PLAYBOOK', defaultValue: 'playbooks/test1.yml', description: 'Playbook path inside the repo')
    string(name: 'LIMIT', defaultValue: 'routers', description: 'Ansible --limit (host or group)')
  }

  environment {
    ANSIBLE_FORCE_COLOR = 'true'
    // Params are null on the first run of a new branch, so default here.
    PLAYBOOK = "${params.PLAYBOOK ?: 'playbooks/test1.yml'}"
    LIMIT = "${params.LIMIT ?: 'routers'}"
  }

  stages {
    stage('Validate parameters') {
      steps {
        script {
          if (!(env.PLAYBOOK ==~ /^playbooks\/[A-Za-z0-9_.\-]+\.ya?ml$/)) {
            error "Invalid PLAYBOOK: ${env.PLAYBOOK}"
          }
          if (!(env.LIMIT ==~ /^[A-Za-z0-9_.,\-]+$/)) {
            error "Invalid LIMIT: ${env.LIMIT}"
          }
        }
      }
    }

    stage('Setup') {
      steps {
        sh '''
          python3 -m venv venv
          . venv/bin/activate
          pip install -q -r requirements.txt
          ansible-galaxy collection install -r requirements.yml
        '''
      }
    }

    stage('Lint') {
      steps {
        sh '''
          . venv/bin/activate
          yamllint .
          ansible-lint playbooks/
        '''
      }
    }

    stage('Syntax check') {
      steps {
        sh '''
          . venv/bin/activate
          ansible-playbook "$PLAYBOOK" --syntax-check
        '''
      }
    }

    stage('Backup and dry run') {
      steps {
        withNetCreds {
          sh '''
            . venv/bin/activate
            ansible-playbook playbooks/backup.yml --limit "$LIMIT"
            ansible-playbook "$PLAYBOOK" --limit "$LIMIT" --check --diff
          '''
        }
      }
    }

    stage('Approve') {
      when { branch 'main' }
      steps {
        timeout(time: 15, unit: 'MINUTES') {
          // Narrow to a network-ops group once RBAC exists (Phase 8).
          input message: "Deploy ${env.PLAYBOOK} to ${env.LIMIT}?", submitter: 'admin'
        }
      }
    }

    stage('Deploy') {
      when { branch 'main' }
      steps {
        withNetCreds {
          sh '''
            . venv/bin/activate
            ansible-playbook "$PLAYBOOK" --limit "$LIMIT"
          '''
        }
      }
    }

    stage('Validate') {
      when { branch 'main' }
      steps {
        withNetCreds {
          sh '''
            . venv/bin/activate
            ansible-playbook playbooks/validate.yml --limit "$LIMIT"
          '''
        }
      }
    }

    stage('Idempotency check') {
      when { branch 'main' }
      steps {
        withNetCreds {
          // A second run must report changed=0.
          sh '''
            . venv/bin/activate
            export ANSIBLE_FORCE_COLOR=false ANSIBLE_NOCOLOR=true
            ansible-playbook "$PLAYBOOK" --limit "$LIMIT" > run2.log 2>&1 || { cat run2.log; exit 1; }
            cat run2.log
            if grep -Eq 'changed=[1-9]' run2.log; then
              echo "Second run made changes: playbook is not idempotent"
              exit 1
            fi
          '''
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'backups/**', allowEmptyArchive: true
      cleanWs()
    }
    failure {
      echo 'Build failed - rollback and notifications are added in later phases.'
    }
  }
}
