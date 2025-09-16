pipeline { 
    agent any

    stages {
	
        stage('CLONE GITHUB CODE') {
            steps {
                echo 'In this stage code will be cloned'
                git branch: 'main', url: 'https://github.com/adarsh0331/Project_9.git'
            }
        }
		
        // stage('BUILDING THE CODE') {
        //     steps {
        //         echo 'In this stage code will be built and mvn artifact will be generated'
        //         sh 'mvn clean install'
        //     }
        // }		
		
        stage('DEPLOY WITH ANSIBLE') {
            steps {
                sh '''
                    ansible-playbook playbook.yml 
                '''
            }
        }
    }
}
