pipeline {
    agent any

    stages {
        stage('Prepare') {
            steps {
                sh 'rm -rf .deploy_git'
                sh 'git config --global user.name "jenkins"'
                sh 'git config --global user.email  "jenkins@aws.com"'
                sh 'git clone git@github.com:wankee/wankee.github.io.git .deploy_git'
            }
        }
        stage('Build') {
            steps {
                sh 'npm install'
            }
        }
        stage('Deploy') {
            steps {
                sh 'hexo g -d'
            }
        }
    }
}