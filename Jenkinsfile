pipeline {

    agent any

    parameters {
        choice(
            name: 'ACTION',
            choices: [
                'BUILD',
                'Deploy_All',
                'Deploy_DB',
                'Deploy_Redis',
                'Deploy_App',
                'Remove_All',
                'Remove_App',
                'Remove_Redis',
                'Remove_DB'
            ],
            description: 'Select Pipeline Action'
        )
    }

    tools {
        maven 'maven'
    }

    environment {
        APP_NAME = "book-my-ticket"
        DOCKER_IMAGE = "ranjitavaddebail/book-my-ticket"
        KUBECONFIG = "/var/lib/jenkins/.kube/config"
    }

    stages {

        stage('Checkout Source Code') {
            steps {
                checkout scm
            }
        }

        stage('Prepare Build') {
            when {
                expression { params.ACTION == 'BUILD' }
            }
            steps {
                echo "Preparing Build..."
                sh "mvn clean"
            }
        }

        stage('Build JAR') {
            when {
                expression { params.ACTION == 'BUILD' }
            }
            steps {
                sh "mvn clean package -DskipTests"
            }
        }

        stage('Build Docker Image') {
            when {
                expression { params.ACTION == 'BUILD' }
            }
            steps {
                sh """
                docker build -t ${DOCKER_IMAGE}:latest .
                """
            }
        }

        stage('Docker Login & Push') {
            when {
                expression { params.ACTION == 'BUILD' }
            }

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'docker-credentials',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {

                    sh """
                    echo \$DOCKER_PASSWORD | docker login -u \$DOCKER_USERNAME --password-stdin
                    docker tag ${DOCKER_IMAGE}:latest ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}
                    docker push ${DOCKER_IMAGE}:latest
                    """
                }
            }
        }

        stage('Clean Docker') {
            when {
                expression { params.ACTION == 'BUILD' }
            }
            steps {
                sh '''
                docker image prune -af
                docker system prune -af
                '''
            }
        }

        stage('Create Namespace') {

            when {

                anyOf {
                    expression { params.ACTION == 'Deploy_All' }
                    expression { params.ACTION == 'Deploy_DB' }
                    expression { params.ACTION == 'Deploy_Redis' }
                    expression { params.ACTION == 'Deploy_App' }
                }
            }

            steps {

                sh '''
                kubectl apply -f k8s/namespace/namespace.yml
                '''
            }
        }

        stage('Deploy Database') {

            when {

                anyOf {
                    expression { params.ACTION == 'Deploy_All' }
                    expression { params.ACTION == 'Deploy_DB' }
                }
            }

            steps {

                sh '''
                kubectl apply -f k8s/database/secret.yml
                kubectl apply -f k8s/database/service.yml
                kubectl apply -f k8s/database/statefulset.yml
                '''
            }
        }

        stage('Deploy Redis') {

            when {

                anyOf {
                    expression { params.ACTION == 'Deploy_All' }
                    expression { params.ACTION == 'Deploy_Redis' }
                }
            }

            steps {

                sh '''
                kubectl apply -f k8s/redis/deployment.yml
                kubectl apply -f k8s/redis/service.yml
                '''
            }
        }

        stage('Deploy Application') {

            when {

                anyOf {
                    expression { params.ACTION == 'Deploy_All' }
                    expression { params.ACTION == 'Deploy_App' }
                }
            }

            steps {

                sh '''
                kubectl apply -f k8s/application/deployment.yml
                kubectl apply -f k8s/application/service.yml
                '''
            }
        }

        stage('Verify Resources') {

            when {

                anyOf {
                    expression { params.ACTION == 'Deploy_All' }
                    expression { params.ACTION == 'Deploy_DB' }
                    expression { params.ACTION == 'Deploy_Redis' }
                    expression { params.ACTION == 'Deploy_App' }
                }
            }

            steps {

                sh '''
                kubectl get pods -n production
                kubectl get deployment -n production
                kubectl get statefulset -n production
                kubectl get svc -n production
                kubectl get pvc -n production
                '''
            }
        }

        stage('Remove Application') {

            when {

                anyOf {
                    expression { params.ACTION == 'Remove_All' }
                    expression { params.ACTION == 'Remove_App' }
                }
            }

            steps {

                sh '''
                kubectl delete -f k8s/application/service.yml --ignore-not-found=true
                kubectl delete -f k8s/application/deployment.yml --ignore-not-found=true
                '''
            }
        }

        stage('Remove Redis') {

            when {

                anyOf {
                    expression { params.ACTION == 'Remove_All' }
                    expression { params.ACTION == 'Remove_Redis' }
                }
            }

            steps {

                sh '''
                kubectl delete -f k8s/redis/service.yml --ignore-not-found=true
                kubectl delete -f k8s/redis/deployment.yml --ignore-not-found=true
                '''
            }
        }

        stage('Remove Database') {

            when {

                anyOf {
                    expression { params.ACTION == 'Remove_All' }
                    expression { params.ACTION == 'Remove_DB' }
                }
            }

            steps {

                sh '''
                kubectl delete -f k8s/database/statefulset.yml --ignore-not-found=true
                kubectl delete -f k8s/database/service.yml --ignore-not-found=true
                kubectl delete -f k8s/database/secret.yml --ignore-not-found=true
                '''
            }
        }

    }

    post {

        success {
            echo "Pipeline executed successfully."
        }

        failure {
            echo "Pipeline execution failed."
        }

        always {
            echo "Pipeline completed."
        }
    }
}