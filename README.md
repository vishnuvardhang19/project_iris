# Project Iris

This project is configured to deploy to AWS App Runner using:

- GitHub Actions
- Amazon ECR
- AWS App Runner

The app image is built from the `Dockerfile` in this repository.
This setup uses GitHub OIDC, so you do not need long-lived AWS access keys in a public repository.

## Local setup

### Create an environment with conda

```bash
conda create -p ./env python=3.11
conda activate env
```

### Create an environment with venv

```bash
python -m venv venv
source venv/bin/activate
```

### Install dependencies

```bash
pip install -r requirements.txt
pip install -e .
```

## Deployment architecture

This is the deployment flow:

1. You push code to the `main` branch.
2. GitHub Actions runs `.github/workflows/deploy-apprunner.yml`.
3. GitHub Actions builds a Docker image.
4. GitHub Actions pushes the image to Amazon ECR.
5. GitHub Actions calls AWS App Runner to start a deployment.
6. App Runner pulls the newest image from ECR and updates the running app.

Files used in deployment:

- `.github/workflows/deploy-apprunner.yml`
- `Dockerfile`

## Before you start

Make sure you have:

- an AWS account
- access to the AWS Console
- access to the GitHub repository settings
- permission to create IAM identity providers, IAM roles, ECR repositories, and App Runner services

## AWS setup: exact order

Follow these steps in the same order.

### Step 1: Choose one AWS region

Pick one AWS region and use it for everything.

Examples:

- `ap-south-1`
- `us-east-1`
- `us-west-2`

Use the same region for:

- Amazon ECR
- AWS App Runner
- GitHub secret `AWS_REGION`

Example:

```text
AWS_REGION=ap-south-1
```

## Step 2: Create the Amazon ECR repository

Amazon ECR stores your Docker images.

In AWS Console:

1. Search for `ECR`
2. Open `Amazon Elastic Container Registry`
3. Confirm you are in the region chosen in Step 1
4. In the left menu, open `Private repositories`
5. Click `Create repository`
6. Choose `Private`
7. Repository name: `project-iris`
8. Keep the default settings unless you want to customize them
9. Click `Create repository`

After creation, you should see a repository URI similar to:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/project-iris
```

You do not put this URI into GitHub secrets for the current workflow, but this confirms the repository exists.

## Step 3: Add GitHub as an AWS OIDC identity provider

This lets GitHub Actions authenticate to AWS without storing `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.

In AWS Console:

1. Search for `IAM`
2. Open `IAM`
3. In the left menu, click `Identity providers`
4. Click `Add provider`
5. Provider type: `OpenID Connect`
6. Provider URL: `https://token.actions.githubusercontent.com`
7. Audience: `sts.amazonaws.com`
8. Click `Add provider`

If this provider already exists in your AWS account, reuse it and do not create a duplicate.

## Step 4: Create the IAM role for GitHub Actions

This IAM role is assumed by GitHub Actions through OIDC.

The workflow needs this role to allow:

- ECR login
- ECR image push
- App Runner deployment start

In AWS Console:

1. Open `IAM`
2. Open `Roles`
3. Click `Create role`
4. Trusted entity type: `Web identity`
5. Identity provider: `token.actions.githubusercontent.com`
6. Audience: `sts.amazonaws.com`
7. Click `Next`
8. For now, you can skip permission policies and create the role first
9. Role name: `GitHubActionsAppRunnerDeployRole`
10. Click `Create role`

After the role is created:

1. Open the new role
2. Open the `Trust relationships` tab
3. Click `Edit trust policy`
4. Replace the trust policy with this

Replace:

- `YOUR_GITHUB_USERNAME` with your GitHub username or org
- `YOUR_REPOSITORY_NAME` with your repository name

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:YOUR_GITHUB_USERNAME/YOUR_REPOSITORY_NAME:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

Also replace:

- `123456789012` with your AWS account ID

This trust policy means:

- only GitHub Actions tokens are accepted
- only your specific repository can assume the role
- only workflows running from the `main` branch can assume the role

## Step 5: Add permissions to the GitHub Actions IAM role

Now attach the AWS permissions that the role needs.

In AWS Console:

1. Open `IAM`
2. Open `Roles`
3. Click `GitHubActionsAppRunnerDeployRole`
4. Open `Permissions`
5. Click `Add permissions`
6. Choose `Create inline policy`
7. Open the `JSON` tab
8. Paste a policy like this

Replace:

