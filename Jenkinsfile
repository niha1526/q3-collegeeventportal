
pipeline {
    agent any

    environment {
        IMAGE = 'niharikarao15/college-event-portal'
    }

    stages {
        stage('Clone Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                powershell '''
                    $image = "$($env:IMAGE):$($env:BUILD_NUMBER)"
                    docker build -t $image .
                    if ($LASTEXITCODE -ne 0) { exit 1 }
                '''
            }
        }

        stage('Test Application') {
            steps {
                powershell '''
                    $image = "$($env:IMAGE):$($env:BUILD_NUMBER)"
                    docker rm -f college-event-ci-test 2>$null
                    docker run -d --name college-event-ci-test -p 18083:80 $image
                    if ($LASTEXITCODE -ne 0) { exit 1 }

                    try {
                        $ready = $false
                        for ($i = 0; $i -lt 12; $i++) {
                            try {
                                $response = Invoke-WebRequest `
                                    -Uri "http://localhost:18083" `
                                    -TimeoutSec 3
                                if ($response.StatusCode -eq 200) {
                                    $ready = $true
                                    break
                                }
                            } catch {
                                Start-Sleep -Seconds 2
                            }
                        }
                        if (-not $ready) {
                            throw "Website health check failed"
                        }
                        Write-Host "Website test passed"
                    } finally {
                        docker rm -f college-event-ci-test
                    }
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKERHUB_USER',
                    passwordVariable: 'DOCKERHUB_TOKEN'
                )]) {
                    powershell '''
                        $env:DOCKERHUB_TOKEN |
                            docker login -u $env:DOCKERHUB_USER --password-stdin
                        if ($LASTEXITCODE -ne 0) { exit 1 }

                        $image = "$($env:IMAGE):$($env:BUILD_NUMBER)"
                        docker push $image
                        if ($LASTEXITCODE -ne 0) { exit 1 }

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(
                    credentialsId: 'kubeconfig',
                    variable: 'KUBECONFIG'
                )]) {
                    powershell '''
                        kubectl apply -f deployment.yaml
                        if ($LASTEXITCODE -ne 0) { exit 1 }

                        $image = "$($env:IMAGE):$($env:BUILD_NUMBER)"
                        kubectl set image deployment/college-event-portal "web=$image"
                        if ($LASTEXITCODE -ne 0) { exit 1 }

                        kubectl rollout status deployment/college-event-portal --timeout=120s
                        if ($LASTEXITCODE -ne 0) {
                            Write-Host "Rollout failed; restoring previous revision"
                            kubectl rollout undo deployment/college-event-portal
                            kubectl rollout status deployment/college-event-portal --timeout=120s
                            exit 1
                        }
                    '''
                }
            }
        }
    }

    post {
        always {
            echo 'CI/CD pipeline finished. Review the stage results above.'
        }
    }
}
