pipeline {
    agent {
        label 'catalogue'
    }

    environment {
        appVersion = ''
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
            def REGION = "us-east-1"
            def ACC_ID = "439481669447"
            def IMAGE = "${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com/catalogue:${appVersion}"

            withAWS(credentials: 'aws-creds', region: REGION) {
                sh """
                    aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ACC_ID}.dkr.ecr.${REGION}.amazonaws.com

                    docker build -t catalogue .

                    docker tag catalogue:latest ${IMAGE}

                    docker push ${IMAGE}
                """
            }
        }
    }
}
    }
}
