pipeline {
  agent { label 'build' }

  environment {
    registry = "guda654/democicd"
    registryCredential = 'dockerhub'
    JAVA_8_HOME = "/usr/lib/jvm/java-1.8.0-amazon-corretto" // Java 8 path
    JAVA_11_HOME = "/usr/lib/jvm/java-11-amazon-corretto.x86_64" // Java 11 path
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

        // Use Java 8 for the build stage
        script {
          sh '''
            export JAVA_HOME=$JAVA_8_HOME
            export PATH=$JAVA_HOME/bin:$PATH
            mvn clean package
          '''
        }
      }
    }

    stage('Stage II: Code Coverage') {
      steps {
        echo "Running Code Coverage ..."

        // Use Java 8 for code coverage
        script {
          sh '''
            export JAVA_HOME=$JAVA_8_HOME
            export PATH=$JAVA_HOME/bin:$PATH
            mvn jacoco:report
          '''
        }
      }
    }

    stage('Stage III: SCA') {
      steps {
        echo "Running Software Composition Analysis using OWASP Dependency-Check ..."

        // Use Java 8 for Software Composition Analysis
        script {
          sh '''
            export JAVA_HOME=$JAVA_8_HOME
            export PATH=$JAVA_HOME/bin:$PATH
            mvn org.owasp:dependency-check-maven:check
          '''
        }
      }
    }

    stage('Stage IV: SAST') {
      steps {
        echo "Running Static Application Security Testing using SonarQube Scanner ..."

        // Use Java 11 for SonarQube scanning
        withSonarQubeEnv('sonarqube') {
          script {
            sh '''
              export JAVA_HOME=$JAVA_11_HOME
              export PATH=$JAVA_HOME/bin:$PATH
              mvn -X sonar:sonar \
                -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml \
                -Dsonar.dependencyCheck.jsonReportPath=target/dependency-check-report.json \
                -Dsonar.dependencyCheck.htmlReportPath=target/dependency-check-report.html \
                -Dsonar.projectName=saiguda_cicd
            '''
          }
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
        sh "trivy image --severity HIGH,CRITICAL --offline-scan ${registry}:latest > trivyresults.txt"
      }
    }

    stage('Stage VIII: Smoke Test') {
      steps {
        echo "Smoke Testing the Docker Image"
        
        // Run the container
        sh "docker run -d --name smokerun -p 8080:8080 ${registry}:latest"

        // Ensure check.sh has execute permissions before running it
        sh 'chmod +x ./check.sh'

        // Run the smoke test script
        sh "sleep 90; ./check.sh"

        // Clean up the Docker container
        sh '''
          container_id=$(docker ps -q -f name=smokerun)
          if [ -n "$container_id" ]; then
            docker rm --force smokerun
          fi
        '''
      }
    }
  }
}
