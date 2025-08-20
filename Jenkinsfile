pipeline {
    agent any

    tools {
        jdk 'jdk17'          // Make sure this matches your Jenkins JDK name
        maven 'maven3'       // Make sure this matches your Jenkins Maven name
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"
        // Docker credentials will be injected via withCredentials
    }

    stages {

        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'git-cred', url: 'https://github.com/devopspractice8/register-app.git'
            }
        }

        stage("Check Maven & Java") {
            steps {
                sh "mvn -version"
                sh "java -version"
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
                    // Replace 'sonarqube-server' with your Jenkins SonarQube server name
                    withSonarQubeEnv('sonar') {
                        sh "mvn sonar:sonar"
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                // Wait for Quality Gate result, fail pipeline if it fails
                waitForQualityGate abortPipeline: true
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    withCredentials([
                        usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')
                    ]) {
                        docker_image = docker.build("${DOCKER_USER}/${APP_NAME}")
                        docker.withRegistry('', "${DOCKER_PASS}") {
                            docker_image.push("${IMAGE_TAG}")
                            docker_image.push('latest')
                        }
                    }
                }
            }
        }

        stage("Trivy Scan") {
            steps {
                script {
                    sh """
                        docker run -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image ${IMAGE_NAME}:${IMAGE_TAG} \
                        --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table
                    """
                }
            }
        }

        stage("Cleanup Artifacts") {
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
                    withCredentials([string(credentialsId: 'JENKINS_API_TOKEN', variable: 'JENKINS_API_TOKEN')]) {
                        sh """
                        curl -v -k --user clouduser:${JENKINS_API_TOKEN} \
                        -X POST \
                        -H 'cache-control: no-cache' \
                        -H 'content-type: application/x-www-form-urlencoded' \
                        --data 'IMAGE_TAG=${IMAGE_TAG}' \
                        'http://ec2-13-232-128-192.ap-south-1.compute.amazonaws.com:8080/job/gitops-register-app-cd/buildWithParameters?token=gitops-token'
                        """
                    }
                }
            }
        }

    } // end of stages

    post {
        failure {
            emailext body: '''${SCRIPT, template="groovy-html.template"}''', 
                     subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Failed", 
                     mimeType: 'text/html', 
                     to: "taufikhshaikh5@gmail.com"
        }
        success {
            emailext body: '''${SCRIPT, template="groovy-html.template"}''', 
                     subject: "${env.JOB_NAME} - Build # ${env.BUILD_NUMBER} - Successful", 
                     mimeType: 'text/html', 
                     to: "taufikhshaikh5@gmail.com"
        }      
    }

}
