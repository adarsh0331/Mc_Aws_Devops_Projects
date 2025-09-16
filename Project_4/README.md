
# CI/CD Pipeline for Java Application with Jenkins, Docker, SonarQube, and Kubernetes (EKS)

## Project Objective

The primary goal of this project is to establish a robust Continuous Integration and Continuous Deployment (CI/CD) pipeline for a Java application, automating the process of building, testing, and deploying the application to a Kubernetes cluster on AWS Elastic Kubernetes Service (EKS). By integrating modern DevOps tools such as Git, Jenkins, Maven, SonarQube, Docker Hub, ArgoCD, and Kubernetes, the pipeline ensures seamless code integration, quality assurance, and deployment to production.

### Goals
1. **Continuous Integration (CI)**:
   - Automate code integration to merge changes into the main codebase.
   - Perform static code analysis and quality checks using SonarQube.
2. **Continuous Deployment (CD)**:
   - Automate deployment to an EKS Kubernetes cluster.
   - Use Docker for consistent, containerized environments across development, testing, and production.
   - Manage deployment configurations via Git for version control and deploy using ArgoCD.

## Architecture Overview

The CI/CD pipeline automates the following workflow:
- **Code Commit**: Developers push code to a GitHub repository.
- **Jenkins Pipeline**: Triggered by a webhook, Jenkins checks out the code, runs quality checks, builds artifacts, creates Docker images, and updates deployment configurations.
- **SonarQube**: Analyzes code quality and reports issues.
- **Docker Hub**: Stores Docker images for deployment.
- **ArgoCD**: Synchronizes Kubernetes manifests from the Git repository to deploy the application on EKS.

## Prerequisites

### Infrastructure
- **AWS Account**: With permissions to create EKS clusters, EC2 instances, and configure IAM roles.
- **EC2 Instance**: 
  - **Type**: t2.large
  - **AMI**: Amazon Linux 2
  - **Storage**: 50 GB EBS volume
  - **VPC**: Default VPC with public subnet
  - **Security Group**: Allow inbound traffic on:
    - SSH (port 22, restrict to your IP for security)
    - HTTP (port 80)
    - Jenkins (port 8080)
    - SonarQube (port 9000)
- **EKS Cluster**: Managed Kubernetes cluster created via `eksctl`.

### Software Requirements
- **Jenkins**: For pipeline orchestration.
- **Docker**: For containerization.
- **Maven**: For building the Java application.
- **SonarQube**: For code quality analysis (run as a Docker container).
- **kubectl**: Kubernetes CLI for cluster management.
- **eksctl**: For creating and managing EKS clusters.
- **AWS CLI**: For AWS interactions.
- **ArgoCD**: For GitOps-based Kubernetes deployments.
- **GitHub Repository**: Containing the Java application, Dockerfile, and Kubernetes manifests.

## Setup Instructions

### 1. EC2 Instance Setup
1. Launch an EC2 instance with the above specifications.
2. SSH into the instance:
   ```
   ssh -i <your-key.pem> ec2-user@<EC2-Public-IP>
   ```
3. Update the system:
   ```
   sudo yum update -y
   ```

### 2. Install Jenkins
Install and configure Jenkins on the EC2 instance:

```
sudo yum install wget -y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum upgrade -y
sudo dnf install java-17-amazon-corretto -y
sudo yum install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

- Access Jenkins at `http://<EC2-Public-IP>:8080`.
- Unlock Jenkins using the initial admin password from `/var/lib/jenkins/secrets/initialAdminPassword`.
- Install suggested plugins and create an admin user.

### 3. Install Docker
Install and configure Docker to work with Jenkins:

```
sudo yum install docker -y
sudo systemctl start docker
sudo usermod -aG docker jenkins
sudo usermod -aG docker ec2-user
sudo systemctl restart docker
sudo chmod 666 /var/run/docker.sock
```

- Log out and back in as `ec2-user` to apply group changes.
- Verify: `docker --version`.

### 4. Install and Configure AWS CLI
Install AWS CLI and configure credentials:

```
sudo yum install awscli -y
aws configure
```

- Enter your AWS Access Key ID, Secret Access Key, region (e.g., `us-east-1`), and output format (e.g., `json`).

### 5. Install kubectl and eksctl
Install tools for managing Kubernetes clusters:

#### kubectl
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256) kubectl" | sha256sum --check
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

