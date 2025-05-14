pipeline {
    agent any
    
    environment {
        NODE_ENV = 'production'
        EC2_USER = 'ubuntu'
        EC2_IP_DEV = '34.239.38.109'
        EC2_IP_QA = '54.160.60.172'
        EC2_IP_PROD = '23.21.175.134'
        REMOTE_PATH_DEV = '/home/ubuntu/JenkinsTest-dev'
        REMOTE_PATH_QA = '/home/ubuntu/JenkinsTest-qa'
        REMOTE_PATH_PROD = '/home/ubuntu/JenkinsTest'
        SSH_KEY = credentials('ssh-key-ec2')
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    // No hacer checkout manual, usar el checkout automático de Jenkins
                    echo "Trabajando en la rama: ${env.BRANCH_NAME}"
                }
            }
        }
        
        stage('Build') {
            steps {
                sh 'rm -rf node_modules'
                sh 'npm ci'
            }
        }
        
        stage('Test') {
            steps {
                sh 'npm test || echo "No tests or tests failed but continuing"'
            }
        }
        
        stage('Deploy to DEV') {
            when {
                expression { env.BRANCH_NAME == 'dev' }
            }
            steps {
                echo "Desplegando en servidor DEV (${EC2_IP_DEV})"
                sh """
                ssh -i \$SSH_KEY -o StrictHostKeyChecking=no \$EC2_USER@\$EC2_IP_DEV '
                    mkdir -p \$REMOTE_PATH_DEV &&
                    cd \$REMOTE_PATH_DEV &&
                    git fetch --all &&
                    git checkout dev &&
                    git reset --hard origin/dev &&
                    npm ci &&
                    pm2 restart health-api-dev || pm2 start server.js --name health-api-dev
                '
                """
                echo 'Despliegue en DEV completado'
            }
        }
        
        stage('Deploy to QA') {
            when {
                expression { env.BRANCH_NAME == 'qa' || env.BRANCH_NAME == 'QA' }
            }
            steps {
                echo "Desplegando en servidor QA (${EC2_IP_QA})"
                sh """
                ssh -i \$SSH_KEY -o StrictHostKeyChecking=no \$EC2_USER@\$EC2_IP_QA '
                    mkdir -p \$REMOTE_PATH_QA &&
                    cd \$REMOTE_PATH_QA &&
                    git fetch --all &&
                    git checkout qa &&
                    git reset --hard origin/qa &&
                    npm ci &&
                    pm2 restart health-api-qa || pm2 start server.js --name health-api-qa
                '
                """
                echo 'Despliegue en QA completado'
            }
        }
        
        stage('Deploy to PROD') {
            when {
                expression { env.BRANCH_NAME == 'main' }
            }
            steps {
                input message: '¿Confirmar despliegue a PRODUCCIÓN?'
                echo "Desplegando en servidor PROD (${EC2_IP_PROD})"
                sh """
                ssh -i \$SSH_KEY -o StrictHostKeyChecking=no \$EC2_USER@\$EC2_IP_PROD '
                    mkdir -p \$REMOTE_PATH_PROD &&
                    cd \$REMOTE_PATH_PROD &&
                    git fetch --all &&
                    git checkout main &&
                    git reset --hard origin/main &&
                    npm ci &&
                    pm2 restart health-api || pm2 start server.js --name health-api
                '
                """
                echo 'Despliegue en PRODUCCIÓN completado'
            }
        }
    }
    
    post {
        success {
            echo "Pipeline ejecutado exitosamente en la rama ${env.BRANCH_NAME}!"
        }
        failure {
            echo "Pipeline fallido en la rama ${env.BRANCH_NAME}, revisa los logs para más información"
        }
    }
}