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
        stage('Run HTML Tests') {
            steps {
            echo "Running HTML and CSS tests..."
            sh 'htmlhint .'
            }
        }
        stage('Link Check') {
            steps {
            echo "🔗 Checking for broken links..."
            bat 'linkinator ./index.html --recurse'
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
         stage('Cleanup') {
            steps {
                echo "🧹 Cleaning up local images..."
                sh "docker rmi ${env.IMAGE_NAME}:latest || exit 0"
            }
        }
    }
}
