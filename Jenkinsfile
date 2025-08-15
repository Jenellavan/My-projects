pipeline {
  agent { label 'infra-build-node' }

  environment {
    AWS_DEFAULT_REGION = 'us-east-1'
    DYNATRACE_API_TOKEN = credentials('Dynatrace_api_token')
    DYNATRACE_ENV_URL = 'https://xpc06585.live.dynatrace.com'
  }

  stages {
    stage('set up one agaent') {
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
              ansible-playbook -i inventory/aws_ec2.yml install_one_agent.yml \
              -e "dynatrace_api_token=${DYNATRACE_API_TOKEN}" \
              -e "dynatrace_env_url=${DYNATRACE_ENV_URL}"
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
      echo 'Dynatrace provisioned successfully.'
    }
    failure {
      echo 'Build failed. Check archived inventory.json for details.'
    }
  }
}
