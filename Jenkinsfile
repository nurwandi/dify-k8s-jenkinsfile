pipeline {
    
    agent {label 'victus'}

    environment {
        // repository
        DIFY_REPOSITORY = "https://github.com/langgenius/dify.git"
        DIFY_YAML_MANIFEST = "https://github.com/nurwandi/dify-k8s.git"
        DIFY_SANDBOX = "https://github.com/langgenius/dify-sandbox.git"
        DIFY_DAEMON_PLUGIN = "https://github.com/langgenius/dify-plugin-daemon.git"

        // path
        BUILDS_PATH = "D:\\TECHNICAL TEST"
        DIFY_REPOSITORY_PATH = "${env.BUILDS_PATH}\\dify"
        DIFY_YAML_MANIFEST_PATH = "${env.BUILDS_PATH}\\dify\\dify-k8s"
        DIFY_SANDBOX_PATH = "${env.BUILDS_PATH}\\dify\\dify-k8s\\dify-sandbox"
        DIFY_DAEMON_PLUGIN_PATH = "${env.BUILDS_PATH}\\dify\\dify-k8s\\dify-plugin-daemon"
    }

    stages {
        stage('Clone or Update Repository') {
            steps {
                script {

                    try {
                        // clone or update main repository
                        if (fileExists(env.DIFY_REPOSITORY_PATH)) {
                            echo "Main repository exists. Pulling latest changes..."
                            dir(env.DIFY_REPOSITORY_PATH) {
                                command("git switch main")
                                command("git fetch origin main")
                                command("git stash --include-untracked")
                                command("git pull --rebase origin main")
                            }
                        } else {
                            dir(env.BUILDS_PATH) {
                                echo "Main repository does not exist. Cloning..."
                                command("git clone --branch main ${env.DIFY_REPOSITORY}")
                            }
                        }

                        // clone or update manifest repo
                        if (fileExists(env.DIFY_YAML_MANIFEST_PATH)) {
                            echo "Manifest repository exists. Pulling latest changes..."
                            dir(env.DIFY_YAML_MANIFEST_PATH) {
                                command("git switch master")
                                command("git fetch origin master")
                                command("git stash --include-untracked")
                                command("git pull --rebase origin master")
                            }
                        } else {
                            dir(env.DIFY_REPOSITORY_PATH) {
                                echo "Manifest repository does not exist. Cloning..."
                                command("git clone --branch master ${env.DIFY_YAML_MANIFEST}")
                            }
                        }

                        // clone or update sandbox repo
                        if (fileExists(env.DIFY_SANDBOX_PATH)) {
                            echo "Sandbox repository exists. Pulling latest changes..."
                            dir(env.DIFY_SANDBOX_PATH) {
                                command("git switch main")
                                command("git fetch origin main")
                                command("git stash --include-untracked")
                                command("git pull --rebase origin main")
                            }
                        } else {
                            dir(env.DIFY_YAML_MANIFEST_PATH) {
                                echo "Sandbox repository does not exist. Cloning..."
                                command("git clone --branch main ${env.DIFY_SANDBOX}")
                            }
                        }

                        // clone or update daemon repo
                        if (fileExists(env.DIFY_DAEMON_PLUGIN_PATH)) {
                            echo "Daemon plugin repository exists. Pulling latest changes..."
                            dir(env.DIFY_DAEMON_PLUGIN_PATH) {
                                command("git switch main")
                                command("git fetch origin main")
                                command("git stash --include-untracked")
                                command("git pull --rebase origin main")
                            }
                        } else {
                            dir(env.DIFY_YAML_MANIFEST_PATH) {
                                echo "Daemon plugin repository does not exist. Cloning..."
                                command("git clone --branch main ${env.DIFY_DAEMON_PLUGIN}")
                            }
                        }


                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        env.FAILURE_MESSAGE = "Failed to clone or update the repository: ${e.message}"
                        error(env.FAILURE_MESSAGE)
                    }
                }
            }
        }   
        stage('Retrieve Credentials from S3') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-credentials', 
                    usernameVariable: 'AWS_ACCESS_KEY_ID', 
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {

                    script {
                        try {
                            dir(env.DIFY_YAML_MANIFEST_PATH) {
                                powershell """
                                    Write-Host "Configuring AWS CLI..."
                                    aws configure set aws_access_key_id "$Env:AWS_ACCESS_KEY_ID"
                                    aws configure set aws_secret_access_key "$Env:AWS_SECRET_ACCESS_KEY"
                                    aws configure set default.region us-east-1

                                    Write-Host "Downloading secrets.yaml from S3..."
                                    aws s3 cp s3://dify-k8s-secret/secrets.yaml ./secrets.yaml
                                """
                            }
                        } catch (Exception e) {
                            currentBuild.result = 'FAILURE'
                            env.FAILURE_MESSAGE = "Failed to download secrets.yaml from S3: ${e.message}"
                            error(env.FAILURE_MESSAGE)
                        }
                    }
                }
            }
        }
        stage('Build Images') {
            steps {
                script {

                    try {
                        // build API image
                        dir("${env.DIFY_REPOSITORY_PATH}\\api"){
                            powershell'''
                            Write-Host "Building API image..."
                            minikube -p minikube docker-env --shell=powershell | Invoke-Expression
                            docker build -t dify-api:local .
                            '''
                        }
                        // build web image
                        dir("${env.DIFY_REPOSITORY_PATH}\\web"){
                            powershell'''
                            Write-Host "Building Web image..."
                            minikube -p minikube docker-env --shell=powershell | Invoke-Expression
                            docker build -t dify-web:local .
                            '''
                        }
                        // build daemon plugin image
                        dir("${env.DIFY_REPOSITORY_PATH}\\dify-k8s\\dify-plugin-daemon"){
                            powershell'''
                            Write-Host "Building Daemon Plugin image..."
                            minikube -p minikube docker-env --shell=powershell | Invoke-Expression
                            docker build -f docker/local.dockerfile -t dify-plugin:local .
                            '''
                        }
                        

                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        env.FAILURE_MESSAGE = "Failed to clone or update the repository: ${e.message}"
                        error(env.FAILURE_MESSAGE)
                    }
                }
            }
        }
        stage('Apply YAML Manifest') {
            steps {
                script {

                    try {
                        dir(env.DIFY_YAML_MANIFEST_PATH){
                            powershell'''
                            Write-Host "Apply YAML manifest..."
                            kubectl create namespace dify --dry-run=client -o yaml | kubectl apply -f -
                            kubectl apply -f .
                            kubectl rollout restart deployment -n dify
                            kubectl rollout restart statefulset -n dify
                            kubectl delete pod --all -n dify --grace-period=0 --force
                            '''
                        }                    

                    } catch (Exception e) {
                        currentBuild.result = 'FAILURE'
                        env.FAILURE_MESSAGE = "Failed to clone or update the repository: ${e.message}"
                        error(env.FAILURE_MESSAGE)
                    }
                }
            }
        }
    }
    post {
        success {
            script {
                dir(env.DIFY_YAML_MANIFEST_PATH){
                    powershell 'Remove-Item -Path ./secrets.yaml -Force'
                    powershell 'docker system prune -f'
                }
            }
        }
    }
}

// executable command
def command(String script) {
    try {
        echo "Running PowerShell script: ${script}"
        def result = powershell(script: script, returnStdout: true, returnStatus: true)

        if (result != 0) {
            echo "Error: Command '${script}' failed with status: ${result}"
            error "Stopping pipeline due to error: ${result}"
        }

        def output = result.toString().trim()
        return output

    } catch (Exception e) {
        echo "Error: Command '${script}' failed with exception: ${e.getMessage()}"
        error "Stopping pipeline due to error: ${e.getMessage()}"
    }
}
