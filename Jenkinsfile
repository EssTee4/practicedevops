pipeline{
    agent any
    stages{
        stage("Checkout code"){
            steps{
                git branch: 'main', url: 'https://github.com/EssTee4/practicedevops/'
            }
        }
        stage("build"){
            steps{
                sh "building"
        }
    }
    }
    post{
        success{
            sh "built sucessfull"
        }
        failure{
            sh "built failed"
        }
    }
}
