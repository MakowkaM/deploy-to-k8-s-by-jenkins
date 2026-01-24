pipeline {
    agent { label 'Jenkins-Agent' }

    tools {
        jdk 'Java21'
        maven 'Maven3'
    }

	triggers {
        GenericTrigger(
            token: 'gitops-token'
        )
    }
    
    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        DOCKER_USER = "majkelowski675"
        DOCKER_PASS = "dockerhub"
        IMAGE_NAME = "${DOCKER_USER}" + "/" + "${APP_NAME}"
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
                git branch: 'main',
                    credentialsId: 'github',
                    url: 'https://github.com/MakowkaM/deploy-to-k8-s-by-jenkins'
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
                withSonarQubeEnv('sonarqube-server') {
                    sh "mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar"
                }
            }
        }

        
        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false
                }
            }
        }
        

        stage("Build and Push Docker Image") {
            steps {
                script {
                    docker.withRegistry('', DOCKER_PASS) {
                        docker_image = docker.build "${IMAGE_NAME}"
                    }
                    docker.withRegistry('', DOCKER_PASS) {
                        docker_image.push("${IMAGE_TAG}")
                        docker_image.push('latest')
                    }
                }
            }
        }

        stage("Trivy Scan") {
           steps {
               script {
	            sh ('docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image majkelowski675/register-app-pipeline:latest --no-progress --scanners vuln  --exit-code 0 --severity HIGH,CRITICAL --format table')
               }
           }
        }

        stage('Cleanup Artifacts') {
            steps {
                script {
                    sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG}"
                    sh "docker rmi ${IMAGE_NAME}:latest"
                }
            }
        }

		stage("Trigger CD Pipeline") {
			steps {
				script {
					sh "curl -v -k --user clouduser:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' 'ec2-ec2-34-229-16-53.compute-1.amazonaws.com:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token'"
				}
			}
		}
    }
}
