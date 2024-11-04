## Install Plugins in Jenkins

1. **Eclipse Temurin Installer**:
   - This plugin enables Jenkins to automatically install and configure the Eclipse Temurin JDK (formerly known as AdoptOpenJDK).
   - To install, go to Jenkins dashboard -> Manage Jenkins -> Manage Plugins -> Available tab.
   - Search for "Eclipse Temurin Installer" and select it.
   - Click on the "Install without restart" button.

2. **Pipeline Maven Integration**:
   - This plugin provides Maven support for Jenkins Pipeline.
   - It allows you to use Maven commands directly within your Jenkins Pipeline scripts.
   - To install, follow the same steps as above, but search for "Pipeline Maven Integration" instead.

3. **Config File Provider**:
   - This plugin allows you to define configuration files (e.g., properties, XML, JSON) centrally in Jenkins.
   - These configurations can then be referenced and used by your Jenkins jobs.
   - Install it using the same procedure as mentioned earlier.

4. **SonarQube Scanner**:
   - SonarQube is a code quality and security analysis tool.
   - This plugin integrates Jenkins with SonarQube by providing a scanner that analyzes code during builds.
   - You can install it from the Jenkins plugin manager as described above.

5. **Kubernetes CLI**:
   - This plugin allows Jenkins to interact with Kubernetes clusters using the Kubernetes command-line tool (`kubectl`).
   - It's useful for tasks like deploying applications to Kubernetes from Jenkins jobs.
   - Install it through the plugin manager.

6. **Kubernetes**:
   - This plugin integrates Jenkins with Kubernetes by allowing Jenkins agents to run as pods within a Kubernetes cluster.
   - It provides dynamic scaling and resource optimization capabilities for Jenkins builds.
   - Install it from the Jenkins plugin manager.

7. **Docker**:
   - This plugin allows Jenkins to interact with Docker, enabling Docker builds and integration with Docker registries.
   - You can use it to build Docker images, run Docker containers, and push/pull images from Docker registries.
   - Install it from the plugin manager.

8. **Docker Pipeline Step**:
   - This plugin extends Jenkins Pipeline with steps to build, publish, and run Docker containers as part of your Pipeline scripts.
   - It provides a convenient way to manage Docker containers directly from Jenkins Pipelines.
   - Install it through the plugin manager like the others.

After installing these plugins, you may need to configure them according to your specific environment and requirements. This typically involves setting up credentials, configuring paths, and specifying options in Jenkins global configuration or individual job configurations. Each plugin usually comes with its own set of documentation to guide you through the configuration process.

## Configure Above Plugins in Jenkins 

## Pipeline 

