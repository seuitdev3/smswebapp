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
        
        // Credentials ID (update with your actual credentials ID)
        WIN_CREDENTIALS_ID = "windows-admin-credentials"
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
                
                // Verify publish directory contents
                sh "ls -la ${PUBLISH_OUTPUT_DIR}"
            }
        }
        
        stage('Prepare Deployment') {
            steps {
                script {
                    // Create deployment package using tar
                    sh """
                    # Create tar.gz package
                    tar -czf deployment.tar.gz ${PUBLISH_OUTPUT_DIR}/
                    echo "Deployment package created:"
                    ls -la deployment.tar.gz
                    """
                    
                    // Archive the package for later reference
                    archiveArtifacts artifacts: 'deployment.tar.gz', fingerprint: true
                }
            }
        }
        
        stage('Deploy to Windows Server') {
            steps {
                script {
                    echo "Starting deployment to Windows server ${WIN_SERVER}"
                    
                    // Step 1: Transfer files to Windows server using SCP
                    withCredentials([usernamePassword(credentialsId: WIN_CREDENTIALS_ID, 
                                   usernameVariable: 'WIN_USERNAME', 
                                   passwordVariable: 'WIN_PASSWORD')]) {
                        
                        // Install sshpass if not available for password-based SCP
                        sh """
                        if ! command -v sshpass &> /dev/null; then
                            echo "Installing sshpass..."
                            apt-get update && apt-get install -y sshpass
                        fi
                        
                        # Copy deployment package to Windows server
                        echo "Copying deployment package to Windows server..."
                        sshpass -p '${WIN_PASSWORD}' scp -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null \
                            deployment.tar.gz ${WIN_USERNAME}@${WIN_SERVER}:C:\\Temp\\deployment.tar.gz
                        """
                    }
                    
                    // Step 2: Execute deployment script on Windows server
                    withCredentials([usernamePassword(credentialsId: WIN_CREDENTIALS_ID, 
                                   usernameVariable: 'WIN_USERNAME', 
                                   passwordVariable: 'WIN_PASSWORD')]) {
                        
                        // Execute PowerShell deployment script
                        bat """
                        powershell -Command "& {
                            \\$securePassword = ConvertTo-SecureString '${WIN_PASSWORD}' -AsPlainText -Force
                            \\$credential = New-Object System.Management.Automation.PSCredential ('${WIN_USERNAME}', \\$securePassword)
                            
                            \\$sessionOptions = New-PSSessionOption -SkipCACheck -SkipCNCheck -SkipRevocationCheck
                            
                            Invoke-Command -ComputerName ${WIN_SERVER} -Credential \\$credential -SessionOption \\$sessionOptions -ScriptBlock {
                                param(\\$DeployPath, \\$TempPath, \\$AppPoolName, \\$SiteName)
                                
                                Write-Host "Starting deployment process..."
                                
                                # Extract deployment package
                                \\$deploymentPackage = "C:\\Temp\\deployment.tar.gz"
                                \\$extractPath = "C:\\Temp\\deployment_extracted"
                                
                                # Create extraction directory
                                if (Test-Path \\$extractPath) {
                                    Remove-Item \\$extractPath -Recurse -Force
                                }
                                New-Item -ItemType Directory -Path \\$extractPath -Force
                                
                                # Extract tar.gz - using 7zip if available
                                if (Get-Command 7z -ErrorAction SilentlyContinue) {
                                    Write-Host "Extracting with 7zip..."
                                    7z x \\$deploymentPackage -o"\\$extractPath" -y
                                } else {
                                    Write-Host "7zip not available, using manual extraction..."
                                    \\$stream = [System.IO.File]::OpenRead(\\$deploymentPackage)
                                    \\$gzipStream = New-Object System.IO.Compression.GZipStream(\\$stream, [System.IO.Compression.CompressionMode]::Decompress)
                                    \\$outFile = "\\$extractPath\\deployment.tar"
                                    \\$outStream = [System.IO.File]::Create(\\$outFile)
                                    \\$gzipStream.CopyTo(\\$outStream)
                                    \\$outStream.Close()
                                    \\$gzipStream.Close()
                                    \\$stream.Close()
                                }
                                
                                # Stop IIS application pool
                                Write-Host "Stopping IIS application pool..."
                                Import-Module WebAdministration
                                Stop-WebAppPool -Name \\$AppPoolName -ErrorAction SilentlyContinue
                                Start-Sleep -Seconds 2
                                
                                # Backup existing deployment
                                Write-Host "Creating backup..."
                                \\$timestamp = Get-Date -Format 'yyyyMMdd_HHmmss'
                                \\$backupPath = "\\${DeployPath}_Backup_\\$timestamp"
                                if (Test-Path \\$DeployPath) {
                                    if (Test-Path \\$backupPath) {
                                        Remove-Item \\$backupPath -Recurse -Force
                                    }
                                    Copy-Item \\$DeployPath \\$backupPath -Recurse -Force
                                    Write-Host "Backup created at: \\$backupPath"
                                }
                                
                                # Remove existing files
                                Write-Host "Cleaning deployment directory..."
                                if (Test-Path \\$DeployPath) {
                                    Remove-Item "\\${DeployPath}\\*" -Recurse -Force
                                } else {
                                    New-Item -ItemType Directory -Path \\$DeployPath -Force
                                }
                                
                                # Copy new files
                                Write-Host "Copying new files..."
                                \\$sourcePath = "\\$extractPath\\publish\\*"
                                if (Test-Path \\$sourcePath) {
                                    Copy-Item \\$sourcePath \\$DeployPath -Recurse -Force
                                    Write-Host "Files copied successfully from \\$sourcePath to \\$DeployPath"
                                } else {
                                    throw "Source files not found at: \\$sourcePath"
                                }
                                
                                # Set proper permissions
                                Write-Host "Setting permissions..."
                                \\$acl = Get-Acl \\$DeployPath
                                \\$accessRule = New-Object System.Security.AccessControl.FileSystemAccessRule(
                                    'IIS_IUSRS', 'ReadAndExecute', 'ContainerInherit,ObjectInherit', 'None', 'Allow')
                                \\$acl.SetAccessRule(\\$accessRule)
                                Set-Acl \\$DeployPath \\$acl
                                
                                # Clean up temp files
                                Write-Host "Cleaning up temporary files..."
                                Remove-Item \\$deploymentPackage -Force -ErrorAction SilentlyContinue
                                Remove-Item \\$extractPath -Recurse -Force -ErrorAction SilentlyContinue
                                
                                # Start IIS application pool
                                Write-Host "Starting IIS application pool..."
                                Start-WebAppPool -Name \\$AppPoolName
                                Start-Website -Name \\$SiteName
                                
                                Write-Host 'Deployment completed successfully!'
                                Write-Host 'Deployed to: ' + \\$DeployPath
                                
                            } -ArgumentList '${WIN_DEPLOY_PATH}', '${WIN_TEMP_PATH}', '${APP_POOL_NAME}', '${SITE_NAME}'
                        }"
                        """
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    echo "Verifying deployment..."
                    
                    // Simple verification by checking if the website responds
                    bat """
                    echo "Testing website availability..."
                    curl -I http://${WIN_SERVER} --connect-timeout 30 --max-time 60 || echo "Website check failed, but deployment may still be successful"
                    """
                    
                    // Additional verification steps
                    withCredentials([usernamePassword(credentialsId: WIN_CREDENTIALS_ID, 
                                   usernameVariable: 'WIN_USERNAME', 
                                   passwordVariable: 'WIN_PASSWORD')]) {
                        bat """
                        powershell -Command "& {
                            \\$securePassword = ConvertTo-SecureString '${WIN_PASSWORD}' -AsPlainText -Force
                            \\$credential = New-Object System.Management.Automation.PSCredential ('${WIN_USERNAME}', \\$securePassword)
                            
                            \\$result = Invoke-Command -ComputerName ${WIN_SERVER} -Credential \\$credential -ScriptBlock {
                                # Check if application pool is running
                                Import-Module WebAdministration
                                \\$appPoolStatus = (Get-WebAppPoolState -Name '${APP_POOL_NAME}').Value
                                \\$siteStatus = (Get-WebsiteState -Name '${SITE_NAME}').Value
                                \\$deployPathExists = Test-Path '${WIN_DEPLOY_PATH}'
                                \\$fileCount = if (\\$deployPathExists) { (Get-ChildItem '${WIN_DEPLOY_PATH}' -Recurse | Measure-Object).Count } else { 0 }
                                
                                return @{
                                    AppPoolStatus = \\$appPoolStatus
                                    SiteStatus = \\$siteStatus
                                    DeployPathExists = \\$deployPathExists
                                    FileCount = \\$fileCount
                                }
                            }
                            
                            Write-Host 'Application Pool Status: ' \\$result.AppPoolStatus
                            Write-Host 'Website Status: ' \\$result.SiteStatus
                            Write-Host 'Deployment Path Exists: ' \\$result.DeployPathExists
                            Write-Host 'Number of files deployed: ' \\$result.FileCount
                            
                            if (\\$result.AppPoolStatus -eq 'Started' -and \\$result.SiteStatus -eq 'Started' -and \\$result.DeployPathExists -eq $true -and \\$result.FileCount -gt 0) {
                                exit 0
                            } else {
                                exit 1
                            }
                        }"
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo 'Build, test, publish, and deployment successful!'
            emailext (
                subject: "SUCCESS: Deployment completed - ${env.JOB_NAME}",
                body: """
                The application was successfully deployed to ${WIN_SERVER}
                
                Details:
                - Server: ${WIN_SERVER}
                - Deployment Path: ${WIN_DEPLOY_PATH}
                - Application Pool: ${APP_POOL_NAME}
                - Website: ${SITE_NAME}
                - Build: ${env.BUILD_NUMBER}
                """,
                to: "m.shaaban@seu.edu.sa"
            )
        }
        failure {
            echo 'Deployment failed!'
            emailext (
                subject: "FAILURE: Deployment failed - ${env.JOB_NAME}",
                body: """
                The deployment to ${WIN_SERVER} failed. Please check Jenkins logs.
                
                Details:
                - Server: ${WIN_SERVER}
                - Deployment Path: ${WIN_DEPLOY_PATH}
                - Application Pool: ${APP_POOL_NAME}
                - Website: ${SITE_NAME}
                - Build: ${env.BUILD_NUMBER}
                """,
                to: "m.shaaban@seu.edu.sa"
            )
        }
        always {
            // Clean up workspace but keep the publish directory for inspection
            sh """
            echo "Cleaning up workspace..."
            rm -f deployment.tar.gz || true
            # Keep publish directory for debugging
            """
        }
    }
}