pipeline {
    agent {
        kubernetes {
            label 'maven-agent'
        }
    }

    environment {
        VERSION = ''
        IMAGE_TAG = ''
    }

    stages {
        stage('Get Version') {
            steps {
                script {
                    def version = sh(script: "mvn help:evaluate -Dexpression=project.version -q -DforceStdout", returnStdout: true).trim()
                    env.VERSION = version
                    env.IMAGE_TAG = "edenginting/springboot-jenkins:${version}"
                    echo "Image tag will be: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('Package') {
            steps {
                sh 'mvn package'
            }
        }

        stage('Docker Build & Push') {
            steps {
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

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    sed 's|__IMAGE__|${env.IMAGE_TAG}|g' k8s/Deployment.yaml | kubectl apply -f -

                    kubectl apply -f k8s/Service.yaml
                """
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
