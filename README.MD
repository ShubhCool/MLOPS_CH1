# MLOps Pipeline: From Local Development to Enterprise Deployment

This guide outlines the transition from a local Jenkins and Docker setup to a scalable, enterprise-grade MLOps architecture. It includes the corrected Docker configurations to resolve the permission and naming errors encountered during development.

***

## 🏗️ Project Overview
This project automates a complete machine learning lifecycle, including:
- **Data Fetching:** Automated retrieval of datasets from Google Cloud Storage (GCS) buckets.
- **Model Training:** Local execution of ML training pipelines with automated weight saving.
- **Service Exposure:** A Flask-based prediction service with a custom HTML/CSS frontend.
- **Containerization:** End-to-end packaging via Docker for consistent execution and deployment.
- **CI/CD:** Automated build and deployment to Google Cloud Run using Jenkins.

***

## 🏢 Enterprise MLOps Workflow
In a professional enterprise environment, the manual local steps are replaced by a **GitOps-driven** and **Cloud-Native** pattern:

1.  **Code Commit:** Developer pushes code to a version-controlled repository (Git).
2.  **CI Trigger:** Jenkins detects changes and spawns an **ephemeral Kubernetes agent** for a clean environment.
3.  **Training Orchestration:** Jenkins triggers **Vertex AI Pipelines** (GCP) for scalable, serverless training using GCS data.
4.  **Model Governance:** Post-training, models are registered in a **Model Registry** and require data scientist approval.
5.  **Secure Artifact Creation:** Jenkins builds the Docker image, injecting the approved model and performing a **Security Scan** (e.g., Snyk or GCR scanning).
6.  **GitOps Deployment:** Jenkins updates a manifest repository, triggering an automated rollout to **Cloud Run** or **GKE**.

***

## 🛠️ Implementation Guide

### Step 1: Optimized Jenkins Setup (Fixing Socket Permissions)
To allow Jenkins to run Docker commands without permission errors, we must align the container's Docker Group ID with your host machine.

1.  **Find your host's Docker GID:**
    ```bash
    # Run this on your host machine
    getent group docker | cut -d: -f3
    ```
    *(Note the output number, e.g., 999)*

2.  **Create the Jenkins Dockerfile (`custom_jenkins/Dockerfile`):**
    ```dockerfile
    FROM jenkins/jenkins:lts
    USER root

    # Install prerequisites using dynamic version detection
    RUN apt-get update && apt-get install -y ca-certificates curl gnupg && \
        install -m 0755 -d /etc/apt/keyrings && \
        curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
        chmod a+r /etc/apt/keyrings/docker.gpg && \
        echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
        $(. /etc/os-release && echo "$VERSION_CODENAME") stable" > /etc/apt/sources.list.d/docker.list && \
        apt-get update && apt-get install -y docker-ce docker-ce-cli containerd.io && \
        apt-get clean

    # Align Group ID (Replace 999 with your host's GID from Step 1)
    RUN groupadd -g 999 host_docker && usermod -aG host_docker jenkins

    USER jenkins
    ```

3.  **Run the Jenkins Container:**
    ```bash
    docker run -d --name jenkins-dind --privileged \
    -p 8080:8080 -p 50000:50000 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v jenkins_home:/var/jenkins_home \
    jenkins-dind
    ```

***

### Step 2: Project Containerization
This Dockerfile handles the ML environment setup, model training, and application serving.

```dockerfile
# Use a lightweight Python image
FROM python:3.10-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

# Install system dependencies for LightGBM/ML libs
RUN apt-get update && apt-get install -y --no-install-recommends \
    libgomp1 && apt-get clean && rm -rf /var/lib/apt/lists/*

COPY . .

# Install and execute training
RUN pip install --no-cache-dir -e .
RUN python pipeline/training_pipeline.py

EXPOSE 5000
CMD ["python", "application.py"]
```

***

### Step 3: Deployment Pipeline (Jenkinsfile)
Ensure your image tags are strictly **lowercase** to avoid Docker registry errors.

```groovy
stage('Build & Push to GCR') {
    steps {
        withCredentials([file(credentialsId: 'gcp-key', variable: 'GOOGLE_APPLICATION_CREDENTIALS')]) {
            script {
                sh '''
                gcloud auth activate-service-account --key-file=${GOOGLE_APPLICATION_CREDENTIALS}
                gcloud auth configure-docker --quiet
                
                # Image name must be lowercase!
                IMAGE_NAME="gcr.io/${GCP_PROJECT}/mlops-project-image:latest"
                
                docker build -t $IMAGE_NAME .
                docker push $IMAGE_NAME
                '''
            }
        }
    }
}
```

***

## 🚨 Troubleshooting Common Errors
- **Permission Denied (`/var/run/docker.sock`):** Ensure the host's GID matches the container's `docker` group ID and that you have restarted the container after the change.
- **Invalid Tag Error:** Docker repositories do not support uppercase letters. Always use lowercase for `${GCP_PROJECT}` and image names.
- **Package Not Found:** Use the updated Docker GPG key methods (using `/etc/apt/keyrings`) as the `apt-key` method is deprecated in newer Debian-based images.