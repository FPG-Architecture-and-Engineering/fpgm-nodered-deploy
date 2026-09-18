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
            description: 'Unique instance name. Used as the Docker container name and to isolate persisted data. Use a different value for each Node-RED instance on the same host.'
        )

        string(
            name: 'NODE_RED_PORT',
            defaultValue: '1880',
            description: 'Host port published to Node-RED (container port stays 1880). Must be unique per instance on the same host.'
        )

        string(
            name: 'DEPLOY_DIR',
            defaultValue: '/home/jenkins/deploy',
            description: 'Base directory for persistent data. Each instance stores flows at <DEPLOY_DIR>/<APP_NAME>/data.'
        )

        text(
            name: 'EXTRA_VOLUMES',
            defaultValue: '',
            description: 'Optional extra host bind mounts, one per line: /host/path:/container/path[:ro]. Leave empty to skip. /data is reserved for Node-RED instance data.'
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

        stage('Validate and Resolve Data Directory') {
            steps {
                script {
                    def appName = params.APP_NAME?.trim()
                    def deployDir = params.DEPLOY_DIR?.trim()?.replaceAll(/\/+$/, '')

                    if (!appName || !(appName ==~ /[a-zA-Z0-9][a-zA-Z0-9_.-]*/)) {
                        error("APP_NAME must be a valid Docker name (start with alphanumeric; only letters, digits, underscore, dash, or dot). Got: '${params.APP_NAME}'")
                    }

                    if (!deployDir || !deployDir.startsWith('/') || deployDir.contains('..')) {
                        error("DEPLOY_DIR must be an absolute path without '..'. Got: '${params.DEPLOY_DIR}'")
                    }

                    def instanceData = "${deployDir}/${appName}/data"
                    def legacyData = "${deployDir}/data"
                    def deployBaseName = deployDir.tokenize('/').last()

                    def instanceExists = sh(script: "test -d '${instanceData}'", returnStatus: true) == 0
                    def legacyExists = sh(script: "test -d '${legacyData}'", returnStatus: true) == 0

                    // Keep the original single-instance layout when this job still
                    // points DEPLOY_DIR at the old per-app path, e.g.
                    // /home/jenkins/deploy/nodered/data for APP_NAME=nodered.
                    if (instanceExists) {
                        env.NODE_RED_DATA_DIR = instanceData
                    } else if (legacyExists && deployBaseName == appName) {
                        env.NODE_RED_DATA_DIR = legacyData
                        echo "Using existing data directory ${legacyData} (legacy layout)"
                    } else {
                        env.NODE_RED_DATA_DIR = instanceData
                    }

                    def extraMounts = []
                    def extraHosts = []
                    def extraRaw = params.EXTRA_VOLUMES ?: ''
                    extraRaw.split('\n').each { rawLine ->
                        def line = rawLine.trim()
                        if (!line || line.startsWith('#')) {
                            return
                        }
                        // Use a String pattern: slashy /.../ regex cannot contain unescaped '/'.
                        def extraVolumePattern = '^(/[A-Za-z0-9._-]+)+:(/[A-Za-z0-9._-]+)+(:[a-zA-Z0-9,_]+)?$'
                        if (!(line ==~ extraVolumePattern)) {
                            error("Invalid EXTRA_VOLUMES entry '${line}'. Use /host/path:/container/path[:ro]")
                        }
                        def parts = line.split(':')
                        def hostPath = parts[0]
                        def containerPath = parts[1]
                        if (hostPath.contains('..') || containerPath.contains('..')) {
                            error("EXTRA_VOLUMES paths must not contain '..': ${line}")
                        }
                        if (containerPath == '/data') {
                            error('EXTRA_VOLUMES cannot mount over /data; that path is reserved for Node-RED instance data.')
                        }
                        extraMounts << line
                        extraHosts << hostPath
                    }

                    env.APP_NAME = appName
                    env.DEPLOY_DIR = deployDir
                    env.EXTRA_VOLUME_ARGS = extraMounts.collect { "-v ${it}" }.join(' ')
                    env.EXTRA_VOLUME_HOSTS = extraHosts.join('|')

                    echo "Running on Jenkins node: ${env.NODE_NAME}"
                    echo "Using label selector: ${params.TARGET_LABEL}"
                    echo "Instance name: ${appName}"
                    echo "Host port: ${params.NODE_RED_PORT}"
                    echo "Node-RED data directory: ${env.NODE_RED_DATA_DIR}"
                    if (extraMounts) {
                        echo "Extra volumes:\n  ${extraMounts.join('\n  ')}"
                    } else {
                        echo 'Extra volumes: none'
                    }
                }
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
                      -v "${NODE_RED_DATA_DIR}:/mnt/data" \
                      alpine:latest sh -c "
                        mkdir -p /mnt/data &&
                        chown -R 1000:1000 /mnt/data &&
                        chmod -R 755 /mnt/data
                      "

                    if [ -n "${EXTRA_VOLUME_HOSTS}" ]; then
                      IFS='|'
                      for host_dir in ${EXTRA_VOLUME_HOSTS}; do
                        [ -z "${host_dir}" ] && continue
                        if [ ! -e "${host_dir}" ]; then
                          docker run --rm -u root \
                            -v "${host_dir}:/mnt/vol" \
                            alpine:latest sh -c "
                              chown 1000:1000 /mnt/vol &&
                              chmod 755 /mnt/vol
                            "
                        fi
                      done
                    fi
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
                      -v "${NODE_RED_DATA_DIR}:/data" \
                      ${EXTRA_VOLUME_ARGS} \
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
            echo "Deployment completed successfully. Data directory: ${env.NODE_RED_DATA_DIR}"
        }
    }
}
