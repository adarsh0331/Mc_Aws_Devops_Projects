---

# DevOps CI/CD Pipeline on AWS with Terraform and Kubernetes

This project implements a CI/CD pipeline for a Node.js application, deployed on AWS infrastructure using Terraform-provisioned EC2 instances and a manually created Kubernetes cluster. The pipeline automates code scanning, building, containerization, and deployment, with monitoring via Prometheus and Grafana. ArgoCD enables GitOps-based deployments to Kubernetes. This `README.md` provides a comprehensive guide to the architecture, workflow, tools, and setup instructions.

## Architecture Diagram

The following Mermaid diagram illustrates the CI/CD pipeline, deployment, and monitoring flows. Render it in a compatible tool (e.g., GitHub, Mermaid Live) to visualize tool icons and connections. Arrows indicate data flow between components.

```mermaid
graph TD
    subgraph AWS
        subgraph EC2 Instances
            EC2_1[EC2 Instance 1 (t2.large): Jenkins, Docker, SonarQube, Trivy, NPM]
            EC2_2[EC2 Instance 2 (t2.xlarge): Prometheus, Grafana (via Helm)]
        end
        ECR[AWS ECR (Docker Image Registry)]
        K8s[Kubernetes Cluster (Manual CLI Creation): ArgoCD Installed]
    end
    GitHub[GitHub Repository]

    %% CI/CD Flow
    GitHub -->|1. Code Commit Triggers| Jenkins[Jenkins on EC2_1]
    Jenkins -->|2. Clone Code| GitHub
    Jenkins -->|3. Code Scan| SonarQube[SonarQube on EC2_1]
    SonarQube -->|Pass| NPM[NPM on EC2_1: Build Node.js App]
    NPM -->|Built Artifacts| Docker[Docker on EC2_1: Build Image]
    Docker -->|4. Security Scan| Trivy[Trivy on EC2_1: Scan Image]
    Trivy -->|Pass| ECR|5. Push Image|
    ECR -->|Image Pushed| GitHub|6. Update deployment.yaml|
    GitHub -->|GitOps Sync| ArgoCD[ArgoCD in K8s: Auto-Sync & Deploy]
    ArgoCD -->|7. Deploy App| K8s

    %% Monitoring Flow
    K8s -->|Scrape Metrics| Prometheus[Prometheus on EC2_2]
    Prometheus -->|Visualize| Grafana[Grafana on EC2_2]
```

## Infrastructure

The infrastructure is hosted on AWS, with EC2 instances provisioned via Terraform and a Kubernetes cluster created manually. Below is a detailed breakdown of components:

### EC2 Instance 1 (t2.large)
- **Purpose**: Hosts CI/CD and build tools.
- **Tools Installed**:
  - **Jenkins**: CI/CD server for pipeline orchestration (port 8080).
  - **Docker**: Builds and manages container images.
  - **SonarQube**: Performs static code analysis for quality and security (port 9000).
  - **Trivy**: Scans Docker images for vulnerabilities.
  - **NPM**: Builds Node.js applications.
- **Terraform Configuration**:
  - Instance type: `t2.large` (2 vCPUs, 8 GB RAM).
  - AMI: Latest Amazon Linux 2 or Ubuntu 20.04.
  - Security groups: Allow inbound traffic on ports 8080 (Jenkins), 9000 (SonarQube), and 22 (SSH).
  - IAM role: Grants access to AWS ECR for pushing/pulling images.
  - User data script: Installs Jenkins, Docker, SonarQube, Trivy, and Node.js/NPM during instance bootstrap.

### EC2 Instance 2 (t2.xlarge)
- **Purpose**: Hosts monitoring tools.
- **Tools Installed**:
  - **Prometheus**: Collects metrics from Kubernetes and the application (port 9090).
  - **Grafana**: Visualizes metrics via dashboards (port 3000).
- **Terraform Configuration**:
  - Instance type: `t2.xlarge` (4 vCPUs, 16 GB RAM) for better performance with monitoring workloads.
  - AMI: Same as EC2 Instance 1.
  - Security groups: Allow inbound traffic on ports 9090 (Prometheus), 3000 (Grafana), and 22 (SSH).
  - User data script: Installs Helm, then deploys Prometheus and Grafana via Helm charts.
- **Helm Charts**:
  - Prometheus: `helm install prometheus prometheus-community/prometheus --set server.service.type=NodePort`.
  - Grafana: `helm install grafana grafana/grafana --set service.type=NodePort`.

