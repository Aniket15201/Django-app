pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'anki15201/django-trivy'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        ERROR_FILE = 'pipeline_error.log'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Checking out code...'
                deleteDir()
                checkout scm
            }
        }

        

        // stage('SonarQube Analysis') {
        //     environment {
        //         SCANNER_HOME = tool 'SonarQube Scanner'
        //     }
        //     steps {
        //         withSonarQubeEnv('SonarQube') {
        //             withEnv(["SONAR_USER_HOME=${env.WORKSPACE}/.sonar"]) {
        //                 sh '''
        //                     mkdir -p "$SONAR_USER_HOME"
        //                     "$SCANNER_HOME/bin/sonar-scanner" \
        //                         -Dsonar.projectKey=dummy \
        //                         -Dsonar.projectName=dummy \
        //                         -Dsonar.projectVersion=1.0 \
        //                         -Dsonar.sources=. \
        //                         -Dsonar.language=py \
        //                         -Dsonar.sourceEncoding=UTF-8
        //                 '''
        //             }
        //         }
        //     }
        // }

        // stage('Quality Gate') {
        //     steps {
        //         timeout(time: 5, unit: 'MINUTES') {
        //             waitForQualityGate abortPipeline: true
        //         }
        //     }
        // }

        stage('Build') {
            steps {
                script {
                    try {
                        echo "Building Docker image with tag: ${IMAGE_TAG}"

                        sh '''
                            if [ -f pod.sh ]; then
                                echo "Fixing line endings in pod.sh..."
                                sed -i 's/\\r$//' pod.sh
                                echo "Line endings fixed."
                            else
                                echo "pod.sh not found, skipping line ending fix."
                            fi
                        '''

                        withCredentials([string(credentialsId: 'dockerhub-token', variable: 'DOCKER_TOKEN')]) {
                            sh '''
                                set -e
                                export DOCKER_CONFIG=$WORKSPACE/.docker_config
                                mkdir -p $DOCKER_CONFIG
                                echo $DOCKER_TOKEN | docker login -u anki15201 --password-stdin
                                
                                docker build --platform linux/amd64\
                                    -t ${DOCKER_IMAGE}:${IMAGE_TAG} \
                                    -t ${DOCKER_IMAGE}:latest \
                                    --load .

                                docker logout
                            '''
                        }
                    } catch (err) {
                        echo "ERROR DETAILS: ${err}"
                        writeFile file: "${ERROR_FILE}", text: "Build stage failed:\n${err}"
                        throw err   // 🔥 instead of error()
                    }
                }
            }
        }

//         stage('Test') {
//             steps {
//                 script {
//                     def testContainerName = "${DOCKER_IMAGE.replace('/', '_')}_${IMAGE_TAG}"

//                     try {
//                         echo 'Starting test container...'

//                         def containerId = sh(
//                             script: "docker run -d -e TEST_MODE=true --name ${testContainerName} ${DOCKER_IMAGE}:${IMAGE_TAG}",
//                             returnStdout: true
//                         ).trim()

//                         echo "Started container: ${containerId}"

//                         sleep 5

//                         def exitCode = sh(
//                             script: "docker inspect ${testContainerName} --format='{{.State.ExitCode}}' || echo 1",
//                             returnStdout: true
//                         ).trim()

//                         if (exitCode != "0") {

//                             echo "Container exited with code ${exitCode}. Collecting logs..."

//                             def logs = sh(
//                                 script: """
//                                     echo "---- Docker Logs ----"
//                                     docker logs ${testContainerName} 2>&1 || true

//                                     echo "---- Flask Log ----"
//                                     docker cp ${testContainerName}:/app/flask.log flask_error.log 2>/dev/null || true
//                                     cat flask_error.log 2>/dev/null || true
//                                 """,
//                                 returnStdout: true
//                             ).trim()

//                             writeFile file: "${ERROR_FILE}", text: """Test stage failed:
// Container Exit Code: ${exitCode}

// ${logs}
// """

//                             error("Test stage failed — container exited with code ${exitCode}")
//                         }

//                         echo "Flask app started successfully."

//                     } catch (err) {

//                         def logs = sh(
//                             script: """
//                                 echo "---- Docker Logs ----"
//                                 docker logs ${testContainerName} 2>&1 || true

//                                 echo "---- Flask Log ----"
//                                 docker cp ${testContainerName}:/app/flask.log flask_error.log 2>/dev/null || true
//                                 cat flask_error.log 2>/dev/null || true
//                             """,
//                             returnStdout: true
//                         ).trim()

//                         writeFile file: "${ERROR_FILE}", text: """Test stage failed:
// ${err}

// ${logs}
// """

