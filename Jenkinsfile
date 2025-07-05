pipeline {
  agent { label 'infra-build-node' }

  environment {
    AWS_DEFAULT_REGION = 'us-east-1'
  }

  stages {
    stage('Install aws-cli on slave') {
      steps {
        sh '''
          set -eu

          # Install base packages
          sudo apt-get update
          sudo apt-get install -y python3-pip jq unzip curl

          # Install AWS CLI v2 manually (since it's not available via apt)
          curl -sSL "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
          unzip -q awscliv2.zip
          sudo ./aws/install --update
          rm -rf awscliv2.zip aws

          # Install/upgrade Python libraries (use --break-system-packages to override restriction)
          pip3 install --upgrade boto3 botocore --break-system-packages

          # Install Node.js 18
          curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
          sudo apt-get install -y nodejs

          # Install Ansible AWS collection
          ansible-galaxy collection install amazon.aws --force
        '''  
      }
    }

    stage('Provision Infrastructure') {
      steps {
        sshagent(credentials: ['ssh-agent-key']) {
          withCredentials([
            usernamePassword(
              credentialsId: 'jenkins-ec2-access',
              usernameVariable: 'AWS_ACCESS_KEY_ID',
              passwordVariable: 'AWS_SECRET_ACCESS_KEY'
            )
          ]) {
            sh '''
              set -eu
              ansible-playbook -i inventory/prod/aws_ec2.yml playbooks/site.yml 
            '''
          }
        }
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'inventory.json', allowEmptyArchive: true
      cleanWs()
    }
    success {
      echo 'Infrastructure provisioned successfully.'
    }
    failure {
      echo 'Build failed. Check archived inventory.json for details.'
    }
  }
}


