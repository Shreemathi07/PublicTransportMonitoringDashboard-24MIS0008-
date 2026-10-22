```groovy
pipeline {
    agent any

    parameters {
        booleanParam(
            name: 'SIMULATE_FAILURE',
            defaultValue: false,
            description: 'Run an optional Kubernetes rollback test'
        )
    }

    environment {
        DOCKER_CLI = 'C:\\Users\\shree\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\docker.exe'
        KUBECTL_CLI = 'C:\\Users\\shree\\AppData\\Local\\Programs\\DockerDesktop\\resources\\bin\\kubectl.exe'

        DOCKER_IMAGE = 'shreemathi07/public-transport-dashboard'
        KUBE_DEPLOYMENT = 'public-transport-dashboard'
        KUBE_CONTAINER = 'public-transport-dashboard'
        APP_URL = 'http://localhost:30081'
    }

    stages {
        stage('Checkout Source Code') {
            steps {
                echo 'Source code checked out from GitHub.'

                powershell '''
                    $ErrorActionPreference = "Stop"

                    foreach ($file in @("index.html", "Dockerfile", "deployment.yaml")) {
                        if (!(Test-Path $file)) {
                            throw "Required file is missing: $file"
                        }
                    }

                    if (!(Test-Path $env:DOCKER_CLI)) {
                        throw "Docker executable not found at $env:DOCKER_CLI"
                    }

                    if (!(Test-Path $env:KUBECTL_CLI)) {
                        throw "kubectl executable not found at $env:KUBECTL_CLI"
                    }

                    Write-Host "Required files and command-line tools verified."
                '''
            }
        }

        stage('Parallel Validation') {
            parallel {
                stage('Validate HTML') {
                    steps {
                        powershell '''
                            $ErrorActionPreference = "Stop"
                            $html = Get-Content "index.html" -Raw

                            if ($html -notmatch '(?i)<html') {
                                throw "HTML opening tag missing."
                            }

                            if ($html -notmatch '(?i)</html>') {
                                throw "HTML closing tag missing."
                            }

                            if ($html -notmatch '(?i)<body') {
                                throw "HTML body missing."
                            }

                            Write-Host "HTML validation passed."
                        '''
                    }
                }

                stage('Validate Deployment Files') {
                    steps {
                        powershell '''
                            $ErrorActionPreference = "Stop"
                            $dockerfile = Get-Content "Dockerfile" -Raw
                            $deployment = Get-Content "deployment.yaml" -Raw

                            if ($dockerfile -notmatch '(?im)^FROM\\s+') {
                                throw "Dockerfile FROM instruction missing."
                            }

                            if ($deployment -notmatch 'kind:\\s*Deployment') {
                                throw "Kubernetes Deployment missing."
                            }

                            if ($deployment -notmatch 'kind:\\s*Service') {
                                throw "Kubernetes Service missing."
                            }

                            Write-Host "Deployment files validated."
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Stop"

                    & $env:DOCKER_CLI build -t "$($env:DOCKER_IMAGE):$($env:BUILD_NUMBER)" .
                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker image build failed."
                    }

                    & $env:DOCKER_CLI tag "$($env:DOCKER_IMAGE):$($env:BUILD_NUMBER)" "$($env:DOCKER_IMAGE):latest"
                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker image tagging failed."
                    }

                    Write-Host "Docker image built successfully."
                '''
            }
        }

        stage('Test Docker Container') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Stop"
                    $containerName = "pt-dashboard-test-$env:BUILD_NUMBER"

                    try {
                        & $env:DOCKER_CLI run -d --name $containerName -p 18080:80 "$($env:DOCKER_IMAGE):$($env:BUILD_NUMBER)"

                        if ($LASTEXITCODE -ne 0) {
                            throw "Unable to start Docker test container."
                        }

                        $ready = $false

                        for ($i = 0; $i -lt 15; $i++) {
                            Start-Sleep -Seconds 2

                            try {
                                $response = Invoke-WebRequest -Uri "http://localhost:18080" -UseBasicParsing -TimeoutSec 3

                                if ($response.StatusCode -eq 200) {
                                    $ready = $true
                                    break
                                }
                            }
                            catch {
                                Write-Host "Waiting for application startup..."
                            }
                        }

                        if (-not $ready) {
                            throw "Docker application health check failed."
                        }

                        Write-Host "Docker container test passed."
                    }
                    finally {
                        & $env:DOCKER_CLI rm -f $containerName 2>$null | Out-Null
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

                        if ([string]::IsNullOrWhiteSpace($env:DOCKER_USER) -or
                            [string]::IsNullOrWhiteSpace($env:DOCKER_TOKEN)) {
                            throw "Docker Hub Jenkins credentials are empty."
                        }

                        $token = ConvertTo-SecureString $env:DOCKER_TOKEN -AsPlainText -Force
                        $credential = New-Object System.Management.Automation.PSCredential($env:DOCKER_USER, $token)

                        $credential.Password | & $env:DOCKER_CLI login --username $env:DOCKER_USER --password-stdin

                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker Hub authentication failed. Check dockerhub-creds."
                        }

                        & $env:DOCKER_CLI push "$($env:DOCKER_IMAGE):$($env:BUILD_NUMBER)"
                        if ($LASTEXITCODE -ne 0) {
                            throw "Failed to push the numbered Docker image."
                        }

                        & $env:DOCKER_CLI push "$($env:DOCKER_IMAGE):latest"
                        if ($LASTEXITCODE -ne 0) {
                            throw "Failed to push the latest Docker image."
                        }

                        & $env:DOCKER_CLI logout

                        Write-Host "Docker Hub push completed successfully."
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

                        & $env:KUBECTL_CLI apply -f deployment.yaml
                        if ($LASTEXITCODE -ne 0) {
                            throw "Failed to apply Kubernetes manifests."
                        }

                        & $env:KUBECTL_CLI set image "deployment/$env:KUBE_DEPLOYMENT" "$($env:KUBE_CONTAINER)=$($env:DOCKER_IMAGE):$($env:BUILD_NUMBER)"
                        if ($LASTEXITCODE -ne 0) {
                            throw "Failed to update the Kubernetes deployment image."
                        }

                        & $env:KUBECTL_CLI rollout status "deployment/$env:KUBE_DEPLOYMENT" --timeout=180s
                        if ($LASTEXITCODE -ne 0) {
                            throw "Kubernetes rollout failed."
                        }

                        & $env:KUBECTL_CLI get deployments
                        & $env:KUBECTL_CLI get services

                        Write-Host "Kubernetes deployment succeeded."
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
                            Write-Host "Waiting for application availability..."
                        }

                        Start-Sleep -Seconds 4
                    }

                    if (-not $ready) {
                        throw "Application health check failed at $env:APP_URL"
                    }

                    Write-Host "Application verification passed."
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

                        & $env:KUBECTL_CLI set image "deployment/$env:KUBE_DEPLOYMENT" "$($env:KUBE_CONTAINER)=$($env:DOCKER_IMAGE):invalid-image"

                        if ($LASTEXITCODE -ne 0) {
                            throw "Could not initiate rollback test."
                        }

                        Start-Sleep -Seconds 20

                        & $env:KUBECTL_CLI rollout undo "deployment/$env:KUBE_DEPLOYMENT"
                        if ($LASTEXITCODE -ne 0) {
                            throw "Kubernetes rollback command failed."
                        }

                        & $env:KUBECTL_CLI rollout status "deployment/$env:KUBE_DEPLOYMENT" --timeout=180s
                        if ($LASTEXITCODE -ne 0) {
                            throw "Deployment did not recover after rollback."
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
            echo 'FAILURE: Inspect the failed stage in Console Output.'
        }
    }
}
```
