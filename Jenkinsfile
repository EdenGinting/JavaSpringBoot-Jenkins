def VERSION = ''
def IMAGE_TAG = ''

pipeline {
    agent {
        kubernetes {
          yaml """
            apiVersion: v1
            kind: Pod
            metadata:
              labels:
                jenkins: maven-agent
            spec:
              serviceAccountName: jenkins
              containers:
                - name: maven
                  image: maven:3.9.6-eclipse-temurin-17
                  command:
                    - cat
                  tty: true
                  volumeMounts:
                    - name: docker-sock
                      mountPath: /var/run/docker.sock
                - name: ubuntu
                  image: ubuntu:22.04
                  command:
                    - cat
                  tty: true
                - name: docker
                  image: docker:24.0.7-cli
                  command:
                    - cat
                  tty: true
                  volumeMounts:
                    - name: docker-sock
                      mountPath: /var/run/docker.sock
              volumes:
                - name: docker-sock
                  hostPath:
                    path: /var/run/docker.sock
                    type: Socket
            """
        }
    }

    stages {
        stage('Get Version') {
            steps {
                container('maven') {
                    script {
                        echo "--- Set Image Tag ---"

                        VERSION = sh script: 'mvn help:evaluate -Dexpression=project.version -q -DforceStdout', returnStdout: true

                        IMAGE_TAG = "edenginting/springboot-jenkins:${VERSION}"

                        echo "--- Image Tag Has Been Set: ${IMAGE_TAG} ---"
                    }
                }
            }
        }

        stage('Build, Test, & Package') {
            steps {
                container('maven') {
                    sh 'mvn clean install'
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                container('docker') {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            echo "Docker hub user: ${DOCKER_USER}"

                            docker build -t "${IMAGE_TAG}" .

                            echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin

                            docker push "${IMAGE_TAG}"
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('ubuntu') {
                    sh """
                        apt-get update && apt-get install -y curl ca-certificates bash

                        curl -LO https://dl.k8s.io/release/v1.32.0/bin/linux/amd64/kubectl

                        install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

                        kubectl version

                        echo "--- Applying Resources ---"

                        sed "s|__IMAGE__|${IMAGE_TAG}|g" k8s/Deployment.yaml | kubectl apply -f -

                        kubectl apply -f k8s/Service.yaml

                        echo "--- Successfully Applying Resources ---"
                    """
                }
            }
        }
    }

    post {
        success {
            echo "✅ Build & Deploy Successful!"
        }

        failure {
            echo "❌ Build or Deploy Failed!"
        }
    }
}
