
# GitHub Actions: Build, Push, and Deploy to EC2


This repository automates the process of building, pushing, and deploying a Docker image to an AWS EC2 instance using GitHub Actions.

## Workflow

This workflow is triggered every time there is a `push` to the `main` branch in the repository. The process is divided into several steps:

1. **Build Docker Image**:
   - Builds the Docker image using the Dockerfile present in the repository.
   - Tags the image with the `latest` tag and a tag based on the build date and time.

2. **Push Docker Image to Docker Hub**:
   - Once the image is built, it is pushed to Docker Hub with the defined tags.

3. **Verify EC2 Instance**:
   - Connects to the EC2 instance via SSH to check if the instance is available.
   - If the instance is unavailable, the workflow will continue without failing, and the deployment will not occur.

4. **Deploy the Image to EC2**:
   - If the EC2 instance is available, the workflow will pull the latest image from Docker Hub and run it on the EC2 instance.
   - If there is a container already using port 80, it will be stopped and removed before running the new container.

## Requirements

Before using this workflow, make sure you have the following configured:

1. **Docker Hub**:
   - Create an account on [Docker Hub](https://hub.docker.com/).
   - Upload the images you want to use or use an existing username and repository.

2. **EC2 Instance on AWS**:
   - You will need an EC2 instance running a compatible distribution (such as Amazon Linux or Ubuntu).
   - The instance should be configured with an SSH key pair to allow access.
   - Open port 80 (HTTP) in the EC2 security group to allow connections from the container.

3. **GitHub Secrets**:
   - **DOCKER_USERNAME**: Your Docker Hub username.
   - **DOCKER_PASSWORD**: Your Docker Hub password (or an access token if using two-factor authentication).
   - **EC2_SSH_KEY**: The private SSH key you will use to access the EC2 instance.
   - **EC2_PUBLIC_IP**: The public IP address of your EC2 instance.

## Setup

### 1. Create Secrets in GitHub

Ensure that all the required secrets are set up in your GitHub repository:

1. Go to the repository page on GitHub.
2. Navigate to **Settings** > **Secrets and variables** > **Actions**.
3. Add the following secrets:
   - `DOCKER_USERNAME`: Your Docker Hub username.
   - `DOCKER_PASSWORD`: Your Docker Hub password.
   - `EC2_SSH_KEY`: The private SSH key you will use to access the EC2 instance.
   - `EC2_PUBLIC_IP`: The public IP address of your EC2 instance.

### 2. Create Dockerfile

Ensure that you have a proper `Dockerfile` for your project. This file should be located in the root of the repository. Here's a basic example:

```Dockerfile
FROM nginx:latest
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

### 3. Set Up the `.yml` File for GitHub Actions

The workflow file (`.yml`) should be located in `.github/workflows/` within your repository. Make sure your `.yml` file looks like the following:

```yaml
name: Build, Push, and Deploy to EC2

on:
  push:
    branches:
      - main

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}

      - name: Get current timestamp
        id: date
        run: echo "::set-output name=timestamp::$(date +'%Y%m%d%H%M')"

      - name: Build and Push Docker image
        uses: docker/build-push-action@v4
        with:
          push: true
          tags: |
            ${{ secrets.DOCKER_USERNAME }}/my-html-image:latest
            ${{ secrets.DOCKER_USERNAME }}/my-html-image:${{ steps.date.outputs.timestamp }}

  check-ec2:
    runs-on: ubuntu-latest
    needs: build-and-push
    steps:
      - name: Configure SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.EC2_SSH_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa

      - name: Check if EC2 instance is available
        run: |
          ssh -o StrictHostKeyChecking=no ec2-user@${{ secrets.EC2_PUBLIC_IP }} "echo 'EC2 is available'" || echo "EC2 instance is not available"

  deploy:
    runs-on: ubuntu-latest
    needs: check-ec2
    if: success()
    steps:
      - name: Configure SSH for EC2 deployment
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.EC2_SSH_KEY }}" > ~/.ssh/id_rsa
          chmod 600 ~/.ssh/id_rsa

      - name: Deploy to EC2
        run: |
          ssh -o StrictHostKeyChecking=no ec2-user@${{ secrets.EC2_PUBLIC_IP }} << 'EOF'
            # Stop any container using port 80
            CONTAINER_ID=$(docker ps -q --filter "publish=80")
            if [ ! -z "$CONTAINER_ID" ]; then
              echo "Stopping the container using port 80..."
              docker stop $CONTAINER_ID
              docker rm $CONTAINER_ID  # Remove the stopped container
            fi

            # Pull the latest image from Docker Hub
            echo "Pulling the latest image from Docker Hub..."
            docker pull ${{ secrets.DOCKER_USERNAME }}/my-html-image:latest

            # Pull the image with the date-based tag
            TIMESTAMP=${{ steps.date.outputs.timestamp }}
            echo "Pulling the image with tag ${TIMESTAMP}..."
            docker pull ${{ secrets.DOCKER_USERNAME }}/my-html-image:$TIMESTAMP

            # Run the container with the latest image (latest) or the date-based image
            echo "Starting the container with the most recent image..."
            docker run -d -p 80:80 ${{ secrets.DOCKER_USERNAME }}/my-html-image:latest
          EOF
```

## Usage

1. **Make changes to the code**: Make changes in the code and push them to the `main` branch of your repository.
2. **Automatic Action**: GitHub Actions will automatically trigger the workflow and perform the build, push, and deployment steps.
3. **EC2 Verification**: The workflow will verify if the EC2 instance is available before attempting the deployment.

4. **Deployment**: If the EC2 instance is available, the workflow will deploy the updated image


# Kevin Guaña