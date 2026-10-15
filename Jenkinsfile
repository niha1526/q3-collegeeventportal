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
                    if ($LASTEXITCODE -ne 0) { throw "Failed to start test container" }

                    try {
                        $ready = $false
                        for ($i = 0; $i -lt 10; $i++) {
                            docker exec college-event-ci-test sh -c "wget -q -O /dev/null http://127.0.0.1/"
                            if ($LASTEXITCODE -eq 0) { $ready = $true; break }
                            Start-Sleep -Seconds 2
                        }
                        if (-not $ready) { throw "Nginx health check failed inside container" }
                        Write-Host "Website test passed: Nginx returned a successful response"
                    }
                    finally {
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
                $u = $env:DOCKERHUB_USER
                $t = $env:DOCKERHUB_TOKEN
                Write-Host "User  : [$u] length=$($u.Length)"
                Write-Host "Token : starts [$($t.Substring(0,[Math]::Min(9,$t.Length)))] length=$($t.Length)"

                $image = "$($env:IMAGE):$($env:BUILD_NUMBER)"

                docker login -u $u -p $t
                if ($LASTEXITCODE -ne 0) { throw "Docker Hub login failed" }

                docker push $image
                if ($LASTEXITCODE -ne 0) { throw "Docker image push failed" }

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
