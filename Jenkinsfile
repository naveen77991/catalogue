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
                    withAWS(credentials: 'awscredntials', region: "${REGION}") {
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
                    withAWS(credentials: 'awscredntials', region: "${REGION}") {
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
                    withAWS(credentials: 'awscredntials', region: "${REGION}") {
                        sh """
                            aws ecr get-login-password --region ${REGION} | \
                            docker login --username AWS --password-stdin \
