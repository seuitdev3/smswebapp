pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                script {
                    if (isUnix()) {
                        sh '''
                            # Update package list and install ICU without sudo
                            apt-get update -o Acquire::AllowInsecureRepositories=true
                            apt-get install -y --allow-unauthenticated libicu-dev
                        '''
                    }
                }
            }
        }
        
        stage('Build') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'dotnet restore'
                        sh 'dotnet build --configuration Release'
                    } else {
                        bat 'dotnet restore'
                        bat 'dotnet build --configuration Release'
                    }
                }
            }
        }
        
        stage('Test') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'dotnet test --no-restore --configuration Release'
                    } else {
                        bat 'dotnet test --no-restore --configuration Release'
                    }
                }
            }
        }
        
        stage('Publish') {
            steps {
                script {
                    if (isUnix()) {
                        sh 'dotnet publish --no-restore --configuration Release --output ./publish'
                    } else {
                        bat 'dotnet publish --no-restore --configuration Release --output .\\publish'
                    }
                }
            }
        }
    }
}
