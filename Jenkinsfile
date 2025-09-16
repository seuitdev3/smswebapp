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
                        // Install ICU library based on Linux distribution
                        sh '''
                            # Detect distribution and install ICU
                            if [ -f /etc/redhat-release ] || [ -f /etc/centos-release ]; then
                                # RedHat/CentOS
                                sudo yum install -y libicu
                            elif [ -f /etc/debian_version ]; then
                                # Debian/Ubuntu
                                sudo apt-get update
                                sudo apt-get install -y libicu-dev
                            elif [ -f /etc/alpine-release ]; then
                                # Alpine Linux
                                apk add --no-cache icu-libs
                            else
                                echo "Unsupported Linux distribution"
                                exit 1
                            fi
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
