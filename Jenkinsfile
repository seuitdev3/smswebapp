pipeline {
    agent none
    
    stages {
        stage('Build and Test') {
            agent {
                docker {
                    image 'mcr.microsoft.com/dotnet/sdk:8.0' // Use your .NET version
                    args '--user root' // Ensure permissions if needed
                    reuseNode true
                }
            }
            stages {
                stage('Checkout') {
                    steps {
                        checkout scm
                    }
                }
                
                stage('Restore') {
                    steps {
                        sh 'dotnet restore'
                    }
                }
                
                stage('Build') {
                    steps {
                        sh 'dotnet build --configuration Release'
                    }
                }
                
                stage('Test') {
                    steps {
                        sh 'dotnet test --no-restore --configuration Release'
                    }
                }
                
                stage('Publish') {
                    steps {
                        sh 'dotnet publish --no-restore --configuration Release --output ./publish'
                    }
                }
            }
        }
        
        stage('Archive Artifacts') {
            agent any
            steps {
                archiveArtifacts artifacts: 'publish/**/*', fingerprint: true
            }
        }
    }
}
