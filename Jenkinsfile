pipeline {
  agent { label 'build' }
  
  environment { 
    registry = "guda654/democicd" 
    registryCredential = 'dockerhub' 
  }

  stages {
    stage('Checkout') {
      steps {
        git credentialsId: 'github', url: 'https://github.com/saiguda654/springboot-build-pipeline.git', branch: 'main'
      }
    }

    stage('Stage I: Build') {
      steps {
        echo "Building Jar Component ..."
        sh "mvn clean package"
      }
    }

    stage('Stage II: Code Coverage') {
      steps {
        echo "Running Code Coverage ..."
        sh "mvn jacoco:report"
      }
    }

    stage('Stage III: SCA') {
      steps { 
        echo "Running Software Composition Analysis using OWASP Dependency-Check ..."
        sh "mvn org.owasp:dependency-check-maven:check"
      }
    }

    stage('Stage IV: SAST') {
      steps { 
        echo "Running Static Application Security Testing using SonarQube Scanner ..."
        withSonarQubeEnv('sonarqube') {
          sh "mvn -X sonar:sonar -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml -Dsonar.dependencyCheck.jsonReportPath=target/dependency-check-report.json -Dsonar.dependencyCheck.htmlReportPath=target/dependency-check-report.html -Dsonar.projectName=saiguda_cicd"
        }
      }
    }

    stage('Stage V: Quality Gates') {
      steps { 
        echo "Running Quality Gates to verify the code quality"
        script {
          timeout(time: 1, unit: 'MINUTES') {
            def qg = waitForQualityGate()
            if (qg.status != 'OK') {
              error "Pipeline aborted due to quality gate failure: ${qg.status}"
            }
          }
        }
      }
    }

    stage('Stage VI: Build Image') {
      steps { 
        echo "Building Docker Image"
        script {
          docker.withRegistry('', registryCredential) { 
            def myImage = docker.build("${registry}:latest")  // Build the image with tag 'latest'
            myImage.push()  // Push the image to Docker registry
          }
        }
      }
    }

    stage('Stage VII: Scan Image') {
      steps { 
        echo "Scanning Image for Vulnerabilities"
        sh "trivy image --scanners vuln --offline-scan ${registry}:latest > trivyresults.txt"
      }
    }

    stage('Stage VIII: Smoke Test') {
      steps { 
        echo "Smoke Testing the Docker Image"
        sh "docker run -d --name smokerun -p 8080:8080 ${registry}:latest"
        sh "sleep 90; ./check.sh"  // Wait for the app to be up and run checks
        sh "docker rm --force smokerun"  // Clean up the container
      }
    }
  }
}
