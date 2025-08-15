pipeline {
    agent { label 'infra-build-node' }

    environment {
        SONARQUBE_SERVER = 'SonarQube'
        NEXUS_URL = 'http://52.202.80.121:8081/'
        NEXUS_REPO = 'ezlearn-release'
        DEPLOY_SERVER = 'ubuntu@172.31.87.178'
        DEPLOY_PATH = '/opt/tomcat/webapps'
        VERSION = '1.0.0'
        MAVEN_HOME = '/opt/maven'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Check Maven Version') {
            steps {
                sh 'export PATH=$MAVEN_HOME/bin:$PATH && mvn -version'
            }
        }

        stage('Build') {
            steps {
                sh 'export PATH=$MAVEN_HOME/bin:$PATH && mvn clean compile'
            }
        }

        stage('Unit Test') {
            steps {
                sh 'export PATH=$MAVEN_HOME/bin:$PATH && mvn test'
            }
        }

        stage('Checkstyle Analysis') {
            steps {
                sh 'export PATH=$MAVEN_HOME/bin:$PATH && mvn checkstyle:check'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh '''
                        export PATH=$MAVEN_HOME/bin:$PATH && \
                        mvn sonar:sonar \
                          -Dsonar.projectKey=ezlearn \
                          -Dsonar.host.url=http://54.224.144.227:9000
                    '''
                }
            }
        }

        stage('Package WAR') {
            steps {
                sh '''
                    export PATH=$MAVEN_HOME/bin:$PATH && \
                    mvn package && \
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

                    sh "cp target/ezlearn.war ${warPath}"

                    withCredentials([usernamePassword(
                        credentialsId: 'nexus-creds',
                        usernameVariable: 'NEXUS_USER',
                        passwordVariable: 'NEXUS_PASS'
                    )]) {
                        sh """
                            export PATH=$MAVEN_HOME/bin:$PATH && \
                            mvn deploy:deploy-file \
                              -DgroupId=com.ezlearn \
                              -DartifactId=ezlearn \
                              -Dversion=${timestamp} \
                              -Dpackaging=war \
                              -Dfile=${warPath} \
                              -DrepositoryId=${NEXUS_REPO} \
                              -Durl=${NEXUS_URL}/repository/${NEXUS_REPO}/ \
                              -DgeneratePom=false \
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
                     set -e
                          cp target/ezlearn.war target/ROOT.war
                          scp -o StrictHostKeyChecking=no target/ROOT.war ${DEPLOY_SERVER}:/tmp/ROOT.war
                          ssh -o StrictHostKeyChecking=no ${DEPLOY_SERVER} '\
                          test -d ${DEPLOY_PATH} || { echo "Deploy path ${DEPLOY_PATH} not found"; exit 2; }; \
                             sudo mv /tmp/ROOT.war ${DEPLOY_PATH}/ROOT.war && \
                             sudo chown tomcat:tomcat ${DEPLOY_PATH}/ROOT.war && \
                             sudo systemctl restart ${TOMCAT_SERVICE} \'
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
