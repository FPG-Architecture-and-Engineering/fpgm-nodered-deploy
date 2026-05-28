pipeline {

    agent {
        label "${params.TARGET_LABEL}"
    }

    parameters {

        choice(
            name: 'TARGET_LABEL',
            choices: [
                'docker && phdcpldev02',
                'docker && payment-dev',
                'docker && payment-platform-prod'
            ],
            description: 'Target deployment server'
        )

        string(
            name: 'APP_NAME',
            defaultValue: 'nodered',
            description: 'Docker container name'
        )

        string(
            name: 'NODE_RED_PORT',
            defaultValue: '1880',
            description: 'Node-RED port'
        )

        string(
            name: 'DEPLOY_DIR',
            defaultValue: '/home/jenkins/deploy/nodered',
            description: 'Persistent data directory'
        )

        string(
            name: 'IMAGE_TAG',
            defaultValue: 'latest',
            description: 'Docker image tag'
        )

        text(
            name: 'NODE_RED_ENV',
            defaultValue: '''\
TZ=Asia/Manila
''',
            description: 'Environment variables'
        )
    }

    environment {
        IMAGE_NAME = 'fpg-nodered'
    }

    stages {

        stage('Show Target') {
            steps {
                echo "Running on Jenkins node: ${env.NODE_NAME}"
                echo "Using label selector: ${params.TARGET_LABEL}"
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                '''
            }
        }

        stage('Prepare Persistent Storage') {
            steps {
                sh '''
                    docker run --rm -u root \
                      -v ${DEPLOY_DIR}:/mnt \
                      alpine:latest sh -c "
                        mkdir -p /mnt/data &&
                        chown -R 1000:1000 /mnt/data &&
                        chmod -R 755 /mnt/data
                      "
                '''
            }
        }

        stage('Generate Env File') {
            steps {
                writeFile file: '.env', text: """
${NODE_RED_ENV}
"""
            }
        }

        stage('Stop Existing Container') {
            steps {
                sh '''
                    docker rm -f ${APP_NAME} || true
                '''
            }
        }

        stage('Deploy Node-RED') {
            steps {
                sh '''
                    docker run -d \
                      --name ${APP_NAME} \
                      --restart unless-stopped \
                      -p ${NODE_RED_PORT}:1880 \
                      --env-file .env \
                      -v ${DEPLOY_DIR}/data:/data \
                      ${IMAGE_NAME}:${IMAGE_TAG}
                '''
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    docker ps --filter name=${APP_NAME}

                    sleep 5

                    curl -I http://localhost:${NODE_RED_PORT} || true
                '''
            }
        }
    }

    post {

        failure {
            sh '''
                docker logs --tail=100 ${APP_NAME} || true
            '''
        }

        success {
            echo 'Deployment completed successfully.'
        }
    }
}
