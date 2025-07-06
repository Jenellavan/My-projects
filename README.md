# EzLearn-Class CI/CD Bootcamp Project

This project provides a full-scale CI/CD pipeline setup for a Java web application (`ezlearn`) using Jenkins, Ansible, Nexus, SonarQube, and Tomcat. Designed as a hands-on project for bootcamp participants, it enables learners with **no prior DevOps experience** to develop real-world skills in infrastructure provisioning, configuration management, CI/CD, and application deployment.

---

## Project Overview

- **Application Name:** `ezlearn`
- **Project Name:** `ezlearn-class`

### Tools and Roles

| Tool         | Role Name            | Purpose                                      |
|--------------|----------------------|----------------------------------------------|
| Java         | `common`             | Installs OpenJDK 11                          |
| Maven        | `maven`              | Installs Maven for Java build                |
| Jenkins      | `jenkins_master`, `jenkins_slave` | Automates builds, testing, deployments       |
| Ansible      | N/A                  | Automates server configuration and provisioning |
| SonarQube    | `sonarqube`          | Analyzes code quality and security           |
| Nexus        | `nexus`              | Stores and versions application artifacts    |
| Tomcat       | `tomcat`             | Hosts the web application                    |
| Deployment   | `deploy_app`         | Deploys the WAR file to Tomcat server        |

---

## Folder Structure

```
ezlearn-class/
├── inventory/
│   ├── nonprod/
│   └── prod/
│       └── hosts.ini
├── playbooks/
│   ├── jenkins_install.yml
│   ├── site.yml
│   └── deploy.yml
├── roles/
│   ├── common/
│   ├── maven/
│   ├── jenkins_master/
│   ├── jenkins_slave/
│   ├── sonarqube/
│   ├── nexus/
│   ├── tomcat/
│   └── deploy_app/
```

---

## Step-by-Step Setup Guide

---

### Phase 1: Provision EC2 Infrastructure to install all required tools in target environments

1. **Login to AWS Console**  
   Go to https://console.aws.amazon.com and sign in and create iam user and give console access.

2. **Create a Key Pair**  
   - Go to EC2 → Key Pairs → Create Key Pair  
   - Name it (e.g., `devops-key`) and download the `.pem` file.  
   - This key allows you to SSH into your servers.

3. **Launch 5 EC2 Instances (Ubuntu 22.04)**  
   - Instance type: `t2.medium`  
   - Image: Ubuntu Server 22.04  
   - Instances needed:  
     - Jenkins Master  
     - Jenkins Slave (Agent)  
     - SonarQube Server  
     - Nexus Repository  
     - Tomcat Application Server

4. **Tag Instances for Identification**  
   - Jenkins Master → `Name = jenkins-master`  
   - Jenkins Slave → `Name = jenkins-slave`  
   - SonarQube → `Name = sonarqube`  
   - Nexus → `Name = nexus`  
   - Tomcat → `Name = tomcat`

5. **Configure Security Groups**  
   Allow these ports:  
   - Port 22: SSH for all  
   - Port 8080: Jenkins & Tomcat  
   - Port 9000: SonarQube  
   - Port 8081: Nexus

6. **Connect via SSH**  
   From your terminal:

   ```bash
   ssh -i /path/to/devops-key.pem ubuntu@<EC2-PUBLIC-IP>
   ```

7. **Update Servers**

   ```bash
   sudo apt update && sudo apt upgrade -y
   ```

---

### Phase 2: Setup Jenkins Slave the Agent to run Ansible + pipeline jobs

1. **Install Java & Ansible on Jenkins Slave**

   ```bash
   sudo apt install fontconfig openjdk-21-jre -y
   sudo apt install software-properties-common -y
   sudo add-apt-repository --yes --update ppa:ansible/ansible
   sudo apt install ansible -y
   ```

2. **Create Jenkins User on slave**

   ```bash
   sudo useradd -m -d /home/jenkins -s /bin/bash jenkins
   sudo mkdir -p /home/jenkins/.ssh
   sudo touch /home/jenkins/.ssh/authorized_keys
   sudo chown -R jenkins:jenkins /home/jenkins/.ssh
   sudo chmod 700 /home/jenkins/.ssh
   sudo chmod 600 /home/jenkins/.ssh/authorized_keys
   echo "jenkins ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/jenkins
   ```



### Phase 3: Setup Ansible Dynamic Inventory so Ansible can discover EC2 servers automatically

1. **Create IAM User in AWS Console**

   - IAM → Users → Add user  
   - Name: `jenkins-ec2-access`  
   - Enable **programmatic access**  
   - Attach policy: `AmazonEC2ReadOnlyAccess`  
   - Create and **download Access Key ID and Secret**


### Phase 4: Install Jenkins on Master  Jenkins will drive all automation

