// Related uploaded file: /mnt/data/9d21f819-57d4-4aa2-b616-fffe272b8faf.png
// Drop this Jenkinsfile at the root of your repository (named exactly "Jenkinsfile").
// This pipeline uses only Jenkins credentials (no hard-coded hosts or secrets).

pipeline {
  agent any

  parameters {
    string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'Image tag to build/push')
  }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  environment {
    APP_NAME = 'cnncls'   // change if your container name differs
  }

  stages {
    stage('integration') {
      agent { docker {
        image 'python:3.10-slim'
        args '-u 0:0'   // run container as root
      } }
      steps {
        checkout scm
        sh '''
          set -e
          python -m pip install --upgrade pip
          pip install -r requirements.txt || true
          echo "Lint step — replace with your linter (flake8/pylint)"
          echo "Unit tests — replace with pytest or your test runner"
        '''
      }
    }

    stage('build-and-push-ecr-image') {
      // this stage should run on a node that can run Docker builds (controller with docker socket or a docker-enabled agent)
      agent { docker {
        image 'python:3.10-slim'
        args '-u 0:0'   // run container as root
      } }
      steps {
        checkout scm

        // gather all secrets as strings
        withCredentials([
          string(credentialsId: 'aws-access-key-id', variable: 'AWS_ACCESS_KEY_ID'),
          string(credentialsId: 'aws-secret-access-key', variable: 'AWS_SECRET_ACCESS_KEY'),
          string(credentialsId: 'aws-region', variable: 'AWS_REGION'),
          string(credentialsId: 'ecr-registry', variable: 'ECR_REGISTRY'),
          string(credentialsId: 'ecr-repo', variable: 'ECR_REPO')
        ]) {
          sh '''
            set -e
            IMAGE_TAG="${IMAGE_TAG}"
            REPO="${ECR_REPO}"
            REGISTRY="${ECR_REGISTRY}"

            echo "Building Docker image ${REPO}:${IMAGE_TAG}"
            docker build -t ${REPO}:${IMAGE_TAG} .
            docker tag ${REPO}:${IMAGE_TAG} ${REGISTRY}/${REPO}:${IMAGE_TAG}

            # ensure aws cli is available; install via pip if missing
            if ! command -v aws >/dev/null 2>&1; then
              echo "aws not found, installing awscli via pip"
              python3 -m pip install --upgrade pip || true
              pip3 install awscli --upgrade || true
            fi

            echo "Configuring AWS CLI and logging into ECR"
            aws configure set aws_access_key_id ${AWS_ACCESS_KEY_ID}
            aws configure set aws_secret_access_key ${AWS_SECRET_ACCESS_KEY}
            aws configure set region ${AWS_REGION}

            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${REGISTRY}

            echo "Pushing ${REGISTRY}/${REPO}:${IMAGE_TAG} to ECR"
            docker push ${REGISTRY}/${REPO}:${IMAGE_TAG}
          '''
        }
      }
    }

    stage('continuous-deployment') {
      agent any
      steps {
        // use ssh key and ec2 host stored as credentials
        withCredentials([
          sshUserPrivateKey(credentialsId: 'ssh-deploy-cred', keyFileVariable: 'SSH_KEY'),
          string(credentialsId: 'ec2-host', variable: 'EC2_SSH'),
          string(credentialsId: 'ecr-registry', variable: 'ECR_REGISTRY'),
          string(credentialsId: 'ecr-repo', variable: 'ECR_REPO'),
          string(credentialsId: 'aws-region', variable: 'AWS_REGION')
        ]) {
          sh '''
            set -e
            IMAGE_TAG="${IMAGE_TAG}"
            REGISTRY="${ECR_REGISTRY}"
            REPO="${ECR_REPO}"
            IMAGE_FULL=${REGISTRY}/${REPO}:${IMAGE_TAG}

            echo "Deploying image: ${IMAGE_FULL} to ${EC2_SSH}"

            chmod 600 ${SSH_KEY}

            # Run remote commands via a single ssh call. We expand variables locally to build the actual commands.
            ssh -o StrictHostKeyChecking=no -i ${SSH_KEY} ${EC2_SSH} \
              "set -e; \
               if command -v aws >/dev/null 2>&1; then \
                 aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${REGISTRY} || true; \
               else \
                 echo 'Warning: aws cli missing on EC2. Ensure EC2 instance role or install awscli for ECR login.'; \
               fi; \
               docker pull ${IMAGE_FULL} || exit 1; \
               docker ps -q --filter \"name=${APP_NAME}\" | grep -q . && docker stop ${APP_NAME} && docker rm -fv ${APP_NAME} || true; \
               docker run -d --restart unless-stopped -p 8080:8080 --name=${APP_NAME} ${IMAGE_FULL}; \
               docker image prune -f || true"
          '''
        }
      }
    }
  }

  post {
    always {
      echo "Pipeline finished"
    }
    success {
      echo "Build and deploy successful"
    }
    failure {
      echo "Pipeline failed — check console output"
    }
  }
}
