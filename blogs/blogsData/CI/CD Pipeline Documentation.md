---
title: "CI/CD Pipeline Documentation"
author: "Suraj Jaiswal"
date: "July 30, 2024"
format: html
categories: ['System']
---

This document outlines the CI/CD pipeline configuration for deploying a Docker container to an EC2 instance using GitHub Actions, Amazon ECR (Elastic Container Registry), and EC2.

## Overview

The CI/CD pipeline is triggered by a push event to the `non_ehr_dev` branch. `non_ehr_dev` branch name is specific to my usecase, but it can be any in your usecase. It comprises two primary jobs:

1. **Build and Push Docker Image**: This job builds a Docker image and pushes it to Amazon ECR.
2. **Deploy to EC2**: This job pulls the Docker image from ECR and deploys it to an EC2 instance.

## Workflow Details

### Triggers

The workflow is initiated by a push to the `non_ehr_dev` branch.

### Environment Variables

- AWS_REGION: The AWS region where resources are located.
- AWS_ACCOUNT_ID: AWS account ID.
- ECR_REPO_NAME: The name of the ECR repository.
- ECR_REPOSITORY: The full URI of your ECR repository.

### Jobs

### Build and Push Docker Image

This job handles the building of a Docker image and its subsequent push to Amazon ECR.

**Steps:**

1. **Checkout**: Retrieves the repository code.
2. **Configure AWS credentials**: Sets up AWS credentials using GitHub secrets.
3. **Login to Amazon ECR**: Authenticates with Amazon ECR.
4. **Build, tag, and push image to Amazon ECR**: Builds the Docker image, tags it, and pushes it to the ECR repository.

### Deploy to EC2

This job manages the deployment of the Docker container to an EC2 instance.

**Steps:**

1. **Configure AWS credentials**: Sets up AWS credentials using GitHub secrets.
2. **Print ECR Repository**: Displays the ECR repository URI.
3. **Login to Amazon ECR**: Authenticates with Amazon ECR.
4. **Pull image from ECR**: Retrieves the Docker image from the ECR repository.
5. **Prune unused Docker images**: Cleans up unused Docker images to free up space.
6. **Stop and remove old container**: Stops and removes the existing Docker container, if any.
7. **Run Docker container**: Deploys the Docker container with the new image.

## GitHub Actions Workflow Configuration

The GitHub Actions workflow is defined in a YAML file. Below is the complete configuration:

```yaml
name: Deploy to EC2

on:
  push:
    branches: [ "non_ehr_dev" ]

env:
  AWS_REGION: ${{ secrets.AWS_REGION }}
  AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
  ECR_REPO_NAME: ${{ secrets.NON_EHR_DEV_ECR_REPO }}
  ECR_REPOSITORY: "${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ secrets.AWS_REGION }}.amazonaws.com/${{ secrets.NON_EHR_DEV_ECR_REPO }}"

permissions:
  contents: read

jobs:
  build:
    name: Build and Push Docker Image
    runs-on: ubuntu-latest

    steps:
    - name: Checkout
      uses: actions/checkout@v3

    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}

    - name: Login to Amazon ECR
      run: |
        aws ecr get-login-password --region ${{ env.AWS_REGION }} | sudo docker login --username AWS --password-stdin ${{ env.ECR_REPOSITORY }}

    - name: Build, tag, and push image to Amazon ECR
      id: build-image
      run: |
        sudo docker build -t ${{ env.ECR_REPO_NAME }} .
        sudo docker tag ${{ env.ECR_REPO_NAME }}:latest ${{ env.ECR_REPOSITORY }}:latest
        sudo docker push ${{ env.ECR_REPOSITORY }}:latest

  deploy:
    name: Deploy to EC2
    needs: build
    runs-on: [self-hosted, non-ehr-dev]

    steps:
    - name: Configure AWS credentials
      uses: aws-actions/configure-aws-credentials@v1
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: ${{ secrets.AWS_REGION }}

    - name: Print ECR Repository
      run: echo ${{ env.ECR_REPOSITORY }}

    - name: Login to Amazon ECR
      run: |
        aws ecr get-login-password --region ${{ env.AWS_REGION }} | sudo docker login --username AWS --password-stdin ${{ env.ECR_REPOSITORY }}

    - name: Pull image from ECR
      run: sudo docker pull ${{ env.ECR_REPOSITORY }}:latest

    - name: Prune unused Docker images
      run: sudo docker image prune -a -f

    - name: Stop and remove old container
      run: sudo docker rm -f python-app-container || true

    - name: Run docker container
      run: |
        sudo docker run -d -e AWS_ACCESS_KEY_ID=${{ secrets.AWS_ACCESS_KEY_ID }} -e AWS_SECRET_ACCESS_KEY=${{ secrets.AWS_SECRET_ACCESS_KEY }} -v /home/ubuntu/private_key:/app/private_key \\\\
        -p 8000:8000 -p 8200:8200 -p 8501:8501 \\\\
        --name python-app-container \\\\
        ${{ env.ECR_REPOSITORY }}:latest

```
## Setting Up the GitHub Actions Runner on EC2

To run the GitHub Actions runner on your EC2 instance and deploy your application, follow these steps:

### Step 1: Create and Configure an EC2 Instance

1. **Launch an EC2 instance** with an appropriate Amazon Machine Image (AMI), such as Ubuntu.
2. **Ensure the instance has internet access** or the necessary VPC settings to communicate with GitHub.
3. **Attach an IAM role** to the instance with the necessary permissions for ECR and ECS, if applicable.
4. **Install Docker** on the EC2 instance:
    
```bash
sudo apt-get update
sudo apt-get install -y docker.io
sudo usermod -aG docker $USER
```
### Step 2: Install GitHub Runner on EC2

1. **Connect to your EC2 instance via SSH**.
2. **Download the runner**:
    
```bash
mkdir actions-runner && cd actions-runner
curl -o actions-runner-linux-x64-2.278.0.tar.gz -L <https://github.com/actions/runner/releases/download/v2.278.0/actions-runner-linux-x64-2.278.0.tar.gz>
tar xzf ./actions-runner-linux-x64-2.278.0.tar.gz
```

3. **Install dependencies**:
```bash
sudo apt-get install -y libicu-dev libssl-dev
```

### Step 3: Configure and Start the GitHub Runner

1. **Configure the runner**:
    - Go to your GitHub repository.
    - Navigate to **Settings** > **Actions** > **Runners**.
    - Click on **Add runner** and follow the instructions to generate a configuration token.
    - Use the provided commands to configure the runner:
    ```bash
    ./config.sh --url <https://github.com/YOUR_GITHUB_USERNAME/YOUR_REPO_NAME> --token YOUR_CONFIGURATION_TOKEN
    ```
2. **Start the runner**:
```bash
./run.sh
```
### Update Your GitHub Actions Workflow

Ensure your GitHub Actions workflow is configured to run on your self-hosted runner by updating the `runs-on` field:
```yaml
deploy:
  name: Deploy to EC2
  needs: build
  runs-on: [self-hosted, non-ehr-dev]
```

By following these steps, you will set up a GitHub Actions runner on your EC2 instance, allowing you to deploy your Docker container directly to EC2 using your CI/CD pipeline.

## Conclusion

This GitHub Actions workflow automates the process of building a Docker image, pushing it to Amazon ECR, and deploying it to an EC2 instance.


PS: This documentation is for my personal reference and the data is taken from net and may require adjustments based on specific project requirements.
