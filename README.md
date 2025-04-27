# cicd-pipeline-eks-jenkins-argocd
✨ Full automation from code integration to production deployment — cloud-native and DevOps-ready!


---

# CI/CD Pipeline with Jenkins, SonarQube, Docker Hub, ArgoCD, and EKS Kubernetes

## 📌 Objective
Build a complete CI/CD pipeline to automate:
- Code integration (CI) with Git, Jenkins, Maven, and SonarQube.
- Application deployment (CD) with Docker Hub, Kubernetes (EKS), and ArgoCD.

---
## 📚 Project Architecture

### 🚀 CI/CD Pipeline - Overall Flow

![CI/CD Pipeline - Architecture](https://raw.githubusercontent.com/laditrinath321/cicd-pipeline-eks-jenkins-argocd/main/CI_CD%20Pipeline_Architecture.jpeg)

---

## 🚀 Project Summary
This project sets up an **end-to-end Jenkins pipeline** that:
- Pulls code from GitHub.
- Scans code with SonarQube.
- Builds artifacts with Maven.
- Builds and pushes Docker images to Docker Hub.
- Updates Kubernetes manifests.
- Deploys the application automatically using ArgoCD on AWS EKS.

---

## 🛠️ Technologies Used
- **GitHub** (Source Control)
- **Jenkins** (CI Tool)
- **Maven** (Build Tool)
- **SonarQube** (Code Quality Scan)
- **Docker** (Containerization)
- **AWS EKS** (Managed Kubernetes)
- **ArgoCD** (Kubernetes GitOps Deployment)
- **Linux Server** (Jenkins, Docker, SonarQube hosting)

---

## ⚙️ Pre-requisites
- Java Application code in GitHub.
- Linux Server (t2.large recommended).
- AWS account with permissions to create EKS clusters.
- Installed:
  - Jenkins
  - Docker
  - Maven
  - SonarQube
  - AWS CLI
  - kubectl
  - eksctl
  - ArgoCD

---

## 📋 Setup Instructions

### 1. Install Jenkins
```bash
sudo yum update -y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum upgrade -y
sudo dnf install java-17-amazon-corretto -y
sudo yum install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

### 2. Install and Configure Docker
```bash
sudo yum install docker -y
sudo systemctl start docker
sudo usermod -aG docker jenkins
sudo usermod -aG docker ec2-user
sudo systemctl restart docker
sudo chmod 666 /var/run/docker.sock
```

### 3. Configure AWS CLI
```bash
aws configure
```
(Provide your AWS Access Key, Secret Key, Region, and Output Format)

### 4. Install kubectl and eksctl
- [kubectl installation guide](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)
- [eksctl installation guide](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)

Example:
```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

### 5. Create an EKS Cluster
```bash
eksctl create cluster --name mcappcluster --nodegroup-name mcng --node-type t3.micro --nodes 8 --managed
```

### 6. Install Required Jenkins Plugins
- Docker
- Git
- Maven Integration
- Kubernetes CLI
- SonarQube Scanner
- Pipeline

### 7. Install SonarQube using Docker
```bash
docker run -itd --name sonar -p 9000:9000 sonarqube
```
Access SonarQube: http://<your-server-ip>:9000  
Login: admin/admin ➔ Create a SonarQube Token

### 8. Create Jenkinsfile
Place the following `Jenkinsfile` in your GitHub repo:

```groovy
pipeline {
    agent any
    tools {
        maven 'maven3'
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/yourrepo/yourapp.git'
            }
        }
        stage('SonarQube Scan') {
            steps {
                sh '''
                mvn sonar:sonar \
                -Dsonar.host.url=http://your-sonarqube-ip:9000 \
                -Dsonar.login=your-sonarqube-token
                '''
            }
        }
        stage('Build Artifact') {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t yourdockerhubuser/yourapp:${BUILD_NUMBER} .'
            }
        }
        stage('Push to Docker Hub') {
            steps {
                withCredentials([string(credentialsId: 'dockerhub', variable: 'dockerhub')]) {
                    sh 'docker login -u yourdockerhubuser -p ${dockerhub}'
                    sh 'docker push yourdockerhubuser/yourapp:${BUILD_NUMBER}'
                }
            }
        }
        stage('Update Deployment File') {
            steps {
                withCredentials([string(credentialsId: 'githubtoken', variable: 'githubtoken')]) {
                    sh '''
                    git config user.email "your-email@example.com"
                    git config user.name "your-github-username"
                    sed -i "s/yourapp:.*/yourapp:${BUILD_NUMBER}/g" deploymentfiles/deployment.yml
                    git add .
                    git commit -m "Update deployment image to version ${BUILD_NUMBER}"
                    git push https://${githubtoken}@github.com/yourgithubuser/yourrepo HEAD:main
                    '''
                }
            }
        }
    }
}
```

---

## 🛡️ Kubernetes Deployment

**deployment.yml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mc-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mc-app
  template:
    metadata:
      labels:
        app: mc-app
    spec:
      containers:
      - name: mc-app
        image: yourdockerhubuser/yourapp:tag
        ports:
        - containerPort: 8080
```

**service.yml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mc-app-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 8080
  selector:
    app: mc-app
```

---

## 🎯 ArgoCD Setup
- Install ArgoCD:
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

- Patch ArgoCD server for LoadBalancer:
```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

- Get ArgoCD Admin Password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

- Access ArgoCD UI ➔ Login ➔ Create New App ➔ Connect your Git repo ➔ Sync ➔ Deploy!

---

## 📈 Flow Diagram
```
GitHub ➔ Jenkins ➔ SonarQube ➔ Docker ➔ Docker Hub ➔ Update GitHub Manifests ➔ ArgoCD ➔ EKS Kubernetes
```

---

## 🤝 Contact
**DevOps Training Hub**  
📧 Email: devopstraininghub@gmail.com  
📞 Phone: +91 7396627149  
🔗 [Training Link](https://bit.ly/3tSjK0i)

---

