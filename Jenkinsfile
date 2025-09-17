pipeline {
    agent any

    environment {
        // Build environment variables
        DOTNET_CLI_HOME = "/usr/share/dotnet"
        BUILD_CONFIGURATION = "Release"
        PUBLISH_OUTPUT_DIR = "./publish"
        
        // Deployment server details
        WIN_SERVER = "172.30.3.103"
        WIN_DEPLOY_PATH = "C:\\inetpub\\wwwroot\\myapp"
        WIN_TEMP_PATH = "C:\\Temp\\Deployments"
        
        // Application details
        APP_POOL_NAME = "MyAppPool01"
        SITE_NAME = "MyWebsite01"
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
                sh "dotnet build --configuration ${BUILD_CONFIGURATION}"
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
            }
        }
        
        stage('Create Deployment Package') {
            steps {
                sh "zip -r deployment.zip ${PUBLISH_OUTPUT_DIR}/"
                archiveArtifacts artifacts: 'deployment.zip', fingerprint: true
            }
        }
        
        stage('Deploy to IIS via WinRM') {
            steps {
                script {
                    // Step 1: Transfer deployment package to Windows server
                    withCredentials([usernamePassword(credentialsId: 'windows-admin-password', 
                                   usernameVariable: 'WIN_USERNAME', 
                                   passwordVariable: 'WIN_PASSWORD')]) {
                        sh """
                        curl -T deployment.zip --user ${WIN_USERNAME}:'${WIN_PASSWORD}' \
                        --negotiate -k "https://${WIN_SERVER}:5986/wsman/upload?path=C:\\Temp\\deployment.zip"
                        """
                    }
                    
                    // Step 2: Execute deployment script on Windows server
                    withCredentials([usernamePassword(credentialsId: 'windows-admin-password', 
                                   usernameVariable: 'WIN_USERNAME', 
                                   passwordVariable: 'WIN_PASSWORD')]) {
                        bat """
                        powershell -Command "& {
                            \\$securePassword = ConvertTo-SecureString '${WIN_PASSWORD}' -AsPlainText -Force
                            \\$credential = New-Object System.Management.Automation.PSCredential ('${WIN_USERNAME}', \\$securePassword)
                            
                            \\$sessionOptions = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
                            
                            Invoke-Command -ComputerName ${WIN_SERVER} -Credential \\$credential -SessionOption \\$sessionOptions -ScriptBlock {
                                param(\\$DeployPath, \\$TempPath, \\$AppPoolName, \\$SiteName, \\$PublishOutputDir)
                                
                                # Create temp directory if it doesn't exist
                                if (!(Test-Path \\$TempPath)) {
                                    New-Item -ItemType Directory -Path \\$TempPath -Force
                                }
                                
                                # Extract deployment package
                                \\$zipPath = 'C:\\Temp\\deployment.zip'
                                if (Test-Path \\$zipPath) {
                                    Expand-Archive -Path \\$zipPath -DestinationPath \\$TempPath -Force
                                } else {
                                    throw 'Deployment package not found: ' + \\$zipPath
                                }
                                
                                # Stop IIS application pool
                                Import-Module WebAdministration
                                Stop-WebAppPool -Name \\$AppPoolName -ErrorAction SilentlyContinue
                                
                                # Backup existing deployment (optional)
                                \\$timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
                                \\$backupPath = "\\${DeployPath}_Backup_\\$timestamp"
                                if (Test-Path \\$DeployPath) {
                                    Copy-Item \\$DeployPath \\$backupPath -Recurse -Force
                                }
                                
                                # Remove existing files
                                if (Test-Path \\$DeployPath) {
                                    Remove-Item "\\${DeployPath}\\*" -Recurse -Force
                                } else {
                                    New-Item -ItemType Directory -Path \\$DeployPath -Force
                                }
                                
                                # Copy new files
                                \\$sourcePath = "\\${TempPath}\\${PublishOutputDir}\\"
                                Write-Host "Copying from: \\$sourcePath"
                                Write-Host "Copying to: \\$DeployPath"
                                
                                if (Test-Path \\$sourcePath) {
                                    Copy-Item "\\${sourcePath}\\*" \\$DeployPath -Recurse -Force
                                } else {
                                    throw 'Source path not found: ' + \\$sourcePath
                                }
                                
                                # Set proper permissions (if needed)
                                \\$acl = Get-Acl \\$DeployPath
                                \\$accessRule = New-Object System.Security.AccessControl.FileSystemAccessRule(
                                    'IIS_IUSRS', 'ReadAndExecute', 'ContainerInherit,ObjectInherit', 'None', 'Allow')
                                \\$acl.SetAccessRule(\\$accessRule)
                                Set-Acl \\$DeployPath \\$acl
                                
                                # Clean up temp files
                                Remove-Item \\$zipPath -Force -ErrorAction SilentlyContinue
                                Remove-Item \\$TempPath -Recurse -Force -ErrorAction SilentlyContinue
                                
                                # Start IIS application pool
                                Start-WebAppPool -Name \\$AppPoolName
                                Start-Website -Name \\$SiteName
                                
                                Write-Host 'Deployment completed successfully!'
                                Write-Host 'Deployed to: ' + \\$DeployPath
                                
                            } -ArgumentList '${WIN_DEPLOY_PATH}', '${WIN_TEMP_PATH}', '${APP_POOL_NAME}', '${SITE_NAME}', '${PUBLISH_OUTPUT_DIR}'
                        }"
                        """
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    bat "curl -I http://${WIN_SERVER} --connect-timeout 30 --max-time 60"
                }
            }
        }
    }

    post {
        success {
            echo 'Build, test, publish, and deployment successful!'
            emailext (
                subject: "SUCCESS: Deployment completed - ${env.JOB_NAME}",
                body: "The application was successfully deployed to ${WIN_SERVER}",
                to: "m.shaaban@seu.edu.sa"
            )
        }
        failure {
            echo 'Deployment failed!'
            emailext (
                subject: "FAILURE: Deployment failed - ${env.JOB_NAME}",
                body: "The deployment to ${WIN_SERVER} failed. Please check Jenkins logs.",
                to: "m.shaaban@seu.edu.sa"
            )
        }
        always {
            cleanWs()
        }
    }
}