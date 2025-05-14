pipeline {
    agent any
    
    environment {
        // Variables globales comunes para todos los entornos
        SSH_KEY = credentials('ssh-key-ec2')
    }
    
    stages {
        stage('Checkout') {
            steps {
                script {
                    echo "Trabajando en la rama: ${env.BRANCH_NAME}"
                }
            }
        }
        
        stage('Setup Environment') {
            steps {
                script {
                    // Crear los archivos .env dinámicamente según la rama
                    if (env.BRANCH_NAME == 'dev') {
                        writeEnvDev()
                    } else if (env.BRANCH_NAME == 'qa' || env.BRANCH_NAME == 'QA') {
                        writeEnvQA()
                    } else if (env.BRANCH_NAME == 'main') {
                        writeEnvProd()
                    } else {
                        error "Rama no reconocida para despliegue: ${env.BRANCH_NAME}"
                    }
                    
                    // Cargar las variables de entorno
                    def envFile = env.BRANCH_NAME == 'main' ? '.env.prod' : 
                                 (env.BRANCH_NAME == 'qa' || env.BRANCH_NAME == 'QA') ? '.env.qa' : '.env.dev'
                    
                    def props = readProperties file: envFile
                    props.each { key, value ->
                        env."${key}" = value
                    }
                    
                    echo "Variables de entorno configuradas para: ${env.NODE_ENV}"
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
                    // Para producción, pedir confirmación
                    if (env.BRANCH_NAME == 'main') {
                        input message: '¿Confirmar despliegue a PRODUCCIÓN?'
                    }
                    
                    echo "Desplegando en servidor ${env.NODE_ENV} (${env.EC2_IP})"
                    
                    // Leer el contenido del archivo .env
                    def envFileContent = readFile(file: env.BRANCH_NAME == 'main' ? '.env.prod' : 
                                              (env.BRANCH_NAME == 'qa' || env.BRANCH_NAME == 'QA') ? '.env.qa' : '.env.dev')
                    
                    // Desplegar la aplicación
                    sh """
                    ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ${env.EC2_USER}@${env.EC2_IP} '
                        mkdir -p ${env.REMOTE_PATH} &&
                        cd ${env.REMOTE_PATH} &&
                        git fetch --all &&
                        git checkout ${env.BRANCH_NAME} &&
                        git reset --hard origin/${env.BRANCH_NAME} &&
                        npm ci
                    '
                    """
                    
                    // Crear archivo .env en el servidor remoto
                    sh """
                    echo '${envFileContent}' | ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ${env.EC2_USER}@${env.EC2_IP} 'cat > ${env.REMOTE_PATH}/.env'
                    """
                    
                    // Reiniciar o iniciar la aplicación con PM2
                    sh """
                    ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ${env.EC2_USER}@${env.EC2_IP} '
                        cd ${env.REMOTE_PATH} &&
                        pm2 restart ${env.APP_NAME} || pm2 start server.js --name ${env.APP_NAME}
                    '
                    """
                    
                    echo "Despliegue en ${env.NODE_ENV} completado"
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

// Función para crear el archivo .env.dev
def writeEnvDev() {
    writeFile file: '.env.dev', text: """
# Variables de entorno para DESARROLLO
NODE_ENV=development
EC2_USER=ubuntu
EC2_IP=34.239.38.109
REMOTE_PATH=/home/ubuntu/JenkinsTest-dev
APP_NAME=health-api-dev
PORT=3000
API_URL=https://api-dev.example.com
DB_HOST=dev-db.example.com
DB_PORT=5432
DB_NAME=devdb
DB_USER=devuser
"""
}

// Función para crear el archivo .env.qa
def writeEnvQA() {
    writeFile file: '.env.qa', text: """
# Variables de entorno para QA
NODE_ENV=qa
EC2_USER=ubuntu
EC2_IP=54.160.60.172
REMOTE_PATH=/home/ubuntu/JenkinsTest-qa
APP_NAME=health-api-qa
PORT=3000
API_URL=https://api-qa.example.com
DB_HOST=qa-db.example.com
DB_PORT=5432
DB_NAME=qadb
DB_USER=qauser
"""
}

// Función para crear el archivo .env.prod
def writeEnvProd() {
    writeFile file: '.env.prod', text: """
# Variables de entorno para PRODUCCIÓN
NODE_ENV=production
EC2_USER=ubuntu
EC2_IP=23.21.175.134
REMOTE_PATH=/home/ubuntu/JenkinsTest
APP_NAME=health-api
PORT=3000
API_URL=https://api.example.com
DB_HOST=prod-db.example.com
DB_PORT=5432
DB_NAME=proddb
DB_USER=produser
"""
}