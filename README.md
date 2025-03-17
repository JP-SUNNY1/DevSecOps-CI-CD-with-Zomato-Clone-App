# Zomato Clone App with DevSecOps CI/CD

![Image](1.png)

**Hello friends, we will be deploying a React Js Zomato-clone. We will be using Jenkins as a CI/CD tool and deploying our application on a Docker container. I hope this detailed blog is useful.**

**Git repo:** [Zomato-Clone](https://github.com/Aj7Ay/Zomato-Clone.git)

![Image](2.png)

## Steps:

1. **Launch an Ubuntu (22.04) T2 Large Instance**
2. **Install Jenkins, Docker, and Trivy. Create a Sonarqube Container using Docker.**
3. **Install Plugins like JDK, Sonarqube Scanner, Nodejs, and OWASP Dependency Check.**
4. **Create a Pipeline Project in Jenkins using a Declarative Pipeline**
5. **Install OWASP Dependency Check Plugins**
6. **Docker Image Build and Push**
7. **Deploy the image using Docker**
8. **Terminate the AWS EC2 Instances**

Now, let’s get started and dig deeper into each of these steps.

---

### **Step 1: Launch an Ubuntu (22.04) T2 Large Instance**

Launch an AWS T2 Large Instance. Use the image as Ubuntu. You can create a new key pair or use an existing one. Enable HTTP and HTTPS settings in the Security Group and open all ports (not best case to open all ports but just for learning purposes it’s okay).

![Image](3.png)

---

### **Step 2: Install Jenkins, Docker, and Trivy**

#### **2A — Install Jenkins**

Connect to your console, and enter these commands to install Jenkins:

```bash
vi jenkins.sh
```

Add the following script:

```bash
#!/bin/bash
sudo apt update -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION\_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins
sudo chmod 777 jenkins.sh
./jenkins.sh
```

Once Jenkins is installed, you will need to go to your AWS EC2 Security Group and open Inbound Port 8080, since Jenkins works on Port 8080.

Now, grab your Public IP Address:

```
EC2 Public IP Address:8080
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

![Image](4.png)

Unlock Jenkins using an administrative password and install the suggested plugins.

![Image](5.png)

Jenkins will now get installed and install all the libraries.

![Image](6.png)

Create a user, click on save and continue.

**Jenkins Getting Started Screen:**

![Image](7.png)

---

#### **2B — Install Docker**

```bash
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER
newgrp docker
sudo chmod 777 /var/run/docker.sock
```

After the Docker installation, we create a Sonarqube container (Remember to add port 9000 in the security group).

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

![Image](8.png)

Now, our Sonarqube is up and running.

![Image](9.png)

Enter username and password, click on login, and change the password.

- **Username:** admin
- **Password:** admin

![Image](10.png)

Update the new password. This is the Sonar Dashboard.

![Image](11.png)

---

#### **2C — Install Trivy**

```bash
vi trivy.sh
```

Add the following script:

```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb\_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```

Next, we will log in to Jenkins and start to configure our Pipeline in Jenkins.

---

### **Step 3: Install Plugins like JDK, Sonarqube Scanner, NodeJs, OWASP Dependency Check**

#### **3A — Install Plugin**

Go to **Manage Jenkins → Plugins → Available Plugins →**

Install the following plugins:

1. **Eclipse Temurin Installer** (Install without restart)
2. **SonarQube Scanner** (Install without restart)
3. **NodeJs Plugin** (Install without restart)

![Image](12.png)
![Image](13.png)

---

#### **3B — Configure Java and Nodejs in Global Tool Configuration**

Go to **Manage Jenkins → Tools → Install JDK (17) and NodeJs (16) → Click on Apply and Save**

![Image](14.png)
![Image](15.png)

---

#### **3C — Create a Job**

Create a job as **Zomato**, select **Pipeline**, and click on **OK**.

---

### **Step 4: Configure Sonar Server in Manage Jenkins**

Grab the Public IP Address of your EC2 Instance. Sonarqube works on Port 9000, so `<Public IP>:9000`. Go to your Sonarqube Server. Click on **Administration → Security → Users → Click on Tokens and Update Token → Give it a name → and click on Generate Token**.

![Image](16.png)

Click on **Update Token**.

![Image](17.png)

Create a token with a name and generate.

![Image](18.png)

Copy the token.

Go to **Jenkins Dashboard → Manage Jenkins → Credentials → Add Secret Text**. It should look like this:

![Image](19.png)

You will see this page once you click on create.

![Image](20.png)

Now, go to **Dashboard → Manage Jenkins → System** and add like the below image.

![Image](21.png)

Click on **Apply and Save**.

**The Configure System option** is used in Jenkins to configure different servers.

**Global Tool Configuration** is used to configure different tools that we install using Plugins.

We will install a Sonar Scanner in the tools.

![Image](22.png)

In the Sonarqube Dashboard, add a quality gate also.

**Administration → Configuration → Webhooks**

![Image](23.png)

Click on **Create**.

![Image](24.png)

Add details.

```bash
# In URL section of quality gate
http://jenkins-public-ip:8080/sonarqube-webhook/
```

![Image](25.png)

Let’s go to our Pipeline and add the script in our Pipeline Script.

```groovy
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/Aj7Ay/Zomato-Clone.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=zomato \
                    -Dsonar.projectKey=zomato '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
    }
}
```

Click on **Build now**, you will see the stage view like this:

![Image](26.png)

To see the report, you can go to the Sonarqube Server and go to **Projects**.

![Image](27.png)

You can see the report has been generated and the status shows as passed. You can see that there are 1.3k lines. To see a detailed report, you can go to **Issues**.

---

### **Step 5: Install OWASP Dependency Check Plugins**

Go to **Dashboard → Manage Jenkins → Plugins → OWASP Dependency-Check**. Click on it and install it without restart.

![Image](28.png)

First, we configured the Plugin, and next, we had to configure the Tool.

Go to **Dashboard → Manage Jenkins → Tools →**

![Image](29.png)

Click on **Apply and Save** here.

Now, go to **Configure → Pipeline** and add this stage to your pipeline and build.

```groovy
stage('OWASP FS SCAN') {
    steps {
        dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
    }
}
stage('TRIVY FS SCAN') {
    steps {
        sh "trivy fs . > trivyfs.txt"
    }
}
```

The stage view would look like this:

![Image](30.png)

You will see that in status, a graph will also be generated and Vulnerabilities.

![Image](31.png)

---

### **Step 6: Docker Image Build and Push**

We need to install the Docker tool in our system. Go to **Dashboard → Manage Plugins → Available plugins → Search for Docker** and install these plugins:

- Docker
- Docker Commons
- Docker Pipeline
- Docker API
- docker-build-step

and click on **Install without restart**.

![Image](32.png)

Now, go to **Dashboard → Manage Jenkins → Tools →**

![Image](33.png)

Add DockerHub Username and Password under **Global Credentials**.

![Image](34.png)

Add this stage to the Pipeline Script:

```groovy
stage("Docker Build & Push"){
    steps{
        script{
            withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                sh "docker build -t zomato ."
                sh "docker tag zomato sevenajay/zomato:latest"
                sh "docker push sevenajay/zomato:latest"
            }
        }
    }
}
stage("TRIVY"){
    steps{
        sh "trivy image sevenajay/zomato:latest > trivy.txt"
    }
}
```

You will see the output below, with a dependency trend.

![Image](35.png)

When you log in to DockerHub, you will see a new image is created.

Now, run the container to see if the app is coming up or not by adding the below stage:

```groovy
stage('Deploy to container'){
    steps{
        sh 'docker run -d --name zomato -p 3000:3000 sevenajay/zomato:latest'
    }
}
```

**Stage view:**

![Image](36.png)

**<Jenkins-public-ip:3000>**

You will get this output:

![Image](37.png)

---

### **Step 8: Terminate Instances**

---

### **Complete Pipeline**

```groovy
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/Aj7Ay/Zomato-Clone.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=zomato \
                    -Dsonar.projectKey=zomato '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                    withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                        sh "docker build -t zomato ."
                        sh "docker tag zomato sevenajay/zomato:latest"
                        sh "docker push sevenajay/zomato:latest"
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image sevenajay/zomato:latest > trivy.txt"
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name zomato -p 3000:3000 sevenajay/zomato:latest'
            }
        }
    }
}
