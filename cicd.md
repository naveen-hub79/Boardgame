#### Here's the Jenkins Pipeline script with added comments explaining the purpose of each stage, the tools, and the required configurations:


pipeline {
    // Specify the agent to run the pipeline. 'any' means it can run on any available agent.
    agent any
    
    // Define the tools needed for the pipeline.
    // these tools jdk, maven , sonar-scanner need to be installed in manage jenkins->tools after installing their required plugins
    // the required plugins fro jdk, maven  and sonar-scanner are 
    // 1. Eclipse Temurin Installer
    // 2. Pipeline Maven Integration
    // 3. maven integration
    // 4. SonarQube Scanner



    tools {
        // Use JDK 17 for this pipeline.
        jdk 'jdk17'
        // Use Maven 3 for this pipeline.
        maven 'maven3'
    }

    // sonar-scanner tool used as part of environment and sonar server need to be configured in manage jenkins->systems along with url and credentials
    environment {
        // Define the environment variable for the SonarQube scanner home directory.
        SCANNER_HOME = tool 'sonar-scanner'
    }
    
    stages {
        stage('Git Checkout') {
            // This stage checks out the code from the Git repository.
            steps {
                // Checkout the code from the specified Git repository and branch.
                git branch: 'main', url: 'https://github.com/naveen-hub79/Boardgame.git'
            }
        }

        stage('Compile') {
            // This stage compiles the project using Maven.
            steps {
                // Run the Maven 'compile' goal.
                sh "mvn compile"
            }
        }

        stage('Maven Test') {
            // This stage runs the unit tests using Maven.
            steps {
                // Run the Maven 'test' goal.
                sh "mvn test"
            }
        }
        // trivy need to be installed on jenkins machines and added to path as pre-requisiste
        stage('Scan File System using Trivy') {
            // This stage scans the file system for vulnerabilities using Trivy.
            steps {
                // Run the Trivy scan and output the results to an HTML report.
                sh "trivy fs --format table -o trivy-fs.report.html ."
            }
        }
         // sonar server was configured in manage jenkins-systems with url and credentials(token) as pre-requisiste
         //this step will generate reports and upload them to sonar server
        stage('SonarQube Analysis') {
            // This stage performs static code analysis using SonarQube.
            steps {
                // Run the SonarQube scanner with the specified project settings.
                withSonarQubeEnv('sonar') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=BoardGame -Dsonar.projectKey=BoardGame \
                            -Dsonar.java.binaries=. '''
                 }
            }
        } 
        // pre-requisite: jenkins webhook need to be pre-created to tell jenkins about quality gate status
        // abortPipeline should be true if you wanted to stop pipeline on gate failure
        stage('Quality Gate Checking') {
            // This stage waits for the quality gate result from SonarQube.
            steps {
                script {
                    // Check the quality gate status and decide whether to proceed or not.
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-token'
                }
            }
        }

        stage('Build Application') {
            // This stage packages the application using Maven.
            steps {
                // Run the Maven 'package' goal.
                sh "mvn package"
            }
        }
        // nexus repositories like maven-release and maven-snashots need to be added in pom.xml
        // plugin: Config File Provider is required to generate setting.xml in jenkins to pass nexus credentials
        // manage jenkins-> managed files and edit servers section to add nexus credentials to publish artifacts
        stage('Publish to Nexus') {
            // This stage publishes the built artifact to Nexus.
            steps {
                withMaven(globalMavenSettingsConfig: 'Global-Settings', jdk: 'jdk17', maven: 'maven3', mavenSettingsConfig: '', traceability: true) {
                    // Run the Maven 'deploy' goal to publish the artifact to Nexus.
                    sh "mvn clean deploy"
                }
            }
        }
        // plugins: Docker and Docker Pipeline Step
        // configure docker credentials
        // docker installed in tools using manage jenkins-> tools
        stage('Build and Tag Docker Image') {
            // This stage builds and tags the Docker image.
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        // Build the Docker image and tag it with the specified name.
                        sh "docker build -t naveensmily79/boardgame:latest ."
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            // This stage scans the Docker image for vulnerabilities using Trivy.
            steps {
                // Run the Trivy scan on the Docker image and output the results to an HTML report.
                sh "trivy image --format table -o trivy-im-report.html naveensmily79/boardgame:latest"
            }
        }

        stage('Push Docker Image') {
            // This stage pushes the Docker image to the Docker registry.
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker-cred', toolName: 'docker') {
                        // Push the Docker image to the specified Docker registry.
                        sh "docker push naveensmily79/boardgame:latest"
                    }
                }
            }
        }
        // Plugins: kubernets CLI and Kubernetes
        // Service account with right permissions need to be created and generate token and add in global credentials
        stage('Deploy to Kubernetes') {
            // This stage deploys the application to a Kubernetes cluster.
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://<api endpoint>:6443') {
                    // Apply the Kubernetes deployment and service configurations.
                    sh "kubectl apply -f deployment-service.yaml"
                }
            }
        }

        stage('Validate Deployment') {
            // This stage validates the deployment by checking the status of the pods and services.
            steps {
                withKubeConfig(caCertificate: '', clusterName: 'kubernetes', contextName: '', credentialsId: 'k8-cred', namespace: 'webapps', restrictKubeConfigAccess: false, serverUrl: 'https://<api endpoint>:6443') {
                    // Get the status of the pods in the specified namespace.
                    sh "kubectl get pods -n webapps"
                    // Get the status of the services in the specified namespace.
                    sh "kubectl get svc -n webapps"
                }
            }
        }        
    }
}


## Tools and Plugins Required
#### Jenkins Tools Configuration:

JDK: Ensure JDK 17 is installed and configured in Jenkins.
Maven: Ensure Maven 3 is installed and configured in Jenkins.
Sonar Scanner: Ensure Sonar Scanner is installed and configured in Jenkins.
#### SonarQube:

A SonarQube server instance.
SonarQube token for authentication.
Configure a SonarQube server in Jenkins under Manage Jenkins -> Configure System -> SonarQube Servers.
#### Trivy:

Trivy installed on the Jenkins agent to scan the file system and Docker images.
#### Nexus:

Nexus repository configured to host Maven artifacts.
Credentials to deploy artifacts to Nexus.
#### Docker:

Docker installed on the Jenkins agent.
Docker registry credentials for pushing images.
#### Kubernetes:

Kubernetes cluster configuration.
Kubernetes credentials configured in Jenkins.

By following the above configurations and comments, you should be able to set up a comprehensive CI/CD pipeline that integrates with SonarQube, Trivy, Docker, Nexus, and Kubernetes.
