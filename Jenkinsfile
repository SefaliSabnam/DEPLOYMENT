pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "sefali26/banking-app"
        AWS_CREDENTIALS = 'AWS-DOCKER-CREDENTIALS'
    }

    stages {
        stage('Checkout') {
            steps {
                git credentialsId: 'github-token', url: 'https://github.com/SefaliSabnam/Demo-docker.git', branch: env.BRANCH_NAME
            }
        }

        stage('Terraform Init') {
            steps {
                script {
                    withAWS(credentials: AWS_CREDENTIALS, region: 'ap-south-1') {
                        dir('terraform') {
                            sh 'terraform init'
                        }
                    }
                }
            }
        }

        stage('Terraform Plan') {
            steps {
                script {
                    withAWS(credentials: AWS_CREDENTIALS, region: 'ap-south-1') {
                        dir('terraform') {
                            sh 'terraform plan -out=tfplan'
                        }
                    }
                }
            }
        }

        stage('Terraform Apply') {
            when {
                expression { env.BRANCH_NAME == 'main' }  
            }
            steps {
                script {
                    withAWS(credentials: AWS_CREDENTIALS, region: 'ap-south-1') {
                        dir('terraform') {
                            sh 'terraform apply -auto-approve tfplan'
                        }

                        // Fetch dynamically created S3 bucket name
                        env.S3_BUCKET = sh(script: "terraform output -raw bucket_name", returnStdout: true).trim()
                        echo "S3 Bucket: ${env.S3_BUCKET}"
                    }
                }
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'DOCKER_HUB_TOKEN') {
                        sh "docker build -t ${env.DOCKER_IMAGE} ."
                        sh "docker push ${env.DOCKER_IMAGE}"
                    }
                }
            }
        }

        stage('Application Deployment to S3') {
            when {
                expression { env.BRANCH_NAME == 'main' }
            }
            steps {
                script {
                    withAWS(credentials: AWS_CREDENTIALS, region: 'ap-south-1') {
                        sh """
                            aws s3 cp index.html s3://${env.S3_BUCKET}/index.html --acl public-read
                        """
                        echo "Application successfully deployed to: http://${env.S3_BUCKET}.s3-website.ap-south-1.amazonaws.com"
                    }
                }
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution completed!'
        }
        success {
            echo 'Deployment to S3 was successful!'
        }
        failure {
            echo 'Deployment failed!'
        }
    }
}
