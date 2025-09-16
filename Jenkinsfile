pipeline {
    agent any
    tools {
        dotnetsdk 'dotnet'  // Uses the globally configured .NET SDK
    }
    stages {
        stage('Checkout') {
            steps {
                checkout scm
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