### Kubernetes Cluster
- **Setup**: Manually created using AWS CLI (e.g., via `eksctl create cluster` or EKS API).
- **Configuration**:
  - Minimum 3 worker nodes (e.g., `t3.medium`) for high availability.
  - VPC and subnets configured to allow communication with EC2 instances.
  - ArgoCD installed in the cluster for GitOps deployments (`kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml`).
  - Configured to pull images from AWS ECR using an IAM role for the Kubernetes nodes.
- **Networking**: Security groups allow Kubernetes API access and communication with Prometheus for metrics scraping.

### AWS ECR
- **Purpose**: Private Docker registry for storing application images.
- **Setup**: Created via Terraform or AWS CLI (`aws ecr create-repository --repository-name myapp`).
- **IAM**: EC2 Instance 1 has permissions to push images; Kubernetes nodes have pull permissions.

### GitHub Repository
- **Purpose**: Stores source code, Dockerfiles, and Kubernetes manifests (`deployment.yaml`, `service.yaml`).
- **Webhook**: Configured to trigger Jenkins on code commits to the main branch.
- **Access**: Jenkins authenticates via SSH or GitHub API token.

### Terraform Setup
- **Modules**: Separate modules for VPC, EC2 instances, security groups, and IAM roles.
- **Directory Structure**:
  ```
  terraform/
  ├── main.tf
  ├── variables.tf
  ├── outputs.tf
  ├── modules/
  │   ├── vpc/
  │   ├── ec2/
  │   ├── iam/
  ```
- **Example `main.tf`** (simplified):
  ```hcl
  module "vpc" {
    source  = "./modules/vpc"
    cidr_block = "10.0.0.0/16"
  }

  module "ec2_instance_1" {
    source        = "./modules/ec2"
    instance_type = "t2.large"
    ami           = "ami-12345678"
    security_groups = [module.vpc.jenkins_sg]
    user_data     = file("scripts/install_ci_tools.sh")
  }

  module "ec2_instance_2" {
    source        = "./modules/ec2"
    instance_type = "t2.xlarge"
    ami           = "ami-12345678"
    security_groups = [module.vpc.monitoring_sg]
    user_data     = file("scripts/install_monitoring_tools.sh")
  }
  ```
- **Apply**: Run `terraform init`, `terraform plan`, and `terraform apply` to provision resources.

## CI/CD Workflow

The CI/CD pipeline automates building, testing, and deploying a Node.js application using Jenkins as the orchestrator. The pipeline is defined in a `Jenkinsfile` (declarative pipeline) in the GitHub repository.

### Pipeline Stages
1. **Trigger on Commit**:
   - A GitHub webhook triggers Jenkins on new commits to the `main` branch.
   - Webhook URL: `http://<jenkins-ip>:8080/github-webhook/`.
2. **Clone Code**:
   - Jenkins clones the repository using Git credentials.
3. **Code Quality Scan**:
   - SonarQube scans the code for bugs, vulnerabilities, and code smells.
   - Configured via `sonar-project.properties` in the repo.
   - Fails if quality gates (e.g., coverage < 80%) are not met.
4. **Build Application**:
   - NPM installs dependencies (`npm install`) and builds the app (`npm run build`).
   - Artifacts are stored temporarily for Docker.
5. **Security Scan (Optional)**:
   - Trivy scans the Docker image for vulnerabilities (`trivy image myapp:latest`).
   - Configured to fail on critical or high-severity vulnerabilities.
6. **Build and Push Docker Image**:
   - Docker builds the image (`docker build -t myapp:$BUILD_NUMBER .`).
   - Tags and pushes to ECR (`docker push <aws-account-id>.dkr.ecr.<region>.amazonaws.com/myapp:$BUILD_NUMBER`).
   - Requires AWS CLI and ECR login (`aws ecr get-login-password`).
7. **Update Manifest**:
   - A script updates `deployment.yaml` in GitHub to reference the new image tag.
   - Example script:
     ```bash
     sed -i "s|image: .*|image: <aws-account-id>.dkr.ecr.<region>.amazonaws.com/myapp:$BUILD_NUMBER|" deployment.yaml
     git commit -m "Update image tag to $BUILD_NUMBER"
     git push origin main
     ```
8. **ArgoCD Sync**:
   - ArgoCD detects the manifest change and syncs the Kubernetes deployment.

