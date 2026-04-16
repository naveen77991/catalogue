pipeline {
    agent {
        label 'catalogue'
    }

    stages {
        stage('Read package.json') {
            steps {
                script {
                    def packageJson = readJSON file: 'package.json'
                    echo "Version: ${packageJson.version}"
                }
            }
        }
    }
}
