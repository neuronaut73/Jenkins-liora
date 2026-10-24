# Jenkins CI/CD Exam – Liora DevOps

## Project Overview

This project implements a Jenkins-based CI/CD pipeline for two FastAPI microservices: **Movie Service** and **Cast Service**.

Each microservice uses its own PostgreSQL database. Jenkins builds and verifies Docker images, pushes them to DockerHub, and deploys the applications to Kubernetes using Helm.

The pipeline automatically deploys to three environments: **DEV, QA, and STAGING**. Deployment to **PRODUCTION** requires manual approval and is restricted to the dedicated Jenkins production job configured to build the `master` branch.

## Architecture

The application consists of:

- **Movie Service:** FastAPI application for managing movies.
- **Cast Service:** FastAPI application for managing casts.
- **Movie PostgreSQL:** Dedicated database for the Movie Service.
- **Cast PostgreSQL:** Dedicated database for the Cast Service.
- **Nginx:** Reverse proxy for the local Docker Compose setup.
- **Jenkins:** CI/CD automation server.
- **DockerHub:** Container image registry.
- **Helm:** Kubernetes application deployment.
- **Kubernetes (k3s):** Container orchestration platform.

The Movie Service communicates with the Cast Service over the Kubernetes service network.

## Repository Structure

```text
Jenkins-liora/
├── Jenkinsfile
├── README.md
├── docker-compose.yml
├── nginx_config.conf
├── movie-service/
│   └── Dockerfile
├── cast-service/
│   └── Dockerfile
├── charts/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
└── k8s/
    └── postgres.example.yaml
```

## Docker Images

The Jenkins pipeline builds and pushes two images:

| Service | DockerHub repository |
|---|---|
| Movie Service | [neuronaut73/jenkins-exam-movie](https://hub.docker.com/r/neuronaut73/jenkins-exam-movie) |
| Cast Service | [neuronaut73/jenkins-exam-cast](https://hub.docker.com/r/neuronaut73/jenkins-exam-cast) |

Images use unique tags derived from the Jenkins job name and build number, preventing tag collisions between the main and production pipelines.

## Kubernetes Environments

The project uses four Kubernetes namespaces:

| Namespace | Deployment |
|---|---|
| `dev` | Automatic |
| `qa` | Automatic |
| `staging` | Automatic |
| `prod` | Manual approval required |

Each namespace contains:

- Movie Service deployment and Kubernetes Service
- Cast Service deployment and Kubernetes Service
- Dedicated PostgreSQL deployments and Services
- PersistentVolumeClaims for database storage
- Kubernetes Secret containing database credentials

Helm releases are named `jenkins-movie` and `jenkins-cast`.

## PostgreSQL Configuration

The repository includes a reusable Kubernetes configuration template:

`k8s/postgres.example.yaml`

It defines the PostgreSQL Secrets, PersistentVolumeClaims, Deployments, and Services.

For security reasons, the actual database configuration file is excluded from Git using `.gitignore`.

To prepare the database infrastructure:

1. Copy `k8s/postgres.example.yaml` to `k8s/postgres.yaml`.
2. Replace the two database password placeholders with appropriate passwords.
3. Ensure the Kubernetes cluster provides the `local-path` StorageClass, or adjust the configuration for the available storage provider.
4. Apply the configuration to the required namespaces.

Example:

```bash
kubectl create namespace dev --dry-run=client -o yaml | kubectl apply -f -

kubectl apply -n dev -f k8s/postgres.yaml
```

Repeat for `qa`, `staging`, and `prod`, using the same configuration only where the database names and credentials are intended to match.

**Security note:** This is an educational deployment. For production use, credentials should be managed through an appropriate secrets-management solution rather than stored as plaintext files.

## Jenkins Pipeline

The pipeline is defined in the repository's `Jenkinsfile`.

### Continuous Integration

1. Check out the Git repository.
2. Build Docker images for both microservices.
3. Verify that both Docker images exist.
4. Authenticate to DockerHub using Jenkins credentials.
5. Push both images to DockerHub.

The DockerHub credentials are stored in Jenkins under the credential ID `dockerhub-credentials`.

### Continuous Deployment

After a successful image push, Jenkins deploys both microservices sequentially to:

1. DEV
2. QA
3. STAGING

Deployments use `helm upgrade --install` and Kubernetes rollout checks.

### Production Approval

The project uses two Jenkins Pipeline jobs:

| Jenkins job | Git branch | Purpose |
|---|---|---|
| `Jenkins-Liora-Exam` | `main` | Automatic CI/CD through STAGING |
| `Jenkins-Liora-Production` | `master` | CI/CD including manual production approval |

The production job verifies that the checked-out commit matches `origin/master`.

Before production deployment, Jenkins pauses at the **Approve Production** stage and requires an authorized user to approve the deployment.

Only after approval does Jenkins execute **Deploy to PROD**.

If approval is rejected or times out, production deployment is not executed.

## Local Development

The application can also be started using Docker Compose:

```bash
docker compose up --build -d
```

The Nginx reverse proxy exposes the FastAPI documentation at:

- Movie API: `http://localhost:8080/api/v1/movies/docs`
- Cast API: `http://localhost:8080/api/v1/casts/docs`

Stop the local environment with:

```bash
docker compose down
```

## Verification

Check application deployments:

```bash
kubectl get deployments -n dev
kubectl get deployments -n qa
kubectl get deployments -n staging
kubectl get deployments -n prod
```

Check production Helm releases:

```bash
helm list -n prod
```

Check production pods:

```bash
kubectl get pods -n prod
```

Successful Jenkins pipeline executions and Kubernetes deployments were verified during the practical exercise.

## Source Code

GitHub repository:

https://github.com/neuronaut73/Jenkins-liora

## Course

Liora / DataScientest – DevOps Training

Practical Jenkins CI/CD examination project.