
pipeline {
    agent any

    parameters {
        booleanParam(
            name: 'SIMULATE_FAILURE',
            defaultValue: false,
            description: 'Test Kubernetes automatic rollback using an invalid image'
        )
    }

    environment {
        DOCKER_IMAGE = 'shreemathi07/public-transport-dashboard'
        KUBE_DEPLOYMENT = 'public-transport-dashboard'
        KUBE_CONTAINER = 'public-transport-dashboard'
        KUBE_SERVICE = 'public-transport-dashboard-service'
        APP_URL = 'http://localhost:30081'
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                echo 'Source code has been checked out from GitHub.'
                powershell '''
                    if (!(Test-Path "index.html")) {
                        throw "index.html was not found."
                    }

                    if (!(Test-Path "Dockerfile")) {
                        throw "Dockerfile was not found."
                    }

                    if (!(Test-Path "deployment.yaml")) {
                        throw "deployment.yaml was not found."
                    }

                    Write-Host "Required project files are present."
                '''
            }
        }

        stage('Parallel Validation') {
            parallel {
                stage('Validate HTML') {
                    steps {
                        powershell '''
                            $html = Get-Content "index.html" -Raw

                            if ($html -notmatch '(?i)<html') {
                                throw "HTML opening tag was not found."
                            }

                            if ($html -notmatch '(?i)</html>') {
                                throw "HTML closing tag was not found."
                            }

                            if ($html -notmatch '(?i)<body') {
                                throw "HTML body was not found."
                            }

                            Write-Host "HTML validation passed."
                        '''
                    }
                }

                stage('Validate Deployment Files') {
                    steps {
                        powershell '''
                            $dockerfile = Get-Content "Dockerfile" -Raw
                            $deployment = Get-Content "deployment.yaml" -Raw

                            if ($dockerfile -notmatch '(?im)^FROM\\s+') {
                                throw "Dockerfile must contain a FROM instruction."
                            }

                            if ($deployment -notmatch 'kind:\\s*Deployment') {
                                throw "Kubernetes Deployment definition not found."
                            }

                            if ($deployment -notmatch 'kind:\\s*Service') {
                                throw "Kubernetes Service definition not found."
                            }

                            Write-Host "Deployment file validation passed."
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Stop"

                    docker build -t "$env:DOCKER_IMAGE:$env:BUILD_NUMBER" .
                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker image build failed."
                    }

                    docker tag "$env:DOCKER_IMAGE:$env:BUILD_NUMBER" "$env:DOCKER_IMAGE:latest"
                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker image tagging failed."
                    }

                    Write-Host "Docker image built and tagged successfully."
                '''
            }
        }

        stage('Test Docker Container') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Stop"
                    $containerName = "pt-dashboard-test-$env:BUILD_NUMBER"

                    try {
                        docker run -d --name $containerName -p 18080:80 "$env:DOCKER_IMAGE:$env:BUILD_NUMBER"

                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker test container failed to start."
                        }

                        $ready = $false

                        for ($i = 0; $i -lt 12; $i++) {
                            Start-Sleep -Seconds 2

                            try {
                                $response = Invoke-WebRequest -Uri "http://localhost:18080" -UseBasicParsing -TimeoutSec 3

                                if ($response.StatusCode -eq 200) {
                                    $ready = $true
                                    break
                                }
                            }
                            catch {
                                Write-Host "Waiting for the application to respond..."
                            }
                        }

                        if (-not $ready) {
                            throw "Docker application did not return HTTP 200."
                        }

                        Write-Host "Docker application test passed."
                    }
                    finally {
                        docker rm -f $containerName 2>$null | Out-Null
                    }
                '''
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    powershell '''
                        $ErrorActionPreference = "Stop"

                        $secureToken = ConvertTo-SecureString $env:DOCKER_TOKEN -AsPlainText -Force
                        $credential = New-Object System.Management.Automation.PSCredential($env:DOCKER_USER, $secureToken)

                        $credential.Password | docker login -u $env:DOCKER_USER --password-stdin

                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker Hub login failed. Check the Jenkins dockerhub-creds credential."
                        }

                        docker push "$env:DOCKER_IMAGE:$env:BUILD_NUMBER"
                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker Hub push of the build image failed."
                        }

                        docker push "$env:DOCKER_IMAGE:latest"
                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker Hub push of the latest image failed."
                        }

                        Write-Host "Docker images pushed to Docker Hub successfully."
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig-creds',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    powershell '''
                        $ErrorActionPreference = "Stop"
                        $env:KUBECONFIG = $env:KUBECONFIG_FILE

                        kubectl apply -f deployment.yaml
                        if ($LASTEXITCODE -ne 0) {
                            throw "Kubernetes manifest application failed."
                        }

                        kubectl set image "deployment/$env:KUBE_DEPLOYMENT" "$env:KUBE_CONTAINER=$env:DOCKER_IMAGE`:$env:BUILD_NUMBER"
                        if ($LASTEXITCODE -ne 0) {
                            throw "Kubernetes image update failed. Check the deployment and container names."
                        }

                        kubectl rollout status "deployment/$env:KUBE_DEPLOYMENT" --timeout=180s
                        if ($LASTEXITCODE -ne 0) {
                            throw "Kubernetes deployment rollout failed."
                        }

                        kubectl get deployments
                        kubectl get services

                        Write-Host "Kubernetes deployment completed successfully."
                    '''
                }
            }
        }

        stage('Verify Application') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Stop"
                    $ready = $false

                    for ($i = 0; $i -lt 15; $i++) {
                        try {
                            $response = Invoke-WebRequest -Uri $env:APP_URL -UseBasicParsing -TimeoutSec 5

                            if ($response.StatusCode -eq 200) {
                                $ready = $true
                                break
                            }
                        }
                        catch {
                            Write-Host "Waiting for the Kubernetes application..."
                        }

                        Start-Sleep -Seconds 4
                    }

                    if (-not $ready) {
                        throw "The application did not respond successfully at $env:APP_URL."
                    }

                    Write-Host "Kubernetes application verification passed."
                '''
            }
        }

        stage('Test Automatic Rollback') {
            when {
                expression {
                    return params.SIMULATE_FAILURE
                }
            }

            steps {
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig-creds',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    powershell '''
                        $ErrorActionPreference = "Stop"
                        $env:KUBECONFIG = $env:KUBECONFIG_FILE

                        $previousImage = kubectl get deployment $env:KUBE_DEPLOYMENT -o "jsonpath={.spec.template.spec.containers[0].image}"

                        if ($LASTEXITCODE -ne 0 -or [string]::IsNullOrWhiteSpace($previousImage)) {
                            throw "Could not determine the current Kubernetes image."
                        }

                        Write-Host "Previous image recorded."

                        kubectl set image "deployment/$env:KUBE_DEPLOYMENT" "$env:KUBE_CONTAINER=$env:DOCKER_IMAGE`:invalid-image"
                        if ($LASTEXITCODE -ne 0) {
                            throw "Could not start the rollback test."
                        }

                        Start-Sleep -Seconds 20

                        kubectl rollout undo "deployment/$env:KUBE_DEPLOYMENT"
                        if ($LASTEXITCODE -ne 0) {
                            throw "Kubernetes rollback command failed."
                        }

                        kubectl rollout status "deployment/$env:KUBE_DEPLOYMENT" --timeout=180s
                        if ($LASTEXITCODE -ne 0) {
                            throw "The deployment did not recover after rollback."
                        }

                        Write-Host "Kubernetes rollback test passed."
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts(
                artifacts: 'index.html,Dockerfile,deployment.yaml,Jenkinsfile',
                allowEmptyArchive: true
            )
        }

        success {
            echo 'SUCCESS: CI/CD pipeline completed successfully.'
        }

        failure {
            echo 'FAILURE: Check the failed stage in the Jenkins console output.'
        }
    }
}
