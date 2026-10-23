pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/amannprakash/StrideX-E-commerce.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm ci'
            }
        }

        stage('Lint') {
            steps {
                sh 'npm run lint'
            }
        }

        stage('Security Audit') {
            steps {
                sh 'npm audit --audit-level=critical'
            }
        }

        stage('Build') {
            steps {
                sh 'npm run build'
            }
        }
    }
}
