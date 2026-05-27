# Assignment 3: Kubernetes Orchestration & Full DevOps Pipeline

**Course:** DevOps and Cloud Computing Foundation
**Semester:** Spring 2026
**Due Date:** 31-05-2026
**Total Marks:** 100
**Submission:** VM Snapshot (.ova) + PDF Report

---

## 1. Introduction & Objectives

In this assignment, you will build and deploy a complete 3-tier microservice application using the full DevOps toolchain covered in this course. You will containerize services, set up CI/CD pipelines, and orchestrate everything on Kubernetes using Minikube.

### Learning Outcomes

Upon completion, you will be able to:

- Use Git for version control with proper branching and collaboration practices
- Write Dockerfiles and use Docker Compose for multi-container applications
- Configure Docker networking (bridge networks) and volumes for persistence
- Build CI/CD pipelines using GitHub Actions to automate image builds and pushes
- Deploy, manage, and scale applications on Kubernetes
- Configure Kubernetes networking (Services), storage (PV/PVC), and configuration management (ConfigMaps/Secrets)

---

## 2. Key Concepts (Quick Reference)

Before starting, understand these core Kubernetes concepts:

| Concept | What It Is |
|---|---|
| **Pod** | Smallest deployable unit — one or more containers running together |
| **Deployment** | Manages a set of identical Pods, handles scaling and updates |
| **Service** | Stable network endpoint to access a set of Pods (Pods are ephemeral, Services are not) |
| **ConfigMap** | Stores non-sensitive configuration as key-value pairs |
| **Secret** | Stores sensitive data (passwords, tokens) in base64-encoded form |
| **PersistentVolume (PV)** | A piece of storage provisioned in the cluster |
| **PersistentVolumeClaim (PVC)** | A request for storage by a Pod — binds to a PV |
| **Namespace** | Virtual cluster within a cluster — isolates resources |
| **NodePort** | Exposes a Service on a static port on each Node's IP |
| **ClusterIP** | Internal-only Service (default) — accessible only within the cluster |

---

## 3. Application Overview

You will work with a simple 3-tier service system:

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│    Nginx    │─────▶│  Flask API  │─────▶│    MySQL    │
│  (Reverse   │      │  (Backend)  │      │ (Database)  │
│   Proxy)    │      │             │      │             │
│  Port: 80   │      │  Port: 5000 │      │  Port: 3306 │
└─────────────┘      └─────────────┘      └─────────────┘
```

- **Nginx** — Reverse proxy that routes requests to the Flask API
- **Flask API** — Python REST API with endpoints for health checks and CRUD operations on items
- **MySQL** — Relational database storing application data

### Application Code

The application code is provided below. Create these files in the specified directory structure.

#### Flask API (`app/flask-api/app.py`)

```python
from flask import Flask, jsonify, request
import mysql.connector
import os
import time

app = Flask(__name__)

def get_db_connection():
    """Create database connection using environment variables."""
    max_retries = 30
    for i in range(max_retries):
        try:
            conn = mysql.connector.connect(
                host=os.environ.get('DB_HOST', 'mysql'),
                user=os.environ.get('DB_USER', 'flaskuser'),
                password=os.environ.get('DB_PASSWORD', 'flaskpass'),
                database=os.environ.get('DB_NAME', 'flaskdb')
            )
            return conn
        except mysql.connector.Error:
            if i < max_retries - 1:
                time.sleep(2)
            else:
                raise

