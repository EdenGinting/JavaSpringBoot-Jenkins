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
                        echo "--- Debugging Maven Version Extraction ---"
                        sh 'pwd' // Show current working directory
                        sh 'ls -l' // List contents to confirm pom.xml presence

                        // Attempt to read the version and capture status
                        def result = sh(
                            script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout",
                            returnStdout: true,
                            returnStatus: true // Crucial for error checking
                        )

                        if (result.status != 0) {
                            error "Maven version evaluation failed. Exit code: ${result.status}. Check previous logs for Maven errors."
                        }

                        def version = result.stdout.trim()

                        if (version.isEmpty()) {
                            error "Maven project version is empty after evaluation. Raw output: '${result.stdout}'"
                        } else {
                            env.VERSION = version
                            env.IMAGE_TAG = "edenginting/springboot-jenkins:${version}"
                            echo "Successfully extracted Maven Project Version: ${env.VERSION}"
                            echo "Calculated Docker Image Tag: ${env.IMAGE_TAG}"
                        }
                        echo "--- Debugging Complete ---"
                    }
                }
            }
        }

        stage('Build, Test, & Package') {
            steps {
                container('maven') {
                    sh 'mvn clean compile'
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

                            docker build -t ${env.IMAGE_TAG} .

                            echo "${DOCKER_USER}" | docker login -u "${DOCKER_USER}" --password-stdin

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
