# CI/CD Pipeline Architecture on AWS
<img width="940" height="365" alt="image" src="https://github.com/user-attachments/assets/2d0e60c2-1347-483b-a1ec-bd371a2bdd16" />


## Brief Documentation Summary

### Components:
- **AWS EC2 Instance (t3.micro)**: The compute host running Jenkins, Ansible, and Docker. Uses default VPC, security groups, and networking. No custom configurations applied.
- **GitHub**: Source code management repository for the Java application. Developers push code here, triggering the CI pipeline.
- **Jenkins**: CI orchestrator installed on the EC2 instance. Uses a Jenkinsfile to define pipeline stages for cloning, building, and deploying.
- **Ansible**: Automation tool running on the EC2 instance. Triggered by Jenkins to execute playbooks that automate deployment steps.
- **Docker**: Containerization tool on the EC2 instance. Builds and runs the application image as a container.

### Steps:
1. **Source Code Management**: Developer commits and pushes Java code to GitHub.
2. **Continuous Integration (CI)**: Jenkins detects the push (via webhook or polling), clones the repo, triggers Ansible playbook, builds Docker image, and runs the container.
3. **Configuration Management & Deployment**: Ansible handles automation (e.g., setup, configs). Docker containerizes and deploys the app on the same EC2.

**Assumptions**: Default AWS settings; all tools on one EC2; no advanced security or scaling.

```markdown
# CI/CD Pipeline for Java Application on AWS

This repository documents a simple CI/CD pipeline for a Java application, deployed on AWS using Jenkins, Ansible, Docker, and GitHub. The entire setup runs on a single AWS EC2 instance for simplicity, with no custom IAM roles, VPCs, or security hardening.

## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Infrastructure Setup](#infrastructure-setup)
- [Tools and Components](#tools-and-components)
- [Workflow Details](#workflow-details)
- [Jenkins Pipeline Configuration](#jenkins-pipeline-configuration)
- [Ansible Playbook Details](#ansible-playbook-details)
- [Docker Configuration](#docker-configuration)
- [Deployment Steps](#deployment-steps)
- [Testing and Validation](#testing-and-validation)
- [Troubleshooting](#troubleshooting)
- [Assumptions and Limitations](#assumptions-and-limitations)
- [Cleanup](#cleanup)

## Overview
This pipeline automates the build and deployment of a Java application:
- **Source Control**: GitHub for code storage.
- **CI Orchestrator**: Jenkins to manage the pipeline.
- **Automation**: Ansible for configuration management and deployment tasks.
- **Containerization**: Docker to package and run the application.
- **Hosting**: AWS EC2 instance (t3.micro) as the single host for all tools and the running container.
```

- **Key Elements**:
  - **Developer**: Initiates the process by pushing code.
  - **GitHub**: Hosts the repository; triggers Jenkins on push.
  - **Jenkins**: Clones repo, runs pipeline stages.
  - **Ansible**: Automates setup and deployment via playbooks.
  - **Docker**: Builds image and runs container.
  - **EC2**: Single instance hosting everything.

## Prerequisites
- AWS account with access to EC2.
- GitHub account and repository for the Java application (e.g., a simple Spring Boot app).
- Basic knowledge of SSH, Linux commands, and YAML/ Groovy syntax.
- Installed tools on local machine: AWS CLI, Git.

## Infrastructure Setup
1. **Launch EC2 Instance**:
   - Go to AWS Console > EC2 > Launch Instance.
   - AMI: Amazon Linux 2 (or Ubuntu 22.04).
   - Instance Type: t3.micro.
   - Network: Default VPC and subnet.
   - Security Group: Allow inbound SSH (port 22), HTTP (port 80/8080 for app), Jenkins (port 8080), and any other needed ports (e.g., Docker).
   - Key Pair: Create or use existing for SSH access.
   - Launch and note the public IP/DNS.

2. **SSH into EC2**:
   ```bash
   ssh -i your-key.pem ec2-user@your-ec2-public-ip
   ```

