pipeline {
    agent any

    parameters {
        booleanParam(
            name: 'SIMULATE_FAILURE',
            defaultValue: false,
            description: 'Simulate a failed Kubernetes release and test automatic rollback'
        )
    }

    environment {
        IMAGE_NAME = 'shreemathi07/student-placement-portal'
        DEPLOYMENT_NAME = 'placement-portal'
        CONTAINER_NAME = 'placement-portal'
        KUBE_NAMESPACE = 'default'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out the application source code'
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                powershell '''
                    $ErrorActionPreference = "Stop"

                    $tag = "v${env:BUILD_NUMBER}"
                    Write-Host "Building image: $env:IMAGE_NAME`:$tag"

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

        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_TOKEN'
                    )
                ]) {
                    powershell '''
                        $ErrorActionPreference = "Stop"

                        $env:DOCKERHUB_TOKEN |
                            docker login -u $env:DOCKERHUB_USER --password-stdin

                        if ($LASTEXITCODE -ne 0) {
                            throw "Docker Hub login failed"
                        }

                        $tag = "v${env:BUILD_NUMBER}"

                        docker push "$env:IMAGE_NAME`:$tag"
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
                withCredentials([
                    file(
                        credentialsId: 'kubeconfig-creds',
                        variable: 'KUBECONFIG_FILE'
                    )
                ]) {
                    powershell '''
                        $ErrorActionPreference = "Stop"
                        $env:KUBECONFIG = $env:KUBECONFIG_FILE

                        $tag = "v${env:BUILD_NUMBER}"
                        $image = "$env:IMAGE_NAME`:$tag"

                        Write-Host "Deploying image: $image"

                        kubectl apply -f deployment.yaml
                        if ($LASTEXITCODE -ne 0) {
                            throw "Kubernetes manifest application failed"
                        }

                        kubectl set image "deployment/$env:DEPLOYMENT_NAME" "$env:CONTAINER_NAME=$image"
                        if ($LASTEXITCODE -ne 0) {
                            throw "Unable to update the deployment image"
                        }

                        kubectl rollout status "deployment/$env:DEPLOYMENT_NAME" --timeout=120s
                        if ($LASTEXITCODE -ne 0) {
                            throw "Deployment rollout failed"
                        }

                        Write-Host "Stable deployment completed successfully."

                        if ("${env:SIMULATE_FAILURE}" -eq "true") {
                            Write-Host "SIMULATION: deploying a deliberately invalid image tag."

                            kubectl set image "deployment/$env:DEPLOYMENT_NAME" "$env:CONTAINER_NAME=$env:IMAGE_NAME`:faulty"
                            if ($LASTEXITCODE -ne 0) {
                                throw "Could not initiate the simulated faulty release"
                            }

                            kubectl rollout status "deployment/$env:DEPLOYMENT_NAME" --timeout=60s

                            if ($LASTEXITCODE -ne 0) {
                                Write-Host "Faulty rollout detected. Rolling back to the previous stable release."

                                kubectl rollout undo "deployment/$env:DEPLOYMENT_NAME"
                                if ($LASTEXITCODE -ne 0) {
                                    throw "Automatic rollback command failed"
                                }

                                kubectl rollout status "deployment/$env:DEPLOYMENT_NAME" --timeout=120s
                                if ($LASTEXITCODE -ne 0) {
                                    throw "Rollback did not restore a healthy deployment"
                                }

                                Write-Host "Automatic rollback completed successfully."
                            }
                            else {
                                throw "The simulated faulty release unexpectedly succeeded."
                            }
                        }
                    '''
                }
            }
        }

        stage('Verify Deployment') {
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

                        kubectl get deployment $env:DEPLOYMENT_NAME
                        if ($LASTEXITCODE -ne 0) {
                            throw "Could not retrieve deployment status"
                        }

                        kubectl get pods -l app=placement-portal
                        if ($LASTEXITCODE -ne 0) {
                            throw "Could not retrieve pod status"
                        }

                        kubectl get service placement-portal-service
                        if ($LASTEXITCODE -ne 0) {
                            throw "Could not retrieve service status"
                        }

                        Write-Host "Deployment verification completed."
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully.'
        }

        failure {
            echo 'CI/CD pipeline failed. Check the Jenkins Console Output for details.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}