pipeline {
    agent any
    tools {
       nodejs 'node'
    }

    stages {
        stage('Checkout') {
            steps {
                git(
                    url: 'https://github.com/manish-g0u74m/wanderlust-3-tier-project.git',
                    branch: 'devops'
                )
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('sonar') {
                    script {
                        def scanner = tool name: 'sonar'
                        sh """
                            ${scanner}/bin/sonar-scanner \
                            -Dsonar.projectKey=${env.JOB_NAME} \
                            -Dsonar.projectName=${env.JOB_NAME} \
                            -Dsonar.sources=.
                        """
                    }
                }
            }
        }

        stage('Trivy file system scan') {
            steps {
                sh "trivy fs --format table -o trivy-fs-report.html ."
            }
        }

        stage('Check SonarQube Quality Gate') {
            steps {
                echo 'Waiting for SonarQube Quality Gate result...'
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --nvdApiKey b588dcc5-7433-4160-b160-8d774e5b866', odcInstallation: 'dc'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage("docker Image build") {
            steps {
                echo "Code Build Stage"
                sh "docker build -t wanderlust-frontend:latest -f ./frontend/Dockerfile ./frontend"
                sh "docker build -t wanderlust-backend:latest -f ./backend/Dockerfile ./backend"
            }
        }

        stage("TRIVY image Scane") {
            steps {
                sh "trivy image wanderlust-frontend:latest > trivy-frontend-image.txt"
                sh "trivy image wanderlust-backend:latest > trivy-backend-image.txt"
            }
        }

        stage("Push To DockerHub") {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "dockerHubCreds",
                    usernameVariable: "dockerHubUser",
                    passwordVariable: "dockerHubPass"
                )]) {
                    sh 'echo $dockerHubPass | docker login -u $dockerHubUser --password-stdin'
                    sh "docker image tag wanderlust-frontend:latest ${env.dockerHubUser}/wanderlust-frontend:latest"
                    sh "docker image tag wanderlust-backend:latest ${env.dockerHubUser}/wanderlust-backend:latest"
                    echo "Push wanderlust frontend image"
                    sh "docker push ${env.dockerHubUser}/wanderlust-frontend:latest"
                    echo "Push wanderlust backend image"
                    sh "docker push ${env.dockerHubUser}/wanderlust-backend:latest"
                }
            }
        }

        stage("Deploy on K8s") {
            steps {
                withKubeConfig([credentialsId: 'kube-cred-id']) {
                    sh "kubectl apply -f ./k8s/wl-namespace.yml"
                    sh "kubectl apply -f ./k8s/mongo/."
                    sh "kubectl apply -f ./k8s/frontend/."
                    sh "kubectl apply -f ./k8s/backend/."
                }
            }
        }
    }

    post {
        success {
            emailext(
                to: 'manish.sharma.devops@gmail.com',
                subject: 'Build Successful for Wanderlust CICD App',
                body: 'The build has completed successfully for Wanderlust CICD App.'
            )
        }
        failure {
            emailext(
                to: 'manish.sharma.devops@gmail.com',
                subject: 'Build Failed for Wanderlust CICD App',
                body: 'The build has failed for Wanderlust CICD App. Please check the Jenkins logs for details.'
            )
        }
    }
}
