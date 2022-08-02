pipeline {
    agent any
    environment {
        PROJECT_NAME = "conversao-temperatura"
        REPOSITORY = "gabriel2012rissi/${env.PROJECT_NAME}"
    }
    stages {
        stage('Checkout') {
            steps {
                script {
                    // Obtendo o hash do commit atual
                    COMMIT = "${GIT_COMMIT.substring(0,8)}"
                    // Criando a tag da imagem
                    IMAGE_TAG = "${env.BRANCH_NAME}"
                }
            }
        }
        stage('Static Analysis') {
            environment {
                sonarHome = tool 'SONAR_SCANNER'
            }
            steps {
                // SonarQube Scanner
                sh """
                   ${sonarHome}/bin/sonar-scanner -e \
                   -Dsonar.projectKey=${env.PROJECT_NAME} \
                   -Dsonar.branch.name=${env.BRANCH_NAME} \
                   -Dsonar.sources=. \
                   -Dsonar.host.url=http://sonarqube:9000 \
                   -Dsonar.login=275ebb556ebdcf5f6ce6ad67e2272a8e6ada4faf
                   """
            }
        }
        stage('Build Application') {
            steps {
                script {
                    appImage = docker.build("${env.REPOSITORY}:${COMMIT}", "./src")
                }
            }
        }
        stage('Run Application') {
            steps {
                script {
                    // Criando a rede 'jenkins_test'
                    sh "docker network create jenkins_test-${env.BUILD_NUMBER} || true"

                    // Criando o container 'api'
                    sh """
                       docker run \
                       --detach \
                       --name app-${env.BUILD_NUMBER} \
                       --publish 8080:8080 \
                       --network jenkins_test-${env.BUILD_NUMBER} \
                       --network-alias app \
                       ${appImage.id}
                       """
                }
            }
            post {
                always {
                    echo 'Cleanning up...'

                    // Removendo o container 'api'
                    sh "docker rm -f app-${env.BUILD_NUMBER} || true"

                    sleep 10

                    // Removendo a rede 'jenkins_test'
                    sh "docker network rm jenkins_test-${env.BUILD_NUMBER} || true"
                }
            }
        }
        stage('Push to Docker Hub') {
            steps {
                script {
                    // Enviar as imagens para o repositório do Docker Hub
                    docker.withRegistry("https://registry.hub.docker.com", "dockerhub") {
                        if ("${env.BRANCH_NAME}" == "master") {
                            appImage.push("latest")
                            appImage.push("${IMAGE_TAG}")
                        } else {
                            appImage.push("${IMAGE_TAG}")
                        }
                    }
                }
            }
        }
        stage('Trigger Manifest Update') {
            steps {
                script {
                    echo 'Triggering Manifest Update...'
                    build job: 'conversao-temperatura-manifest-update', parameters: [
                        string(name: "DOCKER_IMAGE", value: "${env.REPOSITORY}:${IMAGE_TAG}")
                    ]
                }
            }
        }
    }
}