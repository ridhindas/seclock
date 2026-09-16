pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = sh(script: 'aws sts get-caller-identity --query Account --output text', returnStdout: true).trim()
        ECR_REPO_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.ap-south-1.amazonaws.com/seclock"
        IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
        EKS_CLUSTER_NAME = 'seclock-cluster'
        K8S_NAMESPACE = 'seclock-prod'
    }

    stages {
        stage('1. Checkout & Secrets Scan') {
            steps {
                checkout scm
                sh 'docker run --rm -v ${WORKSPACE}:/path zricethezav/gitleaks:latest detect --source /path --redact -v || true'
            }
        }

        stage('2. SCA & SAST') {
            steps {
                sh '''
                    python3 -m venv venv
                    source venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    pip install pip-audit bandit
                    
                    pip-audit --format json > pip-audit-report.json || true
                    bandit -r . -f json -o bandit-report.json || true
                '''
                archiveArtifacts artifacts: 'pip-audit-report.json, bandit-report.json', allowEmptyArchive: true
            }
        }

        stage('3. Unit & E2E Testing') {
            steps {
                sh '''
                    source venv/bin/activate
                    pip install pytest httpx
                    pytest test_e2e.py -v --junitxml=pytest-report.xml || true
                '''
                junit 'pytest-report.xml'
            }
        }

        stage('4. Build Docker Image') {
            steps {
                script {
                    docker.build("${ECR_REPO_URI}:${IMAGE_TAG}", ".")
                }
            }
        }

        stage('5. Container Security Scan (Trivy)') {
            steps {
                sh """
                    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                    aquasec/trivy image --exit-code 1 --severity CRITICAL,HIGH ${ECR_REPO_URI}:${IMAGE_TAG}
                """
            }
        }

        stage('6. AWS ECR Login & Push') {
            steps {
                script {
                    sh '''
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    '''
                    docker.image("${ECR_REPO_URI}:${IMAGE_TAG}").push()
                    docker.image("${ECR_REPO_URI}:${IMAGE_TAG}").push("latest")
                }
            }
        }

        stage('7. Deploy to Amazon EKS') {
            steps {
                script {
                    sh """
                        aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER_NAME}
                        sed -i 's|<AWS_ACCOUNT_ID>.dkr.ecr.<AWS_REGION>.amazonaws.com/seclock:latest|${ECR_REPO_URI}:${IMAGE_TAG}|g' k8s/deployment.yaml
                        kubectl create namespace ${K8S_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                        kubectl apply -f k8s/serviceaccount.yaml
                        kubectl apply -f k8s/deployment.yaml
                        kubectl apply -f k8s/service.yaml
                        kubectl rollout status deployment/seclock-deployment -n ${K8S_NAMESPACE} --timeout=120s
                    """
                }
            }
        }
    }

           post {
           always {
               echo 'Cleaning up workspace...'
               // deleteDir() is more reliable than cleanWs() in Declarative Pipeline
               deleteDir() 
           }
           failure {
               echo '🚨 Pipeline failed! Scroll up in the Console Output to find the actual error (Docker, Trivy, Bandit, or EKS).'
           }
       }
