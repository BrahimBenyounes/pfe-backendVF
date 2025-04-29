pipeline {
    agent any

    tools {
        maven 'M3'
        jdk 'jdk17'
    }

    environment {
        DOCKER_IMAGE_VERSION = '1.0.0'
        DOCKER_COMPOSE_FILE = 'docker-compose.yml'
        MAVEN_HOME = tool name: 'M3', type: 'maven'
        JAVA_HOME = tool name: 'jdk17', type: 'jdk'
        PATH = "${MAVEN_HOME}/bin:${JAVA_HOME}/bin:${env.PATH}"
        SONAR_HOST_URL = 'http://localhost:9000'
        SONAR_TOKEN = 'squ_e8b5f30571241cb357ad5734ed99e18e1807aa81'
    }

    stages {
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build All Maven Projects') {
            steps {
                script {
                    def services = [
                        "discovery-service", "gateway-service", "product-service",
                        "formation-service", "order-service", "notification-service",
                        "login-service", "contact-service"
                    ]
                    services.each { service ->
                        dir(service) {
                            bat 'mvn clean install -DskipTests'
                        }
                    }
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def services = [
                        "discovery-service", "gateway-service", "product-service",
                        "formation-service", "order-service", "notification-service",
                        "login-service", "contact-service"
                    ]
                    services.each { service ->
                        echo "Analyzing project: ${service}"
                        def timestamp = new Date().format("yyyyMMdd-HHmmss", TimeZone.getTimeZone('UTC'))
                        def projectKey = "${service}-${timestamp}"
                        dir(service) {
                            bat 'mvn clean org.jacoco:jacoco-maven-plugin:prepare-agent install -Dmaven.test.failure.ignore=true'
                            bat "mvn sonar:sonar -Dsonar.token=${env.SONAR_TOKEN} -Dsonar.projectKey=${projectKey} -Dsonar.host.url=${SONAR_HOST_URL}"
                        }
                    }
                }
            }
        }

        stage('Upload Artifacts to Nexus') {
            steps {
                script {
                    def version = "1.0-SNAPSHOT"
                    def projects = [
                        "discovery-service", "gateway-service", "product-service",
                        "formation-service", "order-service", "notification-service",
                        "login-service", "contact-service"
                    ]
                    projects.each { projectName ->
                        dir(projectName) {
                            nexusArtifactUploader(
                                nexusVersion: 'nexus3',
                                protocol: 'http',
                                nexusUrl: 'localhost:8081',
                                groupId: 'org.pfe',
                                version: version,
                                repository: 'maven-snapshots',
                                credentialsId: 'nexus-credentials',
                                artifacts: [
                                    [
                                        artifactId: projectName,
                                        classifier: '',
                                        file: "target/${projectName}-${version}.jar",
                                        type: 'jar'
                                    ]
                                ]
                            )
                        }
                    }
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    def services = [
                        "discovery-service", "gateway-service", "product-service",
                        "formation-service", "order-service", "notification-service",
                        "login-service", "contact-service"
                    ]
                    services.each { service ->
                        dir(service) {
                            bat "docker build -t ${service}:${DOCKER_IMAGE_VERSION} -t brahim2025/${service}:latest ."
                        }
                    }
                }
            }
        }

        stage('Deploy Microservices') {
            steps {
                script {
                    bat "docker compose -f ${DOCKER_COMPOSE_FILE} down"
                    bat "docker compose -f ${DOCKER_COMPOSE_FILE} up -d"
                }
            }
        }

        stage('Push Docker Images to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'DOCKER_HUB_PASSWORD', usernameVariable: 'DOCKER_HUB_USERNAME')]) {
                        bat "docker login -u %DOCKER_HUB_USERNAME% -p %DOCKER_HUB_PASSWORD%"

                        def services = [
                            "discovery-service", "gateway-service", "product-service",
                            "formation-service", "order-service", "notification-service",
                            "login-service", "contact-service"
                        ]

                        services.each { service ->
                            def remoteTag = "brahim2025/${service}:latest"
                            catchError(buildResult: 'FAILURE', stageResult: 'FAILURE') {
                                bat "docker push ${remoteTag}"
                            }
                        }
                    }
                }
            }
        }

        /*
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    try {
                        withKubeConfig([credentialsId: 'mykubeconfig', serverUrl: 'https://127.0.0.1:61372']) {
                            bat 'kubectl apply -f Kubernetes'
                        }
                    } catch (Exception e) {
                        error "Kubernetes deployment failed: ${e.getMessage()}"
                    }
                }
            }
        }
        */
    }
}
