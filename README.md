# CI/CD Pipeline for Java Application

## Project Overview

- This project demonstrates the implementation of a (CI/CD) pipeline for a basic Java application using Jenkins, SonarQube and Nexus. 
- The goal is to automate the build, testing, and deployment processes, ensuring that changes made to the codebase are automatically validated and deployed to production.

## Selected Approach: Jenkins Pipeline

- For this project, I chose to use Jenkins as the primary CI/CD tool due to its extensive plugin ecosystem, and flexibility in integrating with various tools and services. Jenkins allows for the creation of complex build pipelines that can be easily configured and extended, making it an ideal choice for managing the entire CI/CD process.

- Gitlab would also be a prefered tool.

## Implementation

- I chose to configure virtual machines using Vagrant rather than relying on a cloud provider like AWS for the following reason:
1. **Cost-Effectiveness** : Being a local solution, Vagrant does not incur any ongoing costs.
Although AWS has a free tier, costs can escalate quickly as resources are used.

### 1. Prerequisites

- Before you starting, I ensured I had the following installed and configured:

- **JDK**:  Java Development Kit (version 8 or above). JDK is installed on Nexus and jenkins servers since they both require Java to run. 
- **Maven**: Build tool for Java projects
- **Jenkins**: CI/CD server
- **SonarQube**: Code quality analysis tool
- **Nexus**: Repository manager for storing artifacts

- The following files in my repo contain the scripts I wrote to automate installation of Jenkins, Sonarcube and Nexus:
1. 'install_jenkins.sh'
2. 'install_nexus.sh'
3. 'install_sonarqube.sh'

### 2. Seting Up GitHub Repository and Pushing Local Code 

- **Login to GitHub** 
- **Create a New Repository**
- **Push Existing Local Code to GitHub** 
    - Initialize Git in my Local Project
    - Adding Remote Repository
    - Add and Commit my Code
    - Pushing to GitHub 

### 3. Launching Virtual Machines for Jenkins, Sonarqube and Nexus 

1. **Set Up the Project Directory**
```bash
mkdir jenkins-cicd
cd jenkins-cicd
```

2. **Initialize a Vagrant environment**
```bash
vagrant init
```

3. **Configure the Vagrantfile**
- Refer to Vagrant folder to see my Vagrantfile configuration

4. **Add provisioning scripts for each VM**
-Create Provisioning Scripts:

5. **launching VMs**
```bash
vagrant up
```

6. **Access the Virtual Machines**
- SSH into each VM to verify the installations.
```bash
vagrant ssh jenkins
vagrant ssh sonarqube
vagrant ssh nexus
```

7. **Verify Access in the Browser**
- Access Jenkins at http://192.168.33.10:8080
- Access SonarQube at http://192.168.33.11
- Access Nexus at http://192.168.33.12:8081

### 4. Installing and Configuring Plugins on Jenkins

- **JDK Plugin**: Manages and configures Java versions needed for building and testing Java projects in Jenkins.

- **SonarQube Scanner Plugin**: Integrates SonarQube with Jenkins for code analysis, checking code quality, bugs, and vulnerabilities.

- **OWASP Dependency Check Plugin**: Scans project dependencies for known security vulnerabilities, helping to identify and mitigate risks.

### 5. Installing and Configuring Plugins on Jenkins  
- Logged into Jenkins and navigated to Manage Plugins.
- Install the JDK, SonarQube Scanner, and OWASP Dependency-Check plugins.
- Configured them under Global Tool Configuration by specifying paths and settings.
- Tested with a build job to verify that everything works as expected

### 6. Creating Pipeline

```groovy
pipeline {
    
	agent any
/*	
	tools {
        maven "maven3"
    }
*/	
    environment {
        NEXUS_VERSION = "nexus3"
        NEXUS_PROTOCOL = "http"
        NEXUS_URL = "172.31.40.209:8081"
        NEXUS_REPOSITORY = "vprofile-release"
	NEXUS_REPO_ID    = "vprofile-release"
        NEXUS_CREDENTIAL_ID = "nexuslogin"
        ARTVERSION = "${env.BUILD_ID}"
    }
	
    stages{
        
        stage('BUILD'){
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

	stage('UNIT TEST'){
            steps {
                sh 'mvn test'
            }
        }

	stage('INTEGRATION TEST'){
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }
		
        stage ('CODE ANALYSIS WITH CHECKSTYLE'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage('CODE ANALYSIS with SONARQUBE') {
          
		  environment {
             scannerHome = tool 'sonarscanner4'
          }

          steps {
            withSonarQubeEnv('sonar-pro') {
               sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                   -Dsonar.projectName=vprofile-repo \
                   -Dsonar.projectVersion=1.0 \
                   -Dsonar.sources=src/ \
                   -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                   -Dsonar.junit.reportsPath=target/surefire-reports/ \
                   -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                   -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
            }

            timeout(time: 10, unit: 'MINUTES') {
               waitForQualityGate abortPipeline: true
            }
          }
        }

        stage("Publish to Nexus Repository Manager") {
            steps {
                script {
                    pom = readMavenPom file: "pom.xml";
                    filesByGlob = findFiles(glob: "target/*.${pom.packaging}");
                    echo "${filesByGlob[0].name} ${filesByGlob[0].path} ${filesByGlob[0].directory} ${filesByGlob[0].length} ${filesByGlob[0].lastModified}"
                    artifactPath = filesByGlob[0].path;
                    artifactExists = fileExists artifactPath;
                    if(artifactExists) {
                        echo "*** File: ${artifactPath}, group: ${pom.groupId}, packaging: ${pom.packaging}, version ${pom.version} ARTVERSION";
                        nexusArtifactUploader(
                            nexusVersion: 'nexus3',
                            protocol: http,
                            nexusUrl: '192.168.33.14:8081',
                            groupId: QA,
                            version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                            repository: jenkins_CI-CD_project-repo,
                            credentialsId: nexuslogin,
                            artifacts: [
                                [artifactId: pom.jenkinsprojectapp,
                                classifier: '',
                                file:  target/jenkisproject.war,
                                type: pom.packaging],
                                [artifactId: pom.jenkinsprojectapp,
                                classifier: '',
                                file: "pom.xml",
                                type: "pom"]
                            ]
                        );
                    } 
		    else {
                        error "*** File: ${artifactPath}, could not be found";
                    }
                }
            }
        }


    }


}
```

### 7. Run and Test the Pipeline

- Manually trigger the pipeline by clicking the Build Now button in Jenkins.
- Alternatively, set up the pipeline to automatically run whenever changes are pushed to the repository using webhooks.

### 8. Check Build Artifacts and Reports:
- Review the build artifacts stored in Nexus by accessing Nexus repository.
- View the SonarQube code analysis reports to assess code quality, bugs, vulnerabilities, and code smells. The report can be accessed from the SonarQube dashboard.
- Check the OWASP Dependency-Check report for any vulnerabilities in project dependencies, which can also be viewed in Jenkins or SonarQube.

### 9. Fix Pipeline Errors:
- If any stage of the pipeline fails (e.g., build errors, failed tests, or SonarQube quality gate violations), review the console output logs and reports.
- Identify the root cause of the failure and make necessary changes to the code or pipeline configuration.
- Rerun the pipeline after fixing the issues to verify that everything works correctly.

### 10. Monitoring and Logging
- I did not implement monitoring and logging. However,  Implementing monitoring and logging is used to track application performance and detect issues in production. Tools like Prometheus and Grafana, are used for real-time insights.


