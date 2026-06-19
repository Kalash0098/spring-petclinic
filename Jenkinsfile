pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'kalash655'
        IMAGE_NAME = "${DOCKERHUB_USER}/petclinic:${BUILD_NUMBER}"
        DOCKER_CREDS = credentials('dockerhub-creds')
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Kalash0098/spring-petclinic.git'
            }
        }

        stage('Maven Build') {
            steps {
                sh 'MAVEN_OPTS="-Xmx1024m" mvn clean package -DskipTests -Dcheckstyle.skip=true'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''mvn sonar:sonar \
                        -Dsonar.projectKey=Kalash0098_spring-petclinic \
                        -Dsonar.organization=kalash0098 \
                        -Dsonar.host.url=https://sonarcloud.io'''
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                sh "docker build -t ${IMAGE_NAME} ."
                sh "echo ${DOCKER_CREDS_PSW} | docker login -u ${DOCKER_CREDS_USR} --password-stdin"
                sh "docker push ${IMAGE_NAME}"
            }
        }

        stage('Terraform Apply') {
            environment {
                AWS_ACCESS_KEY_ID     = credentials('aws-access-key')
                AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
            }
            steps {
                dir('terraform') {
                    sh 'terraform init'
                    sh 'terraform apply -auto-approve'
                    script {
                        env.APP_SERVER_IP = sh(
                            script: 'terraform output -raw app_server_ip',
                            returnStdout: true
                        ).trim()
                    }
                }
            }
        }

        stage('Ansible Deploy') {
            steps {
                sh "echo 'APP_SERVER_IP is: ${APP_SERVER_IP}'"
                sh """
                    printf '[appserver]\\n${APP_SERVER_IP} ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/jenkins-key.pem\\n' > ansible/inventory
                """
                sh "cat ansible/inventory"
                sh """
                    ansible-playbook ansible/playbook.yml \
                        -i ansible/inventory \
                        -e dockerhub_user=${DOCKERHUB_USER} \
                        -e image_tag=${BUILD_NUMBER}
                """
            }
        }

    }

    post {
        success { echo 'Pipeline succeeded! App is live.' }
        failure { echo 'Pipeline failed. Check stage logs.' }
    }
}