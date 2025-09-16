pipeline {
    agent {
        label 'dotnet'  // This matches the label of your custom agent
    }
    
    environment {
        DOTNET_VERSION = '8.0'
        BUILD_CONFIGURATION = 'Release'
        PUBLISH_OUTPUT = './publish'
    }
    
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()  // Clean workspace before starting
            }
        }
        
        stage('Checkout') {
            steps {
                checkout scm  // Checkout source code from SCM
            }
        }
        
        stage('Restore Dependencies') {
            steps {
                script {
                    echo 'Restoring NuGet packages...'
                    sh 'dotnet restore'
                }
            }
        }
        
        stage('Build') {
            steps {
                script {
                    echo 'Building application...'
                    sh "dotnet build --configuration ${env.BUILD_CONFIGURATION} --no-restore"
                }
            }
        }
        
        stage('Test') {
            steps {
                script {
                    echo 'Running tests...'
                    sh "dotnet test --configuration ${env.BUILD_CONFIGURATION} --no-build --verbosity normal"
                }
            }
        }
        
        stage('Publish') {
            steps {
                script {
                    echo 'Publishing application...'
                    sh "dotnet publish --configuration ${env.BUILD_CONFIGURATION} --output ${env.PUBLISH_OUTPUT} --no-build"
                }
            }
        }
        
        stage('Archive Artifacts') {
            steps {
                script {
                    echo 'Archiving build artifacts...'
                    archiveArtifacts artifacts: "${env.PUBLISH_OUTPUT}/**/*", fingerprint: true
                }
            }
        }
    }
    
    post {
        always {
            echo "Build completed: ${currentBuild.currentResult}"
            cleanWs()  // Clean workspace after build
        }
        success {
            echo '✅ Build, test, and publish successful!'
            slackSend channel: '#builds', message: "✅ .NET Build Successful: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        failure {
            echo '❌ Build failed!'
            slackSend channel: '#builds', message: "❌ .NET Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        unstable {
            echo '⚠️ Build unstable!'
        }
    }
}
