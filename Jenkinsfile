 fix that here then pipeline {
    agent { label 'infra-build-node' }

    environment {
        SONARQUBE_SERVER = 'SonarQube'
        NEXUS_URL = 'http://52.202.80.121:8081'
        NEXUS_REPO = 'ezlearn-release'
        DEPLOY_SERVER = 'ubuntu@pipeline {
    agent { label 'infra-build-node' }

    environment {
        SONARQUBE_SERVER = 'SonarQube'
        NEXUS_URL = 'http://52.202.80.121:8081/'
        NEXUS_REPO = 'ezlearn-release'
        DEPLOY_SERVER = 'ubuntu@172.31.87.178'   // use private IP
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
                          -Dsonar.host.url=http://54.224.144.227:9000/
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
}'
        DEPLOY_PATH = '/opt/tomcat/webapps'
        VERSION = '1.0.0'
        SONAR_JACOCO_REPORT_PATH = 'target/site/jacoco/jacoco.xml'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Static Analysis') {
            steps {
                sh 'mvn checkstyle:checkstyle'
                archiveArtifacts artifacts: 'target/checkstyle-result.xml', allowEmptyArchive: true
            }
        }

        stage('Unit Test & Coverage') {
            steps {
                sh 'mvn test jacoco:report'
                junit 'target/surefire-reports/**/*.xml'
                archiveArtifacts artifacts: 'target/site/jacoco/**/*.xml', allowEmptyArchive: true
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh """
                        mvn sonar:sonar \
                          -Dsonar.projectKey=ezlearn \
                          -Dsonar.host.url=http://54.224.144.227:9000 \
                          -Dsonar.qualitygate.wait=true \
                          -Dsonar.coverage.jacoco.xmlReportPaths=${SONAR_JACOCO_REPORT_PATH} \
                          -Dsonar.java.binaries=target/classes \
                          -Dsonar.sources=src/main/java
                    """
                }
            }
        }

        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package WAR') {
            steps {
                sh '''
                    mvn package
                    cp target/ezlearn-${VERSION}.war target/ezlearn.war
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
                            mvn deploy:deploy-file \
                              -DgroupId=com.ezlearn \
                              -DartifactId=ezlearn \
                              -Dversion=${timestamp} \
                              -Dpackaging=war \
                              -Dfile=${warPath} \
                              -DrepositoryId=ezlearn-release \
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
            cleanWs()
            script {
                if (currentBuild.result == 'UNSTABLE') {
                    echo "⚠️ Build unstable due to quality gate warnings."
                } else if (currentBuild.result == 'FAILURE') {
                    echo "❌ Build failed!"
                } else {
                    echo "✅ Pipeline executed successfully!"
                }
            }
        }
    }
}
