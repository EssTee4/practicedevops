pipeline{
    agent any
    stages{
        stage("Checkout code"){
            steps{
                git branch: "main" url: "https://github.com/EssTee4/practicedevops/"
            }
        }
        stage("build"){
            steps{
                bat "building"
        }
    }
    }
    post{
        success{
            bat "built sucessfull"
        }
        failure{
            bat "built failed"
        }
    }
}
