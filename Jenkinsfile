pipeline {
    agent any

    environment {
        IMAGE_NAME = "practicedevops"
        DOCKER_USER = "esstee911"
    }

    stages {

        /* -------- Feature Branch Build -------- */
        stage('Feature Branch Build') {
            when { expression { env.BRANCH_NAME.startsWith('feature/') } }
            steps {
                echo "🚧 Building feature branch: ${env.BRANCH_NAME}"
                checkout scm
                script {
                    def featureTag = env.BRANCH_NAME.replaceAll('[^a-zA-Z0-9_.-]', '-')
                    if (!featureTag) { featureTag = "latest" }
                    echo "Using Docker tag: ${featureTag}"

                    sh "docker build -t ${DOCKER_USER}/${IMAGE_NAME}:${featureTag} ."

                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'dockerUser', passwordVariable: 'dockerPass')]) {
                        sh """
                            echo \$dockerPass | docker login -u \$dockerUser --password-stdin
                            docker push ${DOCKER_USER}/${IMAGE_NAME}:${featureTag}
                            docker logout
                        """
                    }
                }
            }
        }

        /* -------- Develop Branch Build & Deploy (Testing) -------- */
        stage('Develop Build & Deploy') {
            when { branch 'dev' }
            steps {
                echo "🧪 Building and deploying develop branch..."
                checkout scm
                sh "docker build -t ${DOCKER_USER}/${IMAGE_NAME}:develop ."
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'dockerUser', passwordVariable: 'dockerPass')]) {
                    sh """
                        echo \$dockerPass | docker login -u \$dockerUser --password-stdin
                        docker push ${DOCKER_USER}/${IMAGE_NAME}:develop
                        docker stop dev-test || true
                        docker rm dev-test || true
                        docker run -d -p 2222:80 --name dev-test ${DOCKER_USER}/${IMAGE_NAME}:develop
                        docker logout
                    """
                }
            }
        }

        /* -------- Release Branch Build & Deploy to Staging -------- */
        stage('Release Build & Deploy to Staging') {
            when { branch 'release' }
            steps {
                echo "🚀 Building release branch for staging..."
                sh "docker build -t ${DOCKER_USER}/${IMAGE_NAME}:staging ."
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'dockerUser', passwordVariable: 'dockerPass')]) {
                    sh """
                        echo \$dockerPass | docker login -u \$dockerUser --password-stdin
                        docker push ${DOCKER_USER}/${IMAGE_NAME}:staging
                        docker stop staging || true
                        docker rm staging || true
                        docker run -d -p 4444:80 --name staging ${DOCKER_USER}/${IMAGE_NAME}:staging
                        docker logout
                    """
                }

                echo "🔒 Locking develop branch during release..."
                withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'USER', passwordVariable: 'TOKEN')]) {
                    sh """
                        git fetch origin dev:dev || echo "Dev branch not found locally"
                        if git show-ref --verify --quiet refs/heads/dev; then
                            LOCKED_DEV="dev-locked-\$(date +%s)"
                            git branch -m dev \$LOCKED_DEV
                            git push origin \$LOCKED_DEV || echo "Failed to push locked dev branch"
                            git push origin :dev || true
                            echo "✅ Develop branch locked as \$LOCKED_DEV"
                        else
                            echo "⚠️ Dev branch not found, skipping lock"
                            exit 1
                        fi
                    """
                }
            }
        }

        /* -------- Approval Before Merge to Main -------- */
        stage('Approval to Merge Release → Main') {
            when { branch 'release' }
            steps {
                input message: "✅ Approve merging release branch into main for production?"
            }
        }

        /* -------- Merge Release into Main -------- */
        stage('Merge Release → Main') {
            when { branch 'release' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'USER', passwordVariable: 'TOKEN')]) {
                    sh """
                        git config user.name "jenkins"
                        git config user.email "jenkins@ci.local"
                        git fetch origin main
                        git checkout main
                        git merge --no-ff origin/release -m "Merge release into main"
                        git push https://\$USER:\$TOKEN@github.com/EssTee4/practicedevops.git main
                    """
                }
            }
        }

        /* -------- Production Deployment with Rollback -------- */
        stage('Production Deployment with Rollback') {
            when { branch 'main' }
            steps {
                script {
                    echo "🚀 Starting production deployment..."
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'dockerUser', passwordVariable: 'dockerPass')]) {
                        sh """
                            docker pull ${DOCKER_USER}/${IMAGE_NAME}:latest || true
                            docker pull ${DOCKER_USER}/${IMAGE_NAME}:stable || true
                            docker stop prod-live || true
                            docker rm prod-live || true
                            docker build -t ${DOCKER_USER}/${IMAGE_NAME}:latest .
                            docker run -d -p 3333:80 --name prod-live ${DOCKER_USER}/${IMAGE_NAME}:latest

                            sleep 5
                            status=\$(docker ps | grep prod-live | wc -l)
                            if [ "\$status" != "1" ]; then
                                echo "❌ Deployment failed, rolling back..."
                                docker stop prod-live || true
                                docker rm prod-live || true
                                docker run -d -p 3333:80 --name prod-live ${DOCKER_USER}/${IMAGE_NAME}:stable || echo "⚠️ No stable image available"
                                exit 1
                            fi
                        """
                    }
                }
            }
            post {
                success {
                    echo "🏷️ Tagging deployed image as stable..."
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'dockerUser', passwordVariable: 'dockerPass')]) {
                        sh """
                            echo \$dockerPass | docker login -u \$dockerUser --password-stdin
                            docker tag ${DOCKER_USER}/${IMAGE_NAME}:latest ${DOCKER_USER}/${IMAGE_NAME}:stable
                            docker push ${DOCKER_USER}/${IMAGE_NAME}:stable
                            docker logout
                        """
                    }
                }
                failure { echo "Deployment failed. Previous stable image retained." }
            }
        }

        /* -------- Sync & Unlock Develop -------- */
        stage('Sync & Unlock Develop') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'USER', passwordVariable: 'TOKEN')]) {
                    sh """
                        echo "🔄 Syncing main back into develop..."
                        git fetch origin
                        # Recreate or checkout dev branch from main
                        git checkout main
                        git branch -f dev
                        git push https://\$USER:\$TOKEN@github.com/EssTee4/practicedevops.git dev --force
                        echo "🔓 Develop branch unlocked and synced"
                    """
                }
            }
        }
    }

    post {
        success { echo "✅ Pipeline completed successfully for ${env.BRANCH_NAME}" }
        failure { echo "❌ Pipeline failed for ${env.BRANCH_NAME}" }
    }
}
