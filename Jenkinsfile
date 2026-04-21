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
        stage('Configure kubectl') {
            steps {
                script {
                    withAWS(credentials: 'awscredentials', region: "${REGION}") {
                        sh """
                            aws eks update-kubeconfig \
                                --region ${REGION} \
                                --name ${PROJECT}-${params.deploy_to}
                        """
                    }
                }
            }
        }
        stage('Check Status') {
            steps {
                script {
                    withAWS(credentials: 'awscredentials', region: "${REGION}") {
                        def deploymentStatus = sh(
                            returnStdout: true,
                            script: "kubectl rollout status deployment/${COMPONENT} --timeout=30s -n ${PROJECT} || echo FAILED"
                        ).trim()
                        if (deploymentStatus.contains("successfully rolled out")) {
                            echo "Deployment is success"
                        } else {
                            sh """
                                kubectl rollout undo deployment/${COMPONENT} -n ${PROJECT}
                                sleep 20
                            """
                            def rollbackStatus = sh(
                                returnStdout: true,
                                script: "kubectl rollout status deployment/${COMPONENT} --timeout=30s -n ${PROJECT} || echo FAILED"
                            ).trim()
                            if (rollbackStatus.contains("successfully rolled out")) {
                                error "Deployment is Failure, Rollback Success"
                            } else {
                                error "Deployment is Failure, Rollback Failure. Application is not running"
                            }
                        }
                    }
                }
            }
        }
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
                    withAWS(credentials: 'awscredentials', region: "${REGION}") {
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
        stage('Deploy') {
            steps {
                script {
                    withAWS(credentials: 'awscredentials', region: "${REGION}") {
                        sh """
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
        stage('Functional Testing') {
            when {
                expression { params.deploy_to == "dev" }
            }
            steps {
                script {
                    echo "Run functional test cases"
                }
            }
        }
        stage('Integration Testing') {
            when {
                expression { params.deploy_to == "qa" }
            }
            steps {
                script {
                    echo "Run Integration test cases"
                }
            }
        }
        stage('PROD Deploy') {
            when {
                expression { params.deploy_to == "prod" }
            }
            steps {
                script {
                    withAWS(credentials: 'awscredentials', region: "${REGION}") {
                        sh """
                            echo "get cr number"
                            echo "check within the deployment window"
                            echo "is CR approved"
                            echo "trigger PROD deploy"
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
            echo "❌ Failed - Check logs above for details"
        }
    }
}
