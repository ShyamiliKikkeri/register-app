pipeline {
    agent { label 'Jenkins-Agent' }
    tools {
        jdk 'java17'
        maven 'maven3'
    }

    environment {
        APP_NAME       = "register-app-pipeline"
        RELEASE        = "1.0.0"
        DOCKER_USER    = "shyamilikikkeri"
        DOCKER_PASS    = 'dockerhub'
        IMAGE_NAME     = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG      = "${RELEASE}-${BUILD_NUMBER}"
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/ShyamiliKikkeri/register-app'
            }
        }

        stage("Build Application") {
            steps {
                sh "mvn clean package"
            }
        }

        stage("Test Application") {
            steps {
                sh "mvn test"
            }
        }

        stage("SonarQube Analysis") {
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') { 
                        sh "mvn sonar:sonar"
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
                }
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('', DOCKER_PASS) {
                        docker_image = docker.build("${IMAGE_NAME}")
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push("latest")
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh """
                        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image ${IMAGE_NAME}:latest \
                        --no-progress --scanners vuln \
                        --exit-code 0 --severity HIGH,CRITICAL --format table
                    """
                }
            }
        }

        stage("Cleanup Artifacts") {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                    sh "docker rmi ${IMAGE_NAME}:latest || true"
                }
            }
        }

        stage("Trigger CD Pipeline") {
            steps {
                script {
                    withCredentials([string(credentialsId: 'JENKINS_API_TOKEN', variable: 'TOKEN')]) {
                        sh """
                            curl -v -k --user shyamilikikkeri:${TOKEN} \
                            -X POST -H 'cache-control: no-cache' \
                            -H 'content-type: application/x-www-form-urlencoded' \
                            --data 'IMAGE_TAG=${IMAGE_TAG}' \
                            'http://ec2-18-232-110-31.compute-1.amazonaws.com:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token'
                        """
                    }
                }
            }
        }
    }
}
