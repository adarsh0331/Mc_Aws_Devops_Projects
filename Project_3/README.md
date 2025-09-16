## Architecture Diagram

Below is the CI/CD architecture diagram illustrating the flow from code commit to deployment:

<img width="1982" height="720" alt="image" src="https://github.com/user-attachments/assets/2ce9ec72-43eb-4335-932e-70386d4052b5" />


# CI/CD Pipeline for Python Application Deployment on AWS

## Overview

This repository outlines a Continuous Integration and Continuous Deployment (CI/CD) pipeline for deploying a Python web application on AWS using an EC2 instance. The pipeline automates the process of building, testing, and deploying the application using Jenkins, Docker, and Maven, with source code managed in a Git repository (GitHub or GitLab). The application is containerized and deployed on the same EC2 instance, accessible via HTTP on port 80.

The architecture leverages AWS infrastructure, Jenkins for orchestration, Docker for containerization, and Maven for build automation, ensuring a streamlined deployment process.

### Diagram Explanation
- **Developer**: Commits Python application code and Dockerfile to a Git repository (GitHub/GitLab).
- **Git Repository**: Hosts the source code and triggers Jenkins via a webhook on code push.
- **Jenkins**: Installed on an EC2 instance, orchestrates the pipeline with stages for checkout, build, test, and deployment.
- **EC2 Instance**: A t2.micro instance in a default VPC’s public subnet, hosting Jenkins, Docker, and the application container.
- **Docker Build**: Builds a Docker image from the Dockerfile.
- **Docker Run**: Deploys the container on the EC2 instance, exposing the application on port 80.
- **Security Group**: Allows SSH (22), HTTP (80), and Jenkins (8080) traffic.
- **Public Access**: The application is accessible via the EC2 public IP on port 80; Jenkins UI is accessible on port 8080.

## Prerequisites

- **AWS Account**: With permissions to create EC2 instances and configure networking.
- **Git Repository**: Hosted on GitHub or GitLab, containing the Python application and Dockerfile.
- **EC2 Instance**: t2.micro, Amazon Linux 2 AMI (or compatible), in a default VPC with a public subnet.
- **IAM Role** (optional): Attach to EC2 for S3/CloudWatch access if needed.
- **Tools**: Git, Maven, Docker, and Jenkins installed on the EC2 instance.

## AWS Infrastructure Setup

1. **Launch EC2 Instance**:
   - **Instance Type**: t2.micro
   - **AMI**: Amazon Linux 2
   - **VPC**: Default VPC, public subnet
   - **Key Pair**: Create or use an existing SSH key pair
   - **Security Group**:
     - Allow inbound traffic:
       - SSH: Port 22, Source: Your IP or 0.0.0.0/0 (restrict for security)
       - HTTP: Port 80, Source: 0.0.0.0/0
       - Jenkins: Port 8080, Source: Your IP or 0.0.0.0/0 (restrict for security)
   - **IAM Role** (optional): Attach a role with policies for S3/CloudWatch if required.

2. **SSH into EC2**:
   ```
   ssh -i <your-key.pem> ec2-user@<EC2-Public-IP>
   ```

3. **Update System**:
   ```
   sudo yum update -y
   ```

## Installing Dependencies

### Install Jenkins
Install Jenkins on the EC2 instance:

```
sudo yum install wget -y
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum upgrade -y
sudo dnf install java-17-amazon-corretto -y
sudo yum install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

- Access Jenkins at `http://<EC2-Public-IP>:8080`.
- Unlock using the initial admin password from `/var/lib/jenkins/secrets/initialAdminPassword`.
- Install suggested plugins and create an admin user.

### Install Docker
Install and configure Docker:

```
sudo yum install -y docker
sudo service docker start
sudo usermod -aG docker ec2-user
sudo usermod -a -G docker jenkins
```

- Log out and back in as `ec2-user` to apply group changes.
- Verify: `docker --version`.

### Install Git
Ensure Git is installed:

