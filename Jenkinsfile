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
                        echo "No microservices changed."
                        currentBuild.result = 'ABORTED'
                        return
                    } else {
                        echo "Changed microservices: ${env.CHANGED_MICROSERVICES}"
                    }
                }
            }
        }

        stage('Check for Dockerfile') {
            steps {
                script {
                    def validMicroservices = []

                    env.CHANGED_MICROSERVICES.split(',').each { service ->
                        def dockerfilePath = "src/${service}/Dockerfile"
                        def exists = sh(script: "test -f ${dockerfilePath} && echo exists || echo missing", returnStdout: true).trim()

                        if (exists == "exists") {
                            echo "Dockerfile found: ${dockerfilePath}"
                            validMicroservices.add(service)
                        } else {
                            echo "No Dockerfile found in ${service}"
                        }
                    }

                    env.VALID_MICROSERVICES = validMicroservices.join(',')

                    if (validMicroservices.isEmpty()) {
                        echo "No services with Dockerfile found."
                        currentBuild.result = 'ABORTED'
                        return
                    }
                }
            }
        }

        stage('Login to GHCR') {
            steps {
                script {
                    sh """
                        echo '${GHCR_CREDENTIALS_PSW}' | docker login ghcr.io -u '${GHCR_CREDENTIALS_USR}' --password-stdin
                    """
                    echo "Successfully logged in to GHCR"
                }
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                script {
                    env.VALID_MICROSERVICES.split(',').each { service ->
                        def imageName = "ghcr.io/${GHCR_CREDENTIALS_USR}/${service}:latest"
                        def dockerfileDir = "src/${service}"

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
