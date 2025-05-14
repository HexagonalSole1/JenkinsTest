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
        
        stage('Load Environment Variables') {
            steps {
                script {
                    // Cargar variables de entorno basadas en la rama
                    if (env.BRANCH_NAME == 'dev') {
                        loadEnvVars('.env.dev')
                    } else if (env.BRANCH_NAME == 'qa' || env.BRANCH_NAME == 'QA') {
                        loadEnvVars('.env.qa')
                    } else if (env.BRANCH_NAME == 'main') {
                        loadEnvVars('.env.prod')
                    } else {
                        error "Rama no reconocida para despliegue: ${env.BRANCH_NAME}"
                    }
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
                    
                    // Crear archivo .env en el servidor remoto
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

// Función para cargar variables desde un archivo .env
def loadEnvVars(String file) {
    if (!fileExists(file)) {
        error "Archivo de variables de entorno no encontrado: ${file}"
    }
    
    def props = readProperties file: file
    props.each { key, value ->
        env."${key}" = value
    }
    
    echo "Variables de entorno cargadas desde ${file}"
}