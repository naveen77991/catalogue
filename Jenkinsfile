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
    }
}
