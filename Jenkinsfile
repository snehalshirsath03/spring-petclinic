pipeline {
  agent any

  environment {
    REPO_URL = 'https://github.com/snehalshirsath03/spring-petclinic.git'
    BRANCH   = 'main'

    APP_NAME  = 'spring-petclinic'
    IMAGE_TAG = "${BUILD_NUMBER}"

    // ✅ AWS / ECR
    AWS_REGION     = 'ap-south-1'
    ECR_REPOSITORY = 'spring-petclinic'

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
          mvn -B clean package -DskipTests
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
            -Dsonar.login=${SONAR_TOKEN}
        '''
      }
    }

    stage('Create .dockerignore') {
      steps {
        sh '''
          cat > .dockerignore <<'EOF'
.git
target
node_modules
EOF
        '''
      }
    }

    stage('Validate Dockerfile') {
      steps {
        sh '''
          if [ ! -f Dockerfile ]; then
            cat > Dockerfile <<'EOF'
FROM eclipse-temurin:17-jre
WORKDIR /app
COPY target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","app.jar"]
EOF
          fi
        '''
      }
    }

    stage('Docker Build') {
      steps {
        sh '''
          docker build -t ${APP_NAME}:${IMAGE_TAG} .
        '''
      }
    }

    stage('Login to ECR + Push Image') {
      steps {
        sh '''
          AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
          ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

          aws ecr get-login-password --region ${AWS_REGION} | \
            docker login --username AWS --password-stdin ${ECR_REGISTRY}

          aws ecr describe-repositories --repository-names ${ECR_REPOSITORY} --region ${AWS_REGION} >/dev/null 2>&1 || \
            aws ecr create-repository --repository-name ${ECR_REPOSITORY} --region ${AWS_REGION}

          docker tag ${APP_NAME}:${IMAGE_TAG} ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}
          docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}
        '''
      }
    }

    // ✅ ✅ ✅ NEW STAGE (GitOps)
    stage('GitOps – Update Kubernetes Manifests') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: 'github-pat',
          usernameVariable: 'GIT_USER',
          passwordVariable: 'GIT_TOKEN'
        )]) {
          sh '''
            set -eux
            AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
            ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
            IMAGE="${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"

            sed -i "s|image:.*|image: ${IMAGE}|" k8s/deployment.yaml

            git config user.name "jenkins-bot"
            git config user.email "jenkins@ci"

            git add k8s/deployment.yaml
            git commit -m "ci: deploy ${IMAGE}"
            git push https://${GIT_USER}:${GIT_TOKEN}@github.com/snehalshirsath03/spring-petclinic.git HEAD:main
          '''
        }
      }
    }

    stage('Smoke Test') {
      steps {
        sh '''
          docker rm -f petclinic || true
          AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
          ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"

          docker run -d --name petclinic -p 8085:8080 \
            ${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}

          sleep 15
          curl -f http://localhost:8085/ || exit 1
          docker rm -f petclinic
        '''
      }
    }
  }

  post {
    success {
      echo "✅ CI → ECR → GitOps → Argo CD flow SUCCESS"
      sh '''
        docker system prune -af --volumes || true
      '''

    }
    failure {
      echo "❌ Pipeline failed"
    }
  }
}
