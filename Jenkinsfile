// pipeline {
//     agent any

//     environment {
//         DOCKER_IMAGE = "karthik854/devopsexamapp:latest"
//         SCANNER_HOME = tool 'sonar-scanner'
//     }

//     stages {

//         /* ========================
//            GIT CHECKOUT
//         ======================== */
//         stage('Git Checkout') {
//             steps {
//                 git url: 'https://github.com/karthikpyati777/devops-exam-app.git',
//                     branch: 'master'
//             }
//         }

//         /* ========================
//            TRIVY FILE SYSTEM SCAN
//         ======================== */
//         stage('Trivy FS Scan') {
//             steps {
//                 sh '''
//                 trivy fs \
//                   --scanners vuln,misconfig \
//                   --format table \
//                   -o trivy-fs-report.html .
//                 '''
//             }
//         }

//         /* ========================
//            SONARQUBE ANALYSIS
//         ======================== */
//         stage('SonarQube Analysis') {
//             steps {
//                 withSonarQubeEnv('sonar') {
//                     sh """
//                     ${SCANNER_HOME}/bin/sonar-scanner \
//                     -Dsonar.projectName=devops-exam-app \
//                     -Dsonar.projectKey=devops-exam-app \
//                     -Dsonar.sources=. \
//                     -Dsonar.language=py \
//                     -Dsonar.python.version=3 \
//                     -Dsonar.exclusions=trivy-fs-report.html \
//                     -Dsonar.host.url=http://localhost:9000
//                     """
//                 }
//             }
//         }

//         /* ========================
//            VERIFY DOCKER COMPOSE
//         ======================== */
//         stage('Verify Docker Compose') {
//             steps {
//                 sh '''
//                 docker compose version || {
//                     echo "Docker Compose not available"
//                     exit 1
//                 }
//                 '''
//             }
//         }

//         /* ========================
//            BUILD DOCKER IMAGE
//         ======================== */
//         stage('Build Docker Image') {
//             steps {
//                 dir('backend') {
//                     sh '''
//                     docker build -t ${DOCKER_IMAGE} .
//                     '''
//                 }
//             }
//         }

//         /* ========================
//            DOCKER PUSH (SEPARATE STAGE)
//         ======================== */
//         stage('Docker Push') {
//             steps {
//                 withDockerRegistry(credentialsId: 'docker') {
//                     sh '''
//                     docker push ${DOCKER_IMAGE}
//                     '''
//                 }
//             }
//         }

//         /* ========================
//            DOCKER SCOUT SCAN
//         ======================== */
//         stage('Docker Scout Image Analysis') {
//             steps {
//                 withDockerRegistry(credentialsId: 'docker') {
//                     sh '''
//                     docker scout quickview ${DOCKER_IMAGE}
//                     docker scout cves ${DOCKER_IMAGE}
//                     docker scout recommendations ${DOCKER_IMAGE}
//                     '''
//                 }
//             }
//         }

//         /* ========================
//            DEPLOY USING DOCKER COMPOSE
//         ======================== */
//         stage('Deploy with Docker Compose') {
//             steps {
//                 sh '''
//                 docker compose down --remove-orphans || true
//                 docker compose up -d --build

//                 echo "Waiting for MySQL..."
//                 timeout 120s bash -c '
//                 until docker compose exec -T mysql \
//                   mysqladmin ping -uroot -prootpass --silent;
//                 do
//                     sleep 5
//                 done'
//                 '''
//             }
//         }

//         /* ========================
//            VERIFY DEPLOYMENT
//         ======================== */
//         stage('Verify Deployment') {
//             steps {
//                 sh '''
//                 echo "Containers:"
//                 docker compose ps

//                 echo "Testing Flask API..."
//                 curl -I http://localhost:5000 || true
//                 '''
//             }
//         }
//     }

//     /* ========================
//        POST ACTIONS
//     ======================== */
//     post {
//         success {
//             echo '🚀 Pipeline completed successfully!'
//             archiveArtifacts artifacts: 'trivy-fs-report.html', allowEmptyArchive: true
//         }

//         failure {
//             echo '❌ Pipeline failed. Collecting logs...'
//             sh 'docker compose logs --tail=50 || true'
//         }

//         always {
//             sh 'docker compose ps || true'
//         }
//     }
// }



pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "karthik854/devopsexamapp:latest"
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_REGISTRY = "https://index.docker.io/v1/"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        /* ========================
           GIT CHECKOUT
        ======================== */
        stage('Git Checkout') {
            steps {
                git url: 'https://github.com/karthikpyati777/devops-exam-app.git',
                    branch: 'master'
            }
        }

        /* ========================
           TRIVY FILE SYSTEM SCAN
           (DO NOT FAIL PIPELINE)
        ======================== */
        stage('Trivy FS Scan') {
            steps {
                sh '''
                echo "Running Trivy FS scan..."
                trivy fs \
                  --scanners vuln,misconfig \
                  --severity HIGH,CRITICAL \
                  --format table \
                  -o trivy-fs-report.html . || true
                '''
            }
        }

        /* ========================
           SONARQUBE ANALYSIS
        ======================== */
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh """
                    ${SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectName=devops-exam-app \
                    -Dsonar.projectKey=devops-exam-app \
                    -Dsonar.sources=. \
                    -Dsonar.exclusions=trivy-fs-report.html
                    """
                }
            }
        }

        /* ========================
           SONAR QUALITY GATE
        ======================== */
        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        /* ========================
           VERIFY DOCKER COMPOSE
        ======================== */
        stage('Verify Docker Compose') {
            steps {
                sh 'docker compose version'
            }
        }

        /* ========================
           BUILD DOCKER IMAGE
        ======================== */
        stage('Build Docker Image') {
            steps {
                dir('backend') {
                    sh '''
                    set -e
                    echo "Building Docker image..."
                    docker build -t ${DOCKER_IMAGE} .
                    '''
                }
            }
        }

        /* ========================
           IMAGE SECURITY SCAN
           (FAIL ONLY IF CRITICAL)
        ======================== */
        stage('Trivy Image Scan') {
            steps {
                sh '''
                set -e
                echo "Scanning Docker image..."
                trivy image \
                  --severity CRITICAL \
                  --exit-code 1 \
                  ${DOCKER_IMAGE}
                '''
            }
        }

        /* ========================
           DOCKER PUSH
        ======================== */
        stage('Docker Push') {
            steps {
                withDockerRegistry(credentialsId: 'docker', url: "${DOCKER_REGISTRY}") {
                    sh 'docker push ${DOCKER_IMAGE}'
                }
            }
        }

        /* ========================
           DEPLOY USING DOCKER COMPOSE
        ======================== */
        stage('Deploy with Docker Compose') {
            steps {
                sh '''
                docker compose down --remove-orphans || true
                docker compose up -d --build
                '''
            }
        }

        /* ========================
           VERIFY DEPLOYMENT
        ======================== */
        stage('Verify Deployment') {
            steps {
                sh '''
                docker compose ps
                curl -I http://localhost:5000 || true
                '''
            }
        }
    }

    /* ========================
       POST ACTIONS
    ======================== */
    post {

        success {
            echo '🚀 Pipeline completed successfully!'
            archiveArtifacts artifacts: 'trivy-fs-report.html', allowEmptyArchive: true
        }

        failure {
            echo '❌ Pipeline failed!'
            sh 'docker compose logs --tail=50 || true'
        }

        always {
            sh 'docker compose ps || true'
        }
    }
}
