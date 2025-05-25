pipeline {
    agent any

    environment {
        KUBECONFIG = "/home/hongphuc/.kube/config"
        SERVICES = "spring-petclinic-vets-service,spring-petclinic-customers-service,spring-petclinic-visits-service,spring-petclinic-admin-server,spring-petclinic-api-gateway,spring-petclinic-config-server,spring-petclinic-genai-service,spring-petclinic-discovery-server"
    }

    stages {
        stage("Detect Changed Services") {
            steps {
                script {
                    sh 'git fetch origin main:refs/remotes/origin/main'
                    def changedFiles = sh(script: "git diff --name-only origin/main...", returnStdout: true).trim().split("\n")
                    def affectedServices = []

                    env.SERVICES.split(',').each { service ->
                        if (changedFiles.find { it.startsWith(service + "/") }) {
                            affectedServices.add(service)
                        }
                    }

                    if (affectedServices.isEmpty()) {
                        echo "No services changed. Skipping build."
                        currentBuild.result = "SUCCESS"
                        return
                    }

                    env.BUILD_SERVICES = affectedServices.join(',')
                }
            }
        }

        stage("Login to Docker Hub") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    script {
                        env.REPOSITORY_PREFIX = DOCKER_USER
                        sh '''
                            echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        '''
                    }
                }
            }
        }

        stage("Docker Build") {
            when {
                expression { return env.BUILD_SERVICES?.trim() }
            }
            steps {
                script {
                    def commitId = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    env.VERSION = commitId
                    
                    env.BUILD_SERVICES.split(',').each { service ->
                        dir(service) {
                            sh "../mvnw install -D skipTests -P buildDocker"
                        }
                    }
                }
            }
        }

        stage("Tag & Push Images") {
            steps {
                script {
                    env.BUILD_SERVICES.split(',').each { service ->
                        sh "docker tag springcommunity/${service} ${env.REPOSITORY_PREFIX}/${service}:${env.VERSION}"
                        sh "docker push ${env.REPOSITORY_PREFIX}/${service}:${env.VERSION}"
                    }
                }
            }
        }
    }
}
