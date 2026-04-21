pipeline {
    agent {
        label 'catalogue'
    }
    environment {
        appVersion = ''
        REGION     = "us-west-1"
        ACC_ID     = "439481669447"
        REPO       = "catalogue"
        IMAGE      = "${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${REPO}"
        CLUSTER    = "catalogue-cluster1"
        KUBECONFIG = "${WORKSPACE}/kubeconfig"
    }
    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }
    stages {
        stage('Read package.json') {
            steps {
                script {
                    def packageJson = readJSON file: 'package.json'
                    appVersion = packageJson.version
                    echo "Package version: ${appVersion}"
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        stage('Docker Build & Push to ECR') {
            steps {
                script {
                    def FULL_IMAGE = "${IMAGE}:${appVersion}"
                    sh """
                        aws ecr get-login-password --region ${REGION} | \
                        docker login --username AWS --password-stdin \
                        ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com

                        docker build -t ${REPO} .
                        docker tag ${REPO}:latest ${FULL_IMAGE}
                        docker push ${FULL_IMAGE}
                    """
                }
            }
        }
        stage('Deploy to EKS') {
            steps {
                script {
                    def FULL_IMAGE = "${IMAGE}:${appVersion}"
                    sh """
                        aws eks update-kubeconfig \
                            --region ${REGION} \
                            --name ${CLUSTER} \
                            --kubeconfig ${KUBECONFIG}

                        export KUBECONFIG=${KUBECONFIG}

                        kubectl get nodes

                        sed -i 's|IMAGE_PLACEHOLDER|${FULL_IMAGE}|g' deployment.yaml

                        kubectl apply -f deployment.yaml
                        kubectl apply -f service.yaml

                        kubectl rollout status deployment/${REPO} \
                            --timeout=300s
                    """
                }
            }
        }
    }
    post {
        success {
            echo "✅ Build Success - Image pushed & Deployed to EKS"
        }
        failure {
            echo "❌ Build Failed - Rolling back..."
            sh """
                export KUBECONFIG=${KUBECONFIG}
                aws eks update-kubeconfig \
                    --region ${REGION} \
                    --name ${CLUSTER} \
                    --kubeconfig ${KUBECONFIG}

                kubectl rollout undo deployment/${REPO}
                echo "✅ Rollback complete!"
            """
        }
        always {
            echo "🔁 Pipeline completed"
            sh 'docker system prune -f || true'
        }
    }
}
