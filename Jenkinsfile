pipeline {
    agent {
        node {
            label 'docker-springboot-agent'
        }
    }

    stages {
        stage('Build') {
            steps {
                echo 'Building...'
                sh 'mvn clean compile'
                echo 'Building successful'
            }
        }

        stage('Test') {
            steps {
                echo 'Testing...'
                sh 'mvn test'
                echo 'Testing successful'
            }
        }

        stage('Package') {
            steps {
                echo 'Packaging...'
                sh 'mvn package'
                echo 'Packaging successful'
            }
        }
    }

    post {
        success {
            echo 'Build succeed!'
        }

        failure {
            echo 'Build failed!'
        }
    }
}