def init_db():
    """Initialize the database table."""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute('''
        CREATE TABLE IF NOT EXISTS items (
            id INT AUTO_INCREMENT PRIMARY KEY,
            name VARCHAR(255) NOT NULL,
            description TEXT,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    conn.commit()
    cursor.close()
    conn.close()

@app.route('/health', methods=['GET'])
def health():
    """Health check endpoint."""
    return jsonify({'status': 'healthy', 'service': 'flask-api'}), 200

@app.route('/api/items', methods=['GET'])
def get_items():
    """Get all items."""
    conn = get_db_connection()
    cursor = conn.cursor(dictionary=True)
    cursor.execute('SELECT * FROM items ORDER BY created_at DESC')
    items = cursor.fetchall()
    cursor.close()
    conn.close()
    # Convert datetime objects to strings
    for item in items:
        item['created_at'] = item['created_at'].strftime('%Y-%m-%d %H:%M:%S')
    return jsonify(items), 200

@app.route('/api/items', methods=['POST'])
def create_item():
    """Create a new item."""
    data = request.get_json()
    if not data or 'name' not in data:
        return jsonify({'error': 'Name is required'}), 400

    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute(
        'INSERT INTO items (name, description) VALUES (%s, %s)',
        (data['name'], data.get('description', ''))
    )
    conn.commit()
    item_id = cursor.lastrowid
    cursor.close()
    conn.close()
    return jsonify({'id': item_id, 'name': data['name'], 'description': data.get('description', '')}), 201

@app.route('/api/items/<int:item_id>', methods=['DELETE'])
def delete_item(item_id):
    """Delete an item."""
    conn = get_db_connection()
    cursor = conn.cursor()
    cursor.execute('DELETE FROM items WHERE id = %s', (item_id,))
    conn.commit()
    affected = cursor.rowcount
    cursor.close()
    conn.close()
    if affected == 0:
        return jsonify({'error': 'Item not found'}), 404
    return jsonify({'message': 'Item deleted'}), 200

if __name__ == '__main__':
    init_db()
    app.run(host='0.0.0.0', port=5000)
```

#### Flask Requirements (`app/flask-api/requirements.txt`)

```
flask==3.0.0
mysql-connector-python==8.2.0
```

#### Nginx Configuration (`app/nginx/nginx.conf`)

```nginx
upstream flask_api {
    server flask-api:5000;
}

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://flask_api;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

---

## 4. Environment Setup

### 3.1 VM Requirements

| Specification | Value |
|---|---|
| Hypervisor | VirtualBox 7.x |
| OS | Ubuntu 22.04 LTS (Server or Desktop) |
| RAM | 4 GB minimum |
| CPUs | 2 cores minimum |
| Disk | 50 GB |
| Network | NAT (default) |

### 3.2 Install Required Tools

Run the following commands on your Ubuntu VM. After each section, verify the installation.

#### Update System

```bash
sudo apt update && sudo apt upgrade -y
```

#### Install Git

```bash
sudo apt install -y git
```

**Verify:**
```bash
git --version
# Expected: git version 2.x.x
```

#### Install Docker

```bash
# Install prerequisites
sudo apt install -y ca-certificates curl gnupg lsb-release

# Add Docker GPG key
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Add Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add user to docker group (avoids using sudo for docker commands)
sudo usermod -aG docker $USER
newgrp docker
```

**Verify:**
```bash
docker --version
# Expected: Docker version 24.x.x or newer

docker compose version
# Expected: Docker Compose version v2.x.x
```

#### Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl
```

**Verify:**
```bash
kubectl version --client
# Expected: Client Version: v1.x.x
```

#### Install Minikube

```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube
rm minikube-linux-amd64
```

**Verify:**
```bash
minikube version
# Expected: minikube version: v1.x.x
```

### 3.3 Start Minikube

```bash
minikube start --driver=docker --memory=2048 --cpus=2
```

**Verify:**
```bash
minikube status
# Expected: host: Running, kubelet: Running, apiserver: Running

kubectl get nodes
# Expected: minikube   Ready    control-plane   ...
```

---

## 5. Directory Structure

Create the following directory structure on your VM:

```
~/assignment3/
├── app/
│   ├── nginx/
│   │   ├── Dockerfile
│   │   └── nginx.conf
│   ├── flask-api/
│   │   ├── Dockerfile
│   │   ├── app.py
│   │   └── requirements.txt
│   └── docker-compose.yml
├── k8s/
│   ├── namespace.yml
│   ├── mysql-secret.yml
│   ├── mysql-pv.yml
│   ├── mysql-pvc.yml
│   ├── mysql-deployment.yml
│   ├── mysql-service.yml
│   ├── flask-configmap.yml
│   ├── flask-deployment.yml
│   ├── flask-service.yml
│   ├── nginx-configmap.yml
│   ├── nginx-deployment.yml
│   └── nginx-service.yml
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── .gitignore
├── start.sh
└── README.md
```

Create this structure:
```bash
mkdir -p ~/assignment3/{app/{nginx,flask-api},k8s,.github/workflows}
```

---

## 6. Assignment Tasks

---

### Part A: Git & Version Control (10 marks)

**Objective:** Demonstrate proper use of Git for version control.

#### Tasks:

1. **Configure Git and initialize a repository** in `~/assignment3/`
   ```bash
   # Configure your identity (required for commits)
   git config --global user.name "Your Name"
   git config --global user.email "your.email@example.com"

   cd ~/assignment3
   git init
   ```

2. **Create a `.gitignore` file** that excludes:
   - Python bytecode (`__pycache__/`, `*.pyc`)
   - Environment files (`.env`)
   - IDE folders (`.vscode/`, `.idea/`)
   - OS files (`.DS_Store`, `Thumbs.db`)

3. **Use a branching strategy:**
   - `main` branch for stable/final code
   - `develop` branch for integration
   - Feature branches (e.g., `feature/docker-setup`, `feature/k8s-deployment`)

4. **Make meaningful commits** — each commit should represent a logical unit of work with a descriptive message.

5. **Push to GitHub** — Create a repository on GitHub and push your code.

#### Marking Criteria:

| Criteria | Marks |
|---|---|
| Proper `.gitignore` | 2 |
| Branching strategy (at least 3 branches used) | 3 |
| Meaningful commit messages (minimum 10 commits) | 3 |
| Repository pushed to GitHub successfully | 2 |
| **Total** | **10** |

---

### Part B: Containerization with Docker (20 marks)

**Objective:** Containerize all services, configure networking and persistent storage using Docker.

#### Task B1: Dockerfiles (6 marks)

**Nginx Dockerfile** (`app/nginx/Dockerfile`):

Write a Dockerfile that:
- Uses `nginx:alpine` as base image
- Removes the default Nginx config (`/etc/nginx/conf.d/default.conf`)
- Copies your custom `nginx.conf` to `/etc/nginx/conf.d/default.conf`

Example:
```dockerfile
FROM nginx:alpine
RUN rm /etc/nginx/conf.d/default.conf
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

**Flask API Dockerfile** (`app/flask-api/Dockerfile`):

Write a Dockerfile that:
- Uses `python:3.11-slim` as base image
- Sets working directory to `/app`
- Copies and installs `requirements.txt` first (for Docker layer caching)
- Copies the rest of the application code
- Exposes port 5000
- Runs the application with `python app.py`

Example:
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

**MySQL**: Use the official `mysql:8.0` image (no custom Dockerfile needed).

#### Task B2: Docker Networking (4 marks)

Create a custom bridge network and demonstrate inter-container communication:

```bash
# Create a custom bridge network
docker network create assignment3-net

# Run containers on this network and verify they can communicate
```

**Demonstrate:** Containers can reach each other by name on the custom network.

#### Task B3: Docker Volumes (4 marks)

Create a named volume for MySQL data persistence:

```bash
# Create a named volume
docker volume create mysql-data

# Use it when running MySQL container
```

**Demonstrate:** Data survives container restart (insert data, stop container, restart, verify data exists).

#### Task B4: Docker Compose (6 marks)

Write a `docker-compose.yml` that:
- Defines all 3 services (nginx, flask-api, mysql)
- Uses a custom bridge network
- Uses a named volume for MySQL
- Sets environment variables for MySQL (root password, database, user, password)
- Configures proper `depends_on` ordering
- Maps port 80 on the host to Nginx

**Expected `docker-compose.yml` structure:**

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: flaskdb
      MYSQL_USER: flaskuser
      MYSQL_PASSWORD: flaskpass
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  flask-api:
    build: ./flask-api
    environment:
      DB_HOST: mysql
      DB_USER: flaskuser
      DB_PASSWORD: flaskpass
      DB_NAME: flaskdb
    depends_on:
      mysql:
        condition: service_healthy
    networks:
      - app-network

  nginx:
    build: ./nginx
    ports:
      - "80:80"
    depends_on:
      - flask-api
    networks:
      - app-network

networks:
  app-network:
    driver: bridge

volumes:
  mysql-data:
```

#### Verification:

```bash
cd ~/assignment3/app
docker compose up -d

# Test the application
curl http://localhost/health
# Expected: {"service":"flask-api","status":"healthy"}

curl -X POST http://localhost/api/items \
  -H "Content-Type: application/json" \
  -d '{"name": "Test Item", "description": "This is a test"}'
# Expected: {"id":1,"name":"Test Item","description":"This is a test"}

curl http://localhost/api/items
# Expected: List containing the item you just created

# Test persistence - restart and verify data
docker compose down      # Stops and removes containers, but NOT volumes
docker compose up -d     # Recreates containers, volume data is preserved
curl http://localhost/api/items
# Data should still be there because the named volume persists

# NOTE: "docker compose down -v" WOULD delete volumes — don't use -v flag here
```

#### Marking Criteria:

| Criteria | Marks |
|---|---|
| Nginx Dockerfile correct | 2 |
| Flask API Dockerfile correct | 4 |
| Custom bridge network created and functional | 4 |
| Named volume for MySQL with data persistence | 4 |
| Docker Compose orchestrates all services correctly | 6 |
| **Total** | **20** |

---

### Part C: CI/CD with GitHub Actions (15 marks)

**Objective:** Automate Docker image builds and push to DockerHub using GitHub Actions.

#### Prerequisites:

1. Create a free account on [DockerHub](https://hub.docker.com)
2. Create a DockerHub access token (Account Settings > Security > New Access Token)
3. Add secrets to your GitHub repository:
   - Go to Repository Settings > Secrets and variables > Actions
   - Add `DOCKERHUB_USERNAME` (your DockerHub username)
   - Add `DOCKERHUB_TOKEN` (your access token)

#### Task: Create CI/CD Workflow

Create `.github/workflows/ci-cd.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push Flask API image
        uses: docker/build-push-action@v5
        with:
          context: ./app/flask-api
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/flask-api:latest

      - name: Build and push Nginx image
        uses: docker/build-push-action@v5
        with:
          context: ./app/nginx
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/nginx-proxy:latest
```

#### Verification:

- Push to main branch and verify the workflow runs successfully (green checkmark in GitHub Actions tab)
- Verify images appear on your DockerHub account
- Take screenshots of: successful workflow run, DockerHub repositories showing the images

#### Marking Criteria:

| Criteria | Marks |
|---|---|
| Workflow triggers on push to main | 2 |
| DockerHub login step configured correctly | 3 |
| Flask API image built and pushed | 4 |
| Nginx image built and pushed | 4 |
| GitHub Secrets used (no hardcoded credentials) | 2 |
| **Total** | **15** |

---

### Part D: Kubernetes Orchestration on Minikube (55 marks)

**Objective:** Deploy and manage the complete application on Kubernetes using Minikube.

> **Important:** In your Kubernetes manifests, use the DockerHub images you pushed via GitHub Actions.
> Replace `<your-dockerhub-username>` with your actual DockerHub username in all manifests below.

---

#### D1: Namespace & Deployments (10 marks)

**Create a namespace:**

`k8s/namespace.yml`:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: assignment3
```

**Create Deployment manifests for all 3 services.**

Each Deployment should:
- Be in the `assignment3` namespace
- Have appropriate labels (e.g., `app: flask-api`)
- Specify resource requests/limits
- Use your DockerHub images for Flask and Nginx
- Use the official `mysql:8.0` image for MySQL

**Example structure for Flask API Deployment** (`k8s/flask-deployment.yml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-api
  namespace: assignment3
  labels:
    app: flask-api
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-api
  template:
    metadata:
      labels:
        app: flask-api
    spec:
      containers:
        - name: flask-api
          image: <your-dockerhub-username>/flask-api:latest
          ports:
            - containerPort: 5000
          env:
            - name: DB_HOST
              valueFrom:
                configMapKeyRef:
                  name: flask-config
                  key: db_host
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: mysql_user
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: mysql_password
            - name: DB_NAME
              valueFrom:
                configMapKeyRef:
                  name: flask-config
                  key: db_name
          readinessProbe:
            httpGet:
              path: /health
              port: 5000
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /health
              port: 5000
            initialDelaySeconds: 30
            periodSeconds: 10
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "250m"
```

> **Explanation of Probes:**
> - **readinessProbe**: Kubernetes won't send traffic to the Pod until this check passes. Prevents errors while Flask is still connecting to MySQL.
> - **livenessProbe**: Kubernetes restarts the Pod if this check fails repeatedly. Recovers from deadlocks or crashes.
> - `initialDelaySeconds`: Wait time before first check (gives app time to start).
> - `periodSeconds`: How often to repeat the check.

**Complete MySQL Deployment** (`k8s/mysql-deployment.yml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
  namespace: assignment3
  labels:
    app: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
        - name: mysql
          image: mysql:8.0
          ports:
            - containerPort: 3306
          env:
            - name: MYSQL_ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: mysql_root_password
            - name: MYSQL_DATABASE
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: mysql_database
            - name: MYSQL_USER
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: mysql_user
            - name: MYSQL_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mysql-secret
                  key: mysql_password
          volumeMounts:
            - name: mysql-storage
              mountPath: /var/lib/mysql
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
      volumes:
        - name: mysql-storage
          persistentVolumeClaim:
            claimName: mysql-pvc
```

**Complete Nginx Deployment** (`k8s/nginx-deployment.yml`):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: assignment3
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: <your-dockerhub-username>/nginx-proxy:latest
          ports:
            - containerPort: 80
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d
          resources:
            requests:
              memory: "64Mi"
              cpu: "50m"
            limits:
              memory: "128Mi"
              cpu: "100m"
      volumes:
        - name: nginx-config
          configMap:
            name: nginx-config
```

> **Note:** The Nginx Deployment mounts the `nginx-config` ConfigMap (defined in Section D4/Tips) to override the baked-in configuration with one that uses the Kubernetes service name.

**Verification:**
```bash
kubectl apply -f k8s/namespace.yml
kubectl get namespaces
# assignment3 should appear

# After applying all manifests:
kubectl get deployments -n assignment3
# All deployments should show READY
```

#### Marking Criteria:

| Criteria | Marks |
|---|---|
| Namespace created correctly | 1 |
| MySQL Deployment correct | 3 |
| Flask API Deployment correct | 3 |
| Nginx Deployment correct | 3 |
| **Total** | **10** |

---

#### D2: Services & Networking (10 marks)

Create Kubernetes Service manifests for each component. Choose the appropriate Service type based on the component's role:

| Service | Type | Reason |
|---|---|---|
| MySQL | ClusterIP | Internal-only access (database should never be exposed externally) |
| Flask API | ClusterIP | Internal access (Nginx proxies to it) |
| Nginx | NodePort | External access (entry point for users) |

**MySQL Service** (`k8s/mysql-service.yml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
  namespace: assignment3
spec:
  type: ClusterIP
  selector:
    app: mysql
  ports:
    - port: 3306
      targetPort: 3306
```

**Flask API Service** (`k8s/flask-service.yml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: flask-api
  namespace: assignment3
spec:
  type: ClusterIP
  selector:
    app: flask-api
  ports:
    - port: 5000
      targetPort: 5000
```

**Nginx Service** (`k8s/nginx-service.yml`):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx
  namespace: assignment3
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080
```

**In your report, explain:**
- What is the difference between ClusterIP, NodePort, and LoadBalancer service types?
- Why did you choose each type for each service?

**Verification:**
```bash
kubectl get services -n assignment3
# Should show all 3 services with correct types and ports
```

#### Marking Criteria:

| Criteria | Marks |
|---|---|
| MySQL ClusterIP Service correct | 3 |
| Flask API ClusterIP Service correct | 3 |
| Nginx NodePort Service correct | 3 |
| Explanation of service types in report | 1 |
| **Total** | **10** |

---

#### D3: Persistent Storage (10 marks)

Create PersistentVolume and PersistentVolumeClaim for MySQL data.

**PersistentVolume** (`k8s/mysql-pv.yml`):
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  storageClassName: manual
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /data/mysql
```

> **Explanation:** `storageClassName: manual` ensures the PV and PVC bind to each other explicitly (not to Minikube's default dynamic provisioner). `hostPath` stores data on the Minikube node's filesystem. `Retain` means the data is preserved even if the PVC is deleted.

**PersistentVolumeClaim** (`k8s/mysql-pvc.yml`):
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
  namespace: assignment3
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

> **Important:** The `storageClassName` must match between PV and PVC for them to bind. If you omit this, Minikube's default StorageClass may dynamically provision a different volume and your PV will remain unbound.

**In your MySQL Deployment**, reference the PVC:
```yaml
# Inside the container spec:
volumeMounts:
  - name: mysql-storage
    mountPath: /var/lib/mysql

# Inside the pod spec:
volumes:
  - name: mysql-storage
    persistentVolumeClaim:
      claimName: mysql-pvc
```

**Verification:**
```bash
kubectl get pv
# mysql-pv should show STATUS: Bound

kubectl get pvc -n assignment3
# mysql-pvc should show STATUS: Bound

# Test persistence:
# 1. Add data via the API
# 2. Delete the MySQL pod: kubectl delete pod <mysql-pod-name> -n assignment3
# 3. Wait for pod recreation
# 4. Verify data still exists
```

#### Marking Criteria:

| Criteria | Marks |
|---|---|
| PersistentVolume created correctly | 3 |
| PersistentVolumeClaim created correctly | 3 |
| MySQL Deployment uses PVC correctly | 2 |
| Data survives pod deletion (demonstrated in report) | 2 |
| **Total** | **10** |

---

#### D4: ConfigMaps & Secrets (10 marks)

Store configuration and sensitive data properly using Kubernetes ConfigMaps and Secrets.

**Secret for database credentials** (`k8s/mysql-secret.yml`):
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mysql-secret
  namespace: assignment3
type: Opaque
data:
  mysql_root_password: cm9vdHBhc3M=    # base64 encoded "rootpass"
  mysql_user: Zmxhc2t1c2Vy            # base64 encoded "flaskuser"
  mysql_password: Zmxhc2twYXNz        # base64 encoded "flaskpass"
  mysql_database: Zmxhc2tkYg==        # base64 encoded "flaskdb"
```

> **Note:** Encode values using: `echo -n 'value' | base64`

**ConfigMap for application configuration** (`k8s/flask-configmap.yml`):
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: flask-config
  namespace: assignment3
data:
  db_host: "mysql"
  db_name: "flaskdb"
  app_env: "production"
```

**Reference these in your Deployments** using `envFrom` or individual `env` entries with `valueFrom` (as shown in the D1 Flask example).

**MySQL Deployment environment example:**
```yaml
env:
  - name: MYSQL_ROOT_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mysql-secret
        key: mysql_root_password
  - name: MYSQL_DATABASE
    valueFrom:
      secretKeyRef:
        name: mysql-secret
        key: mysql_database
  - name: MYSQL_USER
    valueFrom:
      secretKeyRef:
        name: mysql-secret
        key: mysql_user
  - name: MYSQL_PASSWORD
    valueFrom:
      secretKeyRef:
        name: mysql-secret
        key: mysql_password
```

**Verification:**
```bash
kubectl get secrets -n assignment3
kubectl get configmaps -n assignment3
kubectl describe secret mysql-secret -n assignment3
```

#### Marking Criteria:

| Criteria | Marks |
|---|---|
| Secret created with base64-encoded values | 3 |
| ConfigMap created with app configuration | 3 |
| Deployments reference Secret correctly | 2 |
| Deployments reference ConfigMap correctly | 2 |
| **Total** | **10** |

---

#### D5: Scaling & Updates (10 marks)

Demonstrate Kubernetes scaling and update capabilities.

**Scaling:**

```bash
# Scale Flask API to 3 replicas
kubectl scale deployment flask-api --replicas=3 -n assignment3

# Verify
kubectl get pods -n assignment3 -l app=flask-api
# Should show 3 pods running
```

**Rolling Update:**

```bash
# Update the Flask API image (simulate a new version)
kubectl set image deployment/flask-api flask-api=<your-dockerhub-username>/flask-api:v2 -n assignment3

# Watch the rolling update
kubectl rollout status deployment/flask-api -n assignment3
```

> **How to create a v2 image for this demonstration:**
> 1. Make a small change to `app/flask-api/app.py` (e.g., change the health response to `'version': 'v2'`)
> 2. Rebuild and push: `docker build -t <your-dockerhub-username>/flask-api:v2 ./app/flask-api && docker push <your-dockerhub-username>/flask-api:v2`
> 3. Or update your GitHub Actions workflow to also tag with a version number, then push to trigger the pipeline
>
> Alternatively, you can use any valid public image tag (e.g., `python:3.11-slim`) just to demonstrate the rolling update mechanism, even if the app won't work with that image — the rollback will restore it.

**Rollback:**

```bash
# Check rollout history
kubectl rollout history deployment/flask-api -n assignment3

# Rollback to previous version
kubectl rollout undo deployment/flask-api -n assignment3

# Verify rollback
kubectl rollout status deployment/flask-api -n assignment3
kubectl describe deployment flask-api -n assignment3 | grep Image
```

**Verification & Screenshots required:**
- Show 3 Flask pods running after scaling
- Show rolling update in progress
- Show successful rollback

#### Marking Criteria:

| Criteria | Marks |
|---|---|
| Successfully scaled Flask to 3 replicas | 3 |
| Rolling update performed | 3 |
| Rollback performed successfully | 3 |
| Screenshots and explanation in report | 1 |
| **Total** | **10** |

---

#### D6: End-to-End Verification (5 marks)

Demonstrate the complete system working.

**Access the application:**

```bash
# Get the Minikube service URL
minikube service nginx -n assignment3 --url
# Use this URL to access the application

# Or use NodePort directly
curl http://$(minikube ip):30080/health
curl http://$(minikube ip):30080/api/items
```

**Test full functionality:**

```bash
# Health check
curl http://$(minikube ip):30080/health

# Create items
curl -X POST http://$(minikube ip):30080/api/items \
  -H "Content-Type: application/json" \
  -d '{"name": "Kubernetes Item", "description": "Created on K8s"}'

# List items
curl http://$(minikube ip):30080/api/items
```

**Demonstrate self-healing:**

```bash
# Get pod name
kubectl get pods -n assignment3 -l app=flask-api

# Delete a pod
kubectl delete pod <flask-pod-name> -n assignment3

# Watch Kubernetes recreate it
kubectl get pods -n assignment3 -l app=flask-api -w
# A new pod should appear almost immediately
```

**Demonstrate data persistence:**

```bash
# Delete MySQL pod
kubectl delete pod <mysql-pod-name> -n assignment3

# Wait for recreation
kubectl get pods -n assignment3 -l app=mysql -w

# Verify data still exists
curl http://$(minikube ip):30080/api/items
# Previously created items should still be there
```

#### Marking Criteria:

| Criteria | Marks |
|---|---|
| Application accessible via NodePort | 2 |
| Self-healing demonstrated (pod recreation) | 1.5 |
| Data persistence after pod deletion | 1.5 |
| **Total** | **5** |

---

## 7. Startup Script

Create `~/assignment3/start.sh` that your instructor will use to start the application:

```bash
#!/bin/bash

# Change to the assignment directory
cd ~/assignment3

echo "Starting Minikube..."
minikube start --driver=docker --memory=2048 --cpus=2

echo "Waiting for Minikube to be ready..."
kubectl wait --for=condition=Ready node/minikube --timeout=120s

echo "Applying Kubernetes manifests..."
kubectl apply -f k8s/namespace.yml
sleep 2
kubectl apply -f k8s/mysql-secret.yml
kubectl apply -f k8s/flask-configmap.yml
kubectl apply -f k8s/mysql-pv.yml
kubectl apply -f k8s/mysql-pvc.yml
kubectl apply -f k8s/mysql-deployment.yml
kubectl apply -f k8s/mysql-service.yml

echo "Waiting for MySQL to be ready..."
kubectl wait --for=condition=Ready pod -l app=mysql -n assignment3 --timeout=120s

kubectl apply -f k8s/flask-deployment.yml
kubectl apply -f k8s/flask-service.yml
kubectl apply -f k8s/nginx-configmap.yml
kubectl apply -f k8s/nginx-deployment.yml
kubectl apply -f k8s/nginx-service.yml

echo "Waiting for all pods to be ready..."
kubectl wait --for=condition=Ready pod --all -n assignment3 --timeout=180s

echo ""
echo "=== Deployment Status ==="
kubectl get all -n assignment3

echo ""
echo "=== Access URL ==="
minikube service nginx -n assignment3 --url
```

Make it executable:
```bash
chmod +x ~/assignment3/start.sh
```

> **Why this order matters:**
> 1. **Namespace first** — all other resources are created inside it
> 2. **Secrets & ConfigMaps before Deployments** — Pods reference these at startup; if they don't exist, Pods will fail to create
> 3. **PV before PVC** — PVC needs a PV to bind to
> 4. **MySQL before Flask** — Flask needs the database to be available
> 5. **Flask before Nginx** — Nginx proxies to Flask (though Nginx will retry if Flask isn't ready)

---

## 8. Submission Instructions

### What to Submit:

1. **VM Snapshot (.ova file)**
   - Before exporting: stop Minikube with `minikube stop`
   - In VirtualBox: File > Export Appliance > Select your VM > Export as `.ova`
   - File naming: `RollNumber_Assignment3.ova`

2. **PDF Report** (submit separately)
   - File naming: `RollNumber_Assignment3_Report.pdf`

### VM Requirements Before Export:

- [ ] All code is in `~/assignment3/` directory
- [ ] `start.sh` is executable and works
- [ ] Minikube is stopped (`minikube stop`)
- [ ] Docker images are available locally
- [ ] Git repository has full commit history
- [ ] GitHub repository is accessible (make it public or invite instructor)

### PDF Report Must Include:

1. **Architecture Diagram** — Visual diagram showing all components and their connections
2. **Part A:** Screenshots of git log, branches, GitHub repository
3. **Part B:** Screenshots of Docker containers running, network inspection, volume persistence test
4. **Part C:** Screenshots of GitHub Actions workflow (successful run), DockerHub repositories
5. **Part D:** Screenshots for each sub-task:
   - Running pods, deployments, services
   - PV/PVC bound status
   - ConfigMap and Secret listings
   - Scaling output
   - Rolling update and rollback
   - End-to-end curl outputs
   - Self-healing demonstration
6. **YAML Explanations** — Brief explanation of each Kubernetes manifest (what it does and why)

---

## 9. Evaluation Rubric Summary

| Section | Marks |
|---|---|
| **Part A:** Git & Version Control | 10 |
| **Part B:** Containerization with Docker | 20 |
| **Part C:** CI/CD with GitHub Actions | 15 |
| **Part D:** Kubernetes Orchestration | 55 |
| - D1: Namespace & Deployments | (10) |
| - D2: Services & Networking | (10) |
| - D3: Persistent Storage | (10) |
| - D4: ConfigMaps & Secrets | (10) |
| - D5: Scaling & Updates | (10) |
| - D6: End-to-End Verification | (5) |
| **Total** | **100** |

### Instructor Evaluation Process:

1. Import `.ova` into VirtualBox
2. Start the VM
3. Run `~/assignment3/start.sh`
4. Verify all pods are running
5. Access application via NodePort
6. Test CRUD operations
7. Test persistence (delete MySQL pod, verify data)
8. Check Git history and GitHub repository
9. Verify GitHub Actions workflow ran successfully
10. Review Kubernetes manifests for correctness

---

## 10. Important Notes & Tips

### Resource Management
- Minikube uses Docker as driver — ensure Docker is running before starting Minikube
- If your VM runs slow, close unnecessary applications
- Use `minikube dashboard` for a visual interface to debug issues

### Common Pitfalls
- **MySQL takes time to initialize** — On first start, MySQL creates the database, user, and runs initialization. This can take 30-60 seconds. The Flask app has retry logic (30 retries × 2 seconds) to handle this. In Kubernetes, the readiness probe prevents traffic until the app is ready.
- **Image pull errors** — Ensure DockerHub images are public (or set the repository visibility to public on DockerHub). If private, you'd need `imagePullSecrets` in K8s (not required for this assignment — just make images public).
- **Nginx can't reach Flask** — In Kubernetes, update `nginx.conf` to use the Kubernetes service name (`flask-api`) as the upstream server. Use the Nginx ConfigMap (shown below) to override the config.
- **PVC stuck in Pending** — Make sure: (1) PV is created first, (2) `storageClassName` matches between PV and PVC, (3) capacity and access modes are compatible.
- **Base64 encoding** — Use `echo -n` (no newline) when encoding secrets: `echo -n 'value' | base64`. If you accidentally include a newline (`echo` without `-n`), the password will have a trailing `\n` and MySQL auth will fail.
- **MySQL authentication error** — MySQL 8.0 uses `caching_sha2_password` by default. The `mysql-connector-python==8.2.0` supports this. If you face auth issues, verify your Secret values are correctly base64-encoded.
- **Flask crashes on startup** — If MySQL isn't ready yet, the `init_db()` call will retry. Check Flask pod logs with `kubectl logs <pod> -n assignment3`. The retry logic gives MySQL up to 60 seconds to become available.
- **Minikube out of memory** — If pods stay in `Pending` or `CrashLoopBackOff`, check `kubectl describe pod <pod> -n assignment3` for resource issues. Consider reducing resource requests in your manifests.

### Nginx ConfigMap for Kubernetes

Since the Nginx configuration needs to reference the Kubernetes service name, create a ConfigMap:

`k8s/nginx-configmap.yml`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: assignment3
data:
  default.conf: |
    upstream flask_api {
        server flask-api:5000;
    }
    server {
        listen 80;
        server_name localhost;
        location / {
            proxy_pass http://flask_api;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
    }
```

Mount this in your Nginx Deployment:
```yaml
volumeMounts:
  - name: nginx-config
    mountPath: /etc/nginx/conf.d
volumes:
  - name: nginx-config
    configMap:
      name: nginx-config
```

### Verification Checklist

Before submitting, verify each section:

- [ ] `git log --oneline` shows 10+ meaningful commits
- [ ] `git branch -a` shows multiple branches
- [ ] `docker compose up -d` starts all containers (from `app/` directory)
- [ ] `curl http://localhost/health` returns healthy (Docker Compose)
- [ ] GitHub Actions shows green checkmark
- [ ] DockerHub has your images
- [ ] `kubectl get all -n assignment3` shows all resources running
- [ ] `curl http://$(minikube ip):30080/health` returns healthy (Kubernetes)
- [ ] Data persists after MySQL pod deletion
- [ ] Flask scales to 3 replicas
- [ ] Rollback works

### Useful Commands Reference

```bash
# Docker
docker ps                          # List running containers
docker logs <container>            # View container logs
docker network ls                  # List networks
docker volume ls                   # List volumes

# Kubernetes
kubectl get all -n assignment3     # All resources in namespace
kubectl logs <pod> -n assignment3  # Pod logs
kubectl describe pod <pod> -n assignment3  # Pod details
kubectl exec -it <pod> -n assignment3 -- /bin/bash  # Shell into pod
kubectl port-forward svc/nginx 8080:80 -n assignment3  # Alternative access

# Minikube
minikube status                    # Check Minikube status
minikube dashboard                 # Open web dashboard
minikube ssh                       # SSH into Minikube node
minikube stop                      # Stop Minikube
minikube delete                    # Delete Minikube cluster (use with caution)
```

---

**Good luck! Start early and test frequently.**
