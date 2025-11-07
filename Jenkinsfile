pipeline {
    agent any
    environment {
        Image_name = "esstee911/test"
    }
    stages {
        stage("checkout") {
            steps {
                git branch: "main", url: "https://github.com/EssTee4/practicedevops.git"
            }
        }
        stage("build image") {
            steps {
                sh "docker build -t ${Image_name}:latest ."
            }
        }
        stage("push to docker") {
            steps {
                withCredentials([usernamePassword(credentialsId: "dockerhub", usernameVariable: "dockerUser", passwordVariable: "dockerPass")]) {
                    sh '''
                        echo $dockerPass | docker login -u $dockerUser --password-stdin
                        docker push ${Image_name}:latest
                        docker logout
                    '''
                }
            }
        }
    }
}
