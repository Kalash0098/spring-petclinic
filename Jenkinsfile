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
sh 'mvn clean package -DskipTests'
}
}
stage('SonarQube Analysis') {
steps {
withSonarQubeEnv('sonarqube') {
sh 'mvn sonar:sonar'
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
AWS_ACCESS_KEY_ID = credentials('aws-access-key')
AWS_SECRET_ACCESS_KEY = credentials('aws-secret-key')
}
steps {
dir('terraform') {
sh 'terraform init'
sh 'terraform apply -auto-approve'
script {
env.APP_SERVER_IP = sh(script: 'terraform output -raw app_server_ip', returnStdout:
true).trim()
}
}
}
}
stage('Ansible Deploy') {
steps {
sh """
echo '[appserver]' > ansible/inventory
echo '${APP_SERVER_IP} ansible_user=ubuntu
ansible_ssh_private_key_file=~/.ssh/jenkins-key.pem' >> ansible/inventory
"""
sh """
ansible-playbook ansible/playbook.yml \\-i ansible/inventory \\-e dockerhub_user=${DOCKERHUB_USER} \\-e image_tag=${BUILD_NUMBER}
"""
}
}
}
post {
success { echo 'Pipeline succeeded! App is live.' }
failure { echo 'Pipeline failed. Check stage logs.' }
}
}
