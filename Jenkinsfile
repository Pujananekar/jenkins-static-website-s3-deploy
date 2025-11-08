pipeline {
  agent any

  environment {
    SITE_DIR = "public"
    S3_BUCKET = "my-website-puja"
    AWS_CRED_ID = "aws-jenkins-credentials" // बदल करा
    AWS_REGION = "us-east-1"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Show files') {
      steps {
        sh "ls -la ${SITE_DIR} || true"
      }
    }

    stage('Deploy to S3') {
      steps {
        withCredentials([usernamePassword(credentialsId: env.AWS_CRED_ID, usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
          sh '''
            set -euo pipefail

            # check aws CLI
            if ! command -v aws >/dev/null 2>&1; then
              echo "ERROR: aws CLI not found on this agent. Install awscli or use an agent image with awscli preinstalled."
              exit 1
            fi

            export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
            export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
            export AWS_DEFAULT_REGION=${AWS_REGION}

            # sync files (recommended). If your bucket needs ACLs, add --acl public-read (but prefer bucket policy).
            aws s3 sync ${SITE_DIR}/ s3://${S3_BUCKET}/ --delete --exact-timestamps

            # ensure index.html content-type
            if [ -f "${SITE_DIR}/index.html" ]; then
              aws s3 cp ${SITE_DIR}/index.html s3://${S3_BUCKET}/index.html --content-type "text/html" || true
            fi
            if [ -f "${SITE_DIR}/error.html" ]; then
              aws s3 cp ${SITE_DIR}/error.html s3://${S3_BUCKET}/error.html --content-type "text/html" || true
            fi

            echo "Deployed to s3://${S3_BUCKET}/"
          '''
        }
      }
    }
  }

  post {
    success {
      echo "SUCCESS: http://${env.S3_BUCKET}.s3-website.${env.AWS_REGION}.amazonaws.com"
    }
    failure {
      echo "Deployment failed - check logs"
    }
    always {
      // use deleteDir() instead of cleanWs() to avoid missing plugin
      deleteDir()
    }
  }
}
