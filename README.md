![GitHub Workflow Status](https://github.com/zyderus/deploy-node-app-to-aws-ec2/actions/workflows/deploy.yml/badge.svg)
<br>

# Deploy with GitHub Actions to AWS Beanstalk

## Overview

This repository demonstrates how to automate the deployment of a Node.js application to AWS Elastic Beanstalk using GitHub Actions. The workflow includes testing, staging, and production deployments with manual approval for production.

## Prerequisites

Before you begin, ensure you have the following:

- An AWS account with necessary permissions.
- A GitHub repository with your Node.js application.
- Basic knowledge of AWS Elastic Beanstalk, IAM, and GitHub Actions.
  <br>
  <br>

## AWS Configuration

#### 1. Create Identity Provider

- Provider Type: `OpenID Connect (OIDC)`
- Provider URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`

#### 2. Create IAM Role

- Select the Identity Provider created above.
- Set the Audience to sts.amazonaws.com.
- Specify your GitHub organization name or username.
- Attach the following policies:
  - AmazonS3FullAccess
  - AWSElasticBeanstalkFullAccess
- Name the role: GitHubActionsDeploymentRole

#### 3. Create S3 Bucket for Deployment Packages

- Ensure you are in the correct AWS region.
- Create an S3 bucket named github-zyderus-node-app.

#### 4. Create Elastic Beanstalk Application and Environment

- Ensure you are in the correct AWS region.
- Create an Elastic Beanstalk application named SimpleNodeApp.
- Select the Node.js platform.
- Create and use a new service role.
- Create a separate role for EC2 instances with the following permissions:

  - Web tier
  - Worker tier
  - Multicontainer

- Name the role: aws-elasticbeanstalk-ec2-role
- Select this role in the EC2 instance profile during the Beanstalk setup.
- Skip to review and launch.
- Monitor the progress in CloudFormation.
- Create two environments: stg (staging) and prod (production).
  <br>
  <br>

## GitHub Actions Configuration

#### 1. Create GitHub Actions Workflow

Create a GitHub Actions workflow file `.github/workflows/deploy.yml` with the following content:

#### Define your workflow and set up environment variables

```yaml
name: Deploy to Beanstalk

env:
  APPLICATION_NAME: 'SimpleNodeApp'
  AWS_REGION: 'us-east-1'
```

#### Specify the events to trigger the workflow

```yml
on:
  push:
    branches:
      - main
  workflow_dispatch: # Allows manual triggering of the workflow
```

#### Define the tasks as jobs

```yml
jobs:
```

#### Set up the build phase with tests

```yml
test:
  runs-on: ubuntu-latest

  steps:
    - name: Prints Hello Message in Testing
      run: echo "Hello World from Testing job"
```

#### Set up deployment to the `stg` environment

```yml
deploy-stg:
  runs-on: ubuntu-latest
  needs: [test]
  permissions:
    id-token: write
    contents: read

  steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Node.js
      uses: actions/setup-node@v4
      with:
        node-version: 22

    - name: Install dependencies
      run: npm ci

    - name: Zip application
      run: zip -r application.zip . -x '*.git*' 'node_modules/*'

    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsDeploymentRole
        aws-region: ${{ env.AWS_REGION }}

    - name: Upload to S3
      run: aws s3 cp application.zip s3://${{ env.S3_BUCKET_NAME }}/$GITHUB_SHA.zip

    - name: Create Beanstalk application version
      run: |
        aws elasticbeanstalk create-application-version \
          --application-name ${{ env.APPLICATION_NAME }} \
          --version-label $GITHUB_SHA \
          --source-bundle S3Bucket=${{ env.S3_BUCKET_NAME }},S3Key=$GITHUB_SHA.zip

    - name: Update Beanstalk STG environment
      run: |
        aws elasticbeanstalk update-environment \
          --environment-name ${{ env.ENVIRONMENT_NAME_STG }} \
          --version-label $GITHUB_SHA
```

#### Promote deployment to the `prod` environment with manual approval

**Note**: Ensure you have set up the `prod` environment in AWS Elastic Beanstalk and configured the `prod` environment in your GitHub repository.

```yml
deploy-prod:
  runs-on: ubuntu-latest
  needs: [deploy-stg]
  environment:
    name: production # Requires approval via GitHub Environments
  permissions:
    id-token: write
    contents: read

  steps:
    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsDeploymentRole
        aws-region: ${{ env.AWS_REGION }}

    - name: Promote to PROD environment
      run: |
        aws elasticbeanstalk update-environment \
          --environment-name ${{ env.ENVIRONMENT_NAME_PROD }} \
          --version-label $GITHUB_SHA  # Same version as staging
```
