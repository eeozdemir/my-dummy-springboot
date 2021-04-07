pipeline {
    agent { label "master" }
    environment {
        APP_REPO_NAME= "my-spring-boot"
        PATH="/usr/local/bin/:${env.PATH}"
    }
    stages {
        stage('Build Docker Compile image') {
            steps {
                sh 'docker run --rm -v $HOME/.m2:/root/.m2 -v $WORKSPACE:/app -w /app maven:3.6.3-openjdk-8 mvn clean package'
                sh 'docker image ls'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t <imagename> .'
                sh 'docker image ls'
            }
        }
        stage('Push Image to DockerHub') {
            steps {
                sh 'docker login -u <username> -p <password>'
                sh 'docker push <username>/<imagename>'
            }
        }
        stage('Deploy') {
            steps {
                sh 'docker login -u <username> -p <password>'
                sh 'docker pull <username>/<imagename>'
                // sh 'docker run -d -p 80:8080 <imagename>'
                sh 'kubectl apply -f .'

            }
        }

    }
    post {
        always {
            echo 'Deleting all local images'
            sh 'docker image prune -af'
        }
    }
}