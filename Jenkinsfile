
// SHOP EASE LAB 3
// CI/CD + Manual Approval + Safe Production Deployment + Automatic Rollback

def deploy(String name, int port, String image, boolean bad) {
    def version = image.tokenize(':')[-1]

    sh """
        docker rm -f ${name} || true

        docker run -d \
          --name ${name} \
          --network shopease-net \
          -p ${port}:5000 \
          --memory 256m \
          --cap-drop ALL \
          --security-opt no-new-privileges \
          -e APP_VERSION=${version} \
          -e BREAK_HEALTH=${bad} \
          ${image}
    """
}

def smoke(String host) {
    withEnv(["HOST=${host}"]) {
        sh '''
            for i in 1 2 3 4 5 6 7 8 9 10; do
                if curl -fsS http://$HOST:5000/health; then
                    echo ""
                    echo "HEALTH CHECK PASSED for $HOST"
                    exit 0
                fi

                echo "Waiting for $HOST ($i/10)..."
                sleep 3
            done

            echo "HEALTH CHECK FAILED for $HOST"
            exit 1
        '''
    }
}

def notifyTeam(String status) {
    def msg = "ShopEase order-service | ${status} | build #${env.BUILD_NUMBER} | ${env.BUILD_URL}"

    writeFile file: 'notify.json',
        text: '{"text": "' + msg + '"}'

    withCredentials([
        string(credentialsId: 'notify-token', variable: 'NOTIFY_TOKEN')
    ]) {
        sh '''
            curl -sS -X POST \
              -H "Content-Type: application/json" \
              -H "Authorization: Bearer $NOTIFY_TOKEN" \
              --data @notify.json \
              http://notify-receiver:9000/ || true
        '''
    }
}

pipeline {

    agent any

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    parameters {
        booleanParam(
            name: 'SIMULATE_PROD_FAILURE',
            defaultValue: false,
            description: 'Enable to simulate an unhealthy production release and test rollback.'
        )
    }

    environment {
        IMAGE = "shopease/order-service:${BUILD_NUMBER}"
    }

    stages {

        stage('Quality Gate') {
            steps {
                sh 'docker build --progress=plain --target test .'
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build --target runtime -t $IMAGE .'
            }
        }

        stage('Deploy to Staging') {
            steps {
                script {
                    deploy(
                        'shopease-staging',
                        8081,
                        env.IMAGE,
                        false
                    )
                }
            }
        }

        stage('Smoke Test Staging') {
            steps {
                script {
                    smoke('shopease-staging')
                }
            }
        }

        stage('Approval') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    input(
                        message: 'Deploy build #' + env.BUILD_NUMBER + ' to PRODUCTION?',
                        ok: 'Deploy',
                        submitter: 'admin'
                    )
                }
            }
        }

        stage('Deploy to Production') {
            steps {
                script {

                    // Remember the currently running production image
                    def previousImage = sh(
                        script: "docker inspect --format='{{.Config.Image}}' shopease-prod",
                        returnStdout: true
                    ).trim()

                    echo "Previous production image: ${previousImage}"

                    boolean productionCutoverStarted = false

                    try {

                        // Step 1: Deploy candidate on isolated test port 8085
                        echo "Starting production candidate deployment"

                        deploy(
                            'shopease-prod-candidate',
                            8085,
                            env.IMAGE,
                            params.SIMULATE_PROD_FAILURE
                        )

                        // Step 2: Verify candidate before touching production
                        smoke('shopease-prod-candidate')

                        echo "Candidate health check successful"

                        // Step 3: Replace production only after candidate passes
                        productionCutoverStarted = true

                        deploy(
                            'shopease-prod',
                            8083,
                            env.IMAGE,
                            false
                        )

                        // Step 4: Verify actual production
                        smoke('shopease-prod')

                        echo "PRODUCTION DEPLOYMENT SUCCESSFUL"

                    } catch (err) {

                        echo "PRODUCTION DEPLOYMENT FAILED"
                        echo "Starting rollback to ${previousImage}"

                        // If production replacement started, restore old image
                        if (productionCutoverStarted) {

                            try {
                                deploy(
                                    'shopease-prod',
                                    8083,
                                    previousImage,
                                    false
                                )

                                smoke('shopease-prod')

                                echo "ROLLBACK SUCCESSFUL"

                            } catch (rollbackErr) {

                                echo "CRITICAL: ROLLBACK ALSO FAILED"

                                error(
                                    "Deployment failed and rollback failed: ${rollbackErr}"
                                )
                            }

                        } else {

                            echo "Production was not replaced."
                            echo "Previous production container remains untouched."
                        }

                        // Keep the failed release marked as FAILED
                        throw err

                    } finally {

                        // Remove temporary candidate container
                        sh 'docker rm -f shopease-prod-candidate || true'
                    }
                }
            }
        }
    }

    post {

        always {
            sh '''
                echo "build=$BUILD_NUMBER image=$IMAGE commit=$(git rev-parse --short HEAD)" > build-report.txt
            '''

            archiveArtifacts(
                artifacts: 'build-report.txt',
                allowEmptyArchive: true
            )

            sh 'docker image prune -f || true'
        }

        success {
            script {
                notifyTeam('SUCCESS')
            }
        }

        failure {
            script {
                notifyTeam('FAILED')
            }
        }

        aborted {
            script {
                notifyTeam('ABORTED - not approved')
            }
        }

        cleanup {
            cleanWs()
        }
    }
}
