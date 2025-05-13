pipeline {
    agent any

    environment {
        NODE_ENV = 'production'
        EC2_USER = 'ubuntu'
        EC2_IP = '98.81.245.108'
        REMOTE_PATH = '/home/ubuntu/node-healthcheck'
        SSH_KEY = credentials('ssh-key-ec2')
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/roberto14118927/node-healthcheck.git'
            }
        }

        stage('Build') {
            steps {
                sh 'rm -rf node_modules'
                sh 'npm ci'
            }
        }

        stage('Deploy') {
            steps {
                sh """
                ssh -i $SSH_KEY -o StrictHostKeyChecking=no $EC2_USER@$EC2_IP '
                    cd $REMOTE_PATH &&
                    git pull origin main &&
                    npm ci &&
                    pm2 restart health-api || pm2 start server.js --name health-api
                '
                """
            }
        }
    }
}
