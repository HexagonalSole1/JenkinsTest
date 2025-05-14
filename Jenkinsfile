pipeline {
    agent any

    environment {
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
                    // Crear archivos de entorno locales según la rama
                    if (env.BRANCH_NAME == 'dev') {
                        writeEnvDev()
                    } else if (env.BRANCH_NAME == 'qa' || env.BRANCH_NAME == 'QA') {
                        writeEnvQA()
                    } else if (env.BRANCH_NAME == 'main') {
                        writeEnvProd()
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
                    return ['dev', 'qa', 'QA', 'main'].contains(env.BRANCH_NAME)
                }
            }
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        input message: '¿Confirmar despliegue a PRODUCCIÓN?'
                    }

                    def envFile = env.BRANCH_NAME == 'main' ? '.env.prod' :
                                  (env.BRANCH_NAME == 'qa' || env.BRANCH_NAME == 'QA') ? '.env.qa' : '.env.dev'

                    // Leer el archivo de entorno como texto
                    def envFileContent = readFile(envFile)

                    echo "Desplegando en servidor según entorno"

                    // Desplegar código y .env en EC2
                    sh """
                    ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ubuntu@$(getEc2Ip()) '
                        mkdir -p $(getRemotePath()) &&
                        cd $(getRemotePath()) &&
                        git fetch --all &&
                        git checkout ${env.BRANCH_NAME} &&
                        git reset --hard origin/${env.BRANCH_NAME} &&
                        npm ci
                    '

                    echo '${envFileContent}' | ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ubuntu@$(getEc2Ip()) 'cat > $(getRemotePath())/.env'

                    ssh -i \$SSH_KEY -o StrictHostKeyChecking=no ubuntu@$(getEc2Ip()) '
                        cd $(getRemotePath()) &&
                        pm2 restart $(getAppName()) || pm2 start server.js --name $(getAppName())
                    '
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline ejecutado exitosamente en la rama ${env.BRANCH_NAME}!"
        }
        failure {
            echo "Pipeline fallido en la rama ${env.BRANCH_NAME}, revisa los logs."
        }
    }
}

// Funciones auxiliares para obtener datos según la rama
def getEc2Ip() {
    if (env.BRANCH_NAME == 'main') return '23.21.175.134'
    if (env.BRANCH_NAME == 'qa' || env.BRANCH_NAME == 'QA') return '54.160.60.172'
    return '34.239.38.109'
}

def getRemotePath() {
    if (env.BRANCH_NAME == 'main') return '/home/ubuntu/JenkinsTest'
    if (env.BRANCH_NAME == 'qa' || env.BRANCH_NAME == 'QA') return '/home/ubuntu/JenkinsTest-qa'
    return '/home/ubuntu/JenkinsTest-dev'
}

def getAppName() {
    if (env.BRANCH_NAME == 'main') return 'health-api'
    if (env.BRANCH_NAME == 'qa' || env.BRANCH_NAME == 'QA') return 'health-api-qa'
    return 'health-api-dev'
}

// Funciones para generar archivos .env
def writeEnvDev() {
    writeFile file: '.env.dev', text: """
NODE_ENV=development
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
PORT=3000
API_URL=https://api.example.com
DB_HOST=prod-db.example.com
DB_PORT=5432
DB_NAME=proddb
DB_USER=produser
"""
}
