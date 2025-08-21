pipeline {
    agent any

    tools {
        jdk 'jdk17'          // Make sure this matches your Jenkins JDK name
        maven 'maven3'       // Make sure this matches your Jenkins Maven name
    }

    environment {
        APP_NAME = "register-app-pipeline"
        RELEASE = "1.0.0"
        // DOCKER_USER and DOCKER_PASS will be injected via withCredentials
        JENKINS_API_TOKEN = credentials("JENKINS_API_TOKEN")
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
                    withSonarQubeEnv('sonarqube-server') {
                        sh "mvn clean verify sonar:sonar -Dsonar.projectKey=register-app -Dsonar.host.url=http://15.206.92.240:9000"
                    }
                }
            }
        }

        stage("Quality Gate") {
            steps {
                waitForQualityGate abortPipeline: true
            }
        }

        stage("Build & Push Docker Image") {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        def IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
                        def IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"

                        def docker_image = docker.build("${IMAGE_NAME}:${IMAGE_TAG}")
                        docker.withRegistry('https://index.docker.io/v1/', 'docker-cred') {
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
                    withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        def IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
                        def IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"

                        sh """
                          docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
                          aquasec/trivy image ${IMAGE_NAME}:${IMAGE_TAG} \
                          --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table
                        """
                    }
                }
            }
        }

        stage("Cleanup Artifacts") {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-cred', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        def IMAGE_NAME = "${DOCKER_USER}/${APP_NAME}"
                        def IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}"

                        sh "docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true"
                        sh "docker rmi ${IMAGE_NAME}:latest || true"
                    }
                }
            }
        }

      stage("Trigger CD Pipeline") {
            steps {
                script {
                     sh "curl -v -k --user clouduser:${JENKINS_API_TOKEN} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'IMAGE_TAG=${IMAGE_TAG}' 'ec2-3-110-77-94.ap-south-1.compute.amazonaws.com:8080/job/gitops-register-app/buildWithParameters?token=gitops-token'"
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
