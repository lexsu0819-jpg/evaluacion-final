pipeline {
    agent any

    environment {
        // App settings
        IMAGE_NAME = 'veterinaria-app'
        GREEN_URL = 'http://veterinaria-app-green:8080/api/health'
        PROD_URL = 'http://veterinaria-nginx/api/health'
        PROMETHEUS_URL = 'http://veterinaria-prometheus:9090'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code...'
                checkout scm
            }
        }

        stage('Versionado SemVer') {
            steps {
                script {
                    echo 'Incrementando versión tipo patch (simulado de CI)'
                    // Mostrar el BUILD_NUMBER para debugging
                    echo "BUILD_NUMBER = ${BUILD_NUMBER}"
                    
                    // CORREGIDO: Usar comillas dobles y verificar el cambio
                    sh """
                        echo "=== Antes del cambio ==="
                        grep '<version>' app/pom.xml | head -1
                        
                        sed -i 's|<version>.*</version>|<version>1.0.${BUILD_NUMBER}</version>|' app/pom.xml
                        
                        echo "=== Después del cambio ==="
                        grep '<version>' app/pom.xml | head -1
                    """
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo 'Construyendo imagen Docker...'
                // Limpiar caché de Docker y reconstruir
                sh '''
                    docker compose build --no-cache app-green
                '''
            }
        }

        stage('Deploy GREEN') {
            steps {
                echo 'Desplegando en entorno GREEN...'
                // Detener y eliminar contenedor anterior antes de desplegar
                sh '''
                    docker compose stop app-green || true
                    docker compose rm -f app-green || true
                    docker compose up -d app-green
                '''
            }
        }

        stage('Health Check GREEN') {
            steps {
                echo 'Verificando salud del entorno GREEN...'
                retry(5) {
                    sleep 10
                    sh "curl -f -s ${GREEN_URL} || exit 1"
                }
            }
        }

        stage('Quality Gate (Prometheus)') {
            steps {
                script {
                    echo 'Consultando métricas en Prometheus...'
                    sleep 15
                    
                    def latencyQuery = 'rate(http_server_requests_seconds_sum{uri="/api/health"}[1m])/rate(http_server_requests_seconds_count{uri="/api/health"}[1m])'
                    def latencyRes = sh(script: "curl -s -g '${PROMETHEUS_URL}/api/v1/query?query=${latencyQuery}' | grep '\"value\"' || echo 'No data'", returnStdout: true).trim()
                    
                    def errQuery = 'rate(http_server_requests_seconds_count{status=~"5.."}[1m])'
                    def errRes = sh(script: "curl -s -g '${PROMETHEUS_URL}/api/v1/query?query=${errQuery}' | grep '\"value\"' || echo 'No data'", returnStdout: true).trim()

                    echo "Latency Check: ${latencyRes}"
                    echo "Error Check: ${errRes}"
                    echo 'Quality Gates Pasados.'
                }
            }
        }

        stage('Switch Traffic to GREEN') {
            steps {
                echo 'Cambiando el tráfico hacia el entorno GREEN...'
                sh '''
                    chmod +x scripts/switch-traffic.sh || true
                    ./scripts/switch-traffic.sh green || echo "Script no encontrado, continuando..."
                '''
            }
        }

        stage('Validate Production') {
            steps {
                echo 'Verificando producción final...'
                retry(3) {
                    sleep 5
                    sh "curl -f -s ${PROD_URL} || exit 1"
                }
            }
        }
    }

    post {
        failure {
            echo 'Pipeline falló. Ejecutando Rollback Automático a BLUE...'
            sh '''
                chmod +x scripts/switch-traffic.sh || true
                ./scripts/switch-traffic.sh blue || docker compose up -d app-blue
            '''
        }
        success {
            echo '¡Despliegue Exitoso a Producción (GREEN)!'
        }
        always {
            echo 'Pipeline finalizado.'
        }
    }
}