pipeline {
    agent {
        label 'catalogue'
    }
    environment {
        appVersion = ''
        REGION = "us-west-1"
        ACC_ID = "439481669447"
        COMPONENT = "catalogue"
        CLUSTER = "catalogue-cluster1"
    }
    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }
    parameters {
        booleanParam(name: 'deploy', defaultValue: false, description: 'Toggle to trigger deploy')
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
                sh "npm install"
            }
        }
        stage('Docker Build & Push to ECR') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: 'us-west-1') {
                        sh """
                            aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com
                            docker build -t ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${COMPONENT}:${appVersion} .
                            docker push ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/${COMPONENT}:${appVersion}
                        """
                    }
                }
            }
        }
        stage('Deploy to EKS') {
            when {
                expression { params.deploy }
            }
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: 'us-west-1') {
                        sh """
                            aws eks update-kubeconfig --region ${REGION} --name ${CLUSTER}
                            kubectl get nodes
                            sed -i "s|IMAGE_VERSION|${appVersion}|g" deployment.yaml
                            kubectl apply -f deployment.yaml
                            kubectl rollout status deployment/${COMPONENT} --timeout=120s
                        """
                    }
                }
            }
        }
    }
    post {
        always {
            echo 'Pipeline finished!'
            deleteDir()
        }
        success {
            echo 'Success!'
        }
        failure {
            echo 'Failed!'
        }
    }
}
