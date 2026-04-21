pipeline {
    agent {
        label 'catalogue'
    }
    environment {
        REGION    = "us-west-1"
        ACC_ID    = "439481669447"
        PROJECT   = "roboshop"
        COMPONENT = "catalogue"
        IMAGE     = "${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${COMPONENT}"
    }
    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }
    parameters {
        string(name: 'appVersion', description: 'Image version of the application')
        choice(name: 'deploy_to', choices: ['dev', 'qa', 'prod'], description: 'Pick the Environment')
    }
    stages {
        stage('Read package.json') {
            steps {
                script {
                    def packageJson = readJSON file: 'package.json'
                    env.appVersion = packageJson.version
                    echo "Package version: ${env.appVersion}"
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
                    withAWS(credentials: 'aws-creds', region: "${REGION}") {
                        sh """
                            aws ecr get-login-password --region ${REGION} | \
                            docker login --username AWS --password-stdin \
                            ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com
                            docker build -t ${COMPONENT} .
                            docker tag ${COMPONENT}:latest ${IMAGE}:${env.appVersion}
                            docker push ${IMAGE}:${env.appVersion}
                        """
                    }
                }
            }
        }
        stage('Deploy to EKS') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: "${REGION}") {
                        sh """
                            aws eks update-kubeconfig \
                                --region ${REGION} \
                                --name ${PROJECT}-${params.deploy_to}
                            kubectl get nodes
                            kubectl apply -f 01-namespace.yaml
                            sed -i 's|IMAGE_PLACEHOLDER|${IMAGE}:${env.appVersion}|g' deployment.yaml
                            kubectl apply -f deployment.yaml -n ${PROJECT}
                            kubectl apply -f service.yaml -n ${PROJECT}
                        """
                    }
                }
            }
        }
        stage('Check Rollout Status') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: "${REGION}") {
                        def rolloutStatus = sh(
                            returnStdout: true,
                            script: """
                                kubectl rollout status deployment/${COMPONENT} \
                                    -n ${PROJECT} \
                                    --timeout=300s || echo FAILED
                            """
                        ).trim()

                        if (rolloutStatus.contains("successfully rolled out")) {
                            echo "✅ Deployment is successful"
                        } else {
                            echo "❌ Deployment failed - triggering rollback"
                            sh """
                                kubectl rollout undo deployment/${COMPONENT} \
                                    -n ${PROJECT}
                                sleep 20
                            """
                            def rollbackStatus = sh(
                                returnStdout: true,
                                script: """
                                    kubectl rollout status deployment/${COMPONENT} \
                                        -n ${PROJECT} \
                                        --timeout=120s || echo FAILED
                                """
                            ).trim()

                            if (rollbackStatus.contains("successfully rolled out")) {
                                error "Deployment Failed, Rollback Successful - previous version is running"
                            } else {
                                error "Deployment Failed, Rollback also Failed - application is not running"
                            }
                        }
                    }
                }
            }
        }
        stage('Functional Testing') {
            when {
                expression { params.deploy_to == "dev" }
            }
            steps {
                echo "Running functional test cases on dev"
            }
        }
        stage('Integration Testing') {
            when {
                expression { params.deploy_to == "qa" }
            }
            steps {
                echo "Running integration test cases on qa"
            }
        }
        stage('PROD Deploy Approval') {
            when {
                expression { params.deploy_to == "prod" }
            }
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: "${REGION}") {
                        input message: "Approve PROD deployment of ${COMPONENT}:${env.appVersion}?",
                              ok: "Deploy to PROD"
                        sh """
                            aws eks update-kubeconfig \
                                --region ${REGION} \
                                --name ${PROJECT}-prod
                            sed -i 's|IMAGE_PLACEHOLDER|${IMAGE}:${env.appVersion}|g' deployment.yaml
                            kubectl apply -f deployment.yaml -n ${PROJECT}
                            kubectl apply -f service.yaml -n ${PROJECT}
                        """
                    }
                }
            }
        }
    }
    post {
        always {
            echo "🔁 Pipeline completed"
            sh 'docker system prune -f || true'
            deleteDir()
        }
        success {
            echo "✅ Success - ${COMPONENT}:${env.appVersion} deployed to ${params.deploy_to}"
        }
        failure {
            echo "❌ Failed - Check logs above for rollback details"
        }
    }
}