- Verify: `kubectl version --client`.

#### eksctl
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

- Verify: `eksctl version`.

### 6. Create EKS Cluster
Create an EKS cluster using `eksctl`:

```
eksctl create cluster --name mcappcluster --nodegroup-name mcng --node-type t3.micro --nodes 8 --managed
```

- This creates a managed EKS cluster with 8 `t3.micro` nodes.
- Verify cluster creation: `kubectl get nodes`.

### 7. Install SonarQube
Run SonarQube as a Docker container:

```
sudo docker run -itd --name sonar -p 9000:9000 sonarqube
```

- Check container status: `docker ps`.
- Access SonarQube at `http://<EC2-Public-IP>:9000`.
- Log in with default credentials (`admin`/`admin`) and change the password.
- Generate a SonarQube token:
  - Go to **SonarQube Dashboard > Administration > My Account > Security > Generate Token**.
  - Save the token for Jenkins configuration.

### 8. Install ArgoCD
Install ArgoCD in the EKS cluster:

```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

- Expose ArgoCD server as a LoadBalancer:
  ```
  kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
  ```

- Get the external IP:
  ```
  kubectl get svc argocd-server -n argocd
  ```

- Retrieve the initial admin password:
  ```
  kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
  ```

- Access ArgoCD UI at `http://<EXTERNAL-IP>` with username `admin` and the retrieved password.

### 9. Configure Jenkins Plugins
Install necessary plugins via **Manage Jenkins > Manage Plugins > Available**:
- **Docker Plugin**
- **Docker Pipeline**
- **GitHub Integration**
- **SonarQube Scanner**
- **Maven Integration**

Restart Jenkins after installation.

### 10. Configure Credentials in Jenkins
Add credentials in **Manage Jenkins > Manage Credentials > System > Global credentials**:

- **GitHub Token**:
  - Go to **GitHub > Settings > Developer settings > Personal access tokens**.
  - Generate a token with `repo` and `admin:repo_hook` scopes.
  - In Jenkins:
    - **Kind**: Secret text
    - **Secret**: GitHub token
    - **ID**: `github-token`
    - **Description**: GitHub access token

- **SonarQube Token**:
  - **Kind**: Secret text
  - **Secret**: SonarQube token from step 7
  - **ID**: `sonar-token`
  - **Description**: SonarQube authentication token

- **Docker Hub Credentials**:
  - **Kind**: Username with password
  - **Username**: Your Docker Hub username
  - **Password**: Your Docker Hub password
  - **ID**: `dockerhub`
  - **Description**: Docker Hub credentials

### 11. Configure SonarQube in Jenkins
1. Go to **Manage Jenkins > System Configuration > SonarQube Servers**.
2. Add:
   - **Name**: `SonarQube`
   - **Server URL**: `http://<EC2-Public-IP>:9000`
   - **Credentials**: Select `sonar-token`.
3. Go to **Manage Jenkins > Tools > SonarQube Scanner**.
4. Add:
   - **Name**: `SonarScanner`
   - **Install automatically**: Check or specify path.
5. Save.

