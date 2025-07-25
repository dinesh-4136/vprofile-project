pipeline {
    agent any
    
    tools {
        jdk 'jdk-17'
        maven 'maven'
    }
    
    environment {
    NEXUS_VERSION        = "nexus3"
    NEXUS_PROTOCOL       = "http"
    NEXUS_URL            = "35.154.198.189:8081"
    NEXUS_REPOSITORY     = "my-artifact"
    NEXUS_CREDENTIAL_ID = "Nexus-Login"
    ARTVERSION           = "${BUILD_ID}"
    DOCKER_IMAGE_NAME    = "dinesh4136/vprofile"
    DOCKER_IMAGE_TAG     = "${BUILD_NUMBER}"
    }

    stages {
        stage("Git Checkout") {
            steps{
                checkout scmGit(branches: [[name: '*/main']], extensions: [], userRemoteConfigs: [[credentialsId: 'GitHub-Login', url: 'https://github.com/dinesh-4136/vprofile-project.git']])
            }
        }
        stage("Maven Build") {
            steps {
                sh 'mvn clean package'
            }
        }
        stage('Code Analysis with SonarQube') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh 'mvn sonar:sonar'
            	}
                // timeout(time: 10, unit: 'MINUTES') {
                //    waitForQualityGate abortPipeline: true
                // }
            }
        }
        stage('Upload Artifact to Nexus') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: "${NEXUS_VERSION}",
                    protocol: "${NEXUS_PROTOCOL}",
                    nexusUrl: "${NEXUS_URL}",
                    version: "${ARTVERSION}",
                    groupId: "com.vprofile",
                    repository: "${NEXUS_REPOSITORY}",
                    credentialsId: "${NEXUS_CREDENTIAL_ID}",
                    artifacts: [[
                        artifactId: "vprofile",
                        classifier: "",
                        file: "target/vprofile-v2.war",
                        type: "war"
                    ]]
                )
            }
        }
        stage('Docker Image Build') {
            steps {
                echo "Building Docker image: ${}:${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                sh "docker build -t ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} ."
            }
        }
        stage('Docker Image Scan with Trivy') {
            steps {
                echo "Scanning Docker image with Trivy: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                sh '''
                    IMAGE_NAME="${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                    echo "Scanning image: $IMAGE_NAME"
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        -v $HOME/.cache/trivy:/root/.cache/ \
                        -v $WORKSPACE:/app \
                        aquasec/trivy:latest \
                        image --exit-code 0 --severity CRITICAL,HIGH \
                        -f json -o /app/trivy-report.json \
                        "$IMAGE_NAME"
                '''
                sh 'ls -lh trivy-report.json'
            }
        }
        stage('Docker Push to DockerHub') {
            steps {
                script {
                    echo "Logging into Docker Registry and pushing image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                    docker.withRegistry('https://index.docker.io/v1/', 'Docker-Login') {
                        def image = docker.build("${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}")
                        image.push()
                    }
                }
            }
        }
        stage('Deploy to EKS') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'EKS-Login', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                        export AWS_DEFAULT_REGION=ap-south-1

                        echo "Setting up KUBECONFIG for EKS cluster..."
                        aws eks update-kubeconfig --region ap-south-1 --name my-cluster

                        echo "Deploying to Amazon EKS..."
                        sed -i "s|image: .*|image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}|g" k8s/deployment.yaml

                        kubectl apply -f k8s/
                    '''
                }
            }
        }
    }
    post {
        always {
            echo 'Slack Notifications.'
            slackSend(
                channel: '#devops-deployment',
                message: "*${currentBuild.currentResult}* - Job <${env.BUILD_URL}|#${env.JOB_NAME}>"
            )
        }
    }
}
