pipeline {
    agent { label 'infra-build-node' }
       
    environment {
        SONARQUBE_SERVER = 'SonarQube'
        NEXUS_URL = 'http://54.172.175.151:8081/'
        NEXUS_REPO = 'ezlearn-release'
        DEPLOY_SERVER = 'ubuntu@18.207.173.147'
        DEPLOY_PATH = '/opt/tomcat/webapps'
        VERSION = '1.0.0'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check Maven Version') {
            steps {
                sh 'mvn -version'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                sh 'mvn checkstyle:check'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh '''
                        mvn sonar:sonar \
                          -Dsonar.projectKey=ezlearn \
                          -Dsonar.host.url=http://sonarqube.mitechnology.org:9000
                    '''
                }
            }
        }

        stage('Package WAR') {
            steps {
                sh '''
                    mvn package
                    cp target/ezlearn-1.0.0.war target/ezlearn.war
                '''
            }
        }

        stage('Publish to Nexus') {
            steps {
                script {
                    def timestamp = sh(script: "date +%Y%m%d%H%M%S", returnStdout: true).trim()
                    def warName = "ezlearn-${timestamp}.war"
                    def warPath = "target/${warName}"

                    // Copy WAR with versioned filename
                    sh "cp target/ezlearn.war ${warPath}"

                    withCredentials([usernamePassword(
                        credentialsId: 'nexus-creds',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {
                        sh """
                            mvn deploy:deploy-file \
                              -DgroupId=com.ezlearn \
                              -DartifactId=ezlearn \
                              -Dversion=${timestamp} \
                              -Dpackaging=war \
                              -Dfile=${warPath} \
                              -DrepositoryId=${NEXUS_REPO} \
                              -Durl=${NEXUS_URL}/repository/${NEXUS_REPO}/ \
                              -DgeneratePom=true \
                              --settings jenkins/settings.xml
                        """
                    }    
                }
            }
        }

        stage('Deploy to Tomcat') {
            steps {
                sshagent (credentials: ['ssh-agent-key']) {
                   sh """  
                        cp target/ezlearn.war target/ROOT.war
                        scp -o StrictHostKeyChecking=no target/ROOT.war ${DEPLOY_SERVER}:/tmp/ROOT.war
                        ssh -o StrictHostKeyChecking=no ${DEPLOY_SERVER} 'sudo mv /tmp/ROOT.war ${DEPLOY_PATH}/ROOT.war && sudo chown tomcat:tomcat ${DEPLOY_PATH}/ROOT.war'
                    """    
                }
            }
        }
    }

    post {
        always {
            script {
                def hasReports = fileExists('target/surefire-reports')
                if (hasReports) {
                    junit 'target/surefire-reports/*.xml'
                } else {
                    echo "No test reports found to archive."
                }
            }
        }
        success {
            echo "✅ Pipeline executed successfully!"
        }
        failure {
            echo "❌ Pipeline failed!"
        }
    }
}
