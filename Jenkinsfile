pipeline {
    agent any

    tools {
        maven 'MAVEN_HOME'
    }

    environment {
        // Nexus configuration
        NEXUS_VERSION       = "${env.NEXUS_VERSION ?: 'nexus3'}"
        NEXUS_PROTOCOL      = "${env.NEXUS_PROTOCOL ?: 'http'}"
        NEXUS_URL           = "${env.NEXUS_URL ?: '13.217.7.157:8081'}"
        NEXUS_REPOSITORY    = "${env.NEXUS_REPOSITORY ?: 'devops'}"
        NEXUS_CREDENTIAL_ID = "${env.NEXUS_CREDENTIAL_ID ?: 'Nexus_server'}"

        // SonarQube configuration
        SONARQUBE_SERVER    = "${env.SONARQUBE_SERVER ?: 'MySonar'}"
        SONARQUBE_URL       = "http://35.175.252.12/"

        // Slack notification
        SLACK_CHANNEL       = "${env.SLACK_CHANNEL ?: '#new-channel'}"

        // Git repository
        REPO_URL            = "https://github.com/mubeen-hub78/spring3-mvc-maven-xml-hello-world-1.git"
        GIT_BRANCH          = "main"

        // Docker configuration for your Docker Hub repository
        DOCKER_IMAGE_NAME   = "mubeendochub/java-app"
        DOCKER_REGISTRY     = "docker.io" // Docker Hub registry domain
        DOCKER_CREDENTIALS_ID = "Docker-cred"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: "${env.GIT_BRANCH}", url: "${env.REPO_URL}"
            }
        }

        stage('Build with Maven') {
            steps {
                sh 'mvn -Dmaven.test.failure.ignore=true clean install'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${env.SONARQUBE_SERVER}") {
                    sh 'mvn sonar:sonar -Dsonar.host.url=$SONARQUBE_URL'
                }
            }
        }

        stage('Publish Artifact to Nexus') {
            steps {
                script {
                    def pom = readMavenPom file: 'pom.xml'
                    def artifacts = findFiles(glob: "target/*.${pom.packaging}")

                    if (artifacts.size() == 0) {
                        error "No artifact found for packaging type: ${pom.packaging}"
                    }

                    def artifactPath = artifacts[0].path

                    nexusArtifactUploader(
                        nexusVersion: "${env.NEXUS_VERSION}",
                        protocol: "${env.NEXUS_PROTOCOL}",
                        nexusUrl: "${env.NEXUS_URL}",
                        groupId: pom.groupId,
                        version: "${env.BUILD_NUMBER}-SNAPSHOT",
                        repository: "${env.NEXUS_REPOSITORY}",
                        credentialsId: "${env.NEXUS_CREDENTIAL_ID}",
                        artifacts: [
                            [artifactId: pom.artifactId, classifier: '', file: artifactPath, type: pom.packaging],
                            [artifactId: pom.artifactId, classifier: '', file: 'pom.xml', type: 'pom']
                        ]
                    )
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageTag = "${env.DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh "docker build -t ${imageTag} ."
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    def fullImageName = "${env.DOCKER_REGISTRY}/${env.DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    sh "docker tag ${env.DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER} ${fullImageName}"

                    withCredentials([usernamePassword(credentialsId: env.DOCKER_CREDENTIALS_ID, usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin ${env.DOCKER_REGISTRY}"
                    }

                    sh "docker push ${fullImageName}"
                }
            }
        }

        stage('ArgoCD Deploy') {
            steps {
                withCredentials([string(credentialsId: 'argocd-password', variable: 'ARGOCD_PASS')]) {
                    sh '''
                        argocd login 54.172.117.25:31630 --username admin --password $ARGOCD_PASS --insecure
                        argocd app sync java-app
                        argocd app wait java-app --timeout 600
                    '''
                }
            }
        }
    }

    post {
        success {
            slackSend(channel: "${env.SLACK_CHANNEL}", color: 'good',
                message: "✅ SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (<${env.BUILD_URL}|Open>)")
        }
        failure {
            slackSend(channel: "${env.SLACK_CHANNEL}", color: 'danger',
                message: "❌ FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (<${env.BUILD_URL}|Open>)")
        }
        unstable {
            slackSend(channel: "${env.SLACK_CHANNEL}", color: 'warning',
                message: "⚠️ UNSTABLE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'")
        }
    }
}
