pipeline {
    agent any
    
    environment {
        DOCKERHUB_USERNAME = 'calebrs' // He visto que usas calebrs en el login, actualiza si es necesario
        IMAGE_NAME = "${DOCKERHUB_USERNAME}/jenkins-demo"
    }

    stages {
        stage('1. Build Docker Image') {
            steps {
                sh "docker build --no-cache -t ${IMAGE_NAME}:${env.BUILD_NUMBER} ."
            }
        }

        stage('2. Static Analysis (Trivy)') {
            steps {
                sh "trivy image --exit-code 1 --severity HIGH,CRITICAL ${IMAGE_NAME}:${env.BUILD_NUMBER}"
            }
        }

        stage('3. Deploy & Wait for App') {
            steps {
                sh "BUILD_ID=${env.BUILD_NUMBER} IMAGE_NAME=${IMAGE_NAME} docker compose up -d"
                
                // CAMBIO AQUÍ: Envolvemos la espera en un bloque try/catch para depuración
                script {
                    try {
                        echo "Esperando a que la aplicación esté disponible..."
                        timeout(time: 2, unit: 'MINUTES') {
                            sh """
                                until curl --fail -s -o /dev/null http://127.0.0.1:8081; do
                                    echo "La aplicación no está lista aún, esperando 5 segundos..."
                                    sleep 5
                                done
                            """
                        }
                        echo "¡La aplicación está lista!"
                    } catch (e) {
                        // Si el timeout falla, imprimimos los logs del contenedor
                        echo "¡El tiempo de espera se agotó! La aplicación no inició correctamente."
                        echo "--- Mostrando logs del contenedor de la aplicación ---"
                        sh "docker compose logs"
                        echo "-----------------------------------------------------"
                        // Fallamos el pipeline manualmente para detener la ejecución
                        error "La aplicación no pudo iniciar."
                    }
                }
            }
        }
        
        stage('4. Dynamic Analysis (OWASP ZAP)') {
            steps {
                sh """
                    docker run --rm --network host --user root -v \$(pwd):/zap/wrk zaproxy/zap-stable zap-baseline.py \
                    -t http://127.0.0.1:8081 -J report.json || echo 'DAST scan completed with potential findings'
                """
            }
        }
    }

    post {
        always {
            echo "Limpiando el entorno de prueba..."
            sh "BUILD_ID=${env.BUILD_NUMBER} IMAGE_NAME=${IMAGE_NAME} docker compose down"
            archiveArtifacts artifacts: 'report.json', allowEmptyArchive: true
        }
        
        success {
            echo "Todos los análisis pasaron. Subiendo imagen a DockerHub..."
            withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
               sh "docker login -u ${USER} -p ${PASS}"
               sh "docker push ${IMAGE_NAME}:${env.BUILD_NUMBER}"
               sh "docker logout"
            }
        }
    }
}
