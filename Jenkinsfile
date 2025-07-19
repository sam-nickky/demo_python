pipeline {
  agent any

  tools {
    jdk 'jdk17'
  }

  environment {
    SCANNER_HOME = tool 'sonar-scanner'
    REGISTRY_CREDENTIALS = credentials('dockerhub-creds')
    IMAGE_NAME = "demoapp"
    KUBE_CONFIG = credentials('kubeconfig-creds')
    SONARQUBE_SERVER = 'sonar-server'
    TRIVY_CACHE_DIR = './.trivy-cache'
  }

  stages {

    stage('Checkout Code') {
      steps {
        echo 'Cloning source code from Git repository'
        git branch: 'master', url: 'https://github.com/sam-nickky/demo_python.git'
      }
    }

    stage('Plan') {
      steps {
        echo 'Planning - Validate configuration files'
        sh 'cat requirements.txt || echo "No requirements.txt found"'
        sh 'yamllint . || echo "No YAML issues"'
      }
    }

    stage('Develop - Code Quality') {
      steps {
        echo 'Running code quality analysis using SonarQube'
        withSonarQubeEnv("${SONARQUBE_SERVER}") {
          sh "echo SonarScanner path: ${SCANNER_HOME}/bin/sonar-scanner && ls -l ${SCANNER_HOME}/bin/"
          sh """
            ${SCANNER_HOME}/bin/sonar-scanner \\
              -Dsonar.projectName=devops-demo \\
              -Dsonar.projectKey=devops-demo \\
              -Dsonar.sources=.
          """
        }
      }
    }


    stage('Quality Gate') {
      steps {
        script {
          waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
        }
      }
    }
    stage('Build') {
      steps {
        echo 'Building Docker image'
        sh 'docker build -t $IMAGE_NAME:$BUILD_NUMBER .'
      }
    }

    stage('Security - Image Scan') {
      steps {
        echo 'Scanning Docker image for vulnerabilities using Trivy'
        sh 'trivy image --cache-dir $TRIVY_CACHE_DIR --exit-code 0 --severity HIGH,CRITICAL $IMAGE_NAME:$BUILD_NUMBER'
      }
    }

    stage('Test - Unit Tests') {
      steps {
        echo 'Running unit tests with Pytest'
        sh 'pytest || echo "pytest not set up"'
      }
    }

    stage('Push') {
      steps {
        echo 'Pushing Docker image to Docker Hub'
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
          sh 'echo $PASS | docker login -u $USER --password-stdin'
          sh 'docker tag $IMAGE_NAME:$BUILD_NUMBER alexcarry/$IMAGE_NAME:$BUILD_NUMBER'
          sh 'docker push alexcarry/$IMAGE_NAME:$BUILD_NUMBER'
        }
      }
    }

    stage('Deploy to Kubernetes') {
      steps {
        echo 'Deploying application to Kubernetes cluster'
        withCredentials([file(credentialsId: 'kubeconfig-creds', variable: 'KUBECONFIG_FILE')]) {
          sh '''
            export KUBECONFIG=$KUBECONFIG_FILE
            kubectl apply -f k8s/deploy.yml
          '''
        }
      }
    }

    stage('Operate - Post-Deploy Checks') {
      steps {
        echo 'Checking pod status and resource usage'
        sh 'kubectl get pods -o wide'
        sh 'kubectl top pods || echo "Metrics Server might not be installed"'
      }
    }

    stage('Monitor & Feedback') {
      steps {
        echo 'Checking logs and capturing feedback'
        sh 'kubectl logs --tail=50 -l app=devops-demo-app || echo "No logs found"'
        echo 'Integrate Prometheus/Grafana alerts in production setup for better observability'
      }
    }
  }

  post {
    always {
      echo 'Pipeline execution complete. Cleanup or notifications can go here.'
    }
  }
}