1. **Install Jenkins**

   ```bash
   sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
   sudo echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
   sudo apt update
   sudo apt install jenkins -y
   ```

2. **Install Java**

   ```bash
   sudo apt install fontconfig openjdk-21-jre -y
   ```

3. **Enable & Start Jenkins**

   ```bash
   sudo systemctl enable jenkins
   sudo systemctl start jenkins
   ```

4. **Access Jenkins UI**

   - Visit: `http://<Jenkins-IP>:8080`  
   - Get the unlock password:

     ```bash
     sudo cat /var/lib/jenkins/secrets/initialAdminPassword
     ```

   - Install suggested plugins and create admin user.
   - Install ssh agent plugin.  Click on manage Jenkins -> Plugins -> Available Plugins -> search ssh agent - Check bok and click Install

---

### Phase 5: Connect Jenkins Slave ( Delegate workload to slave agent)

1. **Generate SSH Keys on Jenkins Master**

   ```bash
   # 1. Create .ssh directory
   sudo -u jenkins mkdir -p /var/lib/jenkins/.ssh

   # 2. Generate key pair (will create id_rsa and id_rsa.pub)
   sudo -u jenkins ssh-keygen -t rsa -b 4096 -f /var/lib/jenkins/.ssh/id_rsa -N ""

   # 3. Set proper permissions
   sudo chmod 700 /var/lib/jenkins/.ssh
   sudo chmod 600 /var/lib/jenkins/.ssh/id_rsa
   sudo chmod 644 /var/lib/jenkins/.ssh/id_rsa.pub

# 4. Verify files exist
   ```
   sudo -u jenkins ls -la /var/lib/jenkins/.ssh/
   ```
2. **Add Key to Slave’s `jenkins` User**
Run the following command on master and copy keys over to slave node
   ```bash
   sudo cat /var/lib/jenkins/.ssh/id_rsa.pub
   ```
   Copy the key and go to slave node terminal and insert with the following commands

   ```
   sudo su - jenkins
   vim .ssh/authorized_keys
   ```
   paste the copied keys and save file with :wq! hit enter key to save and exit

4. **Add Jenkins Node in UI**

   - Click Manage Jenkins → Nodes → New Node  
   - Name: `infra-build-node`  Select Permanent Agent then click Create 
   - Remote root: `/home/jenkins`
   - Launch Method click dropdown and select Launch via SSH
   - Host insert private ip of slave
   - Credentials: click Add and select Jenkins
   - Kind: Select SSH Username with private key
   - ID: Jenkins-ssh-key
   - Username: jenkins
   - Private Key = select Enter directly and click Add
   - Copy private SSH key from Jenkins master with the command `sudo cat /var/lib/jenkins/.ssh/id_rsa`
   - Copy entire key and go back to jenkins UI and paste key
   - Host Key Verification Strategy -> Select Non verifying Verification Strategy
   - click save
  
   **Store Credentials in Jenkins**

   - Install **Credentials** and **Credentials Binding** plugins  
   - Go to: Jenkins → Manage → Credentials → Global → Add Credentials  
   - Kind: `Username with password`  
   - ID: `jenkins-ec2-access`  
   - Username: AWS Access Key  
   - Password: AWS Secret Key

---

### Phase 6: Provision DevOps Tools Automatically

1. **Create Jenkins Pipeline Job**

   - Name: `infrastructure-setup`  
   - Type: Pipeline  
   - Definition: Pipeline script from SCM  
   - SCM: Git  
   - Repo URL: `https://github.com/<your-org>/<repo>`  
   - Branch: `*/project-1-cicd`  
   - Script Path: `Jenkinsfile`

2. **Run the Job**

   - Jenkins will use Ansible to provision:\n
     - SonarQube (port 9000)  
     - Nexus (port 8081)  
     - Tomcat (port 8080)

---

## ✅ Final Outcome

After successful execution:

- Jenkins is set up with a connected slave
- SonarQube is accessible for code analysis
- Nexus is hosting artifacts



---

🎯 **Next Steps:**

- Add SonarQube analysis in your Jenkinsfile
- Deploy custom WAR from Maven build to Tomcat
- Add unit tests and reports to Jenkins

## SonarQube Configuration

### Generate Access Token
1. Access SonarQube dashboard:

http://<sonarqube-server-ip>:9000
   - Default credentials: `admin`/`admin`

2. Navigate to:
- Top-right profile icon → "My Account"
- Go to "Security" tab
- Click "Generate Tokens"
- Enter a name (e.g., `jenkins-token`) and generate

3. **Important**: Copy the token and store it securely (it will only be shown once)

---

## Nexus Repository Setup

### Initial Configuration
1. Access Nexus:

http://<nexus-server-ip>:8081

