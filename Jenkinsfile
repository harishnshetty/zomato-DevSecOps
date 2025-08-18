pipeline {
    agent any

    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }

    stages {
        stage("Clean Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Git Checkout") {
            steps {
                git branch: 'main', url: 'https://github.com/harishnshetty/zomato-devsecops.git'
            }
        }

        stage("SonarQube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=zomato \
                        -Dsonar.projectKey=zomato '''
                }
            }
        }

        stage("Quality Gate") {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage("Install NPM Dependencies") {
            steps {
                sh "npm install"
            }
        }
        
        // stage("OWASP FS Scan") {
        //     steps {
        //         dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit',
        //                         odcInstallation: 'dp-check'
        //         dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
        //     }
        // }  




        
        // https://nvd.nist.gov/developers/request-an-api-key  [request an API key]
        stage("OWASP FS Scan") {
            steps {
                dependencyCheck additionalArguments: '''
                    --scan ./ 
                    --disableYarnAudit 
                    --disableNodeAudit 
                    --nvdApiKey 788b28b3-e0a8-4fcf-a6c7-a6c4b772d8a7  
                    ''',
                odcInstallation: 'dp-check'

                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }


        stage("Trivy File Scan") {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }

        stage("Build Docker Image") {
            steps {
                script {
                    env.IMAGE_TAG = "harishnshetty/zomato:${BUILD_NUMBER}"

                    // Optional cleanup
                    sh "docker rmi -f zomato ${env.IMAGE_TAG} || true"

                    sh "docker build -t zomato ."
                }
            }
        }

        stage("Tag & Push to DockerHub") {
            steps {
                script {
                    withCredentials([string(credentialsId: 'docker-cred', variable: 'dockerpwd')]) {
                        sh "docker login -u harishnshetty -p ${dockerpwd}"
                        sh "docker tag zomato ${env.IMAGE_TAG}"
                        sh "docker push ${env.IMAGE_TAG}"

                        // Also push latest
                        sh "docker tag zomato harishnshetty/zomato:latest"
                        sh "docker push harishnshetty/zomato:latest"
                    }
                }
            }
        }

       

        stage("Trivy Scan Image") {
            steps {
                script {
                    sh """
                    echo '🔍 Running Trivy scan on ${env.IMAGE_TAG}'

                    # JSON report
                    trivy image -f json -o trivy-report.json ${env.IMAGE_TAG}

                    # HTML report using built-in HTML format
                    trivy image -f table -o trivy-report.txt ${env.IMAGE_TAG}

                    # Fail build if HIGH/CRITICAL vulnerabilities found
                    # trivy image --exit-code 1 --severity HIGH,CRITICAL ${env.IMAGE_TAG} || true
                """
                }
            }
        }


        stage("Deploy to Container") {
            steps {
                script {
                    sh "docker rm -f zomato || true"
                    sh "docker run -d --name zomato -p 80:3000 ${env.IMAGE_TAG}"
                }
            }
        }
    }

     post {
		always {
              
                        
		    emailext(
		        to: 'harishn662@gmail.com',
		        subject: "📢 Jenkins Build Report: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
		        body: """
		            <html>
		                <body style="font-family: Arial, sans-serif; line-height: 1.5;">
		                    <p>📌 <b>This is a Jenkins Prime-Video CICD pipeline status.</b></p>
		                    <p><b>Project:</b> ${env.JOB_NAME}</p>
		                    <p><b>Build Number:</b> ${env.BUILD_NUMBER}</p>
		                    <p><b>Build Status:</b> ${env.buildStatus}</p>
		                    <p><b>Started by:</b> ${env.buildUser}</p>
		                    <p><b>Build URL:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
		                </body>
		            </html>
		        """,
		        mimeType: 'text/html',
		        attachmentsPattern: 'trivyfs.txt'
		    )
		}
    }
}


