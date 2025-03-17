# Zomato Clone App with DevSecOps CI/CD

![1](https://github.com/user-attachments/assets/a736d0eb-d8cc-4eed-898a-57d0e84c77ac)


**Hello friends, we will be deploying a React Js Zomato-clone. We will be using Jenkins as a CI/CD tool and deploying our application on a Docker container. I hope this detailed blog is useful.**

**Git repo:** [Zomato-Clone](https://github.com/Aj7Ay/Zomato-Clone.git)

![2](https://github.com/user-attachments/assets/19b0c2c2-a44f-40d0-bf66-f1cdd15f4747)


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

![3](https://github.com/user-attachments/assets/ccc4ac8a-d2c0-47b7-a7bf-172730e56b08)

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

![4](https://github.com/user-attachments/assets/097a8e1e-e764-48e4-a918-a24fd1341bcb)



Unlock Jenkins using an administrative password and install the suggested plugins.

![5](https://github.com/user-attachments/assets/5e5d83cd-b1a1-4f6d-a652-396f9fa04685)



Jenkins will now get installed and install all the libraries.

![6](https://github.com/user-attachments/assets/4e51e6d2-301a-4398-aaeb-3254993f2fa9)


Create a user, click on save and continue.

**Jenkins Getting Started Screen:**

![7](https://github.com/user-attachments/assets/b9aa6015-25d3-48a6-8895-e70ef42624ee)

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

![8](https://github.com/user-attachments/assets/b94730d4-6290-4786-9d15-055ed9535998)


Now, our Sonarqube is up and running.

![9](https://github.com/user-attachments/assets/a2a48bb6-403d-40f2-93f7-9799efa6eae4)


Enter username and password, click on login, and change the password.

- **Username:** admin
- **Password:** admin

![10](https://github.com/user-attachments/assets/a2959c89-589f-4d7e-b9d3-3979e5517de0)


Update the new password. This is the Sonar Dashboard.

![11](https://github.com/user-attachments/assets/7e5e4134-c321-4060-84c0-f6de71f72d8e)


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

![12](https://github.com/user-attachments/assets/9ce12b5a-7c9a-4d22-9ee4-1e507e2fbdc4)
![13](https://github.com/user-attachments/assets/cdb031f0-9b41-47bb-b6ef-846aca3db463)


---

#### **3B — Configure Java and Nodejs in Global Tool Configuration**

Go to **Manage Jenkins → Tools → Install JDK (17) and NodeJs (16) → Click on Apply and Save**

![14](https://github.com/user-attachments/assets/6a763e34-d8df-405a-9026-704d9df31f2b)
![15](https://github.com/user-attachments/assets/24e9e666-a795-42af-912a-72029e337234)



---

#### **3C — Create a Job**

Create a job as **Zomato**, select **Pipeline**, and click on **OK**.

---

### **Step 4: Configure Sonar Server in Manage Jenkins**

Grab the Public IP Address of your EC2 Instance. Sonarqube works on Port 9000, so `<Public IP>:9000`. Go to your Sonarqube Server. Click on **Administration → Security → Users → Click on Tokens and Update Token → Give it a name → and click on Generate Token**.

![16](https://github.com/user-attachments/assets/08b92f4a-f044-40a3-9aff-4dc3ee919d43)


Click on **Update Token**.

![17](https://github.com/user-attachments/assets/a8a51d16-8d14-4083-a01a-7978f882a01e)


Create a token with a name and generate.

![18](https://github.com/user-attachments/assets/9c0ebbd5-d0bc-4666-82bb-464e9d529dc2)


Copy the token.

Go to **Jenkins Dashboard → Manage Jenkins → Credentials → Add Secret Text**. It should look like this:

![19](https://github.com/user-attachments/assets/a16ccbd1-0f39-43b8-831e-e0b4488257d9)


You will see this page once you click on create.

![20](https://github.com/user-attachments/assets/54354f5b-cda8-45f0-8c42-97d63b1f0f41)


Now, go to **Dashboard → Manage Jenkins → System** and add like the below image.

![21](https://github.com/user-attachments/assets/fd9e6330-7531-4675-9b04-624b0d68ed4f)


Click on **Apply and Save**.

**The Configure System option** is used in Jenkins to configure different servers.

**Global Tool Configuration** is used to configure different tools that we install using Plugins.

We will install a Sonar Scanner in the tools.

![22](https://github.com/user-attachments/assets/9a5af8b5-755b-480f-8f93-e51fdb6dbb86)

In the Sonarqube Dashboard, add a quality gate also.

**Administration → Configuration → Webhooks**

![23](https://github.com/user-attachments/assets/2a26c4b0-9331-47bc-b7f6-8ce958b45c44)


Click on **Create**.

![24](https://github.com/user-attachments/assets/82d5805d-94f6-4a21-ab86-9b21202f20bd)

Add details.

```bash
# In URL section of quality gate
http://jenkins-public-ip:8080/sonarqube-webhook/
```

![25](https://github.com/user-attachments/assets/aa695794-c5f8-4587-b176-4eb576d0ab5e)

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

![26](https://github.com/user-attachments/assets/889a7ab0-22dc-49c1-9f7d-4d50d0b4efe7)

To see the report, you can go to the Sonarqube Server and go to **Projects**.

![27](https://github.com/user-attachments/assets/20b912a6-0f2b-4487-8afb-7e0bfca4d3e3)

You can see the report has been generated and the status shows as passed. You can see that there are 1.3k lines. To see a detailed report, you can go to **Issues**.

---

### **Step 5: Install OWASP Dependency Check Plugins**

Go to **Dashboard → Manage Jenkins → Plugins → OWASP Dependency-Check**. Click on it and install it without restart.

![28](https://github.com/user-attachments/assets/7eae8cfd-8ebd-410f-95b3-abfc32659155)

First, we configured the Plugin, and next, we had to configure the Tool.

Go to **Dashboard → Manage Jenkins → Tools →**

![29](https://github.com/user-attachments/assets/7a9df491-464b-4809-8c5c-9a8ec7ebf701)

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

![30](https://github.com/user-attachments/assets/52a61923-bf44-4790-9bd7-8647592da995)

You will see that in status, a graph will also be generated and Vulnerabilities.

![31](https://github.com/user-attachments/assets/0e52e09a-d1a4-40c2-8c26-0fa023a772ff)

---

### **Step 6: Docker Image Build and Push**

We need to install the Docker tool in our system. Go to **Dashboard → Manage Plugins → Available plugins → Search for Docker** and install these plugins:

- Docker
- Docker Commons
- Docker Pipeline
- Docker API
- docker-build-step

and click on **Install without restart**.

![32](https://github.com/user-attachments/assets/2b954363-0b93-4124-a2b5-aa917c36be9c)

Now, go to **Dashboard → Manage Jenkins → Tools →**

![33](https://github.com/user-attachments/assets/977db2b8-5368-45ec-a7b8-fbedb2797e14)

Add DockerHub Username and Password under **Global Credentials**.

![34](https://github.com/user-attachments/assets/5812b1e0-30ba-4f63-80f7-ce174baf8908)

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

![35](https://github.com/user-attachments/assets/cd6343de-d5c2-4d8c-8fe0-a0ecd1f461bc)

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

![36](https://github.com/user-attachments/assets/ad3099b4-a899-47a3-b667-19b517e4078d)

**<Jenkins-public-ip:3000>**

You will get this output:

![37](https://github.com/user-attachments/assets/7bb18974-d983-4cab-b67c-2cb0ee0db0fb)

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
