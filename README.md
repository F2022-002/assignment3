# DevOps & Kubernetes Orchestration - Assignment 3

This repository contains the implementation of a 3-tier microservice application orchestrated on Kubernetes (Minikube) with a full CI/CD pipeline using GitHub Actions.

## Repository Structure
- `/app` - Dockerfiles and configurations for Nginx, Flask-API, and Docker Compose.
- `/k8s` - Kubernetes manifests (Deployments, Services, PV, PVC, ConfigMaps, Secrets).
- `.github/workflows` - CI/CD pipeline definition.
- `start.sh` - Entrypoint script to provision and run the application on Minikube.
- `REPORT.md` - Technical report detailing the implementation and architecture.
