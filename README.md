# Deploying a Static Website to AWS S3 using Jenkins or Github Actions  

Complete guide to automate static website deployment to AWS S3 using Jenkins.

---

## Folder Structure

```
my-website/
├─ site/
│  ├─ index.html
│  ├─ css/
│  └─ js/
├─ Jenkinsfile
└─ README.md
```

---

## Step 1 — Create GitHub Repository

1. Create a new repository on GitHub (e.g., `my-website`)
2. Create the folder structure locally
3. Write `site/index.html` (basic HTML) and commit-push:

```bash
git init
git add .
git commit -m "initial commit: static site + jenkinsfile"
git branch -M main
git remote add origin https://github.com/<your-username>/my-website.git
git push -u origin main
```

---

## Step 2 — AWS: Create S3 Bucket and IAM User

> **Important:** Bucket name must be globally unique for production.

### 2.1 Create S3 Bucket

1. AWS Console → S3 → Create bucket
2. Bucket name: `my-website-<your-unique-suffix>` (e.g., `my-website-demo-2025`)
3. Select region (e.g., `us-east-1` or `ap-south-1`)
4. **Uncheck** `Block all public access` if your static website needs to be public (control access properly with bucket policy later)
5. Click Create bucket

### 2.2 Enable Static Website Hosting

1. Go to bucket → Properties → Static website hosting → Enable
2. Index document: `index.html`
3. (Optional) Error document: `error.html`
4. Save changes

### 2.3 Create IAM User (Programmatic Access)

1. AWS Console → IAM → Users → Add user
2. Username: `jenkins-deployer`
3. Access type: **Programmatic access** (you'll get Access Key ID and Secret Access Key)
4. Attach policy: Create a custom policy with minimal permissions

#### Custom IAM Policy (S3 Deploy Only)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:PutObjectAcl",
        "s3:DeleteObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-website-<your-unique-suffix>",
        "arn:aws:s3:::my-website-<your-unique-suffix>/*"
      ]
    }
  ]
}
```

* Replace `<your-unique-suffix>` with your actual bucket name
* After creating the IAM user, **save the Access Key ID and Secret Access Key** — you'll need them in Jenkins

### 2.4 (Optional) S3 Bucket Policy — Public Read Access

If you want your website publicly accessible, add this bucket policy (Bucket → Permissions → Bucket policy):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-website-<your-unique-suffix>/*"
    }
  ]
}
```

> **Warning:** Public access means anyone can view your site. For production, use CloudFront with proper HTTPS.

---

## Step 3 — Jenkins Setup

### 3.1 Running Jenkins with Docker (Simple Method)

```bash
docker run --name jenkins -p 8080:8080 -p 50000:50000 -v jenkins_home:/var/jenkins_home jenkins/jenkins:lts
```

> Open `http://localhost:8080` in your browser and follow the initial setup.

### 3.2 Install Required Plugins

* Git
* Pipeline
* Credentials Binding
* (Optional) AWS Steps or Amazon ECR Plugin — not needed if using AWS CLI

Navigate to: Manage Jenkins → Plugins → Available plugins

---

## Step 4 — Add AWS Credentials to Jenkins

1. Jenkins → Manage Jenkins → Credentials → System → Global credentials → Add Credentials
2. Kind: **Username with password**
   * Username = AWS Access Key ID
   * Password = AWS Secret Access Key
   * ID: `aws-jenkins-credentials` (remember this ID)
   * Description: `AWS S3 Deploy Credentials`
3. Click Create

---

## Step 5 — Create Jenkinsfile

Create a `Jenkinsfile` in your repository root. Copy and paste this, updating your bucket name and credentials ID:

```groovy
pipeline {
  agent any

  environment {
    SITE_DIR = "site"
    S3_BUCKET = "my-website-<your-unique-suffix>"
    AWS_REGION = "us-east-1"  // Change to your region
    AWS_CRED_ID = "aws-jenkins-credentials"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout([$class: 'GitSCM', 
                  branches: [[name: '*/main']], 
                  userRemoteConfigs: [[url: 'https://github.com/<your-username>/my-website.git']]
        ])
      }
    }

    stage('Build') {
      steps {
        echo "Build stage - optional for static sites"
        // Uncomment if you need to build assets
        // sh 'npm install'
        // sh 'npm run build'
      }
    }

    stage('Prepare') {
      steps {
        sh "ls -la ${SITE_DIR} || true"
      }
    }

    stage('Deploy to S3') {
      steps {
        withCredentials([usernamePassword(
          credentialsId: env.AWS_CRED_ID, 
          usernameVariable: 'AWS_ACCESS_KEY_ID', 
          passwordVariable: 'AWS_SECRET_ACCESS_KEY'
        )]) {
          sh '''
            # Install AWS CLI if not present
            which aws || (apt-get update && apt-get install -y awscli)

            # Configure temporary credentials
            export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
            export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
            export AWS_DEFAULT_REGION=${AWS_REGION}

            # Sync files to S3
            aws s3 sync ${SITE_DIR}/ s3://${S3_BUCKET}/ --delete

            # Make files public (if needed)
            aws s3 cp ${SITE_DIR}/ s3://${S3_BUCKET}/ --recursive --acl public-read
          '''
        }
      }
    }
  }

  post {
    success {
      echo "✅ Deploy successful!"
      echo "🌐 Website URL: http://${env.S3_BUCKET}.s3-website-${env.AWS_REGION}.amazonaws.com"
    }
    failure {
      echo "❌ Build or deploy failed"
    }
  }
}
```

> **Note:** Update `S3_BUCKET`, `AWS_REGION`, and GitHub URL with your actual values.

---

## Step 6 — Configure Jenkins Pipeline Job

1. Jenkins → New Item → Pipeline → Name: `deploy-my-website` → OK
2. In Pipeline section, select `Pipeline script from SCM`
   * SCM: Git
   * Repository URL: `https://github.com/<your-username>/my-website.git`
   * Branches to build: `*/main`
   * Script Path: `Jenkinsfile`
3. Click Save

---

## Step 7 — Configure GitHub Webhook (Auto-Trigger)

1. GitHub → Your Repository → Settings → Webhooks → Add webhook
2. Payload URL: `http://<jenkins-host>:8080/github-webhook/`
   * Example: `http://ec2-xx-xx-xx-xx.compute.amazonaws.com:8080/github-webhook/`
3. Content type: `application/json`
4. Which events: `Just the push event`
5. Click Add webhook

> **Note:** Jenkins must be publicly accessible for GitHub webhooks to work. For local development, use ngrok to expose Jenkins: `ngrok http 8080`

---

## Step 8 — Test the Complete Flow

1. Make changes to `site/index.html` locally
2. Commit and push:
   ```bash
   git add .
   git commit -m "update index page"
   git push origin main
   ```
3. GitHub webhook triggers Jenkins → Job runs automatically
4. Check Jenkins console output (job page) to verify all stages pass
5. Open in browser: `http://<your-bucket-name>.s3-website-<region>.amazonaws.com`



**Happy deploying! 🚀**
