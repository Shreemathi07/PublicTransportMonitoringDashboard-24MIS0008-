pipeline {
    agent any

    parameters {
        booleanParam(
            name: 'SIMULATE_FAILURE',
            defaultValue: false,
            description: 'Test automatic Kubernetes rollback using an invalid image'
        )
    }

    environment {
        IMAGE_NAME = 'shreemathi07/public-transport-dashboard'
        K8S_DEPLOYMENT = 'public-transport-dashboard'
        K8S_CONTAINER = 'public-transport-dashboard'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo 'Source code checked out successfully.'
            }
        }

        stage('Parallel Validation') {
            parallel {
                stage('Validate HTML') {
                    steps {
                        powershell '''
                            if (!(Test-Path "index.html")) {
                                throw "index.html is missing"
                            }

                            $html = Get-Content "index.html" -Raw

                            if ($html -notmatch "<html" -or $html -notmatch "</html>") {
                                throw "HTML validation failed"
                            }

                            Write-Output "HTML validation passed."
                        '''
                    }
                }

                stage('Validate Docker and Kubernetes Files') {
                    steps {
                        powershell '''
                            if (!(Test-Path "Dockerfile")) {
                                throw "Dockerfile is missing"
                            }

                            if (!(Test-Path "deployment.yaml")) {
                                throw "deployment.yaml is missing"
                            }

                            Write-Output "Required deployment files exist."
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                powershell '''
                    $tag = "$env:BUILD_NUMBER"
                    docker build -t "$env:IMAGE_NAME`:$tag" .
                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker image build failed"
                    }

                    docker tag "$env:IMAGE_NAME`:$tag" "$env:IMAGE_NAME`:latest"
                    if ($LASTEXITCODE -ne 0) {
                        throw "Docker image tagging failed"
                    }
                '''
            }
        }

        stage('Test Docker Container') {
            steps {
                powershell '''
                    docker rm -f transport-dashboard-ci 2>$null | Out-Null

                    docker run -d --name transport-dashboard-ci -p 18081:80 "$env:IMAGE_NAME`:latest"
                    if ($LASTEXITCODE -ne 0) {
                        throw "Container failed to start"
                    }

                    try {
                        Start-Sleep -Seconds 5
                        $response = Invoke-WebRequest -Uri "http://localhost:18081" -UseBasicParsing -TimeoutSec 15

                        if ($response.StatusCode -ne 200) {
                            throw "Web application test failed"
                        }

                        Write-Output "Docker container HTTP test passed."
                    }
                    finally {
                        docker rm -f transport-dashboard-ci 2>$null | Out-Null
                    }
                '''
            }
        }

        stage('Push Image to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_TOKEN'
                )]) {
                    powershell '''
                        $env:DOCKER_TOKEN | docker login -u $env:DOCKER_USER --password-stdin
                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker Hub login failed"
                        }

                        docker push "$env:IMAGE_NAME`:$env:BUILD_NUMBER"
                        if ($LASTEXITCODE -ne 0) {
                            throw "Versioned image push failed"
                        }

                        docker push "$env:IMAGE_NAME`:latest"
                        if ($LASTEXITCODE -ne 0) {
                            throw "Latest image push failed"
                        }

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig-creds',
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    powershell '''
                        $env:KUBECONFIG = $env:KUBECONFIG_FILE

                        kubectl apply -f deployment.yaml
                        if ($LASTEXITCODE -ne 0) {
                            throw "Kubernetes manifest apply failed"
                        }

                        kubectl set image "deployment/$env:K8S_DEPLOYMENT" "$env:K8S_CONTAINER=$env:IMAGE_NAME`:$env:BUILD_NUMBER"
                        if ($LASTEXITCODE -ne 0) {
                            throw "Could not update Kubernetes image"
                        }

                        kubectl rollout status "deployment/$env:K8S_DEPLOYMENT" --timeout=120s
                        if ($LASTEXITCODE -ne 0) {
                            kubectl rollout undo "deployment/$env:K8S_DEPLOYMENT"
                            throw "Deployment rollout failed; rollback requested"
                        }

                        Write-Output "Kubernetes deployment completed successfully."
                    '''
                }
            }
        }

        stage('Test Automatic Rollback') {
            when {
                expression { return params.SIMULATE_FAILURE }
            }
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig-creds',
                    variable: 'KUBECONFIG_FILE'
                )]) {
                    powershell '''
                        $env:KUBECONFIG = $env:KUBECONFIG_FILE

                        kubectl set image "deployment/$env:K8S_DEPLOYMENT" "$env:K8S_CONTAINER=$env:IMAGE_NAME`:faulty"
                        if ($LASTEXITCODE -ne 0) {
                            throw "Could not introduce the test failure"
                        }

                        kubectl rollout status "deployment/$env:K8S_DEPLOYMENT" --timeout=45s

                        if ($LASTEXITCODE -ne 0) {
                            Write-Output "Faulty rollout detected. Rolling back to the previous stable release."

                            kubectl rollout undo "deployment/$env:K8S_DEPLOYMENT"
                            if ($LASTEXITCODE -ne 0) {
                                throw "Automatic rollback command failed"
                            }

                            kubectl rollout status "deployment/$env:K8S_DEPLOYMENT" --timeout=120s
                            if ($LASTEXITCODE -ne 0) {
                                throw "Rollback did not become healthy"
                            }

                            Write-Output "Automatic rollback succeeded."
                        }
                        else {
                            throw "The faulty rollout unexpectedly succeeded."
                        }

                        $response = Invoke-WebRequest -Uri "http://localhost:30081" -UseBasicParsing -TimeoutSec 15
                        if ($response.StatusCode -ne 200) {
                            throw "Application did not respond after rollback"
                        }

                        Write-Output "Application is responding after rollback."
                    '''
                }
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'index.html,Dockerfile,deployment.yaml,Jenkinsfile', fingerprint: true
        }

        success {
            echo 'Pipeline completed successfully.'
        }

        failure {
            echo 'Pipeline failed. Review the stage logs in Console Output.'
        }
    }