- First-time password is in: `/opt/sonatype-work/nexus3/admin.password`
- Change password when prompted

### Create Maven Repositories
1. Navigate to: ⚙️ Settings → Repositories → Create repository
2. Create these repositories:

| Repository Name   | Type    | Version Policy | Blob Store |
|-------------------|---------|----------------|------------|
| `ezlearn-release`  | hosted  | Release        | default    |
| `ezlearn-snapshot` | hosted  | Snapshot       | default    |
| `ezlearn-public`    | group   | -              | default    |

3. For the group repository (`maven-public`):
- Add both `maven-releases` and `maven-snapshots` as members

---

## Jenkins Configuration

### Install Required Plugins
1. Go to: Dashboard → Manage Jenkins → Plugins → Available
2. Search and install:
- SonarQube Scanner
- Nexus Artifact Uploader
- Deploy to container

### Configure System Settings
1. **SonarQube Server**:
- Manage Jenkins → System → SonarQube servers
- Add server:
  - Name: `SonarQube`
  - Server URL: `http://<sonarqube-ip>:9000`
  - Server authentication token:
  - Click on ADD and select Jenkins
  - Kind = secret text
  - ID = sonar-token
  - Secret = paste token from SonarQube
  - Click Add

2. **Tool Configuration**:
- Manage Jenkins → Tools
- Add SonarQube Scanner installation
- select Install automatically i.e jenkins will install sonarqube for you
- Click Apply and save

### Set Up Credentials
1. **Nexus Credentials**:
- Kind: Username with password
- ID: `nexus-creds`
- Username: `admin`
- Password: [your Nexus admin password]

2. **Tomcat Credentials**:
- Kind: Username with password
- ID: `tomcat-deployer`
- Username: `deployer`
- Password: `deploy123`

---

2. **Install and Configure maven on jenkins slave node**:

```bash
# Update package lists
sudo apt update

# Install required tools
sudo apt install -y wget ca-certificates curl gnupg lsb-release

# Download Apache Maven 3.6.3 from the official archive
wget https://archive.apache.org/dist/maven/maven-3/3.6.3/binaries/apache-maven-3.6.3-bin.tar.gz

# Extract Maven to /opt
sudo tar -xvf apache-maven-3.6.3-bin.tar.gz -C /opt

# Rename to /opt/maven for consistency
sudo mv /opt/apache-maven-3.6.3 /opt/maven

# Create environment variables script
sudo bash -c 'cat <<EOF > /etc/profile.d/maven.sh
export M2_HOME=/opt/maven
export MAVEN_HOME=/opt/maven
export PATH=\$PATH:\$M2_HOME/bin
EOF'

# Make the script executable (optional but good practice)
sudo chmod +x /etc/profile.d/maven.sh

# Load the new environment variables into the current shell
source /etc/profile.d/maven.sh

# Verify installation
mvn -version


```



## Pipeline App Deployment

Please change branch to app-release

### Sample Jenkinsfile
```groovy
pipeline {
 agent any
 environment {
     SONAR_SCANNER = tool 'SonarQubeScanner'
     NEXUS_URL = 'http://<nexus-ip>:8081'
     TOMCAT_URL = 'http://<tomcat-ip>:8080/manager/text'
 }
 stages {
     stage('Build & Test') {
         steps {
             sh 'mvn clean package'
         }
     }
     stage('Code Quality Scan') {
         steps {
             withSonarQubeEnv('SonarQube') {
                 sh """
                     ${SONAR_SCANNER}/bin/sonar-scanner \
                     -Dsonar.projectKey=myapp \
                     -Dsonar.java.binasets=target/classes
                 """
             }
         }
     }
     stage('Deploy to Nexus') {
         steps {
             withCredentials([usernamePassword(
                 credentialsId: 'nexus-creds',
                 usernameVariable: 'NEXUS_USER',
                 passwordVariable: 'NEXUS_PASS'
             )]) {
                 sh '''
                     mvn deploy:deploy-file \
                     -Durl=${NEXUS_URL}/repository/maven-releases/ \
                     -DrepositoryId=nexus \
                     -Dfile=target/*.war \
                     -DgroupId=com.myapp \
                     -DartifactId=myapp \
                     -Dversion=1.0
                 '''
             }
         }
     }
     stage('Deploy to Tomcat') {
         steps {
             withCredentials([usernamePassword(
                 credentialsId: 'tomcat-deployer',
                 usernameVariable: 'TOMCAT_USER',
                 passwordVariable: 'TOMCAT_PASS'
             )]) {
                 sh '''
                     curl -u "${TOMCAT_USER}:${TOMCAT_PASS}" \
                     -T target/*.war \
                     "${TOMCAT_URL}/deploy?path=/myapp&update=true"
                 '''
             }
         }
     }
 }
}
