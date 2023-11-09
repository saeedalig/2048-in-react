## Deploying a React.js 2048 Game using a CI/CD pipeline with Jenkins and deploying it on a Kubernetes cluster. The goal is to automate the build, test, and deployment processes to ensure a streamlined and reliable deployment of the application.

## Workflow of Project
In this project I am utilizing **Git, Jenkins, SonarQube, Maven, Trivy, OWASP, Docker and Kubernetes**. 

The project involves deploying a React.js 2048 game using a comprehensive CI/CD pipeline with several essential tools. It begins with version control using Git, where the source code is stored and tracked. Jenkins, the CI/CD orchestrator, monitors the Git repository for changes and triggers a series of automated steps. The React.js application is built, dependencies are installed, and production-ready bundles are generated. SonarQube is integrated to analyze code quality and ensure adherence to coding standards. Docker is employed to containerize the application. The image built with the help of Docker is then scanned by Trivy to check vulnerabilities. The project also integrates OWASP for security checks, ensuring the game's resilience against potential threats. Docker images are pushed to a container registry, making them available for deployment. Kubernetes is then utilized for continuous deployment, orchestrating and scaling containers. 

The pipeline provides a robust and automated development and deployment process for the React.js 2048 game.

## Server Setup (Installation)
- **AWS EC2 Instance:** t2.medium 
- **Jenkins**
```
sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y

# Password
sudo cat /var/lib/jenkins/secrets/initialAdminPassword

```

- **Docker**
```
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER   #my case is ubuntu
newgrp docker
sudo chmod 777 /var/run/docker.sock

```

- **Sonarqube**
```
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```
You can access Sonarqube on port `9000`. Username and Password is same `admin`

- **Trivy**

```

sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y

```

- **Install Plugins** 
    - JDK
    - Sonarqube Scanner
    - NodeJs
    - OWASP Dependency Check
    - Docker
    - Kubernetes

 and **configure** them.

- ## Kuberenetes Setup (kubeadm) -> Master and Worker Node
  
**Installing Kubectl**
```
sudo apt update
sudo apt install curl
curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
```


**Commands to be executed on both master and worker node**
```
sudo apt-get update 

sudo apt-get install -y docker.io
sudo usermod –aG docker Ubuntu
newgrp docker
sudo chmod 777 /var/run/docker.sock

sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

sudo tee /etc/apt/sources.list.d/kubernetes.list <<EOF
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo snap install kube-apiserver

```

**Only on Master node** without root priviliges
```
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

**Only on Worker node**
```
sudo kubeadm join <master-node-ip>:<master-node-port> --token <token> --discovery-token-ca-cert-hash <hash>
```

## Pipeline Script
```
pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
        DOCKERHUB = "asa96"
        APP_NAME = "2048-game"
        IMAGE_TAG = "${BUILD_NUMBER}"
        IMAGE_NAME = "${DOCKERHUB}" + "/" + "${APP_NAME}"
        REGISTRY_CREDS = "dockerhub"
    }

    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }

        stage('Git Checkout'){
            steps{
                git branch: 'master', url: 'https://github.com/saeedalig/2048-in-react.git'
            }
        }

        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Game \
                    -Dsonar.projectKey=Game '''
                }
            }
        }

        stage("Quality Gate"){
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

        stage('Build Docker Image'){
            steps{
                script {
                    docker_image = docker.build("${IMAGE_NAME}")
                }
            }
        }
        
        stage('Push Docker Image'){
            steps {
                script{
                    docker.withRegistry('', REGISTRY_CREDS ){
                        docker_image.push("${BUILD_NUMBER}")
                        docker_image.push('latest')
                    }
                }
            }
        }

        stage("TRIVY Scan Image"){
            steps{
                sh "trivy image sevenajay/2048:latest > trivy.txt" 
            }
        }

        stage('Deploy to kubernets'){
            steps{
                script{
                    withKubeConfig(caCertificate: '',
                        clusterName: '',
                        contextName: '',
                        credentialsId: 'k8s',
                        namespace: '',
                        restrictKubeConfigAccess: false,
                        serverUrl: '') {
                           sh 'kubectl apply -f deployment.yaml'
                    }
                }
            }
        }
    }
}

```
