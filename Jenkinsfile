pipeline{
    agent any

    environment{
        // Define environment variables here
        // For example:
        // MY_VAR = 'value'
        PYTHON_VERSION = "3.10"
        VENV_NAME = "venv"
        AZURE_WEBAPP_NAME = "cicd-webapp-dev"
        AZURE_RESOURCE_GROUP = "cicd-webapp"

        // Azure credentials (for login) - these should ideally be stored securely in Jenkins credentials and accessed via environment variables
        SUBSCRIPTION_ID = 'fe8af34a-02eb-4b77-9796-eee984bbad83'
        TENANT_ID       = '0dba4b73-81df-44a7-9539-08ec0252d600'
        AZURE_CLI_PATH  = 'C:\\Program Files\\Microsoft SDKs\\Azure\\CLI2\\wbin\\az.cmd'
    }
    stages{
        stage('Checkout'){
            steps{
                checkout scm
            }
        }
        stage('Setup Python and Dependencies'){
            steps{
                powershell '''
                # Install Python if not already installed
                Write-Host "Python version:"
                python --version

                # create virtual environment
                python -m venv $env:VENV_NAME

                # activate virtual environment and install dependencies
                & .\\$env:VENV_NAME\\Scripts\\Activate.ps1
                pip install --upgrade pip
                pip install -r requirements.txt

                # Entwicklungspakete installieren (falls vorhanden)
                if (Test-Path "requirements-dev.txt") {
                    pip install -r requirements-dev.txt
                }
                '''
            }
        }
        stage('Test'){
            steps{
                powershell '''
                # activate virtual environment and run tests
                & .\\$env:VENV_NAME\\Scripts\\Activate.ps1
                Write-Host "Running tests..."
                python -m pytest --junitxml=test-results.xml --cov=. --cov-report=xml
                '''
            }
            post{
                always{
                    junit 'test-results.xml'
                }
            }
        }
        stage('Code Quality'){
            steps{
                powershell '''
                # activate virtual environment and run code quality checks
                & .\\$env:VENV_NAME\\Scripts\\Activate.ps1
                Write-Host "Running code quality checks..."
                python -m pylint --exit-zero **/*.py > pylint-report.txt
                python -m flake8 --exit-zero **/*.py > flake8-report.txt
                '''
            }
        }
        stage('Create Deployment Package'){
            steps{
                powershell '''
                Write-Host "Creating Deployment Package..."
                New-Item -ItemType Directory -Force -Path publish | Out-Null

                # activate virtual environment and create deployment package
                & .\\$env:VENV_NAME\\Scripts\\Activate.ps1

                # Create a zip file of the application
                Compress-Archive -Path .\\* -DestinationPath publish\\app.zip -Force
                Write-Host "Deployment package created: app.zip"
                '''
                archiveArtifacts artifacts: 'publish/app.zip', fingerprint: true
            }
        }
        stage('Deploy to Dev') {
            when { branch 'main' }
            steps {
                withCredentials([usernamePassword(credentialsId: 'azure-sp-credentials',
                                  usernameVariable: 'AZURE_CLIENT_ID',
                                  passwordVariable: 'AZURE_CLIENT_SECRET')]) {
                    
                    powershell '''
                        Write-Host "=== Azure Login ==="
                        & $env:AZURE_CLI_PATH login --service-principal `
                            -u $env:AZURE_CLIENT_ID `
                            -p $env:AZURE_CLIENT_SECRET `
                            --tenant $env:TENANT_ID
                        
                        & $env:AZURE_CLI_PATH account set --subscription $env:SUBSCRIPTION_ID
                        
                        Write-Host "=== Deploying Python App ==="
                        cd publish
                        Compress-Archive -Path * -DestinationPath app.zip -Force
                        
                        & $env:AZURE_CLI_PATH webapp deployment source config-zip `
                            --resource-group $env:RESOURCE_GROUP `
                            --name $env:APP_NAME_DEV `
                            --src app.zip
                    '''
                }
            }
        }
    }
    post{
        always{
            deleteDir()
        }
    }
}