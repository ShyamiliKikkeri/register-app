pipeline {
  agent { label 'Jenkins-Agent' }
  tools {
      jdk 'java17'
      maven 'maven3'
  }
  environment {
      APP_NAME = "register-app-pipeline"
      RELEASE = "1.0.0"
      DOCKER_USER = "shyamilikikkeri"
      DOCKER_PASS = "dockerhub"   // must be Jenkins credentialsId
      IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
      IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
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
              sh 'mvn clean package -DskipTests -T 2C'
          }
      }

      stage("Test Application") {
          steps {
              sh "mvn test"
          }
      }

      stage("SonarQube Analysis"){
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
              timeout(time: 1, unit: 'MINUTES') {
                  waitForQualityGate abortPipeline: true
              }
          }
      }

      stage("Build & Push Docker Image") {
          steps {
              script {
                  docker.withRegistry('https://index.docker.io/v1/', DOCKER_PASS) {
                      def docker_image = docker.build("${IMAGE_NAME}")
                      docker_image.push("${IMAGE_TAG}")
                      docker_image.push('latest')
                  }
              }
          }
      }

      stage("Trivy Scan") {
          steps {
              sh """
                  docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                  aquasec/trivy image ${IMAGE_NAME}:latest \
                  --no-progress --scanners vuln \
                  --exit-code 0 --severity HIGH,CRITICAL --format table
              """
          }
      }
  }
}
