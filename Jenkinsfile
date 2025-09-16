pipeline {
    agent any

    environment {
        DOTNET_CLI_HOME = "/usr/share/dotnet"  // Update to the correct path for Linux
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Restore Dependencies') {
            steps {
                script {
                    // Restoring dependencies
                    sh "dotnet restore"  // Use sh instead of bat for Linux
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    // Building the application
                    sh "dotnet build --configuration Release"
                }
            }
        }

        stage('Test') {
            steps {
                script {
                    // Running tests
                    sh "dotnet test --no-restore --configuration Release"
                }
            }
        }

        stage('Publish') {
            steps {
                script {
                    // Publishing the application
                    sh "dotnet publish --no-restore --configuration Release --output ./publish"
                }
            }
        }
    }

    post {
        success {
            echo 'Build, test, and publish successful!'
        }
    }
}
