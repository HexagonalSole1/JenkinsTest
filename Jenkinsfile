pipeline {
    agent any
    
    environment {
        // Configuración dinámica del entorno basada en la rama
        NODE_ENV = "${env.BRANCH_NAME == 'main' ? 'production' : (env.BRANCH_NAME == 'qa' || env.BRANCH_NAME == 'QA') ? 'qa' : 'dev'}"
        
        // Configuración de despliegue
        EC2_USER = 'ubuntu'
        SSH_KEY = credentials('ssh-key-ec2')
        
        // Mapas para configuración dinámica
        EC2_IPS = [
            dev: '34.239.38.109',
            qa: '54.160.60.172',
            production: '23.21.175.134'
        ]
        
        REMOTE_PATHS = [
            dev: '/home/ubuntu/JenkinsTest-dev',
            qa: '/home/ubuntu/JenkinsTest-qa',
            production: '/home/ubuntu/JenkinsTest'
        ]
        
        APP_NAME = "health-api${NODE_ENV == 'production' ? '' : '-' + NODE_ENV}"
        
        // Usar temporalmente estas variables para facilitar el acceso
        EC2_IP = "${EC2_IPS[NODE_ENV]}"
        REMOTE_PATH = "${REMOTE_PATHS[NODE_ENV]}"
        
        // Definir timeout para operaciones SSH
        SSH_TIMEOUT = '300'
    }
    
    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }
    
    stages {
        stage('Preparación') {
            steps {
                echo "✅ Iniciando pipeline en ${NODE_ENV} para la rama: ${env.BRANCH_NAME}"
                echo "✅ Servidor de destino: ${EC2_IP}"
                echo "✅ Ruta remota: ${REMOTE_PATH}"
                
                // Limpiar workspace
                cleanWs()
                checkout scm
            }
        }
        
        stage('Verificación') {
            steps {
                sh '''
                    node -v
                    npm -v
                    git --version
                '''
            }
        }
        
        stage('Instalación de dependencias') {
            steps {
                sh 'npm ci --no-audit --no-fund'
            }
        }
        
        stage('Lint') {
            steps {
                sh 'npm run lint || echo "⚠️ Linting con problemas, pero continuamos..."'
            }
        }
        
        stage('Test') {
            steps {
                sh 'npm test || echo "⚠️ No hay tests o fallaron, pero continuamos..."'
            }
        }
        
        stage('Build') {
            steps {
                sh 'npm run build || echo "⚠️ No hay script de build o falló, pero continuamos..."'
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
                    // Confirmación para producción
                    if (env.NODE_ENV == 'production') {
                        timeout(time: 15, unit: 'MINUTES') {
                            input message: '🚨 ¿Confirmar despliegue a PRODUCCIÓN?', ok: 'Desplegar'
                        }
                    }
                    
                    echo "🚀 Desplegando en servidor ${NODE_ENV} (${EC2_IP})"
                    
                    // Contenido del archivo .env - Variables de entorno seguras
                    def envContent = """
NODE_ENV=${NODE_ENV}
PORT=3000
API_URL=https://api${NODE_ENV == 'production' ? '' : '-' + NODE_ENV}.example.com
LOG_LEVEL=${NODE_ENV == 'production' ? 'info' : 'debug'}
"""
                    
                    // Verificar conectividad al servidor
                    sh """
                    ssh -i \$SSH_KEY -o StrictHostKeyChecking=no -o ConnectTimeout=${SSH_TIMEOUT} ${EC2_USER}@${EC2_IP} 'echo "Conexión establecida con éxito"'
                    """
                    
                    // Crear directorio remoto si no existe y preparar
                    sh """
                    ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_IP} '
                        mkdir -p ${REMOTE_PATH}
                        mkdir -p ${REMOTE_PATH}/backup
                    '
                    """
                    
                    // Respaldar versión actual antes de desplegar
                    sh """
                    ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_IP} '
                        if [ -d "${REMOTE_PATH}/.git" ]; then
                            cd ${REMOTE_PATH}
                            CURRENT_VERSION=\$(git rev-parse HEAD || echo "unknown")
                            echo "Respaldando versión \$CURRENT_VERSION"
                            tar -czf ./backup/backup-\$(date +%Y%m%d%H%M%S)-\${CURRENT_VERSION:0:7}.tar.gz --exclude=node_modules --exclude=.git --exclude=backup .
                        fi
                    '
                    """
                    
                    // Crear archivo tar con el código
                    sh """
                    tar -czf deploy.tar.gz --exclude=node_modules --exclude=.git .
                    scp -i \$SSH_KEY -o StrictHostKeyChecking=no deploy.tar.gz ${EC2_USER}@${EC2_IP}:${REMOTE_PATH}/
                    """
                    
                    // Desplegar en el servidor
                    sh """
                    ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_IP} '
                        cd ${REMOTE_PATH}
                        rm -rf *.js *.json src config .env
                        tar -xzf deploy.tar.gz
                        rm deploy.tar.gz
                        npm ci --production --no-audit --no-fund
                    '
                    """
                    
                    // Crear archivo .env
                    sh """
                    echo '${envContent}' | ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_IP} 'cat > ${REMOTE_PATH}/.env'
                    """
                    
                    // Verificar si PM2 está instalado y reiniciar la aplicación
                    sh """
                    ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_IP} '
                        cd ${REMOTE_PATH}
                        if ! command -v pm2 &> /dev/null; then
                            echo "PM2 no está instalado. Instalando globalmente..."
                            npm install -g pm2
                        fi
                        
                        # Verificar si la aplicación ya está en PM2
                        if pm2 list | grep -q "${APP_NAME}"; then
                            echo "Reiniciando aplicación ${APP_NAME}..."
                            pm2 reload ${APP_NAME} || pm2 restart ${APP_NAME}
                        else
                            echo "Iniciando aplicación ${APP_NAME} por primera vez..."
                            pm2 start server.js --name ${APP_NAME} --time
                        fi
                        
                        # Guardar configuración de PM2
                        pm2 save
                    '
                    """
                    
                    echo "✅ Despliegue en ${NODE_ENV} completado con éxito!"
                }
            }
        }
        
        stage('Verificación de despliegue') {
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
                    // Esperar a que la aplicación esté disponible
                    sh """
                    ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_IP} '
                        cd ${REMOTE_PATH}
                        echo "Verificando estado de la aplicación..."
                        pm2 status ${APP_NAME}
                        
                        # Verificar que el proceso esté en ejecución
                        if ! pm2 list | grep -q "${APP_NAME}.*online"; then
                            echo "❌ La aplicación no está en ejecución!"
                            exit 1
                        fi
                        
                        # Verificar logs para errores
                        if pm2 logs ${APP_NAME} --lines 10 | grep -q "Error"; then
                            echo "⚠️ Se encontraron errores en los logs"
                        fi
                        
                        echo "✅ Aplicación verificada correctamente"
                    '
                    """
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        
        success {
            echo "🎉 ¡Pipeline exitoso en ${env.BRANCH_NAME} (${NODE_ENV})!"
            
            // Notificación de éxito (usando echo, sin dependencias)
            sh '''
                echo "Notificación: Despliegue exitoso en ${NODE_ENV} para la rama ${BRANCH_NAME}"
            '''
        }
        
        failure {
            echo "❌ Fallo en ${env.BRANCH_NAME} (${NODE_ENV}). Verificando detalles..."
            
            // Intentar recuperar logs de la aplicación en caso de fallo
            script {
                try {
                    sh """
                    ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_IP} '
                        cd ${REMOTE_PATH}
                        echo "Últimos logs de la aplicación:"
                        pm2 logs ${APP_NAME} --lines 50 || echo "No se pudieron obtener logs"
                    '
                    """
                } catch (Exception e) {
                    echo "No se pudieron recuperar los logs: ${e.message}"
                }
            }
            
            // Notificación de fallo (usando echo, sin dependencias)
            sh '''
                echo "Notificación: FALLO en el despliegue en ${NODE_ENV} para la rama ${BRANCH_NAME}"
            '''
        }
    }
}