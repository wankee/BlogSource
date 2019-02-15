pipeline {
    agent any

    stages {
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