### Jenkinsfile Example
```groovy
pipeline {
    agent any
    stages {
        stage('Clone') {
            steps {
                git 'https://github.com/<user>/<repo>.git'
            }
        }
        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh 'sonar-scanner'
                }
            }
        }
        stage('Build App') {
            steps {
                sh 'npm install && npm run build'
            }
        }
        stage('Security Scan') {
            steps {
                sh 'trivy image --exit-code 1 --severity CRITICAL,HIGH myapp:latest'
            }
        }
        stage('Build & Push Docker') {
            steps {
                sh '''
                    aws ecr get-login-password | docker login --username AWS --password-stdin <aws-account-id>.dkr.ecr.<region>.amazonaws.com
                    docker build -t myapp:$BUILD_NUMBER .
                    docker tag myapp:$BUILD_NUMBER <aws-account-id>.dkr.ecr.<region>.amazonaws.com/myapp:$BUILD_NUMBER
                    docker push <aws-account-id>.dkr.ecr.<region>.amazonaws.com/myapp:$BUILD_NUMBER
                '''
            }
        }
        stage('Update Manifest') {
            steps {
                sh '''
                    git config user.email "jenkins@ci.com"
                    git config user.name "Jenkins"
                    sed -i "s|image: .*|image: <aws-account-id>.dkr.ecr.<region>.amazonaws.com/myapp:$BUILD_NUMBER|" deployment.yaml
                    git add deployment.yaml
                    git commit -m "Update image tag to $BUILD_NUMBER"
                    git push origin main
                '''
            }
        }
    }
}
```

## Monitoring

Monitoring ensures observability of the Kubernetes cluster and application.

### Components
- **Prometheus**:
  - Scrapes metrics from Kubernetes pods, nodes, and the application (if instrumented with `/metrics` endpoint).
  - Configured with scrape jobs in `prometheus.yml`:
    ```yaml
    scrape_configs:
      - job_name: 'kubernetes'
        kubernetes_sd_configs:
        - role: pod
          namespaces:
            names: [default, argocd]
      - job_name: 'application'
        static_configs:
        - targets: ['<app-service>:8080']
    ```
  - Installed via Helm: `helm install prometheus prometheus-community/prometheus`.
- **Grafana**:
  - Connects to Prometheus as a data source.
  - Dashboards display CPU/memory usage, pod health, deployment status, and custom app metrics.
  - Installed via Helm: `helm install grafana grafana/grafana`.
  - Access: `http://<ec2-ip>:3000`, default login (admin/admin).

### Monitoring Flow
1. Kubernetes pods expose metrics (via kube-state-metrics, node-exporter, or app-specific endpoints).
2. Prometheus scrapes metrics at regular intervals.
3. Grafana queries Prometheus to visualize metrics in dashboards.
4. Alerts (optional): Configure in Prometheus with Alertmanager for notifications (e.g., Slack, email) on high CPU or failed deployments.

## Tools Used

| Tool         | Purpose                              | Version (Recommended) | Location              |
|--------------|--------------------------------------|-----------------------|-----------------------|
| Jenkins      | CI/CD pipeline orchestration         | 2.426.x (LTS)         | EC2 Instance 1        |
| SonarQube    | Code quality and security scanning   | 9.9.x (Community)     | EC2 Instance 1        |
| Docker       | Container image building            | 24.x                  | EC2 Instance 1        |
| Trivy        | Image vulnerability scanning         | 0.45.x                | EC2 Instance 1        |
| NPM          | Node.js app building                 | 10.x (with Node.js 20)| EC2 Instance 1        |
| ArgoCD       | GitOps-based Kubernetes deployments  | 2.8.x                 | Kubernetes Cluster    |
| Prometheus   | Metrics collection and alerting      | 2.47.x (via Helm)     | EC2 Instance 2        |
| Grafana      | Metrics visualization                | 10.x (via Helm)       | EC2 Instance 2        |
| GitHub       | Code and manifest storage           | N/A                   | External              |
| AWS ECR      | Docker image registry                | N/A                   | AWS                   |
| Kubernetes   | Container orchestration              | 1.28.x                | AWS (Manual Cluster)  |

All tools are open-source except AWS services. Use latest stable versions unless specified.

## Deployment Flow

The deployment process is GitOps-driven using ArgoCD:

