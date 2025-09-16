*I have taken instance type as t2.large and i attched volume 50gb

* After the server was configured initially i installed jenkins using below commands
      "sudo yum install wget ",
      "sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo",
      "sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key",	    
      "sudo yum upgrade -y",
      "sudo dnf install java-17-amazon-corretto -y",	  
      "sudo yum install jenkins -y",
 	    "sudo systemctl start jenkins",
      "sudo systemctl enable jenkins",


   Then i installed docker using below commands
      sudo yum update -y
      sudo yum install -y docker
      sudo service docker start
      sudo usermod -aG docker ec2-user
      sudo usermod -a -G docker jenkins



   Then i created sonarqube container from docker image by using below command
      "sudo docker run -itd --name sonar -p 9000:9000 sonarqube"

      
         After the creation of sonarqube container we have to intigrate it with pipeline
         for that follow below process

         













   I installed trivy bu using below command to scan the image
            "sudo rpm -ivh https://github.com/aquasecurity/trivy/releases/download/v0.18.3/trivy_0.18.3_Linux-64bit.rpm",



   Alos i installed OWASP Dependency-Check using below commands

      sudo yum update -y

      Download Dependency-Check:
      cd /opt
      sudo curl -L -O https://github.com/jeremylong/DependencyCheck/releases/download/v8.4.2/dependency-check-8.4.2-release.zip

      nstall unzip & extract:
      sudo yum install unzip -y
      sudo unzip dependency-check-8.4.2-release.zip
      sudo mv dependency-check /opt/dependency-check

      Add to PATH:
      echo 'export PATH=$PATH:/opt/dependency-check/bin' | sudo tee -a /etc/profile.d/dependency-check.sh
      source /etc/profile.d/dependency-check.sh



   I integrated OWASP with jenkins using below process:
   
   Step 1: Install Dependency-Check Plugin in Jenkins
   Go to Jenkins Dashboard → Manage Jenkins → Plugins → Available.
   Search for “OWASP Dependency-Check”.
   Install it and restart Jenkins.


   Step 2: Configure Dependency-Check Tool
   Go to Manage Jenkins → Tools.
   Under Dependency-Check installations, click Add Dependency-Check.
   Give it a name: dependency-check
   Set the installation directory if you installed it manually (/opt/dependency-check).


   step3: Add below code in your pipeline
   stage('Dependency Check') {
    steps {
        sh '''
          /opt/dependency-check/bin/dependency-check.sh \
          --project "SpringBootApp" \
          --scan java-cicd-project/spring-boot-app \
          --format HTML \
          --data /var/lib/jenkins/odc-data \
          --out dependency-check-report
        '''
    }
}



   





   










# Spring Boot based Java web application
 
This is a simple Sprint Boot based Java application that can be built using Maven. Sprint Boot dependencies are handled using the pom.xml 
at the root directory of the repository.

This is a MVC architecture based application where controller returns a page with title and message attributes to the view.

## Execute the application locally and access it using your browser

Checkout the repo and move to the directory

```
git clone https://github.com/iam-veeramalla/Jenkins-Zero-To-Hero/java-maven-sonar-argocd-helm-k8s/sprint-boot-app
cd java-maven-sonar-argocd-helm-k8s/sprint-boot-app
```

Execute the Maven targets to generate the artifacts

```
mvn clean package
```

The above maven target stroes the artifacts to the `target` directory. You can either execute the artifact on your local machine
(or) run it as a Docker container.

 


## Plugins to install
   sonar sonar
   docker
   stage view
   OWASP Dependency-Check



   credentails that can be used 
   dockerHub
   soanrqube



   🔧 Step 1: Install Dependency-Check Plugin in Jenkins

Go to Jenkins Dashboard → Manage Jenkins → Plugins → Available.

Search for “OWASP Dependency-Check”.

Install it and restart Jenkins.

🔧 Step 2: Configure Dependency-Check Tool
Go to Manage Jenkins → Tools.
Under Dependency-Check installations, click Add Dependency-Check.
Give it a name: dependency-check
Set the installation directory if you installed it manually (/opt/dependency-check).






🔧 Step 3: Using Dependency-Check in Jenkins Freestyle Jobs
Create a Freestyle job.
In Build Environment, check:
✅ Provide Dependency-Check installation (select the one you configured).
Add a Build Step → Invoke Dependency-Check analysis.
Project name: MyApp
Scan path: . (or path to your source code).
Output format: HTML or XML.
Add a Post-build Action → Publish Dependency-Check results.
This will show a vulnerability report inside Jenkins.







🔧 Step 4: Using Dependency-Check in Jenkins Pipeline

If you use a Jenkinsfile, you can run Dependency-Check with a shell step.

Example:

pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/your-repo.git'
            }
        }
        stage('Dependency Check') {
            steps {
                sh '''
                  dependency-check.sh \
                    --project "MyApp" \
                    --scan . \
                    --format HTML \
                    --out dependency-check-report
                '''
            }
        }
        stage('Publish Report') {
            steps {
                publishHTML([
                  allowMissing: false,
                  alwaysLinkToLastBuild: true,
                  keepAll: true,
                  reportDir: 'dependency-check-report',
                  reportFiles: 'dependency-check-report.html',
                  reportName: 'OWASP Dependency-Check Report'
                ])
            }
        }
    }
}


