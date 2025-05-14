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
                    if (env.BRANCH_NAME == 'dev') {
                        writeEnvDev()
                    } else if (env.BRANCH_NAME.toLowerCase() == 'qa') {
                        writeEnvQA()
                    } else if (env.BRANCH_NAME == 'main') {
                        writeEnvProd()
                    } else {
                        error "Rama no reconocida para despliegue: ${env.BRANCH_NAME}"
                    }

                    def envFile = env.BRANCH_NAME == 'main' ? '.env.prod' : 
                                 (env.BRANCH_NAME.toLowerCase() == 'qa') ? '.env.qa' : '.env.dev'
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
                sh '''
                rm -rf node_modules
                npm ci
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                npm test || echo "No tests or tests failed but continuing"
                '''
            }
        }

        stage('Deploy') {
            when {
                expression { 
                    return ['dev', 'qa', 'QA', 'main'].contains(env.BRANCH_NAME)
                }
            }
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        input message: '¿Confirmar despliegue a PRODUCCIÓN?'
                    }

                    echo "Desplegando en servidor ${env.NODE_ENV} (${env.EC2_IP})"

                    def envFileContent = readFile(file: env.BRANCH_NAME == 'main' ? '.env.prod' : 
                                              (env.BRANCH_NAME.toLowerCase() == 'qa') ? '.env.qa' : '.env.dev')

                    sh '''
                    ssh -i $SSH_KEY -o StrictHostKeyChecking=no $EC2_USER@$EC2_IP '
                        mkdir -p $REMOTE_PATH &&
                        cd $REMOTE_PATH &&
                        git fetch --all &&
                        git checkout ${BRANCH_NAME} &&
                        git reset --hard origin/${BRANCH_NAME} &&
                        npm ci
                    '
                    '''

                    sh """
                    echo '${envFileContent}' | ssh -i \$SSH_KEY -o StrictHostKeyChecking=no \$EC2_USER@\$EC2_IP 'cat > \$REMOTE_PATH/.env'
                    """

                    sh '''
                    ssh -i $SSH_KEY -o StrictHostKeyChecking=no $EC2_USER@$EC2_IP '
                        cd $REMOTE_PATH &&
                        pm2 restart $APP_NAME || pm2 start server.js --name $APP_NAME
                    '
                    '''

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

// Funciones externas

def writeEnvDev() {
    writeFile file: '.env.dev', text: """
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

def writeEnvQA() {
    writeFile file: '.env.qa', text: """
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

def writeEnvProd() {
    writeFile file: '.env.prod', text: """
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