1. **Image Push**: Jenkins builds and pushes the Docker image to AWS ECR.
2. **Manifest Update**: Jenkins updates `deployment.yaml` in GitHub with the new image tag.
   - Example `deployment.yaml`:
     ```yaml
     apiVersion: apps/v1
     kind: Deployment
     metadata:
       name: myapp
       namespace: default
     spec:
       replicas: 3
       selector:
         matchLabels:
           app: myapp
       template:
         metadata:
           labels:
             app: myapp
         spec:
           containers:
           - name: myapp
             image: <aws-account-id>.dkr.ecr.<region>.amazonaws.com/myapp:<tag>
             ports:
             - containerPort: 8080
     ```
3. **ArgoCD Sync**:
   - ArgoCD monitors the GitHub repo for changes.
   - Detects the updated `deployment.yaml` and triggers a sync.
   - Applies the manifest to Kubernetes using `kubectl apply`.
4. **Kubernetes Deployment**:
   - Kubernetes pulls the image from ECR (authenticated via IAM).
   - Rolls out new pods with a rolling update strategy for zero downtime.
5. **Monitoring**:
   - Prometheus scrapes metrics from the new pods.
   - Grafana visualizes metrics in real-time.

### Rollback
- Revert the `deployment.yaml` commit in GitHub to the previous image tag.
- ArgoCD auto-syncs, rolling back the deployment.
- Alternatively, use `kubectl rollout undo deployment/myapp`.

## Setup Instructions

1. **Provision Infrastructure**:
   - Clone the Terraform repo: `git clone <terraform-repo>`.
   - Configure AWS credentials (`aws configure`).
   - Run `terraform init`, `terraform plan`, and `terraform apply`.
2. **Create Kubernetes Cluster**:
   - Use `eksctl create cluster --name my-cluster --region <region> --nodegroup-name workers --node-type t3.medium --nodes 3`.
   - Install ArgoCD: `kubectl create namespace argocd && kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml`.
3. **Configure Jenkins**:
   - Access Jenkins at `http://<ec2-1-ip>:8080`.
   - Install plugins: Git, Pipeline, SonarQube Scanner, Docker, AWS Credentials.
   - Configure GitHub webhook and credentials.
   - Add SonarQube server in Jenkins global config.
4. **Set Up Monitoring**:
   - SSH into EC2 Instance 2 and install Helm.
   - Deploy Prometheus and Grafana via Helm commands (see Monitoring section).
   - Configure Grafana with Prometheus data source.
5. **Create GitHub Repository**:
   - Add source code, `Dockerfile`, `Jenkinsfile`, and Kubernetes manifests.
   - Configure webhook to point to Jenkins.
6. **Create ECR Repository**:
   - Run `aws ecr create-repository --repository-name myapp`.
   - Assign IAM roles to EC2 Instance 1 and Kubernetes nodes.
7. **Run Pipeline**:
   - Commit code to GitHub to trigger Jenkins.
   - Monitor pipeline in Jenkins UI, ArgoCD UI, and Grafana dashboards.

## Maintenance

- **Upgrades**: Regularly update tools (Jenkins, ArgoCD, etc.) to stable versions.
- **Backups**: Backup Jenkins home (`~/.jenkins`), SonarQube data, and Grafana dashboards.
- **Security**: Rotate AWS IAM credentials and GitHub tokens periodically.
- **Scaling**: Add more Kubernetes nodes or upgrade EC2 instances for higher workloads.
- **Logs**: Use Kubernetes logs (`kubectl logs`) and Grafana Loki (optional) for log aggregation.

## Troubleshooting

- **Jenkins Failure**: Check pipeline logs in Jenkins UI or `/var/lib/jenkins/logs`.
- **SonarQube Issues**: Verify quality gate settings and network access (port 9000).
- **Trivy Failures**: Adjust severity thresholds if scans are too strict.
- **ArgoCD Sync Issues**: Check ArgoCD UI for sync errors or GitHub connectivity.
- **Monitoring Gaps**: Ensure Prometheus scrape targets are correct and pods expose metrics.

## Contact

For issues or improvements, contact the DevOps team at `<your-email>` or raise a GitHub issue in the project repository.

---

This `README.md` provides a complete guide for setting up, running, and maintaining the CI/CD pipeline. It includes practical details like Terraform snippets, Jenkinsfile, and Kubernetes manifests, ensuring developers and DevOps engineers can replicate or extend the setup. Let me know if you need additional sections, specific configurations, or further clarification!
