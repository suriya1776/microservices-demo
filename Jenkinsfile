pipeline {
    agent any

    stages {
        stage('Find Changed Microservices') {
            steps {
                script {
                    // Get changed files from the last commit (adjust HEAD~1 as needed)
                    def changedFiles = sh(script: "git diff --name-only HEAD~1", returnStdout: true).trim()
                    echo "Changed files:\n${changedFiles}"

                    // List to hold the names of changed microservices (dynamic folder names under src/)
                    def changedMicroservices = []

                    // Iterate over each changed file and look for changes in any folder under src/
                    changedFiles.split("\\n").each { filePath ->
                        if (filePath.startsWith("src/")) {
                            // Tokenize the path using "/" as delimiter
                            def tokens = filePath.tokenize("/")
                            if (tokens.size() > 1) {
                                // The folder immediately under src/ is considered the microservice name
                                changedMicroservices.add(tokens[1])
                            }
                        }
                    }
                    
                    // Remove duplicates
                    changedMicroservices = changedMicroservices.unique()

                    if (changedMicroservices.isEmpty()) {
                        echo "No microservices changed."
                    } else {
                        echo "Changed microservices: ${changedMicroservices.join(', ')}"
                        echo "Number of changed microservices: ${changedMicroservices.size()}"

                        // Find Dockerfile for each changed microservice
                        changedMicroservices.each { service ->
                            def dockerfilePath = "src/${service}/Dockerfile"
                            def exists = sh(script: "test -f ${dockerfilePath} && echo exists || echo missing", returnStdout: true).trim()
                            
                            if (exists == "exists") {
                                echo "Dockerfile found: ${dockerfilePath}"
                            } else {
                                echo "No Dockerfile found in ${service}"
                            }
                        }
                    }
                }
            }
        }
    }
}
