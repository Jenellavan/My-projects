pipeline {
  agent { label 'infra-build-node' }

  environment {
    AWS_DEFAULT_REGION = 'us-east-1'
  }

  stages {
    stage('Install aws-cli on slave') {
      steps {
        sh '''
            sudo apt update
            sudo apt install -y python3-pip jq curl unzip

            # Install AWS CLI v2 manually
            curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
            unzip awscliv2.zip
            sudo ./aws/install
            rm -rf aws awscliv2.zip
            pip3 install --upgrade boto3 botocore
            ansible-galaxy collection install amazon.aws --force
            curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
            sudo apt install -y nodejs
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