//                         error("Test stage failed")
//                     } finally {
//                         sh "docker stop ${testContainerName} || true"
//                         sh "docker rm ${testContainerName} || true"
//                     }
//                 }
//             }
//         }

        stage('Trivy Scan') {
            steps {
                script {
                    try {
                        echo "Running Trivy scan on Docker image..."

                        sh '''
                            trivy image \
                            --exit-code 0 \
                            --severity HIGH,CRITICAL \
                            --format template \
                            --template "@/usr/local/share/trivy/templates/html.tpl" \
                            -o trivy-report.html \
                            ${DOCKER_IMAGE}:${IMAGE_TAG}
                        '''

                        echo "Trivy scan completed"

                        // Archive report in Jenkins
                        archiveArtifacts artifacts: 'trivy-report.html', fingerprint: true

                        //show trivy report in UI
                        publishHTML([
                            reportDir: '.',
                            reportFiles: 'trivy-report.html',
                            reportName: 'Trivy Security Report'
                        ])

                    } catch (err) {
                        writeFile file: "${ERROR_FILE}", text: "Trivy scan failed:\n${err}"
                        error("Trivy scan failed")
                    }
                }
            }
        }

        stage('Push Image') {
            steps {
                script {
                    try {
                        echo "Pushing Docker image ${DOCKER_IMAGE}:${IMAGE_TAG} to Docker Hub..."
                        withCredentials([string(credentialsId: 'dockerhub-token', variable: 'DOCKER_TOKEN')]) {
                            sh '''
                                set -e
                                export DOCKER_CONFIG=$WORKSPACE/.docker_config
                                echo $DOCKER_TOKEN | docker login -u anki15201 --password-stdin
                                docker push ${DOCKER_IMAGE}:${IMAGE_TAG}
                                docker push ${DOCKER_IMAGE}:latest
                                docker logout
                            '''
                        }
                    } catch (err) {
                        writeFile file: "${ERROR_FILE}", text: "Push stage failed:\n${err}"
                        error("Push stage failed")
                    }
                }
            }
        }

        // stage('Deploy') {
        //     steps {
        //         script {
        //             try {
        //                 echo 'Deploying to Kubernetes...'
        //                 sh """
        //                     set -e
        //                     sed -i 's|image: .*|image: ${DOCKER_IMAGE}:${IMAGE_TAG}|' deployment.yaml
        //                     kubectl apply -f deployment.yaml
        //                     kubectl rollout status -f deployment.yaml
        //                 """
        //             } catch (err) {
        //                 writeFile file: "${ERROR_FILE}", text: "Deploy stage failed:\n${err}"
        //                 error("Deploy stage failed")
        //             }
        //         }
        //     }
        // }
    }

    post {
        always {
            echo 'Cleaning up Docker environment...'
            sh '''
                KEEP_IMAGE="${DOCKER_IMAGE}:${IMAGE_TAG}"

                # Stop & remove any container using this image
                docker ps -a --filter ancestor=$DOCKER_IMAGE -q | xargs -r docker rm -f || true

                # Remove all old images except the one to keep
                ALL_IMAGES=$(docker images "$DOCKER_IMAGE" --format "{{.Repository}}:{{.Tag}}" | grep -v "<none>" | grep -v "$KEEP_IMAGE")
                for img in $ALL_IMAGES; do
                    docker rmi -f "$img" || true
                done

                # Prune dangling images, builders, volumes
                docker image prune -f || true
                docker builder prune -f || true
                docker volume prune -f || true
            '''
        }

        // success {
        //     script {
        //         emailext(
        //             subject: "✅ Jenkins Pipeline Success: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        //             to: 'singale838@gmail.com',
        //             from: 'jenkins-success@proceedit.com',
        //             mimeType: 'text/html',
        //             body: "<h2>Build Successful</h2>"
        //         )
        //     }
        // }

        // failure {
        //     script {
        //         def errorMsg = fileExists("${ERROR_FILE}") ? readFile("${ERROR_FILE}") : "Unknown error (no details captured)"
        //         emailext(
        //             subject: "❌ Jenkins Pipeline Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
        //             to: 'singale838@gmail.com',
        //             from: 'jenkins-failure@proceedit.com',
        //             mimeType: 'text/html',
        //             body: "<pre>${errorMsg}</pre>"
        //         )
        //     }
        // }

        success {
            script {
                emailext(
                    subject: "✅ Jenkins Pipeline Success: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    to: 'ankikolwal15@gmail.com',
                    from: 'aniketbusiness15@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-report.html',
                    body: "<h2>Build Successful</h2><p>Trivy report attached.</p>"
                )
            }
        }

        failure {
            script {
                def errorMsg = fileExists("${ERROR_FILE}") ? readFile("${ERROR_FILE}") : "Unknown error (no details captured)"

                emailext(
                    subject: "❌ Jenkins Pipeline Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                    to: 'ankikolwal15@gmail.com',
                    from: 'aniketbusiness15@gmail.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-report.html',
                    body: """
                    <h2>Build Failed</h2>
                    <pre>${errorMsg}</pre>
                    <p>Trivy report attached.</p>
                    """
                )
            }
        }
    }
}
