pipeline {
    agent any

    stages {
        stage('Prepare') {
            steps {
                sh 'hexo clean'
                sh 'rm -rf .deploy_git'
                sh 'git config --global user.name "jenkins"'
                sh 'git config --global user.email  "wang24118@qq.com"'
                sh 'git clone git@github.com:wankee/wankee.github.io.git .deploy_git'
            }
        }
        stage('Build') {
            steps {
                sh 'npm install'
                sh 'hexo g'
            }
        }
        stage('Deploy') {
            steps {
                sh 'hexo d'
            }
        }
    }
}