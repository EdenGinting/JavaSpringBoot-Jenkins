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
            - name: kubectl
              image: bitnami/kubectl:latest
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

    environment {
        VERSION = ''
        IMAGE_TAG = ''
    }

    stages {
        stage('Get Version') {
            steps {
                container('maven') {
                    script {
                        def version = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
                        env.VERSION = version
                        env.IMAGE_TAG = "edenginting/springboot-jenkins:${version}"
                        echo "Image tag will be: ${env.IMAGE_TAG}"
                    }
                }
            }
        }

        stage('Build') {
            steps {
                container('maven') {
                    sh 'mvn clean compile'
                }
            }
        }

        stage('Test') {
            steps {
                container('maven') {
                    sh 'mvn test'
                }
            }
        }

        stage('Package') {
            steps {
                container('maven') {
                    sh 'mvn package'
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
                            echo 'Docker hub user: \$DOCKER_USER'

                            docker build -t ${env.IMAGE_TAG} .

                            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin

                            docker push ${env.IMAGE_TAG}
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    sh """
                        sed 's|__IMAGE__|${env.IMAGE_TAG}|g' k8s/Deployment.yaml | kubectl apply -f -

                        kubectl apply -f k8s/Service.yaml
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
