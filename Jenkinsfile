pipeline {
    agent any

    environment {
        IMAGE_NAME = "practicedevops"
        DOCKER_USER = "esstee911"
    }

    stages {

        /* ---------- Feature Branch CI ---------- */
        stage('Feature Branch CI') {
            when {
                branch pattern: "feature/.*", comparator: "REGEXP"
            }
            steps {
                echo "Running CI for Feature Branch..."
                sh '''
                    echo "Running lint, unit tests, and static analysis..."
                    # add your feature branch validation commands here
                '''
            }
        }

        /* ---------- Dev Build & Deploy ---------- */
        stage('Dev Build & Deploy') {
            when {
                branch 'dev'
            }
            steps {
                echo "Building and deploying to Dev environment..."
                sh '''
                    docker build -t ${DOCKER_USER}/${IMAGE_NAME}:dev .
                    docker push ${DOCKER_USER}/${IMAGE_NAME}:dev
                '''
            }
        }

        /* ---------- Release Build & Staging ---------- */
        stage('Release Build & Staging') {
            when {
                branch 'release'
            }
            steps {
                echo "Building and deploying release for staging..."
                sh '''
                    docker build -t ${DOCKER_USER}/${IMAGE_NAME}:release .
                    docker push ${DOCKER_USER}/${IMAGE_NAME}:release
                '''
            }
        }

        /* ---------- Approval: Merge Release → Main ---------- */
        stage('Approval: Merge Release → Main') {
            when {
                branch 'release'
            }
            steps {
                input message: "Approve to merge release into main and deploy to production?"
            }
        }

        /* ---------- Merge Release → Main ---------- */
        stage('Merge Release → Main') {
            when {
                branch 'release'
            }
            steps {
                withCredentials([string(credentialsId: 'github', variable: 'TOKEN')]) {
                    script {
                        sh '''
                            set -e
                            git config user.name "jenkins"
                            git config user.email "jenkins@ci.local"

                            echo "Fetching latest branches..."
                            git fetch origin main
                            git fetch origin release

                            echo "Checking out main..."
                            git checkout main

                            echo "Pulling latest main to prevent non-fast-forward issue..."
                            git pull origin main --rebase

                            echo "Merging release into main..."
                            git merge --no-ff origin/release -m "Merge release into main"

                            echo "Pushing updated main branch..."
                            git push https://${TOKEN}@github.com/EssTee4/practicedevops.git main
                        '''
                    }
                }
            }
        }

        /* ---------- Production Deployment ---------- */
        stage('Production Deployment') {
            when {
                branch 'release'
            }
            steps {
                echo "Deploying to Production..."
                sh '''
                    docker pull ${DOCKER_USER}/${IMAGE_NAME}:release
                    docker tag ${DOCKER_USER}/${IMAGE_NAME}:release ${DOCKER_USER}/${IMAGE_NAME}:latest
                    docker push ${DOCKER_USER}/${IMAGE_NAME}:latest
                '''
            }
        }

        /* ---------- Dev Unlock & Sync ---------- */
        stage('Dev Unlock & Sync') {
            when {
                branch 'release'
            }
            steps {
                echo "Syncing Dev branch with latest main..."
                withCredentials([string(credentialsId: 'github', variable: 'TOKEN')]) {
                    script {
                        sh '''
                            git fetch origin dev
                            git checkout dev
                            git merge --no-ff origin/main -m "Sync dev with main after release"
                            git push https://${TOKEN}@github.com/EssTee4/practicedevops.git dev
                        '''
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline completed successfully!"
        }
        failure {
            echo "❌ Pipeline failed — rollback may be required!"
            // (Optional) Implement rollback logic here if production fails
        }
    }
}
