
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

            if ($LASTEXITCODE -ne 0) {
                throw "Failed to start test container"
            }

            try {
                $ready = $false

                for ($i = 0; $i -lt 10; $i++) {
                    docker exec college-event-ci-test sh -c "wget -q -O /dev/null http://127.0.0.1/"

                    if ($LASTEXITCODE -eq 0) {
                        $ready = $true
                        break
                    }

                    Start-Sleep -Seconds 2
                }

                if (-not $ready) {
                    throw "Nginx health check failed inside container"
                }

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
                $ErrorActionPreference = "Stop"
                $image = "$($env:IMAGE):$($env:BUILD_NUMBER)"

                2> $token = $env:DOCKERHUB_TOKEN
PS C:\Windows\System32> $username = $env:DOCKERHUB_USER
PS C:\Windows\System32>
PS C:\Windows\System32> if ([string]::IsNullOrWhiteSpace($token)) {
>>     throw "Docker Hub token was not received from Jenkins credentials"
>> }
Docker Hub token was not received from Jenkins credentials
At line:2 char:5
+     throw "Docker Hub token was not received from Jenkins
credentials ...
+     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : OperationStopped: (Docker Hub toke...ins
    credentials:String) [], RuntimeException
    + FullyQualifiedErrorId : Docker Hub token was not received from J
   enkins credentials

PS C:\Windows\System32>
PS C:\Windows\System32> if ([string]::IsNullOrWhiteSpace($username)) {
>>     throw "Docker Hub username was not received from Jenkins credentials"
>> }
Docker Hub username was not received from Jenkins credentials
At line:2 char:5
+     throw "Docker Hub username was not received from Jenkins
credenti ...
+     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : OperationStopped: (Docker Hub user...ins
    credentials:String) [], RuntimeException
    + FullyQualifiedErrorId : Docker Hub username was not received fro
   m Jenkins credentials

PS C:\Windows\System32>
PS C:\Windows\System32> $token | docker login --username $username --password-stdin
error: cannot perform an interactive login from a non-TTY device
PS C:\Windows\System32>
PS C:\Windows\System32> if ($LASTEXITCODE -ne 0) {
>>     throw "Docker Hub login failed. Check the Jenkins credential values and token permissions."
>> }
Docker Hub login failed. Check the Jenkins credential values and token
permissions.
At line:2 char:5
+     throw "Docker Hub login failed. Check the Jenkins credential
valu ...
+     ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
    + CategoryInfo          : OperationStopped: (Docker Hub logi...en
   permissions.:String) [], RuntimeException
    + FullyQualifiedErrorId : Docker Hub login failed. Check the Jenki
   ns credential values and token permissions.

PS C:\Windows\System32>
PS C:\Windows\System32>

                if ($LASTEXITCODE -ne 0) {
                    throw "Docker Hub login failed"
                }

                docker push $image

                if ($LASTEXITCODE -ne 0) {
                    throw "Docker image push failed"
                }

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
