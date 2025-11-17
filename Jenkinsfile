pipeline {
    agent any

    environment {
        IMAGE_NAME = "practicedevops"
        DOCKER_USER = "esstee911"
    }

    stages {

        /* -------- Feature Branch -------- */
        stage('Feature Branch CI') {
            when { expression { env.BRANCH_NAME?.startsWith('feature/') } }
            steps {
                echo "🚧 Feature branch: ${env.BRANCH_NAME}"
                checkout scm

                // Install & lint
                sh "npm install || true"
                sh "npm run lint || true"

                script {
                    def featureTag = env.BRANCH_NAME.replaceAll('[^a-zA-Z0-9_.-]', '-')
                    echo "Docker tag: ${featureTag}"

                    // Build and scan Docker image
                    sh "docker build -t ${DOCKER_USER}/${IMAGE_NAME}:${featureTag} ."
                    sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${DOCKER_USER}/${IMAGE_NAME}:${featureTag} || true"

                    // Push feature tag
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'dockerUser', passwordVariable: 'dockerPass')]) {
                        sh """
                            echo \$dockerPass | docker login -u \$dockerUser --password-stdin
                            docker push ${DOCKER_USER}/${IMAGE_NAME}:${featureTag}
                            docker logout || true
                        """
                    }
                }
            }
        }

        /* -------- Dev Branch -------- */
        stage('Dev Build & Deploy') {
            when { branch 'dev' }
            steps {
                echo "🧪 Dev branch build & deploy"
                checkout scm
                sh "docker build -t ${DOCKER_USER}/${IMAGE_NAME}:dev ."

                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'dockerUser', passwordVariable: 'dockerPass')]) {
                    sh """
                        echo \$dockerPass | docker login -u \$dockerUser --password-stdin
                        docker push ${DOCKER_USER}/${IMAGE_NAME}:dev
                        docker logout || true
                    """
                }

                // Run container & health check
                sh "docker stop dev-test || true && docker rm dev-test || true"
                sh "docker run -d -p 2222:80 --name dev-test ${DOCKER_USER}/${IMAGE_NAME}:dev"
                sh "sleep 10"
                sh "docker exec dev-test curl -f http://localhost/ || (echo 'Dev health check failed' && exit 1)"
            }
        }

        /* -------- Release Branch -------- */
        stage('Release Build & Staging') {
            when { branch 'release' }
            steps {
                echo "🚀 Release branch staging deploy"
                checkout scm
                sh "docker build -t ${DOCKER_USER}/${IMAGE_NAME}:staging ."

                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'dockerUser', passwordVariable: 'dockerPass')]) {
                    sh """
                        echo \$dockerPass | docker login -u \$dockerUser --password-stdin
                        docker push ${DOCKER_USER}/${IMAGE_NAME}:staging
                        docker logout || true
                    """
                }

                // Run staging container & health check
                sh "docker stop staging || true && docker rm staging || true"
                sh "docker run -d -p 4444:80 --name staging ${DOCKER_USER}/${IMAGE_NAME}:staging"
                sh "sleep 5"
                sh "docker exec staging curl -f http://localhost/ || (echo 'Staging health check failed' && exit 1)"

                echo "The image has been deployed for performance and regression testing to the following link https://practicedevops-staging.onrender.com"

                // Optional: lock dev branch
                echo "🔒 Locking dev branch (optional)"
                withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'USER', passwordVariable: 'TOKEN')]) {
                    sh """
                        git fetch origin dev:dev || true
                        if git show-ref --verify --quiet refs/heads/dev; then
                            git push https://\$USER:\$TOKEN@github.com/EssTee4/practicedevops.git :dev || true
                        fi
                    """
                }
            }
        }

        /* -------- Approval: Merge Release → Main -------- */
        stage('Approval: Merge Release → Main') {
            when { branch 'release' }
            steps {
                input message: "✅ Approve merging release to main?"
            }
        }

        /* -------- Merge Release into Main -------- */
        stage('Merge Release → Main') {
            when { branch 'release' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'USER', passwordVariable: 'TOKEN')]) {
                    sh '''
                        git config user.name "jenkins"
                        git config user.email "jenkins@ci.local"
                        git fetch origin main
                        git checkout main
                        git pull origin main --rebase || true
                        git merge --no-ff origin/release -m "Merge release into main"
                        git push https://$USER:$TOKEN@github.com/EssTee4/practicedevops.git main
                    '''
                }
            }
        }

        /* -------- Main Branch Production Deploy -------- */
        stage('Production Deployment') {
            when { branch 'main' }
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'dockerUser', passwordVariable: 'dockerPass')]) {
                        sh """
                            echo \$dockerPass | docker login -u \$dockerUser --password-stdin

                            # Build latest and push
                            docker build -t ${DOCKER_USER}/${IMAGE_NAME}:latest .
                            docker push ${DOCKER_USER}/${IMAGE_NAME}:latest

                            # Run production container
                            docker stop prod-live || true
                            docker rm prod-live || true
                            docker run -d -p 3333:80 --name prod-live ${DOCKER_USER}/${IMAGE_NAME}:latest

                            # Tag stable for rollback
                            docker tag ${DOCKER_USER}/${IMAGE_NAME}:latest ${DOCKER_USER}/${IMAGE_NAME}:stable
                            docker push ${DOCKER_USER}/${IMAGE_NAME}:stable
                            docker logout || true
                        """
                    }

                    // Verify deployment
                    sh "sleep 5"
                    sh '''
                        status=$(docker ps | grep prod-live | wc -l)
                        if [ "$status" != "1" ]; then
                          echo "❌ Deployment failed, rolling back..."
                          docker stop prod-live || true
                          docker rm prod-live || true
                          docker pull ${DOCKER_USER}/${IMAGE_NAME}:stable || true
                          docker run -d -p 3333:80 --name prod-live ${DOCKER_USER}/${IMAGE_NAME}:stable || echo "⚠️ No stable image"
                          exit 1
                        fi
                    '''
                }
            }
            post {
                success { echo "🏷️ Production deployment succeeded and stable image updated" }
                failure { echo "⚠️ Production deployment failed. Stable image retained." }
            }
        }

        /* -------- Dev Unlock & Sync -------- */
        stage('Dev Unlock & Sync') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'USER', passwordVariable: 'TOKEN')]) {
                    sh '''
                        echo "🔄 Sync main back to dev..."
                        git fetch origin
                        git checkout main
                        git branch -f dev
                        git push https://$USER:$TOKEN@github.com/EssTee4/practicedevops.git dev --force
                        echo "✅ Dev branch unlocked and synced"
                    '''
                }
            }
        }
    }

    post {
        success { echo "✅ Pipeline completed for ${env.BRANCH_NAME}" }
        failure { echo "❌ Pipeline failed for ${env.BRANCH_NAME}" }
    }
}

