pipeline{
    agent any
    stages{
        stage("Checkout code"){
            steps{
                git "https://github.com/EssTee4/practicedevops/"
            }
        }
        stage("build"){
            step{
                bat "building"
        }
    }
    post{
        sucess{
            bat "built sucessfull"
            
        failure{
            bat "built failed"
    }
}
