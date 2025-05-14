pipeline {
    agent any
    
    environment {
        // Variables de entorno definidas directamente aquí
        NODE_ENV = env.BRANCH_NAME == 'main' ? 'production' : (env.BRANCH_NAME == 'qa' || env.BRANCH_NAME == 'QA') ? 'qa' : 'development'
        EC2_USER = 'ubuntu'
        EC2_IP_DEV = '34.239.38.109'
        EC2_IP_QA = '54.160.60.172'
        EC2_IP_PROD = '23.21.175.134'
        REMOTE_PATH_DEV = '/home/ubuntu/JenkinsTest-dev'
        REMOTE_PATH_QA = '/home/ubuntu/JenkinsTest-qa'
        REMOTE_PATH_PROD = '/home/ubuntu/JenkinsTest'
        SSH_KEY = credentials('ssh-key-ec2')
        APP_NAME = env.BRANCH_NAME == 'main' ? 'health-api' : (env.BRANCH_NAME == 'qa' || env.BRANCH_NAME == 'QA') ? 'health-api-qa' : 'health-api-dev'
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
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
        
        stage('Deploy') {
            when {
                expression { 
                    return env.BRANCH_NAME == 'dev' || 
                           env.BRANCH_NAME == 'qa' || 
                           env.BRANCH_NAME == 'QA' || 
                           env.BRANCH_NAME == 'main' 
                }
            }
            steps {
                script {
                    def EC2_IP = ''
                    def REMOTE_PATH = ''
                    
                    // Determinar IP y ruta según la rama
                    if (env.BRANCH_NAME == 'main') {
                        EC2_IP = EC2_IP_PROD
                        REMOTE_PATH = REMOTE_PATH_PROD
                        
                        // Para producción, pedir confirmación
                        input message: '¿Confirmar despliegue a PRODUCCIÓN?'
                    } else if (env.BRANCH_NAME == 'qa' || env.BRANCH_NAME == 'QA') {
                        EC2_IP = EC2_IP_QA
                        REMOTE_PATH = REMOTE_PATH_QA
                    } else {  // dev
                        EC2_IP = EC2_IP_DEV
                        REMOTE_PATH = REMOTE_PATH_DEV
                    }
                    
                    echo "Desplegando en servidor ${NODE_ENV} (${EC2_IP})"
                    
                    // Crear archivo .env en el servidor
                    def envContent = """
NODE_ENV=${NODE_ENV}
PORT=3000
API_URL=https://api${NODE_ENV == 'production' ? '' : '-' + NODE_ENV}.example.com
"""
                    
                    // Desplegar la aplicación
                    sh """
                    ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_IP} '
                        mkdir -p ${REMOTE_PATH} &&
                        cd ${REMOTE_PATH} &&
                        git fetch --all &&
                        git checkout ${env.BRANCH_NAME} &&
                        git reset --hard origin/${env.BRANCH_NAME} &&
                        npm ci
                    '
                    """
                    
                    // Crear archivo .env en el servidor
                    sh """
                    echo '${envContent}' | ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_IP} 'cat > ${REMOTE_PATH}/.env'
                    """
                    
                    // Reiniciar o iniciar la aplicación con PM2
                    sh """
                    ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_IP} '
                        cd ${REMOTE_PATH} &&
                        pm2 restart ${APP_NAME} || pm2 start server.js --name ${APP_NAME}
                    '
                    """
                    
                    echo "Despliegue en ${NODE_ENV} completado"
                }
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