- `ap-south-1` with your region
- `123456789012` with your AWS account ID

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:BatchGetImage",
        "ecr:CompleteLayerUpload",
        "ecr:InitiateLayerUpload",
        "ecr:PutImage",
        "ecr:UploadLayerPart"
      ],
      "Resource": "arn:aws:ecr:ap-south-1:123456789012:repository/project-iris"
    },
    {
      "Effect": "Allow",
      "Action": [
        "apprunner:StartDeployment"
      ],
      "Resource": "*"
    }
  ]
}
```

9. Click `Review policy`
10. Name it `GitHubActionsDeployProjectIris`
11. Click `Create policy`

Notes:

- `ecr:GetAuthorizationToken` must stay on `*`
- the ECR repository ARN should match your real repository
- `apprunner:StartDeployment` can be narrowed later to your exact service ARN after the service exists

## Step 6: Create the App Runner access role for ECR

This is a different AWS identity from the IAM user above.

Why it exists:

- GitHub Actions pushes the image to ECR
- App Runner needs permission to pull the image from ECR

In AWS Console:

1. Open `IAM`
2. Open `Roles`
3. Click `Create role`
4. For trusted entity type, choose `AWS service`
5. For use case, choose `App Runner`
6. Click `Next`
7. Attach the managed policy `AWSAppRunnerServicePolicyForECRAccess`
8. Click `Next`
9. Role name: `AppRunnerECRAccessRole`
10. Click `Create role`

This role is selected during App Runner service creation.

## Step 7: Create the App Runner service

This creates the hosted web app.

Important:

- the service should be in the same region as the ECR repository
- the service must listen on port `8501` because your app runs on that port

In AWS Console:

1. Search for `App Runner`
2. Open `AWS App Runner`
3. Make sure the region matches Step 1
4. Click `Create service`
5. For source, choose `Container registry`
6. For provider, choose `Amazon ECR`
7. Browse and select the repository `project-iris`
8. For image tag, choose `latest`
9. For deployment settings, choose `Manual`
10. For ECR access role, choose `AppRunnerECRAccessRole`
11. Click `Next`
12. Service name: `project-iris-service`
13. Port: `8501`
14. Leave other settings as default unless you need custom CPU or memory
15. Click `Next`
16. Review the settings
17. Click `Create and deploy`

Wait for the service status to become `Running`.

After creation:

1. Open the App Runner service details page
2. Copy the `Service ARN`
3. Copy the `Default domain`

The default domain is your public app URL.

The service ARN looks like this:

```text
arn:aws:apprunner:ap-south-1:123456789012:service/project-iris-service/xxxxxxxxxxxx
```

This value becomes the GitHub secret:

- `AWS_APP_RUNNER_SERVICE_ARN`

## Step 8: Add GitHub repository secrets

In GitHub:

1. Open the repository
2. Click `Settings`
3. Click `Secrets and variables`
4. Click `Actions`
5. Click `New repository secret`

Create these three secrets:

1. `AWS_REGION`
2. `AWS_ROLE_ARN`
3. `AWS_APP_RUNNER_SERVICE_ARN`

Example:

```text
AWS_REGION=ap-south-1
AWS_ROLE_ARN=arn:aws:iam::123456789012:role/GitHubActionsAppRunnerDeployRole
AWS_APP_RUNNER_SERVICE_ARN=arn:aws:apprunner:ap-south-1:123456789012:service/project-iris-service/xxxxxxxxxxxx
```

## Step 9: Push code to main

After the AWS resources and GitHub secrets are ready:

1. commit your code
2. push to the `main` branch

GitHub Actions will then:

1. check out the repo
2. authenticate to AWS
3. log in to ECR
4. build the Docker image
5. push the image to ECR with:
   - the commit SHA tag
   - the `latest` tag
6. call App Runner to start a new deployment

## Step 10: Verify the deployment

In GitHub:

1. Open the repository
2. Click `Actions`
3. Open the latest workflow run
4. Confirm all steps pass

In AWS:

1. Open `App Runner`
2. Open your service
3. Check that service status is `Running`
4. Open the `Default domain`
5. Confirm the app loads successfully

## First deployment note

Your workflow uses:

- a SHA tag for each build
- a `latest` tag

The App Runner workflow then starts a manual deployment so the service fetches the newest image from ECR.

This means:

- the App Runner service must already exist
- the GitHub workflow does not create the App Runner service for you
- the App Runner service must already be connected to the `project-iris` ECR repository

## Common problems

### Problem: GitHub Actions cannot log in to AWS

Possible causes:

- wrong `AWS_ROLE_ARN`
- wrong `AWS_REGION`
- role trust policy does not match the repo or branch
- workflow is missing `id-token: write`

### Problem: Docker image push to ECR fails

Possible causes:

- ECR repository `project-iris` does not exist
- IAM policy does not allow `ecr:GetAuthorizationToken`
- IAM policy does not allow image push actions
- wrong AWS region

### Problem: App Runner cannot pull the image

Possible causes:

- App Runner service is not using the `AppRunnerECRAccessRole`
- ECR repository and App Runner service are in different regions
- wrong image tag was selected during service setup

### Problem: App Runner deployment command fails

Possible causes:

- wrong `AWS_APP_RUNNER_SERVICE_ARN`
- IAM role does not have `apprunner:StartDeployment`
- the App Runner service does not exist

### Problem: App loads poorly after deployment

Possible causes:

- the container is listening on a port different from `8501`
- the application crashes during startup
- required Python packages are missing from `requirements.txt`

## Quick checklist

Before first deployment, verify all of these are true:

- ECR repository `project-iris` exists
- GitHub OIDC provider exists in IAM
- IAM role `GitHubActionsAppRunnerDeployRole` exists
- IAM role has ECR push permission
- IAM role has `apprunner:StartDeployment`
- trust policy matches your repo and the `main` branch
- App Runner ECR access role exists
- App Runner service exists
- App Runner service uses port `8501`
- App Runner service is in the same region as ECR
- all three GitHub secrets are added
- you are pushing to the `main` branch

## Files used in this repository

- `.github/workflows/deploy-apprunner.yml` is the GitHub Actions deployment workflow
- `Dockerfile` defines the container image
- `kubernetes-deploy.yaml` is not used for the AWS App Runner path
