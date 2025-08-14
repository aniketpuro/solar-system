pipeline {
    // This agent is used for stages that run outside Docker, like OWASP and Trivy.
    // It must have Docker, Trivy, and the Node.js tool installed.
    agent {
        label 'ubuntu-docker'
    }

    tools {
        // Defines the NodeJS tool configured in Jenkins Global Tool Configuration.
        nodejs 'nodejs-22-6-0'
    }

    environment {
        // IMPORTANT: The credential 'mongo-db-uri' should be a "Secret Text" credential in Jenkins
        // containing the full MongoDB connection string.
        // e.g., mongodb+srv://<user>:<password>@cluster-name.mongodb.net/my-database
        // The application's code MUST be written to read this specific environment variable (MONGO_URI).
        MONGO_URI = credentials('mongo-db-uri')
    }

    stages {
        stage('Installing Dependencies') {
            // This stage runs inside a clean Node.js container to install dependencies.
            agent {
                docker {
                    image 'node:24'
                    args '-u root:root'
                }
            }
            steps {
                sh 'npm install --no-audit'
            }
        }

        stage('Dependency Scanning') {
            parallel {
                stage('NPM Dependency Audit') {
                    agent {
                        docker {
                            image 'node:24'
                            args '-u root:root'
                        }
                    }
                    steps {
                        // Catches errors from npm audit. If critical vulnerabilities are found,
                        // it marks the stage as UNSTABLE but allows the pipeline to continue.
                        catchError(buildResult: 'SUCCESS', stageResult: 'UNSTABLE') {
                            sh 'npm audit --audit-level=critical'
                        }
                    }
                }
                stage('OWASP Dependency Check') {
                    // FIX: This stage now runs on the main Jenkins agent ('ubuntu-docker'),
                    // NOT inside a Docker container. This is because the OWASP tool ('OWASP-DepCheck-12')
                    // is installed on the agent itself, not in the 'node:24' image.
                    agent any
                    steps {
                        dependencyCheck additionalArguments: '''
                            --scan ./
                            --format ALL
                            --prettyPrint
                        ''', odcInstallation: 'OWASP-DepCheck-12'

                        // This will now correctly find the report and fail the build if thresholds are met.
                        dependencyCheckPublisher pattern: 'dependency-check-report.xml'
                    }
                }
            }
        }

        stage('Unit Testing') {
            // FIX: This stage runs in Docker and automatically inherits the MONGO_URI
            // environment variable from the top-level definition, fixing the "undefined" URI error.
            agent {
                docker {
                    image 'node:24'
                    args '-u root:root'
                }
            }
            options {
                retry(2)
            }
            steps {
                sh 'npm test'
                junit 'test-results.xml'
            }
        }

        stage('Code Coverage') {
            agent {
                docker {
                    image 'node:24'
                    args '-u root:root'
                }
            }
            steps {
                sh 'npm run coverage'
                publishHTML(target: [
                    reportName: 'Code Coverage Report',
                    reportDir: 'coverage/lcov-report',
                    reportFiles: 'index.html',
                    allowMissing: true
                ])
            }
        }

        stage('Build Docker Image') {
            // This runs on the main agent, which has access to the Docker daemon.
            agent any
            steps {
                sh 'docker build -t aniketpuro/solar-system:$GIT_COMMIT .'
            }
        }

        stage('Trivy Vulnerability Scanner') {
            // This also runs on the main agent where Trivy is installed.
            agent any
            steps {
                // Scan the image and fail the build if critical vulnerabilities are found.
                catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                    sh "trivy image --severity CRITICAL --exit-code 1 aniketpuro/solar-system:$GIT_COMMIT"
                }
            }
        }

        stage('Push Docker Image') {
            agent any
            steps {
                // Pushes the image to a Docker registry using stored credentials.
                withDockerRegistry(credentialsId: 'docker-hub-credentials', url: "") {
                    sh 'docker push aniketpuro/solar-system:$GIT_COMMIT'
                }
            }
        }
    }
}