### 12. Application Structure
The GitHub repository should contain:
- **Java Application**: Source code (e.g., Spring Boot).
- **pom.xml**: Maven configuration for building the app.
- **Dockerfile**: Defines the container runtime.
- **deploymentfiles/**: Directory with Kubernetes manifests (`deployment.yml`, `service.yml`).

#### Sample Dockerfile
```docker
FROM openjdk:17-jdk-slim
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

#### Sample Kubernetes Manifests
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mc-app
  labels:
    app: mc-app
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
        image: devopshubg333/batch13:tag
        ports:
        - containerPort: 8080
```

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mc-app-service
spec:
  type: LoadBalancer
  ports:
  - name: http
    port: 80
    targetPort: 8080
    protocol: TCP
  selector:
    app: mc-app
```

### 13. Jenkins Pipeline
Create a `Jenkinsfile` in the repository root:

```groovy
pipeline {
    agent any
    tools {
        maven 'maven3'
    }
    stages {
        stage('Checkout') {
            steps {
                echo 'Cloning GitHub Repo'
                git credentialsId: 'github-token', url: 'https://github.com/devopstraininghub/mindcircuit13.git'
            }
        }
        stage('SonarQube Scan') {
            steps {
                echo 'Running SonarQube Scan'
                withSonarQubeEnv('SonarQube') {
                    sh 'mvn sonar:sonar'
                }
            }
        }
        stage('Build Artifact') {
            steps {
                echo 'Building Maven Artifact'
                sh 'mvn clean package'
            }
        }
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker Image'
                sh 'docker build -t devopshubg333/batch13:${BUILD_NUMBER} .'
            }
        }
        stage('Push to Docker Hub') {
            steps {
                echo 'Pushing to Docker Hub'
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh 'docker login -u $DOCKER_USER -p $DOCKER_PASS'
                    sh 'docker push devopshubg333/batch13:${BUILD_NUMBER}'
                }
            }
        }
        stage('Update Deployment File') {
            steps {
                echo 'Updating Deployment File'
                withCredentials([string(credentialsId: 'github-token', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                    git config user.email "jenkins@ci.com"
                    git config user.name "Jenkins CI"
                    sed -i "s/batch13:.*/batch13:${BUILD_NUMBER}/g" deploymentfiles/deployment.yml
                    git add .
                    git commit -m "Update deployment image to version ${BUILD_NUMBER}"
                    git push https://${GITHUB_TOKEN}@github.com/devopstraininghub/mindcircuit13.git HEAD:main
                    '''
                }
            }
        }
    }
}
```

### Pipeline Stages Explanation
| Stage                | Description                                                                 |
|----------------------|-----------------------------------------------------------------------------|
| **Checkout**         | Clones the GitHub repository using the provided credentials.                |
| **SonarQube Scan**   | Runs Maven-based SonarQube analysis for code quality and security checks.   |
| **Build Artifact**   | Builds the Java application using `mvn clean package`, producing a JAR/WAR. |
| **Build Docker Image**| Creates a Docker image tagged with the Jenkins build number.                |
| **Push to Docker Hub**| Logs into Docker Hub and pushes the image to the repository.                |
| **Update Deployment File** | Updates the Kubernetes `deployment.yml` with the new image tag and pushes changes to GitHub. |

### 14. ArgoCD Application Configuration
1. Access ArgoCD UI at `http://<EXTERNAL-IP>`.
2. Log in with `admin` and the password from step 8.
3. Create a new application:
   - **Application Name**: `my-app`
   - **Project**: `default`
   - **Sync Policy**: Manual (or Automatic for auto-sync)
   - **Source**:
     - **Repository URL**: `https://github.com/devopstraininghub/mindcircuit13.git`
     - **Revision**: `HEAD`
     - **Path**: `deploymentfiles`
   - **Destination**:
     - **Cluster URL**: `https://kubernetes.default.svc`
     - **Namespace**: `default`
4. Click **Create**.
5. Sync the application:
   - Select the application (`my-app`) in the ArgoCD dashboard.
   - Click **Sync** and then **Synchronize**.
   - Verify the status changes to **Healthy**.

## Accessing the Application
- **Application**: Access via the LoadBalancer external IP from the `mc-app-service` service:
  ```
  kubectl get svc mc-app-service
  ```
  Open `http://<EXTERNAL-IP>` in a browser.
- **Jenkins UI**: `http://<EC2-Public-IP>:8080`
- **SonarQube**: `http://<EC2-Public-IP>:9000`
- **ArgoCD UI**: `http://<ArgoCD-EXTERNAL-IP>`

## Troubleshooting
- **Jenkins issues**: Check logs (`/var/log/jenkins/jenkins.log`) or status (`sudo systemctl status jenkins`).
- **Docker permissions**: Ensure `jenkins` and `ec2-user` are in the `docker` group.
- **EKS cluster issues**: Verify cluster status with `eksctl get cluster` and node status with `kubectl get nodes`.
- **SonarQube not accessible**: Confirm container is running (`docker ps`) and port 9000 is open in the security group.
- **ArgoCD sync failures**: Check ArgoCD logs (`kubectl logs -n argocd <pod-name>`) and ensure manifests are correct.
- **Application not running**: Inspect pod logs (`kubectl logs <pod-name>`) and service status (`kubectl get svc`).

## Next Steps
- Add unit tests and integrate with SonarQube quality gates.
- Implement automated rollback with ArgoCD.
- Set up monitoring with Prometheus and Grafana.
- Configure notifications for pipeline failures (e.g., Slack, email).
- Scale the EKS cluster for production workloads.

For issues, refer to logs or open an issue in the GitHub repository.

</xaiArtifact>
