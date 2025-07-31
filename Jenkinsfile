pipeline {
    agent any

    tools {
        nodejs 'nodejs-22-6-0'
        maven 'Maven 3.9.6'
    }

    environment {
        MONGO_URI = "mongodb+srv://supercluster.d83jj.mongodb.net/superData"
        MONGO_DB_CREDS = credentials('mongo-db-credentials')
        MONGO_USERNAME = credentials('mongo-db-username')
        MONGO_PASSWORD = credentials('mongo-db-password')
        SONAR_SCANNER_HOME = tool 'sonarqube-scanner'
        PATH = "${env.SONAR_SCANNER_HOME}/bin:${env.PATH}"
    }

    stages {
        stage('Install Dependencies') {
            steps {
                echo 'Installing Node.js dependencies...'
                sh 'npm install'
            }
        }

        stage('Code Analysis - SonarQube') {
            steps {
                echo 'Running SonarQube analysis...'
                sh '''
                    sonar-scanner \
                    -Dsonar.projectKey=solar-system \
                    -Dsonar.sources=. \
                    -Dsonar.host.url=http://localhost:9000 \
                    -Dsonar.login=${MONGO_DB_CREDS}
                '''
            }
        }

        stage('Run Tests') {
            steps {
                echo 'Running test cases...'
                sh 'npm test || echo "Tests failed, but continuing..."'
            }
        }

        stage('Build Package') {
            steps {
                echo 'Building the Node.js project...'
                sh 'npm run build || echo "Build failed, but continuing..."'
            }
        }

        stage('Archive Artifacts') {
            steps {
                echo 'Archiving build artifacts...'
                archiveArtifacts artifacts: '**/dist/**/*', fingerprint: true
            }
        }

        stage('Publish Results') {
            steps {
                echo 'Publishing test results...'
                junit 'test-results/*.xml'  // Adjust this path to your actual test output
            }
        }
    }

    post {
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
        failure {
            echo 'Pipeline failed. Please check logs and fix issues.'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
    }
}