3. **Install Dependencies on EC2**:
   - Update system: `sudo yum update -y` (Amazon Linux) or `sudo apt update && sudo apt upgrade -y` (Ubuntu).
   - Install Java (for Jenkins and app): `sudo yum install java-11-amazon-corretto -y` or `sudo apt install openjdk-11-jdk -y`.
   - Install Git: `sudo yum install git -y` or `sudo apt install git -y`.

## Tools and Components
| Tool       | Purpose                          | Installation Command (on EC2) |
|------------|----------------------------------|-------------------------------|
| **GitHub** | Source code repository           | N/A (web-based)              |
| **Jenkins**| CI pipeline orchestrator        | See below                    |
| **Ansible**| Configuration automation         | `sudo yum install ansible -y` or `sudo apt install ansible -y` |
| **Docker** | Containerization and runtime     | See below                    |
| **AWS EC2**| Compute host                     | Launched via AWS Console     |

- **Install Jenkins**:
  ```bash
  sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
  sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
  sudo yum upgrade -y
  sudo yum install jenkins -y
  sudo systemctl start jenkins
  sudo systemctl enable jenkins
  ```
  Access Jenkins at `http://your-ec2-ip:8080`. Unlock with initial admin password from `/var/lib/jenkins/secrets/initialAdminPassword`.

- **Install Docker**:
  ```bash
  sudo yum install docker -y  # Amazon Linux
  # or sudo apt install docker.io -y (Ubuntu)
  sudo systemctl start docker
  sudo systemctl enable docker
  sudo usermod -aG docker ec2-user  # Add user to docker group
  ```

## Workflow Details
1. **Developer Pushes Code**: Commit Java app code (e.g., Maven/Gradle build) to GitHub.
2. **Jenkins Trigger**: Configured webhook or poll SCM triggers the pipeline.
3. **Pipeline Execution**:
   - Clone GitHub repo.
   - Trigger Ansible playbook for setup.
   - Build Docker image.
   - Run Docker container.
4. **Deployment**: App runs as a container on EC2.

## Jenkins Pipeline Configuration
Create a `Jenkinsfile` in your GitHub repo root:

```groovy
pipeline {
    agent any
    stages {
        stage('Clone Repository') {
            steps {
                git url: 'https://github.com/your-repo/your-java-app.git', branch: 'main'
            }
        }
        stage('Trigger Ansible') {
            steps {
                ansiblePlaybook playbook: 'deploy.yml', inventory: 'hosts.ini'
            }
        }
    }
}
```

- Configure Jenkins job as Pipeline, pointing to GitHub repo.

## Ansible Playbook Details
Create `deploy.yml` in repo:

```yaml
---
- name: Deploy Java App
  hosts: localhost
  tasks:
    - name: Ensure Maven is installed
      package:
        name: maven
        state: present
    - name: Build Java App
      command: mvn clean package
      args:
        chdir: "{{ playbook_dir }}"
    # Add more tasks for config management, e.g., copy files, set env vars
```

- Inventory `hosts.ini`: `[localhost] localhost ansible_connection=local`

## Docker Configuration
Create `Dockerfile` in repo:

```dockerfile
FROM openjdk:11-jre-slim
COPY target/your-app.jar /app.jar
CMD ["java", "-jar", "/app.jar"]
EXPOSE 8080
```

## Deployment Steps
1. Set up GitHub webhook in Jenkins (under job config > Build Triggers).
2. Push code to GitHub.
3. Monitor Jenkins dashboard for pipeline run.
4. Access app at `http://your-ec2-ip:8080` (assuming app listens on 8080).


## Troubleshooting
- **Jenkins Not Starting**: Check `sudo systemctl status jenkins`.
- **Docker Permission Issues**: Ensure user in docker group; logout/login.
- **Network Errors**: Verify security group allows ports.
- **Ansible Failures**: Run with `-vvv` for verbose output.
- **Common Errors**: Maven build fails? Ensure `pom.xml` is correct. Container not running? Check `docker ps`.


## Cleanup
- Stop/terminate EC2 instance via AWS Console.
- Remove GitHub webhook.
- Delete Docker images/containers: `docker system prune -a`.
```
```
