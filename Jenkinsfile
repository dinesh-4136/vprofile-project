pipeline {
    agent any
    
    tools {
        jdk 'jdk-17'
        maven 'maven'
    }
    
    environment {
        NEXUS_VERSION        = "nexus3"
        NEXUS_PROTOCOL       = "http"
        NEXUS_URL            = "34.93.214.142:8081"
        NEXUS_REPOSITORY     = "artifact-hub"
        NEXUS_CREDENTIAL_ID  = "Nexus-Cred"
        ARTVERSION           = "${BUILD_ID}"
        DOCKER_IMAGE_NAME    = "dinesh4136/vprofile"
        DOCKER_IMAGE_TAG     = "${BUILD_NUMBER}"
    }
    
    stages {
        stage('Clean Workspace') {
            steps {
                sh '''
                    echo "Cleaning up Docker and workspace..."
                    docker system prune -af || true
                    rm -rf /tmp/trivy* || true
                    df -h
                '''
            }
        }
        stage("Git Checkout") {
            steps {
                echo "Pulling source code from Github"
                checkout scmGit(branches: [[name: '*/gcp-deploy']], extensions: [], userRemoteConfigs: [[credentialsId: 'GitHub-Cred', url: 'https://github.com/dinesh-4136/vprofile-project.git']])
            }
        }
        stage("Maven Build") {
            steps {
                sh 'mvn clean package'
            }
        }
        stage("Code Analysis with Sonar") {
            steps {
                withSonarQubeEnv ('sonar') {
                    sh 'mvn sonar:sonar'
                }
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    /*    stage('Upload Artifact to Nexus') {
            steps {
                nexusArtifactUploader(
                    nexusVersion: "${NEXUS_VERSION}",
                    protocol: "${NEXUS_PROTOCOL}",
                    nexusUrl: "${NEXUS_URL}",
                    version: "${ARTVERSION}",
                    groupId: "com",
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
    */
        stage("Docker Image Build") {
            steps {
                echo "Building Docker Image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                sh "docker build -t ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG} ."
            }
        }
        stage("Docker Image Scan with Trivy") {
            steps {
                echo "Scanning Docker image with Trivy: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                sh """
                docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                  -v \$(pwd):/root/reports aquasec/trivy:latest image \
                  --exit-code 0 --severity CRITICAL,HIGH \
                  --format json -o /root/reports/trivy-report.json ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}
                """
                sh 'ls -lh trivy-report.json'
            }
        }
        stage('Docker Push to DockerHub') {
            steps {
                script {
                    echo "Logging into Docker Registry and pushing image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"
                    docker.withRegistry('https://index.docker.io/v1/', 'DockerHub-Cred') {
                        def image = docker.build("${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}")
                        image.push()
                    }
                }
            }
        }
        stage('Deploy to GKE') {
            steps {
                withCredentials([file(credentialsId: 'gcp-service-account-key', variable: 'GCLOUD_KEY')]) {
                    sh '''
                        echo "Authenticating with GCP..."
                        gcloud auth activate-service-account --key-file=$GCLOUD_KEY
                        gcloud config set project premium-arc-464115-m0
                        gcloud config set compute/zone asia-south1
                        gcloud container clusters get-credentials my-cluster

                        echo "Updating Kubernetes manifests..."
						sed -i "s|image: .*|image: ${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}|g" k8s/deployment.yaml

                        echo "Applying Kubernetes manifests..."
                        kubectl apply -f k8s/
                    '''
                }
            }
        }
    }
    post {
        always {
            archiveArtifacts artifacts: 'trivy-report.json'
        }
    }
}
