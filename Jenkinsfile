pipeline {
    agent any

    environment {
        GHCR_CREDENTIALS = credentials('GHCR_CREDENTIALS') // Store GHCR credentials in Jenkins
    }

    stages {
        stage('Find Changed Microservices') {
            steps {
                script {
                    def changedFiles = sh(script: "git diff --name-only HEAD~1", returnStdout: true).trim()
                    echo "Changed files:\n${changedFiles}"

                    def changedMicroservices = []

                    changedFiles.split("\\n").each { filePath ->
                        if (filePath.startsWith("src/")) {
                            def tokens = filePath.tokenize("/")
                            if (tokens.size() > 1) {
                                changedMicroservices.add(tokens[1])
                            }
                        }
                    }

                    changedMicroservices = changedMicroservices.unique()
                    env.CHANGED_MICROSERVICES = changedMicroservices.join(',')

                    if (changedMicroservices.isEmpty()) {
                        echo "No microservices changed. Aborting build..."
                        currentBuild.result = 'ABORTED'
                        error("No microservices changed.")
                    } else {
                        echo "Changed microservices: ${env.CHANGED_MICROSERVICES}"
                    }
                }
            }
        }

        stage('Find Dockerfile Paths') {
            steps {
                script {
                    def validMicroservices = [:]

                    env.CHANGED_MICROSERVICES.split(',').each { service ->
                        def dockerfilePath = sh(
                            script: "find src/${service} -type f -name 'Dockerfile' | head -n 1",
                            returnStdout: true
                        ).trim()

                        if (dockerfilePath) {
                            echo "Dockerfile found: ${dockerfilePath}"
                            validMicroservices[service] = dockerfilePath
                        } else {
                            echo "No Dockerfile found in ${service}"
                        }
                    }

                    if (validMicroservices.isEmpty()) {
                        echo "No services with Dockerfile found. Aborting build."
                        currentBuild.result = 'ABORTED'
                        error("No services with Dockerfile found.")
                    }

                    env.VALID_MICROSERVICES = validMicroservices.collect { k, v -> "$k:$v" }.join(',')
                }
            }
        }

        stage('Login to GHCR') {
            when {
                expression { env.VALID_MICROSERVICES?.trim() }
            }
            steps {
                script {
                    sh """
                        echo '${GHCR_CREDENTIALS_PSW}' | docker login ghcr.io -u '${GHCR_CREDENTIALS_USR}' --password-stdin
                    """
                    echo "Successfully logged in to GHCR.."
                }
            }
        }

        stage('Build and Push Docker Image') {
            when {
                expression { env.VALID_MICROSERVICES?.trim() }
            }
            steps {
                script {
                    env.VALID_MICROSERVICES.split(',').each { entry ->
                        def (service, dockerfilePath) = entry.tokenize(":")
                        def imageName = "ghcr.io/${GHCR_CREDENTIALS_USR}/${service}:latest"
                        def dockerfileDir = dockerfilePath.replace('/Dockerfile', '')

                        echo "Building and pushing image for ${service}..."
                        sh """
                            docker build -t ${imageName} ${dockerfileDir}
                            docker push ${imageName}
                        """
                        echo "Successfully pushed ${imageName}"
                    }
                }
            }
        }
    }
}
