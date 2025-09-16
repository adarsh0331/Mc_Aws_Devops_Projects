pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "my-devops-app:latest"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git url: 'https://github.com/adarsh0331/project10.git', branch: 'main'
            }
        }

        stage('BUILDING THE CODE') {
            steps {
                echo 'In this stage code will be built and mvn artifact will be generated'
                sh 'mvn clean install'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "sudo docker build -t $DOCKER_IMAGE ."
            }
        }

        stage('Deploy App') {
            steps {
                sh """
                    sudo docker ps -q --filter "name=my-devops-app" | grep -q . && docker stop my-devops-app && docker rm my-devops-app || true
                    sudo docker run -d --name my-devops-app -p 5000:5000 $DOCKER_IMAGE
                """
            }
        }
    }

    post {
        always {
            echo "Pipeline finished!"
        }
        failure {
            echo "Pipeline failed ❌"
        }
        success {
            echo "Pipeline completed successfully ✅"
        }
    }
}
