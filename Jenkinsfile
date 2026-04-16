pipeline {
  agent any

  environment {
    REPO_URL = 'https://github.com/snehalshirsath03/spring-petclinic.git'
    BRANCH   = 'main'

    APP_NAME  = 'spring-petclinic'
    IMAGE_TAG = "${BUILD_NUMBER}"

    // ✅ AWS / ECR
    AWS_REGION = 'ap-south-1'
    ECR_URI    = '059325865468.dkr.ecr.ap-south-1.amazonaws.com/spring-petclinic'

    // ✅ SonarQube
    SONAR_HOST  = 'http://13.233.166.113:9000'
    SONAR_TOKEN = credentials('sonar-token')
  }

  stages {

    stage('Pre-Build Cleanup') {
      steps {
        sh 'docker image prune -af || true'
      }
    }

    stage('Checkout') {
      steps {
        cleanWs()
        git branch: "${BRANCH}", url: "${REPO_URL}"
      }
    }

    stage('Maven Build') {
      steps {
        sh '''
          set -eux
          mvn -B clean package -DskipTests -Dcheckstyle.skip=true
          ls -lah target/*.jar
        '''
      }
    }

    stage('SonarQube Scan') {
      steps {
        sh '''
          set -eux
          mvn -B sonar:sonar \
            -Dsonar.projectKey=spring-petclinic \
            -Dsonar.host.url=${SONAR_HOST} \
            -Dsonar.login=${SONAR_TOKEN} \
            -Dcheckstyle.skip=true
        '''
      }
    }

    stage('Create .dockerignore') {
      steps {
        sh '''
          cat > .dockerignore <<'EOF'
.git
node_modules
EOF
        '''
      }
    }

    stage('Ensure Runtime Dockerfile') {
      steps {
        sh '''
          cat > Dockerfile <<'EOF'
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
EOF
          cat Dockerfile
        '''
      }
    }

    stage('Verify JAR exists') {
      steps {
        sh '''
          set -eux
          ls -lah target || true
          ls -lah target/*.jar
        '''
      }
    }

    stage('Docker Build') {
      steps {
        sh '''
          set -eux
          docker build -t ${APP_NAME}:${IMAGE_TAG} .
        '''
      }
    }

    stage('Login to ECR + Push Image') {
      steps {
        sh '''
          set -eux
          aws ecr get-login-password --region ${AWS_REGION} | \
            docker login --username AWS --password-stdin 059325865468.dkr.ecr.ap-south-1.amazonaws.com

          docker tag ${APP_NAME}:${IMAGE_TAG} ${ECR_URI}:${IMAGE_TAG}
          docker push ${ECR_URI}:${IMAGE_TAG}
        '''
      }
    }

    stage('GitOps – Update Kubernetes Manifests') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'github-pat',
          usernameVariable: 'GIT_USER',
          passwordVariable: 'GIT_TOKEN'
        )]) {
          sh '''
            set -eux

            IMAGE="${ECR_URI}:${IMAGE_TAG}"

            git reset --hard
            git clean -fd

            git fetch origin ${BRANCH}
            git rebase origin/${BRANCH}

            sed -i -E "s|^(\\s*image:\\s*).*$|\\1${IMAGE}|g" k8s/deployment.yaml

            git config user.name "jenkins-bot"
            git config user.email "jenkins@ci"

            git add k8s/deployment.yaml
            git diff --cached --quiet || git commit -m "ci: deploy ${IMAGE}"

            git push https://${GIT_USER}:${GIT_TOKEN}@github.com/snehalshirsath03/spring-petclinic.git HEAD:${BRANCH}
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅ CI → ECR → GitOps → Argo CD flow SUCCESS"
      sh 'docker system prune -af --volumes || true'
    }
    failure {
      echo "❌ Pipeline failed"
    }
  }
}
