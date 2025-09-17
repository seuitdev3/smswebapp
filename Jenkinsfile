pipeline {
    agent any

    environment {
        // Build environment variables
        DOTNET_CLI_HOME = "/usr/share/dotnet"
        BUILD_CONFIGURATION = "Release"
        PUBLISH_OUTPUT_DIR = "./publish"
        
        // Deployment server details
        WIN_SERVER = "172.30.3.103"                   // Windows Server IP or hostname
        WIN_DEPLOY_PATH = "C:\\inetpub\\wwwroot\\myapp" // IIS application path
        WIN_TEMP_PATH = "C:\\Temp\\Deployments"        // Temporary directory on server
        
        // Application details
        APP_POOL_NAME = "MyAppPool01"                    // IIS Application Pool name
        SITE_NAME = "MyWebsite01"                        // IIS Site name
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Restore Dependencies') {
            steps {
                sh "dotnet restore"
            }
        }

        stage('Build') {
            steps {
                sh "dotnet build --configuration ${BUILD_CONFIGURATION} --no-restore"
            }
        }

        stage('Test') {
            steps {
                sh "dotnet test --no-restore --configuration ${BUILD_CONFIGURATION}"
            }
        }

        stage('Publish') {
            steps {
                sh "dotnet publish --no-restore --configuration ${BUILD_CONFIGURATION} --output ${PUBLISH_OUTPUT_DIR}"
                
                // List the publish directory contents to verify
                sh "ls -la ${PUBLISH_OUTPUT_DIR}"
            }
        }
        
        stage('Create Deployment Package') {
            steps {
                script {
                    // Create deployment package using tar
                    sh """
                    # Create tar.gz package
                    tar -czf deployment.tar.gz ${PUBLISH_OUTPUT_DIR}/
                    ls -la deployment.tar.gz
                    """
                    
                    // Archive the package for later reference
                    archiveArtifacts artifacts: 'deployment.tar.gz', fingerprint: true
                }
            }
        }
        
        stage('Verify Files') {
            steps {
                script {
                    // Show workspace contents to verify files exist
                    sh """
                    echo "Workspace contents:"
                    ls -la
                    echo "Publish directory contents:"
                    ls -la ${PUBLISH_OUTPUT_DIR}/
                    echo "Deployment package size:"
                    du -sh deployment.tar.gz
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Build, test, publish, and deployment successful!'
            // Optional: Send notification
            emailext (
                subject: "SUCCESS: Deployment completed - ${env.JOB_NAME}",
                body: "The application was successfully deployed to ${WIN_SERVER}",
                to: "m.shaaban@seu.edu.sa"
            )
        }
        failure {
            echo 'Deployment failed!'
            // Optional: Send failure notification
            emailext (
                subject: "FAILURE: Deployment failed - ${env.JOB_NAME}",
                body: "The deployment to ${WIN_SERVER} failed. Please check Jenkins logs.",
                to: "m.shaaban@seu.edu.sa"
            )
        }
        always {
            // Clean up workspace (commented out for debugging)
            // cleanWs()
            
            // Instead, just clean specific files to preserve the publish directory for inspection
            sh """
            echo "Cleaning up temporary files but keeping publish directory for inspection..."
            rm -f deployment.tar.gz || true
            """
        }
    }
}