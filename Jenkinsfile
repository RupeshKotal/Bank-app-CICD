pipeline {
    agent any
    
    tools {
        maven 'maven3'
    }
    
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        IMAGE_TAG = "v${BUILD_NUMBER}"
    }

    stages {
        stage('Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/RupeshKotal/Bank-app-CICD.git'
            }
        }
        
        stage('Compile') {
            steps {
                sh "mvn compile"
            }
        }
        
        stage('Testing') {
            steps {
                sh "mvn test"
            }
        }
        
        stage('Trivy FS Scan') {
            steps {
                sh "trivy fs --format table -o fs-report.html ."
            }
        }
        
        stage('Sonar Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=gcbank \
                        -Dsonar.projectName=gcbank \
                        -Dsonar.java.binaries=target
                    '''
                }
            }
        }
        
        stage('Quality Gate Check') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }
        
        stage('Build') {
            steps {
                sh "mvn package"
            }
        }
        
        stage('Publish To Nexus') {
            steps {
                withMaven(globalMavenSettingsConfig: 'maven-config', maven: 'maven3', traceability: true) {
                    sh "mvn deploy"
                }
            }
        }
        
        stage('Docker Image Build & Tag') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker build -t ruxs123/bankapp:$IMAGE_TAG ."
                    }
                }
            }
        }
        
        stage('Scan Image') {
            steps {
                sh "trivy image --format table -o image-report.html ruxs123/bankapp:$IMAGE_TAG"
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred') {
                        sh "docker push ruxs123/bankapp:$IMAGE_TAG"
                    }
                }
            }
        }
        
        stage('Update Manifest File in Mega-Project-CD') {
            steps {
                script {
                    // Clean workspace before starting
                    cleanWs()

                    sh '''
                            # Clone the Mega-Project-CD repository
                            git clone https://github.com/RupeshKotal/Ultimate-mega-project.git
                            
                            # Update the image tag in the manifest.yaml file
                            cd Ultimate-mega-project/Mega-Project-CD-main/Mega-Project-CD-main
                            sed -i "s|ruxs123/bankapp:.*|ruxs123/bankapp:${IMAGE_TAG}|" Manifest/manifest.yaml
                            
                            # Confirm changes
                            echo "Updated manifest file contents:"
                            cat Manifest/manifest.yaml
                            
                            # Commit and push the changes
                            git config user.name "Jenkins"
                            git config user.email "jenkins@example.com"
                            git add Manifest/manifest.yaml
                            git commit -m "Update image tag to ${IMAGE_TAG}"
                            git push origin main
                        '''
                    
                }
            }
        }
    }
    
    
post {
    always {
        script {
            def jobName = env.JOB_NAME
            def buildNumber = env.BUILD_NUMBER
            def pipelineStatus = currentBuild.result ?: 'UNKNOWN'
            def bannerColor = pipelineStatus.toUpperCase() == 'SUCCESS' ? 'green' : 'red'

            def body = """
                <html>
                <body>
                <div style="border: 4px solid ${bannerColor}; padding: 10px;">
                <h2>${jobName} - Build ${buildNumber}</h2>
                <div style="background-color: ${bannerColor}; padding: 10px;">
                <h3 style="color: white;">Pipeline Status: ${pipelineStatus.toUpperCase()}</h3>
                </div>
                <p>Check the <a href="${BUILD_URL}">console output</a>.</p>
                </div>
                </body>
                </html>
            """

            emailext (
                subject: "${jobName} - Build ${buildNumber} - ${pipelineStatus.toUpperCase()}",
                body: body,
                to: '567adddi.jais@gmail.com',
                from: 'jenkins@devopsshack.com',
                replyTo: 'jenkins@devopsshack.com',
                mimeType: 'text/html',
               
            )
        }
    }
}
}
