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
            when {
                branch 'dev'
            }
            steps {
            echo "Running HTML and CSS tests..."
            sh 'htmlhint .'
            }
        }
        
        stage('Link Check') {
            when {
                branch 'dev'
            }
            steps {
            echo "🔗 Checking for broken links..."
            sh 'linkinator ./index.html --recurse'
            }
        }

        stage('Merge to Main') {
            when {
                branch 'dev'
            }
            steps {
            withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'USER', passwordVariable: 'TOKEN')]) {
            sh '''
                git remote set-url origin https://${USER}:${TOKEN}@github.com/EssTee4/practicedevops.git
                git fetch origin main
                git checkout main

                git merge origin/dev --no-edit

                git push origin main
            '''
            }
        }
    }

        stage("build image") {
            when {
                branch 'main'
            }
            steps {
                sh "docker build -t ${Image_name}:latest ."
            }
        }
        stage("push to docker") {
            when {
                branch 'main'
            }
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
             when {
                branch 'main'
            }
            steps {
                echo "🧹 Cleaning up local images..."
                sh "docker rmi ${env.IMAGE_NAME}:latest || exit 0"
            }
        }
    }
}
