def runCommand(String unixCommand, String windowsCommand) {
    if (isUnix()) {
        sh unixCommand
    } else {
        bat windowsCommand
    }
}

pipeline {
    agent any

    tools {
        jdk 'jdk21'
        nodejs 'NodeJS-20'
    }

    parameters {
        string(name: 'BRANCH_NAME', defaultValue: 'jenkins_new', description: 'Git branch to build')
        string(name: 'DOCKERHUB_USERNAME', defaultValue: 'hoang2207', description: 'Docker Hub account that owns the images')
        booleanParam(name: 'ENABLE_SONAR', defaultValue: true, description: 'Run SonarQube analysis and wait for its Quality Gate')
    }

    environment {
        GIT_REPO_URL = 'https://github.com/Darktanyan123/jenkins_setup.git'
        DOCKERHUB_CREDENTIALS = 'dockerhub-credentials'
    }

    stages {
        stage('1. Check Code') {
            steps {
                script {
                    if (!(params.BRANCH_NAME ==~ /[A-Za-z0-9][A-Za-z0-9._/-]*/) || params.BRANCH_NAME.contains('..')) {
                        error('BRANCH_NAME contains unsupported characters.')
                    }
                    if (!(params.DOCKERHUB_USERNAME ==~ /[a-z0-9][a-z0-9_-]{0,29}/)) {
                        error('DOCKERHUB_USERNAME must be a valid lowercase Docker Hub username.')
                    }
                }
                echo "Checking out branch ${params.BRANCH_NAME}"
                git branch: params.BRANCH_NAME, url: env.GIT_REPO_URL
                script {
                    env.GIT_COMMIT_SHORT = isUnix()
                        ? sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        : bat(script: '@echo off\r\ngit rev-parse --short HEAD', returnStdout: true).trim()
                    env.VERSION = "v0.${env.BUILD_NUMBER}"
                    env.VERSION_COMMIT = "${env.VERSION}-${env.GIT_COMMIT_SHORT}"
                    env.FRONTEND_IMAGE = "${params.DOCKERHUB_USERNAME}/resume-frontend"
                    env.BACKEND_IMAGE = "${params.DOCKERHUB_USERNAME}/resume-backend"
                    env.MONGO_IMAGE = "${params.DOCKERHUB_USERNAME}/resume-mongodb"
                    echo "Build version: ${env.VERSION_COMMIT}"
                }
            }
        }

        stage('2. Test Code') {
            parallel {
                stage('Test Frontend') {
                    steps {
                        dir('frontend') {
                            script {
                                if (isUnix()) {
                                    sh 'npm ci'
                                    sh 'npm test'
                                } else {
                                    bat 'call npm ci'
                                    bat 'call npm test'
                                }
                            }
                        }
                    }
                }
                stage('Test Backend') {
                    steps {
                        dir('backend') {
                            script {
                                if (isUnix()) {
                                    sh 'chmod +x ./mvnw'
                                    sh './mvnw -B test'
                                } else {
                                    bat 'call mvnw.cmd -B test'
                                }
                            }
                        }
                    }
                }
            }
        }

        stage('3. SonarQube Analysis') {
            when {
                expression { params.ENABLE_SONAR }
            }
            steps {
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('SonarQube') {
                        if (isUnix()) {
                            sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectVersion=${env.VERSION_COMMIT}"
                        } else {
                            bat "\"${scannerHome}\\bin\\sonar-scanner.bat\" -Dsonar.projectVersion=${env.VERSION_COMMIT}"
                        }
                    }
                }
            }
        }

        stage('4. Quality Gate') {
            when {
                expression { params.ENABLE_SONAR }
            }
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('5. Build & Push Docker Images') {
            steps {
                script {
                    withDockerRegistry(credentialsId: env.DOCKERHUB_CREDENTIALS) {
                        dir('frontend') {
                            runCommand(
                                "docker build -t ${env.FRONTEND_IMAGE}:${env.VERSION} -t ${env.FRONTEND_IMAGE}:${env.VERSION_COMMIT} .",
                                "docker build -t ${env.FRONTEND_IMAGE}:${env.VERSION} -t ${env.FRONTEND_IMAGE}:${env.VERSION_COMMIT} ."
                            )
                            runCommand(
                                "docker push ${env.FRONTEND_IMAGE}:${env.VERSION}\ndocker push ${env.FRONTEND_IMAGE}:${env.VERSION_COMMIT}",
                                "docker push ${env.FRONTEND_IMAGE}:${env.VERSION}\r\ndocker push ${env.FRONTEND_IMAGE}:${env.VERSION_COMMIT}"
                            )
                        }

                        dir('backend') {
                            runCommand(
                                "docker build -t ${env.BACKEND_IMAGE}:${env.VERSION} -t ${env.BACKEND_IMAGE}:${env.VERSION_COMMIT} .",
                                "docker build -t ${env.BACKEND_IMAGE}:${env.VERSION} -t ${env.BACKEND_IMAGE}:${env.VERSION_COMMIT} ."
                            )
                            runCommand(
                                "docker push ${env.BACKEND_IMAGE}:${env.VERSION}\ndocker push ${env.BACKEND_IMAGE}:${env.VERSION_COMMIT}",
                                "docker push ${env.BACKEND_IMAGE}:${env.VERSION}\r\ndocker push ${env.BACKEND_IMAGE}:${env.VERSION_COMMIT}"
                            )
                        }

                        dir('mongo-init') {
                            runCommand(
                                "docker build -t ${env.MONGO_IMAGE}:${env.VERSION} -t ${env.MONGO_IMAGE}:${env.VERSION_COMMIT} .",
                                "docker build -t ${env.MONGO_IMAGE}:${env.VERSION} -t ${env.MONGO_IMAGE}:${env.VERSION_COMMIT} ."
                            )
                            runCommand(
                                "docker push ${env.MONGO_IMAGE}:${env.VERSION}\ndocker push ${env.MONGO_IMAGE}:${env.VERSION_COMMIT}",
                                "docker push ${env.MONGO_IMAGE}:${env.VERSION}\r\ndocker push ${env.MONGO_IMAGE}:${env.VERSION_COMMIT}"
                            )
                        }
                    }
                }
            }
        }

        stage('6. Deploy Containers') {
            steps {
                script {
                    runCommand(
                        """echo "FRONTEND_IMAGE=${env.FRONTEND_IMAGE}" > .env
echo "BACKEND_IMAGE=${env.BACKEND_IMAGE}" >> .env
echo "MONGO_IMAGE=${env.MONGO_IMAGE}" >> .env
echo "VERSION=${env.VERSION_COMMIT}" >> .env""",
                        """@echo FRONTEND_IMAGE=${env.FRONTEND_IMAGE}> .env
@echo BACKEND_IMAGE=${env.BACKEND_IMAGE}>> .env
@echo MONGO_IMAGE=${env.MONGO_IMAGE}>> .env
@echo VERSION=${env.VERSION_COMMIT}>> .env"""
                    )
                    runCommand(
                        'docker compose down --remove-orphans',
                        'docker compose down --remove-orphans'
                    )
                    runCommand(
                        'docker compose up -d --force-recreate',
                        'docker compose up -d --force-recreate'
                    )
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully. Deployed version: ${env.VERSION_COMMIT}"
        }
        failure {
            echo 'Pipeline failed. Check the stage logs above for the failing command.'
        }
    }
}