```
sudo yum install -y git
```

- Verify: `git --version`.

## Jenkins Configuration

1. **Install Plugins**:
   - Go to **Manage Jenkins > Manage Plugins > Available**.
   - Install:
     - **Git Plugin**: For repository integration
     - **Pipeline: Stage View**: Visualize pipeline stages
     - **Docker Pipeline**: For Docker build/run steps
   

3. **Configure Git Repository**:
   - Ensure Jenkins has access to your GitHub/GitLab repository.
   - Add repository credentials (if private) in **Manage Jenkins > Manage Credentials > System > Global credentials**.
     - **Kind**: Username with password
     - **ID**: `git-credentials` (or any identifier)
     - **Username/Password**: Your GitHub/GitLab credentials

## Application Structure

The Git repository should contain:

- **Python Application**: Core application code (e.g., Flask or FastAPI).
- **Dockerfile**: Defines the Python runtime environment.


### Sample Dockerfile
If not already present, use the following example:
```

FROM python:3.9-slim
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
EXPOSE 80
CMD ["python", "app.py"]

```
- Assumes `app.py` is the main application file and `requirements.txt` lists dependencies.
- Update `CMD` and `EXPOSE` based on your application.


## Jenkins Pipeline Configuration

Create a new Pipeline job in Jenkins (e.g., `python-cicd`):
- **Source Code Management**: Git
  - **Repository URL**: Your GitHub/GitLab repository
  - **Credentials**: Select `git-credentials`
- **Build Triggers**: Check `GitHub hook trigger for GITScm polling` (or equivalent for GitLab)
- **Pipeline**: Pipeline script (inline or from SCM)

Use the following Jenkinsfile:

```

pipeline {
    agent any
   
    stages {
        stage('Code Checkout') {
            steps {
                git credentialsId: 'git-credentials', url: '<your-repo-url>'
                echo 'Checked out code from Git'
            }
        }
       
        stage('Docker Build') {
            steps {
                echo 'Building Docker image'
                sh 'docker build -t python-app:${BUILD_NUMBER} .'
            }
        }
        stage('Docker Run') {
            steps {
                echo 'Deploying container'
                sh 'docker stop python-app || true'
                sh 'docker rm python-app || true'
                sh 'docker run -d --name python-app -p 80:80 python-app:${BUILD_NUMBER}'
            }
        }
    }
    post {
        always {
            echo 'Cleaning up'
            // Optional cleanup steps
        }
    }
}

```

### Pipeline Stages Explanation
| Stage            | Description                                                                 |
|------------------|-----------------------------------------------------------------------------|
| Code Checkout    | Pulls code from the Git repository using provided credentials.               |
| Docker Build     | Builds a Docker image tagged with the Jenkins build number.                  |
| Docker Run       | Stops/Removes any existing container and runs a new one on port 80.          |

## Accessing the Application

- **Application**: `http://<EC2-Public-IP>:5000`
- **Jenkins UI**: `http://<EC2-Public-IP>:8080`
- **SSH Access**: Use your key pair for troubleshooting.

## Troubleshooting

- **Jenkins UI inaccessible**: Verify security group allows port 8080; check `sudo systemctl status jenkins`.
- **Docker issues**: Ensure `jenkins` user is in `docker` group; restart Jenkins.
- **Git issues**: Validate repository URL and credentials; test webhook connectivity.
- **Application not running**: Check Docker container logs: `docker logs python-app`.
- **Port conflicts**: Ensure no other service uses port 80 or 8080.

## Next Steps

- Add unit tests with `pytest` and integrate into the pipeline.
- Push Docker images to a registry (e.g., Docker Hub, Amazon ECR).
- Implement monitoring with CloudWatch.
- Scale with ECS or EKS for production.
- Add notifications for pipeline failures (e.g., email, Slack).

For issues, check Jenkins logs (`/var/log/jenkins/jenkins.log`) or open a GitHub/GitLab issue.