```groovy

pipeline {
    agent any
    
    tools {
        jdk 'jdk17'
        maven 'maven3'
       
    }

    environment {
        SCANNER_HOME= tool 'sonar-scanner'
        SONARQUBE_SERVER = 'sonar'  
        GITHUB_REPO = 'https://github.com/samiouerfelli/DEVOPS_PROJECT_G2.git'
        SONARQUBE_TOKEN = credentials('sonar-token')
        GITHUB_TOKEN = 'git-cred'
        GITHUB_BRANCH = 'AhmedDridi_5ARCTIC1_G2'
        DOCKERHUB_CREDENTIALS = "docker-cred" 
        DOCKER_IMAGE_NAME = 'hamouda1412/foyerproject'
        NEXUS_REPO_ID = 'maven-releases' // Name of the raw Nexus repository
        NEXUS_URL = '172.19.132.87:8081' // Nexus server URL'
        NEXUS_CREDENTIALS_ID = 'nexus-cred'
    
    }




    stages {
        

        
        
        stage('Clean Workspace') {
            steps {
                script {
                    try {
                        echo 'Cleaning workspace...'
                        cleanWs()
                    } catch (Exception e) {
                        echo "Error in Clean Workspace stage: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }

        stage('Git Checkout') {
            steps {
                script {
                    try {
                        echo 'Checking out Git repository...'
                        git branch: "${GITHUB_BRANCH}", credentialsId: "${GITHUB_TOKEN}", url: "${GITHUB_REPO}"
                    } catch (Exception e) {
                        echo "Error in Git Checkout stage: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }
        
        stage('Maven Clean') {
            steps {
                script {
                    try {
                        echo 'Running mvn clean...'
                        sh 'mvn clean'
                    } catch (Exception e) {
                        echo "Error in Maven Clean stage: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }

        stage('Compile') {
            steps {
                script {
                    try {
                        echo 'Compiling source code...'
                        sh 'mvn compile'
                    } catch (Exception e) {
                        echo "Error in Compile stage: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }
        
        stage('MySQL Container start') {
            steps {
                script {
                    sh 'docker compose up -d'
                    sleep 10
                }
            }
        }
        
        stage('Test') {
            steps {
                script {
                    try {
                        echo 'Running tests...'
                        sh 'mvn test org.jacoco:jacoco-maven-plugin:report'
                    } catch (Exception e) {
                        echo "Error in Test stage: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }
      
        stage('OWASP Scan') {
            steps {
                script {
                    try {
                        echo 'Starting OWASP dependency check...'
                        dependencyCheck additionalArguments: ''' 
                            -o './'
                            -s './'
                            -f 'ALL' 
                            --prettyPrint''', odcInstallation: 'owaspDC'
                        dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                    } catch (Exception e) {
                        echo "Error in OWASP Scan stage: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }
        
        stage('File System Scan') {
            steps {
                script {
                    try {
                        echo 'Scanning file system with Trivy...'
                        sh "trivy fs --format table -o trivy-fs-report.html ."
                    } catch (Exception e) {
                        echo "Error in File System Scan stage: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                script {
                    try {
                        echo 'Starting SonarQube analysis...'
                        withSonarQubeEnv("${SONARQUBE_SERVER}") {
                            sh '''
                                $SCANNER_HOME/bin/sonar-scanner \
                                -Dsonar.projectName=FoyerProject \
                                -Dsonar.projectKey=FoyerProject \
                                -Dsonar.sources=src/main/java \
                                -Dsonar.java.binaries=target/classes \
                                -Dsonar.exclusions=**/test/**,**/resources/**,**/*.spec.java
                            '''
                        }
                    } catch (Exception e) {
                        echo "Error in SonarQube Analysis stage: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                script {
                    try {
                        echo 'Waiting for SonarQube Quality Gate...'
                        waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'  /// check
                    } catch (Exception e) {
                        echo "Error in Quality Gate stage: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }
        
        stage('MySQL Container stop') {
            steps {
                script {
                    sh 'docker compose down'
                }
            }
        }
        
        stage('Build') {
            steps {
                script {
                    try {
                        echo 'Building project package...'
                        sh 'mvn package -DskipTests=true'
                    } catch (Exception e) {
                        echo "Error in Build stage: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }
        
       
        
      stage('Publish To Nexus') {
            steps {
                script {
                    try {
                        echo 'Publishing package and reports to Nexus...'
                        
                        // Publish JAR and other Maven artifacts
                        withMaven(globalMavenSettingsConfig: 'global-settings', jdk: 'jdk17', maven: 'maven3', traceability: true) {
                            sh 'mvn deploy -DskipTests=true'
                        }

                        // Publish reports to Nexus using nexusArtifactUploader
                         nexusArtifactUploader(
                        nexusVersion: 'nexus3',
                        protocol: 'http',
                        nexusUrl: "${NEXUS_URL}",
                        groupId: 'tn.esprit.tp-foyer',
                        version: '1.0', // Update this as per your versioning strategy
                        repository: "${NEXUS_REPO_ID}",
                        credentialsId: "${NEXUS_CREDENTIALS_ID}",
                        artifacts: [
                            [
                                artifactId: 'dependency-check-report',
                                file: "${env.WORKSPACE}/dependency-check-report.xml",
                                type: 'xml',
                                classifier: '',
                                version: '1.0'
                            ],
                            [
                                artifactId: 'trivy-fs-report',
                                file: "${env.WORKSPACE}/trivy-fs-report.html",
                                type: 'html',
                                classifier: '',
                                version: '1.0'
                            ]
                        ]
                    )

                    } catch (Exception e) {
                        echo "Error in Publish To Nexus stage: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }
        
        stage('Build & Tag Docker Image') {
            steps {
                script {
                    try {
                        echo 'Building and tagging Docker image...'
                        withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') { /// check 
                        
                            sh "docker build -t hamouda1412/foyerproject-ahmeddridi:$BUILD_ID ."
                            sh "docker image tag hamouda1412/foyerproject-ahmeddridi:$BUILD_ID hamouda1412/foyerproject-ahmeddridi:latest"
                        }
                    } catch (Exception e) {
                        echo "Error in Build & Tag Docker Image stage: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }
        
        stage('Trivy Cache Clean') {
            steps {
                script {
                    try {
                        echo 'Cleaning Trivy cache...'
                        sh 'trivy clean --java-db'
                    } catch (Exception e) {
                        echo "Error in Trivy Cache Clean stage: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }
        
        stage('Docker Image Scan') {
            steps {
                script {
                    try {
                        echo 'Scanning Docker image with Trivy...'
                        sh 'trivy image --format table -o trivy-image-report.html hamouda1412/foyerproject_ahmeddridi:latest'
                    } catch (Exception e) {
                        echo "Error in Docker Image Scan stage: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    try {
                        echo 'Pushing Docker image to registry...'
                        withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') { //check 
                            sh 'docker push hamouda1412/foyerproject-ahmeddridi:$BUILD_ID'
                            sh 'docker push hamouda1412/foyerproject-ahmeddridi:latest'
                        }
                    } catch (Exception e) {
                        echo "Error in Push Docker Image stage: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }
        
        
        
        // pushh to nexus 
        
        stage('Deploy To Kubernetes') {
            steps {
                script {
                    try {
                        echo 'Deploying to Kubernetes...'
                        withKubeConfig(caCertificate: '', clusterName: 'kind-kind', credentialsId: 'k8-cred', namespace: 'webapps', serverUrl: 'https://127.0.0.1:34063') {
                            sh 'kubectl apply -f deployment-service.yaml -n webapps' 
                            sleep 30
                        }
                    } catch (Exception e) {
                        echo "Error in Deploy To Kubernetes stage: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    try {
                        echo 'Verifying Kubernetes deployment...'
                        withKubeConfig(caCertificate: '', clusterName: 'kind-kind', credentialsId: 'k8-cred', namespace: 'webapps', serverUrl: 'https://127.0.0.1:34063') {
                            sh 'kubectl get pods -n webapps'
                            sh 'kubectl get svc -n webapps'
                        }
                    } catch (Exception e) {
                        echo "Error in Verify Deployment stage: ${e.getMessage()}"
                        throw e
                    }
                }
            }
        }
        
        
        //zap 
        
        // nodeport exporter 
    }
  
    post {
        always {
             sendSlackNotifcation()
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
                    to: 'ahmed.dridi04@gmail.com',
                    from: 'jenkins@example.com',
                    replyTo: 'jenkins@example.com',
                    mimeType: 'text/html',
                    attachmentsPattern: 'trivy-image-report.html'
                )
            }
        }
    }
}


def sendSlackNotifcation()
{
    if ( currentBuild.currentResult == "SUCCESS" ) {
        buildSummary = "Job_name: ${env.JOB_NAME}\n Build_id: ${env.BUILD_ID} \n Status: *SUCCESS*\n Build_url: ${BUILD_URL}\n Job_url: ${JOB_URL} \n"
        slackSend( channel: "#devops-project", token: 'slack-cred', color: 'good', message: "${buildSummary}")
    }
    else {
        buildSummary = "Job_name: ${env.JOB_NAME}\n Build_id: ${env.BUILD_ID} \n Status: *FAILURE*\n Build_url: ${BUILD_URL}\n Job_url: ${JOB_URL}\n  \n "
        slackSend( channel: "#devops-project", token: 'slack-cred', color : "danger", message: "${buildSummary}")
    }
}
```
