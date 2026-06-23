# Cloud / DevOps Interview Questions

**Your Self-Assessment:** 6/10  
**Focus:** Build from basics, practical scenarios, GCP/AWS patterns (your weak areas)

---

## Question 1: Docker Multi-Stage Builds

**Difficulty:** Intermediate  
**Category:** Gap Identification

What is a Docker multi-stage build? Write an optimized Dockerfile for a Python FastAPI application.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Definition:**

A **multi-stage build** uses multiple `FROM` statements in a single Dockerfile. Each stage can use a different base image, and you can copy artifacts from one stage to another. This results in smaller, more secure final images.

**Without Multi-Stage (Large Image):**

```dockerfile
# Single stage - includes build tools in final image
FROM python:3.11

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y gcc musl-dev
RUN pip install --no-cache-dir fastapi uvicorn

# Copy source
COPY . .

# Build (if needed)
RUN python -m compileall .

CMD ["uvicorn", "main:app", "--host", "0.0.0.0"]
# Final image: ~1.2 GB (includes gcc, build tools, etc.)
```

**With Multi-Stage (Optimized):**

```dockerfile
# Stage 1: Build dependencies
FROM python:3.11-slim AS builder

WORKDIR /app

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc \
    musl-dev \
    && rm -rf /var/lib/apt/lists/*

# Create virtual environment
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt

# Stage 2: Runtime
FROM python:3.11-slim

WORKDIR /app

# Copy virtual environment from builder
COPY --from=builder /opt/venv /opt/venv

# Copy application code
COPY . .

# Set environment
ENV PATH="/opt/venv/bin:$PATH" \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

# Create non-root user
RUN useradd -m -u 1000 appuser && \
    chown -R appuser:appuser /app
USER appuser

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')" || exit 1

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
# Final image: ~200 MB (only runtime dependencies)
```

**Benefits:**

1. **Smaller Images:** Build tools not included in final image
2. **Better Security:** Fewer packages = smaller attack surface
3. **Layer Caching:** Dependencies cached separately from code
4. **Faster Deploys:** Smaller images = faster pulls

**Key Optimization Techniques:**

```dockerfile
# 1. Use slim/alpine base images
FROM python:3.11-slim  # vs python:3.11 (saves ~800MB)

# 2. Order layers by change frequency (least to most)
# Build cache invalidates from changed layer down
COPY requirements.txt .          # Changes rarely
RUN pip install -r requirements.txt
COPY . .                         # Changes frequently

# 3. Combine RUN commands to reduce layers
RUN apt-get update && \
    apt-get install -y package1 package2 && \
    rm -rf /var/lib/apt/lists/*

# 4. Use .dockerignore to exclude unnecessary files
# .dockerignore:
# __pycache__
# *.pyc
# .git
# .env
# tests/
# *.md
# .vscode
```

**Complete Production Example:**

```dockerfile
# syntax=docker/dockerfile:1.7

# Build stage
FROM python:3.11-slim AS builder

WORKDIR /build

# Install build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    gcc=12.2.0-14 \
    musl-dev=1.2.3-r1 \
    && rm -rf /var/lib/apt/lists/*

# Create venv
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir \
    --upgrade pip \
    wheel \
    && pip install --no-cache-dir -r requirements.txt

# Runtime stage
FROM python:3.11-slim AS runtime

# Install runtime dependencies only
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl=7.88.1-10 \
    && rm -rf /var/lib/apt/lists/*

# Copy venv
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

WORKDIR /app

# Copy application
COPY --chown=appuser:appuser . .

# Create and switch to non-root user
RUN useradd --create-home --shell /bin/bash --uid 1000 appuser && \
    chown -R appuser:appuser /app
USER appuser

# Environment
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PORT=8000

EXPOSE 8000

# Use exec form for proper signal handling
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "4"]
```

**Key Concepts:**
- Multi-stage builds reduce final image size
- Use slim/alpine base images
- Order layers by change frequency (cache optimization)
- Combine RUN commands to reduce layers
- Use `.dockerignore` to exclude unnecessary files
- Run as non-root user for security
- Use exec form for CMD/ENTRYPOINT (proper signal handling)

**Common Mistakes:**
- Including build tools in final image (use multi-stage)
- Not using `.dockerignore` (large build context)
- Running as root (security risk)
- Not pinning package versions (reproducibility)
- Copying entire project before installing dependencies (cache invalidation)
- Using shell form for CMD (poor signal handling)

**Interview Tip:**
> "I use multi-stage builds to keep final images small. Stage 1 installs build dependencies and creates a virtual environment. Stage 2 copies only the venv and application code, resulting in a ~200MB image vs 1.2GB. I order layers by change frequency—dependencies first, code last—to maximize cache reuse. I always run as non-root user and use `.dockerignore` to exclude unnecessary files."

</details>

---

## Question 2: Kubernetes Pods, Deployments, and Services

**Difficulty:** Intermediate  
**Category:** Learning

Explain Kubernetes core concepts: Pods, Deployments, Services, and Ingress.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Pods:**

The smallest deployable unit. Contains one or more containers that share network and storage.

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  containers:
  - name: app
    image: myapp:1.0
    ports:
    - containerPort: 8000
    resources:
      requests:
        memory: "256Mi"
        cpu: "250m"
      limits:
        memory: "512Mi"
        cpu: "500m"
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: url
```

**Deployments:**

Manages a replicated set of Pods. Handles rolling updates and rollbacks.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: myapp:1.0
        ports:
        - containerPort: 8000
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 5
          periodSeconds: 5
```

**Services:**

Expose Pods to network traffic. Provides stable IP and DNS.

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  selector:
    app: my-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8000
  type: ClusterIP  # Internal only
  # type: LoadBalancer  # External (cloud provider)
  # type: NodePort  # Expose on each node
```

**Ingress:**

Routes external HTTP/HTTPS traffic to Services.

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app-service
            port:
              number: 80
```

**ConfigMap and Secrets:**

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  LOG_LEVEL: "INFO"
  MAX_CONNECTIONS: "100"

# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  # Base64 encoded
  url: cG9zdGdyZXM6Ly91c2VyOnBhc3NAdGJjbG91ZC9teWRi
```

**Key Concepts:**
- Pod: Smallest unit, contains containers
- Deployment: Manages replicated Pods
- Service: Stable network endpoint for Pods
- Ingress: External HTTP/HTTPS routing
- ConfigMap: Non-sensitive configuration
- Secret: Sensitive data (base64 encoded)
- Rolling updates: Zero-downtime deployments
- Health checks: Liveness (restart) and readiness (traffic)

**Common Mistakes:**
- Not setting resource limits (resource starvation)
- No health checks (broken pods not detected)
- Using NodePort in production (security)
- Not using namespaces (resource conflicts)
- Hardcoding config (use ConfigMap/Secret)

**Interview Tip:**
> "Kubernetes has four core concepts: Pods (smallest unit), Deployments (manage replicas), Services (stable network), and Ingress (external routing). I always set resource requests/limits to prevent resource starvation, use health checks (liveness and readiness) for automatic recovery, and use ConfigMap/Secret for configuration. For production, I use ClusterIP services with Ingress, not NodePort, for security."

</details>

---

## Question 3: CI/CD Pipeline Design

**Difficulty:** Advanced  
**Category:** Learning

Design a CI/CD pipeline for a microservices application. What are the key stages?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Pipeline Stages:**

```
Code → Build → Test → Security Scan → Deploy to Staging → Integration Tests → Deploy to Production → Monitor
```

**GitHub Actions Example:**

```yaml
# .github/workflows/cicd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # Stage 1: Lint and Format
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install black flake8 mypy
      
      - name: Run black
        run: black --check .
      
      - name: Run flake8
        run: flake8 . --max-line-length=100
      
      - name: Run mypy
        run: mypy .

  # Stage 2: Unit Tests
  test:
    runs-on: ubuntu-latest
    needs: lint
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: testpass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-cov
      
      - name: Run tests
        env:
          DATABASE_URL: postgresql://postgres:testpass@localhost:5432/testdb
        run: pytest --cov=. --cov-report=xml --cov-report=html
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage.xml

  # Stage 3: Security Scan
  security:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
      
      - name: Upload Trivy results
        if: always()
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
      
      - name: Run Bandit (Python security)
        run: |
          pip install bandit
          bandit -r . -f json -o bandit-results.json
      
      - name: Check secrets
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.ref }}
          head: HEAD

  # Stage 4: Build and Push Docker Image
  build:
    runs-on: ubuntu-latest
    needs: security
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  # Stage 5: Deploy to Staging
  deploy-staging:
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: staging
      url: https://staging.example.com
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          method: kubeconfig
          kubeconfig: ${{ secrets.KUBE_CONFIG_STAGING }}
      
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/my-app \
            my-app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -n staging
          kubectl rollout status deployment/my-app -n staging

  # Stage 6: Integration Tests
  integration-test:
    runs-on: ubuntu-latest
    needs: deploy-staging
    steps:
      - uses: actions/checkout@v4
      
      - name: Run integration tests
        run: |
          pip install pytest requests
          pytest tests/integration/ --base-url=https://staging.example.com

  # Stage 7: Deploy to Production
  deploy-production:
    runs-on: ubuntu-latest
    needs: integration-test
    environment:
      name: production
      url: https://example.com
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy with canary (10% traffic)
        run: |
          kubectl apply -f k8s/canary.yaml
          kubectl set image deployment/my-app-canary \
            my-app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -n production
          
          # Monitor for 5 minutes
          sleep 300
          
          # Check error rate
          ERROR_RATE=$(kubectl logs -l app=my-app-canary -n production --tail=1000 | grep -c "ERROR")
          if [ $ERROR_RATE -lt 10 ]; then
            # Promote canary to full deployment
            kubectl set image deployment/my-app \
              my-app=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
              -n production
            kubectl delete deployment/my-app-canary -n production
          else
            # Rollback
            kubectl rollout undo deployment/my-app -n production
            exit 1
          fi
```

**Key Concepts:**
- Automated testing (unit, integration, e2e)
- Security scanning (vulnerabilities, secrets)
- Container image building and versioning
- Progressive deployment (canary, blue-green)
- Health checks and automatic rollback
- Environment isolation (dev, staging, prod)
- Secrets management

**Common Mistakes:**
- No automated tests (manual testing bottlenecks)
- Deploying directly to production (no staging)
- No rollback strategy (can't recover from bad deploys)
- Not scanning for vulnerabilities (security risk)
- Hardcoded secrets in code (security breach)
- No monitoring post-deploy (blind to issues)

**Interview Tip:**
> "My CI/CD pipeline has 7 stages: lint, test, security scan, build, deploy to staging, integration tests, and deploy to production. I use GitHub Actions with matrix builds for multiple services. For production, I use canary deployments—deploy to 10% of traffic, monitor for 5 minutes, then promote or rollback. The key is automation, security scanning, and progressive deployment with automatic rollback."

</details>

---

## Question 4: AWS Lambda Deployment

**Difficulty:** Intermediate  
**Category:** Learning

How do you deploy a Python FastAPI application to AWS Lambda? Explain the serverless pattern.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Option 1: Lambda with API Gateway (REST)**

```python
# handler.py
from mangum import Mangum
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
    return {"Hello": "World"}

@app.get("/items/{item_id}")
def read_item(item_id: int):
    return {"item_id": item_id}

# Mangum adapts ASGI (FastAPI) to Lambda
handler = Mangum(app)
```

**Deployment with SAM:**

```yaml
# template.yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  FastAPIFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: ./
      Handler: handler.handler
      Runtime: python3.11
      MemorySize: 512
      Timeout: 30
      Environment:
        Variables:
          DATABASE_URL: !Ref DatabaseURL
      Events:
        Api:
          Type: Api
          Properties:
            Path: /{proxy+}
            Method: ANY
```

**Deploy:**

```bash
sam build
sam deploy --guided
```

**Option 2: Lambda with ALB (Container Support)**

```yaml
# template.yaml
Resources:
  FastAPIFunction:
    Type: AWS::Serverless::Function
    Properties:
      PackageType: Image
      ImageUri: ./
      MemorySize: 1024
      Timeout: 60
```

**Dockerfile for Lambda:**

```dockerfile
FROM public.ecr.aws/lambda/python:3.11

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

CMD ["handler.handler"]
```

**Option 3: Lambda Function URLs (Simpler)**

```python
# handler.py with Function URL
def lambda_handler(event, context):
    # Parse event
    path = event.get(\"rawPath\", \"/\")
    method = event.get(\"requestContext\", {}).get(\"http\", {}).get(\"method\", \"GET\")
    body = event.get(\"body\", \"\")
    
    # Your logic
    return {
        \"statusCode\": 200,
        \"body\": json.dumps({\"message\": \"Hello from Lambda\"}),
        \"headers\": {\"Content-Type\": \"application/json\"}
    }
```

**Cold Start Optimization:**

```python
# 1. Keep package small
# 2. Use Lambda Layers for dependencies
# 3. Provisioned Concurrency for critical functions
# 4. Initialize outside handler

import boto3
from fastapi import FastAPI

# Initialize once (cold start)
app = FastAPI()
s3_client = boto3.client('s3')
db_pool = create_db_pool()

# This is created on cold start
@app.get(\"/\")
def read_root():
    # Use initialized resources
    return {\"data\": s3_client.list_buckets()}
```

**Key Concepts:**
- Mangum adapts ASGI/WSGI to Lambda
- Cold starts: first invocation is slow
- API Gateway or ALB for HTTP triggers
- Lambda Layers for shared dependencies
- Provisioned Concurrency eliminates cold starts
- Pay only for execution time (cost-effective)

**Your E-Commerce Project (Go microservices on Lambda):**

This matches the pattern used in your project. Key considerations:
- Use Mangum or AWS Lambda Web Adapter
- Keep deployment package small
- Use layers for shared dependencies
- Monitor cold starts with CloudWatch

**Common Mistakes:**
- Not optimizing for cold starts (slow first request)
- Large deployment packages (slower cold starts)
- No provisioned concurrency for critical functions
- Not using layers (duplicate dependencies)
- Synchronous long-running tasks (use Step Functions)

**Interview Tip:**
> "I deploy FastAPI to Lambda using Mangum to adapt ASGI to Lambda's event model. For deployment, I use SAM (Serverless Application Model) which simplifies packaging and infrastructure. To optimize cold starts, I keep packages small, use Lambda Layers for shared dependencies, and enable Provisioned Concurrency for critical functions. The trade-off is cold start latency vs cost—Lambda is cost-effective for sporadic workloads but EC2/ECS is better for consistent high traffic."

</details>

---

## Question 5: GCP Cloud Run Deployment

**Difficulty:** Intermediate  
**Category:** Strength Validation

How do you deploy a containerized application to GCP Cloud Run? What's the architecture?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Architecture:**

```
Internet → Cloud Load Balancer → Cloud Run (auto-scaling containers) → Cloud SQL/Other Services
```

**Dockerfile:**

```dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# Non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

# Cloud Run uses PORT env variable
ENV PORT=8080
EXPOSE 8080

CMD [\"uvicorn\", \"main:app\", \"--host\", \"0.0.0.0\", \"--port\", \"8080\"]
```

**Deploy with gcloud:**

```bash
# 1. Set project
gcloud config set project my-project-id

# 2. Enable APIs
gcloud services enable run.googleapis.com \
    sqladmin.googleapis.com \
    containerregistry.googleapis.com

# 3. Build and push to Container Registry
gcloud builds submit --tag gcr.io/my-project-id/my-app

# 4. Deploy to Cloud Run
gcloud run deploy my-app \\
    --image gcr.io/my-project-id/my-app \\
    --platform managed \\
    --region us-central1 \\
    --allow-unauthenticated \\
    --memory 512Mi \\
    --cpu 1 \\
    --min-instances 0 \\
    --max-instances 10 \\
    --concurrency 80 \\
    --timeout 60 \\
    --set-env-vars DATABASE_URL=$DATABASE_URL \\
    --set-secrets DB_PASSWORD=db-password:latest

# 5. Get URL
gcloud run services describe my-app --region us-central1 --format='value(status.url)'
```

**Deploy with Cloud Build (CI/CD):**

```yaml
# cloudbuild.yaml
steps:
  # Build image
  - name: 'gcr.io/cloud-builders/docker'
    args: ['build', '-t', 'gcr.io/$PROJECT_ID/my-app:$COMMIT_SHA', '.']

  # Push to Container Registry
  - name: 'gcr.io/cloud-builders/docker'
    args: ['push', 'gcr.io/$PROJECT_ID/my-app:$COMMIT_SHA']

  # Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    entrypoint: gcloud
    args:
      - 'run'
      - 'deploy'
      - 'my-app'
      - '--image'
      - 'gcr.io/$PROJECT_ID/my-app:$COMMIT_SHA'
      - '--region'
      - 'us-central1'
      - '--platform'
      - 'managed'

images:
  - 'gcr.io/$PROJECT_ID/my-app:$COMMIT_SHA'

options:
  logging: CLOUD_LOGGING_ONLY
```

**Connect to Cloud SQL:**

```bash
# Option 1: Direct connection (requires Cloud SQL Auth Proxy)
gcloud run deploy my-app \\
    --add-cloudsql-instances my-project:us-central1:mydb \\
    --update-env-vars INSTANCE_CONNECTION_NAME=my-project:us-central1:mydb

# Option 2: Use Unix socket in code
import os

def get_db_connection():
    # Cloud Run with Cloud SQL Proxy
    unix_socket = f'/cloudsql/{os.environ[\"INSTANCE_CONNECTION_NAME\"]}'
    conn = psycopg2.connect(
        host=unix_socket,
        database=os.environ[\"DB_NAME\"],
        user=os.environ[\"DB_USER\"],
        password=os.environ[\"DB_PASSWORD\"]
    )
    return conn
```

**Terraform Deployment:**

```hcl
# main.tf
resource \"google_cloud_run_service\" \"my_app\" {
  name     = \"my-app\"
  location = \"us-central1\"

  template {
    spec {
      containers {
        image = \"gcr.io/my-project-id/my-app:latest\"
        
        resources {
          limits {
            cpu    = \"1\"
            memory = \"512Mi\"
          }
        }
        
        env {
          name  = \"DATABASE_URL\"
          value = var.database_url
        }
        
        env {
          name = \"DB_PASSWORD\"
          value_from {
            secret_key_ref {
              name = google_secret_manager_secret.db_password.secret_id
              key  = \"latest\"
            }
          }
        }
      }
    }
  }
  
  traffic {
    percent         = 100
    latest_revision = true
  }
}

# Allow public access
resource \"google_cloud_run_service_iam_member\" \"public\" {
  service  = google_cloud_run_service.my_app.name
  location = google_cloud_run_service.my_app.location
  role     = \"roles/run.invoker\"
  member   = \"allUsers\"
}
```

**Key Concepts:**
- Cloud Run: Serverless containers, auto-scaling
- Pay only for actual usage (per request)
- Auto-scales from 0 to N instances
- Built-in HTTPS, custom domains
- Integrates with Cloud SQL, Pub/Sub, etc.
- Cold starts (slower first request)
- Max 60 minutes per request

**Your E-Commerce Project:**

This pattern matches your e-commerce microservices architecture. Benefits:
- Auto-scaling based on traffic
- Pay per use (cost-effective for variable traffic)
- No infrastructure management
- Built-in HTTPS and load balancing

**Common Mistakes:**
- Not setting max-instances (cost explosion)
- Storing secrets in env vars (use Secret Manager)
- Not connecting to Cloud SQL properly (use Auth Proxy)
- No health checks (broken containers not detected)
- Not setting concurrency (resource exhaustion)

**Interview Tip:**
> "I deploy containerized apps to Cloud Run for serverless auto-scaling. The Dockerfile uses Python 3.11-slim, and I deploy with `gcloud run deploy` or Cloud Build for CI/CD. I set resource limits, max instances (cost control), and concurrency. For Cloud SQL, I use the Cloud SQL Auth Proxy or Unix sockets. Cloud Run is perfect for variable traffic—scales to zero when idle, scales up under load, and you pay only for actual usage."

</details>

---

## Question 6: Kubernetes Networking

**Difficulty:** Advanced  
**Category:** Learning

How do services communicate in Kubernetes? Explain DNS, Services, and Ingress.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Service Types:**

**1. ClusterIP (Default - Internal Only):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
```

**Communication:** `http://backend-service:80` (within cluster)

**2. NodePort (Exposed on Node IP):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: NodePort
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30000  # 30000-32767
```

**Communication:** `http://<node-ip>:30000` (external)

**3. LoadBalancer (Cloud Load Balancer):**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 8080
```

**Communication:** Cloud provider creates external LB

**DNS Resolution:**

Kubernetes automatically creates DNS records for Services:

```bash
# Format: <service-name>.<namespace>.svc.cluster.local
# Short form (same namespace): <service-name>
# Across namespaces: <service-name>.<namespace>

# Examples:
backend-service                    # Same namespace
backend-service.default            # Default namespace
backend-service.default.svc        # Full form
backend-service.default.svc.cluster.local  # FQDN
```

**Pod-to-Pod Communication:**

```yaml
# Pods can communicate using Service DNS
apiVersion: v1
kind: Pod
metadata:
  name: client-pod
spec:
  containers:
  - name: client
    image: curlimages/curl
    command: ['sh', '-c', 'curl http://backend-service:80/api']
```

**Ingress (External HTTP/HTTPS Routing):**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      # Frontend - serves React app
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
      
      # API routes to backend
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-service
            port:
              number: 80
      
      # WebSocket endpoint
      - path: /ws
        pathType: Prefix
        backend:
          service:
            name: websocket-service
            port:
              number: 80
```

**Network Policies (Security):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-policy
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow traffic from frontend
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # Allow DNS
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: UDP
      port: 53
  # Allow database
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
```

**Service Mesh (Istio):**

```yaml
# With Istio, services can have advanced features
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: backend-vs
spec:
  hosts:
  - backend-service
  http:
  - match:
    - headers:
        x-version:
          exact: v2
    route:
    - destination:
        host: backend-service
        subset: v2
  - route:
    - destination:
        host: backend-service
        subset: v1
      weight: 90
    - destination:
        host: backend-service
        subset: v2
      weight: 10  # Canary deployment
```

**Key Concepts:**
- ClusterIP: Internal cluster communication
- NodePort: Expose on node IP (dev/testing)
- LoadBalancer: Cloud provider LB (production)
- Ingress: HTTP/HTTPS routing with TLS
- DNS: Automatic service discovery
- Network Policies: Micro-segmentation
- Service Mesh: Advanced traffic management

**Common Mistakes:**
- Using NodePort in production (security)
- Not using Network Policies (no isolation)
- Hardcoding pod IPs (use Service DNS)
- Not setting up Ingress (multiple LoadBalancers expensive)
- Missing health checks (broken pods get traffic)

**Interview Tip:**
> "Kubernetes networking has three layers: Pod-to-Pod (via Services), Service-to-Service (via DNS), and External (via Ingress). I use ClusterIP for internal services, LoadBalancer for external APIs, and Ingress for HTTP/HTTPS routing with TLS. I always set Network Policies for security—default deny with explicit allow rules. For production, I use a service mesh like Istio for canary deployments and observability."

</details>

---

## Question 7: Container Orchestration Comparison

**Difficulty:** Intermediate  
**Category:** Learning

Compare Kubernetes, Docker Swarm, and AWS ECS. When would you use each?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Comparison:**

| Feature | Kubernetes | Docker Swarm | AWS ECS |
|---------|-----------|--------------|---------|
| **Complexity** | High | Low | Medium |
| **Learning curve** | Steep | Gentle | Medium |
| **Ecosystem** | Huge | Limited | AWS-only |
| **Auto-scaling** | Excellent | Basic | Good |
| **Service mesh** | Istio, Linkerd | Limited | App Mesh |
| **Multi-cloud** | Yes | Yes | No (AWS only) |
| **Community** | Massive | Small | Large (AWS) |
| **Best for** | Complex apps | Simple apps | AWS ecosystem |

**Kubernetes:**

```yaml
# Pros:
# - Industry standard
# - Massive ecosystem (Helm, Istio, ArgoCD)
# - Powerful auto-scaling
# - Multi-cloud portability
# - Advanced networking (Network Policies, Ingress)

# Cons:
# - Complex setup
# - Steep learning curve
# - Resource-intensive
# - Overkill for simple apps

# Use when:
# - Complex microservices (10+ services)
# - Multi-cloud strategy
# - Need advanced features (service mesh, GitOps)
# - Large team with K8s expertise
```

**Docker Swarm:**

```bash
# Initialize swarm
docker swarm init

# Deploy service
docker service create \\
    --name web \\
    --replicas 3 \\
    --publish 80:80 \\
    nginx

# Scale
docker service scale web=5

# Pros:
# - Simple setup (built into Docker)
# - Easy to learn
# - Good for small deployments
# - Low resource overhead

# Cons:
# - Limited ecosystem
# - Fewer features
# - Declining popularity
# - No advanced networking

# Use when:
# - Simple applications
# - Small team
# - Quick deployment
# - Docker-only environment
```

**AWS ECS:**

```json
// task-definition.json
{
  "family": "my-app",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "my-app:latest",
      "memory": 512,
      "cpu": 256,
      "essential": true,
      "portMappings": [
        {
          "containerPort": 8000,
          "protocol": "tcp"
        }
      ]
    }
  ]
}
```

```bash
# Create service
aws ecs create-service \\
    --cluster my-cluster \\
    --service-name my-app \\
    --task-definition my-app:1 \\
    --desired-count 3 \\
    --launch-type FARGATE

# Pros:
# - AWS-native (tight integration)
# - Fargate (no server management)
# - Good AWS ecosystem integration
# - Simpler than Kubernetes

# Cons:
# - AWS-only (vendor lock-in)
# - Limited multi-cloud
# - Smaller ecosystem than K8s

# Use when:
# - AWS-centric infrastructure
# - Want serverless containers (Fargate)
# - Already using AWS services heavily
# - Need AWS-specific integrations
```

**Decision Framework:**

```python
# Use Kubernetes when:
# - 10+ microservices
# - Multi-cloud or hybrid cloud
# - Need service mesh (Istio)
# - Large team with K8s expertise
# - Complex deployment strategies (canary, blue-green)

# Use Docker Swarm when:
# - 1-5 simple services
# - Small team
# - Quick deployment priority
# - Don't need advanced features

# Use AWS ECS when:
# - AWS-centric infrastructure
# - Want Fargate (no server management)
# - 5-20 services
# - Tight AWS integration needed
# - Don't need multi-cloud
```

**Your E-Commerce Project:**

Your e-commerce project uses Go microservices deployed to AWS Lambda. This is another option:
- Lambda: Serverless functions (not containers)
- Best for: Event-driven, sporadic workloads
- Cost: Pay per execution

**Key Concepts:**
- Kubernetes: Industry standard, complex, powerful
- Docker Swarm: Simple, limited, declining
- ECS: AWS-native, Fargate, good for AWS shops
- Choice depends on scale, team expertise, cloud strategy

**Common Mistakes:**
- Choosing Kubernetes for simple apps (overkill)
- Using Docker Swarm for complex apps (insufficient)
- Vendor lock-in with ECS (can't migrate easily)
- Not considering serverless (Lambda, Cloud Run)
- Ignoring operational complexity

**Interview Tip:**
> "I choose the orchestrator based on scale and complexity. For complex microservices (10+ services) with multi-cloud needs, I use Kubernetes. For simple deployments (1-5 services) with small teams, I use Docker Swarm. For AWS-centric shops, I use ECS with Fargate. For event-driven workloads, I use Lambda. The key is matching the tool to the use case—Kubernetes is powerful but overkill for simple apps."

</details>

---

## Question 8: Monitoring and Logging

**Difficulty:** Advanced  
**Category:** Learning

Design a monitoring and logging strategy for a production microservices application.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**The Three Pillars:**

1. **Metrics:** Numerical data (CPU, latency, error rate)
2. **Logs:** Event records (application logs, access logs)
3. **Traces:** Request flow across services

**Metrics with Prometheus + Grafana:**

```yaml
# prometheus-config.yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'fastapi-app'
    static_configs:
      - targets: ['app:8000']
    metrics_path: /metrics

# Application exposes metrics
```

```python
# FastAPI app with Prometheus metrics
from prometheus_client import Counter, Histogram, generate_latest
from prometheus_fastapi_instrumentator import Instrumentator

app = FastAPI()

# Add automatic instrumentation
Instrumentator().instrument(app).expose(app)

# Custom metrics
request_count = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint']
)

@app.middleware(\"http\")
async def track_metrics(request, call_next):
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start
    
    request_count.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()
    
    request_duration.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(duration)
    
    return response
```

**Centralized Logging with ELK Stack:**

```python
# Structured JSON logging
import logging
import json
from pythonjsonlogger import jsonlogger

# Configure
logHandler = logging.StreamHandler()
formatter = jsonlogger.JsonFormatter()
logHandler.setFormatter(formatter)
logger = logging.getLogger()
logger.addHandler(logHandler)
logger.setLevel(logging.INFO)

# Log structured data
logger.info(
    \"Request processed\",
    extra={
        \"user_id\": \"user-123\",
        \"request_id\": \"req-456\",
        \"duration_ms\": 245,
        \"endpoint\": \"/api/users\"
    }
)
```

**Docker Compose for ELK:**

```yaml
# docker-compose.logging.yml
version: '3.8'
services:
  elasticsearch:
    image: elasticsearch:8.5.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - \"9200:9200\"
    volumes:
      - es-data:/usr/share/elasticsearch/data
  
  logstash:
    image: logstash:8.5.0
    volumes:
      - ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf
    ports:
      - \"5000:5000\"
    depends_on:
      - elasticsearch
  
  kibana:
    image: kibana:8.5.0
    ports:
      - \"5601:5601\"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    depends_on:
      - elasticsearch

volumes:
  es-data:
```

**Distributed Tracing with Jaeger:**

```python
# OpenTelemetry instrumentation
from opentelemetry import trace
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.trace import TracerProvider

# Setup
trace.set_tracer_provider(TracerProvider())
jaeger_exporter = JaegerExporter(
    agent_host_name=\"jaeger\",
    agent_port=6831,
)
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

# Instrument FastAPI
FastAPIInstrumentor.instrument_app(app)
RequestsInstrumentor().instrument()

# Custom spans
tracer = trace.get_tracer(__name__)

@app.get(\"/users/{user_id}\")
async def get_user(user_id: str):
    with tracer.start_as_current_span(\"get_user\") as span:
        span.set_attribute(\"user.id\", user_id)
        
        # Database query (will be traced)
        user = await db.get_user(user_id)
        return user
```

**Alerting:**

```yaml
# prometheus-alerts.yml
groups:
- name: app_alerts
  rules:
  - alert: HighErrorRate
    expr: |
      rate(http_requests_total{status=~\"5..\"}[5m]) > 0.05
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: \"High error rate detected\"
      description: \"Error rate is {{ $value }} (threshold: 0.05)\"
  
  - alert: HighLatency
    expr: |
      histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: \"High p95 latency\"
      description: \"p95 latency is {{ $value }}s\"
  
  - alert: PodCrashLooping
    expr: rate(kube_pod_container_status_restarts_total[15m]) > 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: \"Pod is crash looping\"
```

**Key Components:**

1. **Prometheus + Grafana:** Metrics and dashboards
2. **ELK Stack:** Centralized logging
3. **Jaeger/Zipkin:** Distributed tracing
4. **PagerDuty/Opsgenie:** Alerting and on-call
5. **Datadog/New Relic:** All-in-one SaaS

**Key Concepts:**
- Metrics: Numerical time-series data
- Logs: Discrete event records
- Traces: Request flow across services
- Use structured logging (JSON)
- Set up alerts for SLO violations
- Monitor golden signals (latency, traffic, errors, saturation)

**Common Mistakes:**
- No structured logging (hard to query)
- Missing distributed tracing (can't debug cross-service)
- Too many alerts (alert fatigue)
- Not monitoring golden signals
- Storing logs forever (expensive)
- No runbooks for alerts

**Interview Tip:**
> "I use the three pillars of observability: Prometheus + Grafana for metrics, ELK stack for centralized logging, and Jaeger for distributed tracing. I instrument applications with OpenTelemetry for automatic tracing. I set up alerts for golden signals (latency, traffic, errors, saturation) with proper thresholds to avoid alert fatigue. The key is structured logging, end-to-end tracing, and actionable alerts with runbooks."

</details>

---

## Question 9: Infrastructure as Code (Terraform)

**Difficulty:** Advanced  
**Category:** Learning

Demonstrate Terraform for deploying a FastAPI application to AWS with RDS, ECS, and ALB.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Project Structure:**

```
terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── modules/
│   ├── vpc/
│   ├── ecs/
│   ├── rds/
│   └── alb/
└── environments/
    ├── dev/
    ├── staging/
    └── prod/
```

**VPC Module:**

```hcl
# modules/vpc/main.tf
resource \"aws_vpc\" \"main\" {
  cidr_block           = \"10.0.0.0/16\"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = \"${var.project}-vpc\"
  }
}

# Public subnets (for ALB)
resource \"aws_subnet\" \"public\" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = \"10.0.${count.index}.0/24\"
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
  
  tags = {
    Name = \"${var.project}-public-${count.index + 1}\"
  }
}

# Private subnets (for ECS, RDS)
resource \"aws_subnet\" \"private\" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = \"10.0.${count.index + 10}.0/24\"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  tags = {
    Name = \"${var.project}-private-${count.index + 1}\"
  }
}

# Internet Gateway
resource \"aws_internet_gateway\" \"main\" {
  vpc_id = aws_vpc.main.id
}

# NAT Gateway (for private subnet internet access)
resource \"aws_eip\" \"nat\" {
  domain = \"vpc\"
}

resource \"aws_nat_gateway\" \"main\" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public[0].id
}

# Route tables
resource \"aws_route_table\" \"public\" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = \"0.0.0.0/0\"
    gateway_id = aws_internet_gateway.main.id
  }
}

resource \"aws_route_table\" \"private\" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block     = \"0.0.0.0/0\"
    nat_gateway_id = aws_nat_gateway.main.id
  }
}
```

**RDS Module:**

```hcl
# modules/rds/main.tf
resource \"aws_db_subnet_group\" \"main\" {
  name       = \"${var.project}-db-subnet\"
  subnet_ids = var.private_subnet_ids
}

resource \"aws_security_group\" \"rds\" {
  name        = \"${var.project}-rds-sg\"
  description = \"RDS security group\"
  vpc_id      = var.vpc_id
  
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = \"tcp\"
    security_groups = [var.ecs_security_group_id]
  }
}

resource \"aws_db_instance\" \"main\" {
  identifier             = \"${var.project}-db\"
  engine                 = \"postgres\"
  engine_version         = \"15.4\"
  instance_class         = var.db_instance_class
  allocated_storage      = 20
  max_allocated_storage  = 100
  storage_encrypted      = true
  
  db_name  = var.db_name
  username = var.db_username
  password = var.db_password
  
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
  
  backup_retention_period = 7
  backup_window           = \"03:00-04:00\"
  maintenance_window      = \"Mon:00:00-Mon:03:00\"
  
  skip_final_snapshot = false
  final_snapshot_identifier = \"${var.project}-db-final\"
  
  tags = {
    Name = \"${var.project}-db\"
  }
}
```

**ECS Module:**

```hcl
# modules/ecs/main.tf
resource \"aws_ecs_cluster\" \"main\" {
  name = \"${var.project}-cluster\"
  
  setting {
    name  = \"containerInsights\"
    value = \"enabled\"
  }
}

resource \"aws_ecs_task_definition\" \"app\" {
  family                   = \"${var.project}-app\"
  network_mode             = \"awsvpc\"
  requires_compatibilities = [\"FARGATE\"]
  cpu                      = \"512\"
  memory                   = \"1024\"
  execution_role_arn       = aws_iam_role.ecs_execution.arn
  task_role_arn            = aws_iam_role.ecs_task.arn
  
  container_definitions = jsonencode([
    {
      name      = \"app\"
      image     = var.container_image
      essential = true
      
      portMappings = [
        {
          containerPort = 8000
          protocol      = \"tcp\"
        }
      ]
      
      environment = [
        { name = \"DATABASE_URL\", value = \"postgresql://${var.db_username}:${var.db_password}@${var.db_endpoint}/${var.db_name}\" },
        { name = \"LOG_LEVEL\", value = \"INFO\" }
      ]
      
      logConfiguration = {
        logDriver = \"awslogs\"
        options = {
          awslogs-group         = \"/ecs/${var.project}\"
          awslogs-region        = var.aws_region
          awslogs-stream-prefix = \"ecs\"
        }
      }
    }
  ])
}

resource \"aws_ecs_service\" \"main\" {
  name            = \"${var.project}-service\"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.desired_count
  launch_type     = \"FARGATE\"
  
  network_configuration {
    subnets          = var.private_subnet_ids
    security_groups  = [aws_security_group.ecs.id]
    assign_public_ip = false
  }
  
  load_balancer {
    target_group_arn = var.target_group_arn
    container_name   = \"app\"
    container_port   = 8000
  }
  
  depends_on = [var.alb_listener_arn]
}
```

**Main Configuration:**

```hcl
# main.tf
module \"vpc\" {
  source = \"./modules/vpc\"
  project = \"myapp\"
}

module \"rds\" {
  source              = \"./modules/rds\"
  project             = \"myapp\"
  vpc_id              = module.vpc.vpc_id
  private_subnet_ids  = module.vpc.private_subnet_ids
  ecs_security_group_id = module.ecs.security_group_id
  db_instance_class   = \"db.t3.micro\"
  db_name             = \"myapp\"
  db_username         = \"admin\"
  db_password         = var.db_password
}

module \"alb\" {
  source             = \"./modules/alb\"
  project            = \"myapp\"
  vpc_id             = module.vpc.vpc_id
  public_subnet_ids  = module.vpc.public_subnet_ids
}

module \"ecs\" {
  source                = \"./modules/ecs\"
  project               = \"myapp\"
  vpc_id                = module.vpc.vpc_id
  private_subnet_ids    = module.vpc.private_subnet_ids
  container_image       = var.container_image
  db_endpoint           = module.rds.endpoint
  db_name               = \"myapp\"
  db_username           = \"admin\"
  db_password           = var.db_password
  target_group_arn      = module.alb.target_group_arn
  alb_listener_arn      = module.alb.listener_arn
  desired_count         = 2
  aws_region            = \"us-east-1\"
}
```

**Workflow:**

```bash
# Initialize
terraform init

# Plan (preview changes)
terraform plan -out=tfplan

# Apply
terraform apply tfplan

# Show outputs
terraform output

# Destroy (careful!)
terraform destroy
```

**Key Concepts:**
- Infrastructure as Code (IaC)
- Modules for reusability
- State management (remote backend)
- Variables for environment-specific config
- Outputs for cross-module references
- Plan before apply (preview changes)
- Version control your Terraform code

**Common Mistakes:**
- Not using remote state (team collaboration issues)
- Hardcoding secrets (use variables/Secrets Manager)
- Not using modules (code duplication)
- No environment separation (dev/prod)
- Skipping terraform plan (surprise changes)

**Interview Tip:**
> "I use Terraform for Infrastructure as Code with a modular structure. I create separate modules for VPC, RDS, ECS, and ALB, then compose them in main configuration. I use remote state (S3 + DynamoDB) for team collaboration, variables for environment-specific values, and always run `terraform plan` before apply. The key is modularity, version control, and state management. For secrets, I use AWS Secrets Manager, not Terraform variables."

</details>

---

## Question 10: Secrets Management

**Difficulty:** Intermediate  
**Category:** Learning

How do you manage secrets in production? Compare environment variables, secrets managers, and Vault.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Comparison:**

| Method | Security | Convenience | Use Case |
|--------|----------|-------------|----------|
| **Environment Variables** | Low | High | Dev only |
| **Docker Secrets** | Medium | Medium | Swarm/K8s |
| **AWS Secrets Manager** | High | High | AWS workloads |
| **HashiCorp Vault** | Highest | Medium | Multi-cloud, complex |
| **Kubernetes Secrets** | Medium | High | K8s only |

**Bad: Hardcoded Secrets:**

```python
# NEVER DO THIS
DATABASE_URL = \"postgresql://user:password123@db.com/mydb\"
OPENAI_API_KEY = \"sk-...\"  # Hardcoded!
```

**Better: Environment Variables:**

```python
import os

DATABASE_URL = os.getenv(\"DATABASE_URL\")
OPENAI_API_KEY = os.getenv(\"OPENAI_API_KEY\")
```

**Issue:** Visible in process list, not encrypted at rest, no rotation

**Best: AWS Secrets Manager:**

```python
import boto3
import json

def get_secret(secret_name: str) -> dict:
    \"\"\"Fetch secret from AWS Secrets Manager\"\"\"
    client = boto3.client('secretsmanager')
    
    try:
        response = client.get_secret_value(SecretId=secret_name)
        return json.loads(response['SecretString'])
    except Exception as e:
        raise Exception(f\"Failed to fetch secret: {e}\")

# Usage
secrets = get_secret(\"myapp/database\")
DATABASE_URL = secrets[\"url\"]
```

**With Caching:**

```python
from functools import lru_cache
import time

class SecretsManager:
    def __init__(self):
        self.client = boto3.client('secretsmanager')
        self.cache = {}
        self.cache_ttl = 3600  # 1 hour
    
    def get_secret(self, secret_name: str) -> dict:
        # Check cache
        if secret_name in self.cache:
            value, timestamp = self.cache[secret_name]
            if time.time() - timestamp < self.cache_ttl:
                return value
        
        # Fetch from AWS
        response = self.client.get_secret_value(SecretId=secret_name)
        secret = json.loads(response['SecretString'])
        
        # Cache
        self.cache[secret_name] = (secret, time.time())
        return secret

# Singleton
secrets_manager = SecretsManager()

# Usage
db_config = secrets_manager.get_secret(\"myapp/database\")
```

**HashiCorp Vault:**

```python
import hvac

# Initialize client
client = hvac.Client(
    url='https://vault.example.com',
    token='s.xxxxx'  # From auth method
)

# Read secret
response = client.secrets.kv.v2.read_secret_version(
    path='myapp/database',
    mount_point='secret'
)

DATABASE_URL = response['data']['data']['url']
```

**Kubernetes Secrets:**

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  url: postgresql://user:password@db:5432/mydb
  api-key: sk-...
```

```python
# In pod, mount as environment variables
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: myapp:latest
        envFrom:
        - secretRef:
            name: db-secret
```

**Better: External Secrets Operator:**

```yaml
# Syncs from AWS Secrets Manager to K8s Secret
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secret
spec:
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-secret  # K8s secret name
  data:
  - secretKey: url
    remoteRef:
      key: myapp/database
      property: url
```

**Rotation:**

```python
# AWS Secrets Manager automatic rotation
import boto3

def rotate_secret(secret_name: str):
    client = boto3.client('secretsmanager')
    
    # Configure rotation
    client.rotate_secret(
        SecretId=secret_name,
        RotationLambdaARN='arn:aws:lambda:...:function:rotate-secret',
        RotationRules={
            'AutomaticallyAfterDays': 30
        }
    )
```

**Best Practices:**

```python
# 1. Never log secrets
logger.info(f\"Connecting to database\")  # Don't log URL!

# 2. Use IAM roles, not access keys (AWS)
# In EC2/ECS/Lambda, use IAM roles
# No need for AWS_ACCESS_KEY_ID environment variables

# 3. Encrypt at rest
# AWS Secrets Manager encrypts by default
# K8s Secrets should be encrypted with KMS

# 4. Audit access
# CloudTrail logs all Secrets Manager access
# Vault audit logs

# 5. Principle of least privilege
# Only fetch secrets you need
# Don't pass secrets between services
```

**Key Concepts:**
- Never hardcode secrets in code
- Use environment variables for dev only
- Use secrets managers for production (AWS, Vault)
- Rotate secrets regularly (30-90 days)
- Audit access (CloudTrail, Vault logs)
- Use IAM roles, not access keys
- Encrypt secrets at rest
- Mount secrets, don't pass them around

**Common Mistakes:**
- Hardcoding secrets in code
- Committing .env files to Git
- Not rotating secrets
- Using long-lived access keys
- Logging secrets in application logs
- Not encrypting secrets at rest
- Passing secrets in URLs/query params

**Interview Tip:**
> "I use AWS Secrets Manager for production secrets. The application fetches secrets at startup with caching (1-hour TTL) to reduce API calls. I enable automatic rotation (30 days) and audit access via CloudTrail. For Kubernetes, I use External Secrets Operator to sync from AWS Secrets Manager to K8s Secrets. The key is never hardcoding secrets, using IAM roles instead of access keys, and rotating regularly. I never log secrets or pass them in URLs."

</details>

---

## Question 11: Auto-Scaling Strategies

**Difficulty:** Advanced  
**Category:** Learning

Compare auto-scaling strategies: reactive vs predictive vs scheduled. Implement auto-scaling for Kubernetes.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Auto-Scaling Types:**

**1. Reactive (Scale based on metrics):**

```yaml
# Kubernetes HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  # CPU-based
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  # Memory-based
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  # Custom metric (requests per second)
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: \"1000\"
  
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5min before scaling down
    scaleUp:
      stabilizationWindowSeconds: 0    # Scale up immediately
```

**2. Predictive (ML-based):**

```python
# Using Kubernetes with KEDA (predictive scaling)
# Based on event sources, schedules, or external metrics

# KEDA scaler for RabbitMQ queue length
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: rabbitmq-scaler
spec:
  scaleTargetRef:
    name: my-app
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
  - type: rabbitmq
    metadata:
      queueName: my-queue
      mode: QueueLength
      value: \"100\"  # Scale when queue > 100 messages
```

**3. Scheduled (Time-based):**

```yaml
# Scale up before known traffic spike
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: scheduled-scaler
spec:
  scaleTargetRef:
    name: my-app
  minReplicas: 2
  maxReplicas: 50
  # Use KEDA for scheduled scaling
```

```yaml
# With KEDA Cron scaler
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: cron-scaler
spec:
  scaleTargetRef:
    name: my-app
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
  - type: cron
    metadata:
      timezone: America/New_York
      start: 0 8 * * *    # 8 AM
      end: 0 18 * * *     # 6 PM
      desiredReplicas: \"10\"
```

**4. Cluster Autoscaler (Node-level):**

```yaml
# Add nodes when pods can't be scheduled
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler
data:
  config.yaml: |
    nodes:
      min: 1
      max: 10
```

**5. Vertical Pod Autoscaler (VPA):**

```yaml
# Adjust CPU/memory requests
apiVersion: autoscaling/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: \"Auto\"
```

**Comparison:**

| Strategy | Response Time | Cost | Complexity | Use Case |
|----------|--------------|------|------------|----------|
| **Reactive** | 1-5 min | Medium | Low | Variable traffic |
| **Predictive** | Proactive | Low | High | Known patterns |
| **Scheduled** | Proactive | Low | Low | Predictable spikes |
| **Cluster** | 2-10 min | High | Medium | Resource constraints |
| **Vertical** | Continuous | Optimal | Medium | Resource optimization |

**Best Practice: Combine Strategies**

```yaml
# Multi-layer scaling
# 1. Scheduled: Scale up before business hours
# 2. Reactive: Handle unexpected spikes
# 3. Cluster: Add nodes when needed
# 4. VPA: Optimize resource requests

# Example: E-commerce platform
# - Scheduled: Scale to 50 pods at 9 AM (business hours)
# - Reactive: Auto-scale based on CPU/RPS during the day
# - Cluster: Add nodes when all pods are scheduled
# - VPA: Continuously optimize memory requests
```

**Custom Metrics with Prometheus Adapter:**

```yaml
# prometheus-adapter-config.yaml
rules:
- seriesQuery: 'http_requests_total{namespace!=\"\"}'
  resources:
    overrides:
      namespace:
        resource: namespace
  name:
    matches: \"^(.*)_total$\"
    as: \"${1}_per_second\"
  metricsQuery: 'rate(<<.Series>>{<<.LabelMatchers>>}[2m])'
```

```yaml
# HPA using custom metric
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  scaleTargetRef:
    name: my-app
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Object
    object:
      metric:
        name: http_requests_per_second
      describedObject:
        apiVersion: v1
        kind: Service
        name: my-app-service
      target:
        type: Value
        value: \"1000\"  # Requests per second per pod
```

**Key Concepts:**
- Reactive: Scale based on current metrics (CPU, memory, RPS)
- Predictive: ML-based forecasting
- Scheduled: Time-based for known patterns
- Combine multiple strategies for optimal scaling
- Set proper min/max replicas
- Use stabilization windows to prevent flapping
- Monitor scaling events

**Common Mistakes:**
- Too aggressive scaling (cost explosion)
- No stabilization window (flapping)
- Setting min replicas to 0 (cold starts)
- Not monitoring scaling events
- Scaling based on single metric
- No cluster autoscaler (pods pending)

**Interview Tip:**
> "I use a multi-layer scaling strategy: scheduled for predictable traffic (scale up before business hours), reactive for unexpected spikes (CPU/RPS-based HPA), and cluster autoscaler for node provisioning. I set proper min/max replicas, use stabilization windows (5min for scale down) to prevent flapping, and monitor scaling events. For custom metrics, I use Prometheus Adapter to scale on business-specific metrics like requests per second or queue length. The key is combining strategies and setting appropriate thresholds."

</details>

---

## Question 12: Database Scaling Strategies

**Difficulty:** Advanced  
**Category:** Learning

Compare database scaling strategies: vertical scaling, read replicas, sharding, and connection pooling.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Scaling Strategies:**

**1. Vertical Scaling (Scale Up):**

```yaml
# AWS RDS - Increase instance size
# Before: db.t3.micro (1 vCPU, 1 GB RAM)
# After: db.r5.2xlarge (8 vCPU, 64 GB RAM)

# Pros: Simple, no code changes
# Cons: Expensive, single point of failure, hard limits
# Use for: Small to medium databases
```

**2. Read Replicas (Horizontal Read Scaling):**

```python
# Separate read and write connections
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

# Write (primary)
write_engine = create_engine(
    \"postgresql://user:pass@primary-db:5432/mydb\"
)

# Read (replicas)
read_engine = create_engine(
    \"postgresql://user:pass@replica-db:5432/mydb\"
)

# Router
class DatabaseRouter:
    def __init__(self):
        self.write_session = sessionmaker(bind=write_engine)
        self.read_session = sessionmaker(bind=read_engine)
    
    def get_session(self, operation: str):
        if operation in [\"read\", \"select\"]:
            return self.read_session()
        return self.write_session()

# Usage
db = DatabaseRouter()

# Read from replica
session = db.get_session(\"read\")
users = session.query(User).all()

# Write to primary
session = db.get_session(\"write\")
session.add(new_user)
session.commit()
```

**Pros:** Scales reads, simple to implement  
**Cons:** Eventual consistency, doesn't help writes  
**Use for:** Read-heavy workloads (90% reads, 10% writes)

**3. Sharding (Horizontal Write Scaling):**

```python
# Shard by user_id
def get_shard(user_id: int) -> int:
    return user_id % 4  # 4 shards

class ShardedDatabase:
    def __init__(self):
        self.shards = {
            0: create_engine(\"postgresql://user:pass@shard0:5432/mydb\"),
            1: create_engine(\"postgresql://user:pass@shard1:5432/mydb\"),
            2: create_engine(\"postgresql://user:pass@shard2:5432/mydb\"),
            3: create_engine(\"postgresql://user:pass@shard3:5432/mydb\"),
        }
    
    def get_session(self, user_id: int):
        shard_id = get_shard(user_id)
        return sessionmaker(bind=self.shards[shard_id])()
    
    def get_user(self, user_id: int):
        session = self.get_session(user_id)
        return session.query(User).filter(User.id == user_id).first()
    
    def create_user(self, user: User):
        session = self.get_session(user.id)
        session.add(user)
        session.commit()

# Pros: Scales writes, handles huge datasets
# Cons: Complex, cross-shard queries hard, rebalancing
# Use for: Very large scale (billions of rows)
```

**4. Connection Pooling:**

```python
# PgBouncer (most common)
# Configured at infrastructure level

# Application-level with SQLAlchemy
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

engine = create_engine(
    \"postgresql://user:pass@db:5432/mydb\",
    poolclass=QueuePool,
    pool_size=20,          # Connections to keep open
    max_overflow=10,       # Additional connections under load
    pool_pre_ping=True,    # Verify connections before use
    pool_recycle=3600,     # Recycle connections after 1 hour
)

# Usage (automatically pooled)
session = sessionmaker(bind=engine)()
```

**Pros:** Reduces connection overhead, better resource utilization  
**Cons:** Doesn't increase capacity, just efficiency  
**Use for:** All production apps

**5. Caching Layer (Redis):**

```python
# Cache frequently accessed data
import redis
import json

cache = redis.Redis(host='redis', port=6379, db=0)

def get_user(user_id: int):
    # Try cache first
    cached = cache.get(f\"user:{user_id}\")
    if cached:
        return json.loads(cached)
    
    # Fetch from database
    user = db.query(User).filter(User.id == user_id).first()
    
    # Cache for 1 hour
    cache.setex(
        f\"user:{user_id}\",
        3600,
        json.dumps(user.to_dict())
    )
    
    return user

# Cache invalidation
def update_user(user_id: int, data: dict):
    db.update(User).filter(User.id == user_id).update(data)
    db.commit()
    
    # Invalidate cache
    cache.delete(f\"user:{user_id}\")
```

**Pros:** Reduces database load dramatically, fast reads  
**Cons:** Cache invalidation complexity, consistency  
**Use for:** Read-heavy, infrequently changing data

**6. Database Partitioning:**

```sql
-- Partition by date
CREATE TABLE orders (
    id BIGSERIAL,
    user_id INT,
    created_at TIMESTAMP,
    total DECIMAL
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE orders_2026_01 PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE orders_2026_02 PARTITION OF orders
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- Indexes on each partition
CREATE INDEX ON orders_2026_01 (user_id);
CREATE INDEX ON orders_2026_02 (user_id);
```

**Pros:** Faster queries on large tables, easier maintenance  
**Cons:** Complex setup, limited to certain use cases  
**Use for:** Time-series data, large tables

**Decision Framework:**

```python
# Use Vertical Scaling when:
# - Database is small (< 100 GB)
# - Simple to manage
# - Can afford expensive hardware
# Examples: Small SaaS apps, startups

# Use Read Replicas when:
# - Read-heavy workload (90%+ reads)
# - Can tolerate eventual consistency
# - Need to scale reads without complexity
# Examples: Content sites, analytics dashboards

# Use Sharding when:
# - Massive scale (TB+ data, millions of writes/sec)
# - Single database can't handle load
# - Can handle complexity
# Examples: Social networks, IoT platforms

# Use Caching when:
# - Frequent reads of same data
# - Can tolerate slight staleness
# - Want to reduce database load
# Examples: User profiles, product catalogs, sessions

# Use Partitioning when:
# - Time-series or large tables
# - Queries often filter by partition key
# Examples: Logs, orders, events

# Combine strategies:
# - Caching + Read Replicas (most common)
# - Sharding + Caching (large scale)
# - Partitioning + Read Replicas (analytics)
```

**Key Concepts:**
- Vertical: Simple but expensive, has limits
- Read Replicas: Scales reads, eventual consistency
- Sharding: Scales writes, complex
- Connection Pooling: Efficiency, not capacity
- Caching: Reduces load, invalidation complexity
- Partitioning: Faster queries, time-series use cases

**Common Mistakes:**
- Not using connection pooling (connection exhaustion)
- Sharding too early (premature complexity)
- No caching (database overload)
- Read-after-write on replicas (consistency issues)
- Not monitoring database metrics
- Vertical scaling beyond reasonable limits

**Interview Tip:**
> "I use a layered approach: connection pooling (PgBouncer + SQLAlchemy) for efficiency, Redis caching for hot data, and read replicas for read-heavy workloads. For massive scale (TB+ data, millions of writes), I use sharding by user_id with application-level routing. I avoid sharding until necessary—it's complex. I monitor key metrics: connection count, query latency, cache hit rate, and replica lag. The key is starting simple (vertical + caching) and scaling complexity only when needed."

</details>

---

## Question 13: Zero-Downtime Deployments

**Difficulty:** Advanced  
**Category:** Learning

Implement zero-downtime deployment strategies: blue-green, canary, and rolling updates.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**1. Rolling Updates (Default in Kubernetes):**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Max pods above desired count
      maxUnavailable: 0  # Never less than desired count
  template:
    metadata:
      labels:
        app: my-app
        version: v2
    spec:
      containers:
      - name: app
        image: myapp:v2
```

**Process:**
```
Initial:  [v1] [v1] [v1] [v1]
Step 1:   [v1] [v1] [v1] [v2]  # Add v2
Step 2:   [v1] [v1] [v2] [v2]  # Remove v1
Step 3:   [v1] [v2] [v2] [v2]  # Remove v1
Step 4:   [v2] [v2] [v2] [v2]  # Remove v1
```

**Pros:** Simple, built-in, gradual  
**Cons:** Mixed versions during deployment, slow rollback  
**Use for:** Standard deployments

**2. Blue-Green Deployment:**

```yaml
# Blue (current production)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: blue
  template:
    metadata:
      labels:
        app: my-app
        version: blue
    spec:
      containers:
      - name: app
        image: myapp:v1

---
# Green (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
      version: green
  template:
    metadata:
      labels:
        app: my-app
        version: green
    spec:
      containers:
      - name: app
        image: myapp:v2

---
# Service routes to blue
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
    version: blue  # Switch to green to deploy
  ports:
  - port: 80
      targetPort: 8080
```

**Deployment Script:**

```bash
# Deploy green
kubectl apply -f green-deployment.yaml

# Wait for green to be ready
kubectl wait --for=condition=ready pod -l version=green --timeout=300s

# Run smoke tests
./scripts/smoke-test.sh green.myapp.com

# Switch traffic
kubectl patch service my-app -p '{\"spec\":{\"selector\":{\"version\":\"green\"}}}'

# Keep blue running for rollback
# If issues: kubectl patch service my-app -p '{\"spec\":{\"selector\":{\"version\":\"blue\"}}}'
```

**Pros:** Instant switch, easy rollback, full testing  
**Cons:** Requires double resources, database migrations tricky  
**Use for:** Critical applications, major releases

**3. Canary Deployment:**

```yaml
# Stable (90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: my-app
      track: stable
  template:
    metadata:
      labels:
        app: my-app
        track: stable
    spec:
      containers:
      - name: app
        image: myapp:v1

---
# Canary (10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      track: canary
  template:
    metadata:
      labels:
        app: my-app
        track: canary
    spec:
      containers:
      - name: app
        image: myapp:v2
```

**With Istio (Advanced Canary):**

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app
spec:
  hosts:
  - my-app
  http:
  - match:
    - headers:
        x-version:
          exact: v2
    route:
    - destination:
        host: my-app
        subset: v2
  - route:
    - destination:
        host: my-app
        subset: v1
      weight: 90
    - destination:
        host: my-app
        subset: v2
      weight: 10
```

**Deployment Process:**

```bash
# 1. Deploy canary (10% traffic)
kubectl apply -f canary-deployment.yaml

# 2. Monitor metrics for 5-10 minutes
./scripts/monitor.sh canary

# 3. Check error rate
ERROR_RATE=$(kubectl logs -l track=canary --tail=1000 | grep -c ERROR)

if [ $ERROR_RATE -lt 10 ]; then
  # 4a. Promote canary to stable
  kubectl set image deployment/my-app-stable app=myapp:v2
  kubectl scale deployment/my-app-canary --replicas=0
else
  # 4b. Rollback
  kubectl scale deployment/my-app-canary --replicas=0
fi
```

**Pros:** Gradual rollout, real-world testing, minimal risk  
**Cons:** Complex setup, longer deployment time  
**Use for:** High-risk changes, gradual rollouts

**4. A/B Testing:**

```python
# Route based on user attributes
@app.get(\"/\")
async def home(request: Request):
    user_id = get_user_id(request)
    
    # A/B test: 50% see new version
    if user_id % 2 == 0:
        return render_v1()
    else:
        return render_v2()
```

**Comparison:**

| Strategy | Risk | Rollback Speed | Resource Cost | Use Case |
|----------|------|----------------|---------------|----------|
| **Rolling** | Medium | Slow | Low | Standard |
| **Blue-Green** | Low | Instant | 2x | Critical apps |
| **Canary** | Lowest | Fast | 1.1x | High-risk changes |
| **A/B Testing** | Low | Medium | Variable | Feature testing |

**Key Concepts:**
- Rolling: Gradual pod replacement
- Blue-Green: Switch all traffic at once
- Canary: Route small % to new version
- A/B Testing: Route based on user attributes
- Always have rollback plan
- Monitor during deployment

**Common Mistakes:**
- No health checks (deploy broken version)
- No rollback plan
- Database migrations not backward compatible
- Not monitoring during deployment
- Deploying without testing
- Not using feature flags

**Interview Tip:**
> "I use different strategies based on risk: rolling updates for standard deployments (zero-downtime, gradual), blue-green for critical apps (instant rollback), and canary for high-risk changes (10% traffic first, monitor, then promote). The key is health checks (liveness/readiness probes), backward-compatible database migrations, and automatic rollback on error rate spikes. I always keep the previous version running for at least 1 hour after deployment for quick rollback."

</details>

---

## Question 14: API Gateway Patterns

**Difficulty:** Advanced  
**Category:** Learning

Design an API Gateway for microservices. What features should it include?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**API Gateway Features:**

1. **Routing:** Route requests to appropriate services
2. **Authentication:** Centralized auth (JWT, OAuth)
3. **Rate Limiting:** Prevent abuse
4. **Caching:** Reduce backend load
5. **Load Balancing:** Distribute traffic
6. **Monitoring:** Metrics, logging, tracing
7. **Transformation:** Request/response mapping
8. **Circuit Breaking:** Fail fast on errors

**Architecture:**

```
Clients → API Gateway → Microservices
         (Kong/AWS API Gateway/Envoy)
```

**Kong Example:**

```yaml
# kong.yml
_format_version: \"2.1\"

services:
  - name: user-service
    url: http://user-service:8000
    routes:
      - name: user-route
        paths:
          - /api/users
        methods:
          - GET
          - POST
        strip_path: true
    
    plugins:
      - name: jwt
        config:
          secret_is_base64: false
          key_claim_name: kid
      
      - name: rate-limiting
        config:
          minute: 100
          hour: 1000
          policy: redis
      
      - name: cors
        config:
          origins:
            - https://example.com
          methods:
            - GET
            - POST
          credentials: true

  - name: order-service
    url: http://order-service:8000
    routes:
      - name: order-route
        paths:
          - /api/orders
        methods:
          - GET
          - POST
          - PUT
          - DELETE

plugins:
  - name: prometheus
    config:
      per_consumer: true
```

**AWS API Gateway:**

```yaml
# serverless.yml
service: my-api

provider:
  name: aws
  runtime: python3.11
  stage: prod
  apiGateway:
    throttle:
      burstLimit: 200
      rateLimit: 100

functions:
  users:
    handler: handlers.users
    events:
      - http:
          path: users/{proxy+}
          method: any
          cors: true
          authorizer:
            type: jwt
            config:
              issuer: https://auth.example.com
              audience: https://api.example.com

  orders:
    handler: handlers.orders
    events:
      - http:
          path: orders/{proxy+}
          method: any
          cors: true
          authorizer:
            type: jwt
```

**Custom API Gateway (FastAPI + Middleware):**

```python
from fastapi import FastAPI, Request, HTTPException
from fastapi.middleware.cors import CORSMiddleware
import jwt
import time
import redis

app = FastAPI()
redis_client = redis.Redis()

# CORS
app.add_middleware(
    CORSMiddleware,
    allow_origins=[\"https://example.com\"],
    allow_credentials=True,
    allow_methods=[\"*\"],
    allow_headers=[\"*\"],
)

# Authentication
async def authenticate(request: Request):
    auth_header = request.headers.get(\"Authorization\")
    if not auth_header or not auth_header.startswith(\"Bearer \"):
        raise HTTPException(401, \"Missing token\")
    
    token = auth_header[7:]
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[\"HS256\"])
        request.state.user = payload
    except jwt.PyJWTError:
        raise HTTPException(401, \"Invalid token\")

# Rate Limiting
async def rate_limit(request: Request):
    user_id = getattr(request.state, \"user\", {}).get(\"sub\", \"anonymous\")
    key = f\"rate:{user_id}\"
    
    current = redis_client.incr(key)
    if current == 1:
        redis_client.expire(key, 60)
    
    if current > 100:  # 100 requests per minute
        raise HTTPException(429, \"Rate limit exceeded\")

# Request Logging
async def log_requests(request: Request, call_next):
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start
    
    logger.info(
        \"Request\",
        extra={
            \"method\": request.method,
            \"path\": request.url.path,
            \"status\": response.status_code,
            \"duration_ms\": duration * 1000,
            \"user_id\": getattr(request.state, \"user\", {}).get(\"sub\")
        }
    )
    
    return response

# Circuit Breaker
class CircuitBreaker:
    def __init__(self, failure_threshold=5, timeout=60):
        self.failure_threshold = failure_threshold
        self.timeout = timeout
        self.failures = 0
        self.last_failure = None
        self.state = \"closed\"
    
    def call(self, func, *args, **kwargs):
        if self.state == \"open\":
            if time.time() - self.last_failure > self.timeout:
                self.state = \"half-open\"
            else:
                raise HTTPException(503, \"Service unavailable\")
        
        try:
            result = func(*args, **kwargs)
            if self.state == \"half-open\":
                self.state = \"closed\"
                self.failures = 0
            return result
        except Exception as e:
            self.failures += 1
            self.last_failure = time.time()
            if self.failures >= self.failure_threshold:
                self.state = \"open\"
            raise

# Register middleware
app.middleware(\"http\")(log_requests)

# Apply to routes
@app.middleware(\"http\")
async def auth_and_rate_limit(request: Request, call_next):
    # Skip for public endpoints
    if request.url.path.startswith(\"/public\"):
        return await call_next(request)
    
    # Authenticate
    await authenticate(request)
    
    # Rate limit
    await rate_limit(request)
    
    return await call_next(request)
```

**Service Routing:**

```python
import httpx

# Service URLs
SERVICES = {
    \"users\": \"http://user-service:8000\",
    \"orders\": \"http://order-service:8000\",
    \"products\": \"http://product-service:8000\"
}

@app.api_route(\"/api/{service}/{path:path}\", methods=[\"GET\", \"POST\", \"PUT\", \"DELETE\"])
async def gateway(service: str, path: str, request: Request):
    if service not in SERVICES:
        raise HTTPException(404, \"Service not found\")
    
    # Forward request to service
    async with httpx.AsyncClient() as client:
        response = await client.request(
            method=request.method,
            url=f\"{SERVICES[service]}/{path}\",
            headers=dict(request.headers),
            content=await request.body()
        )
    
    return response.json()
```

**Key Concepts:**
- Single entry point for all clients
- Centralized cross-cutting concerns (auth, rate limiting)
- Service discovery and routing
- Load balancing
- Circuit breaking
- Monitoring and observability
- Request/response transformation

**Common Mistakes:**
- Business logic in gateway (should be in services)
- Tight coupling between gateway and services
- No rate limiting (abuse potential)
- Missing authentication checks
- No circuit breakers (cascading failures)
- Not monitoring gateway itself

**Interview Tip:**
> "I use Kong or AWS API Gateway for production. The gateway handles cross-cutting concerns: authentication (JWT), rate limiting (Redis-based), CORS, logging, and circuit breaking. Services stay focused on business logic. For custom needs, I build a FastAPI gateway with middleware for auth, rate limiting, and request forwarding. The key is keeping the gateway thin and stateless, with services handling business logic."

</details>

---

## Question 15: Disaster Recovery

**Difficulty:** Advanced  
**Category:** Learning

Design a disaster recovery strategy. Compare backup/restore, pilot light, warm standby, and multi-site.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**DR Strategies (AWS):**

**1. Backup and Restore (RTO: hours, RPO: hours):**

```bash
# Automated backups
aws backup create-backup-plan \\
    --backup-plan '{
        \"Rules\": [{
            \"RuleName\": \"DailyBackup\",
            \"TargetBackupVault\": \"my-vault\",
            \"ScheduleExpression\": \"cron(0 5 ? * * *)\",
            \"Lifecycle\": {
                \"DeleteAfterDays\": 30
            }
        }]
    }'

# RDS automated backups
aws rds modify-db-instance \\
    --db-instance-identifier mydb \\
    --backup-retention-period 7 \\
    --preferred-backup-window \"03:00-04:00\"
```

**Process:**
- Daily backups of databases, files, configs
- Store in S3 (cross-region replication)
- Restore manually when disaster occurs
- Downtime: Hours
- Data loss: Up to 24 hours (RPO)

**Pros:** Cheapest ($)  
**Cons:** Slowest recovery, significant data loss

**2. Pilot Light (RTO: 10s of minutes, RPO: minutes):**

```yaml
# Core services running in DR region
# Database: Read replica (synchronized)
# Compute: Minimal instances (can scale up)
# Data: Continuously replicated

# RDS Cross-Region Read Replica
aws rds create-db-instance-read-replica \\
    --db-instance-identifier mydb-replica \\
    --source-db-instance-identifier mydb \\
    --source-region us-east-1 \\
    --region us-west-2

# S3 Cross-Region Replication
aws s3api put-bucket-replication \\
    --bucket my-bucket \\
    --replication-configuration '{
        \"Rules\": [{
            \"Status\": \"Enabled\",
            \"Destination\": {
                \"Bucket\": \"arn:aws:s3:::my-bucket-dr\"
            }
        }]
    }'
```

**Process:**
- Database replica running in DR region (minimal cost)
- Core services stopped (only data replicated)
- When disaster: Start services, promote replica
- Downtime: 10-60 minutes
- Data loss: Minutes (replication lag)

**Pros:** Moderate cost ($$), faster recovery  
**Cons:** Manual promotion, some downtime

**3. Warm Standby (RTO: minutes, RPO: seconds):**

```yaml
# Reduced-scale environment running in DR region
# Database: Synchronous replication
# Compute: Minimal instances running
# Traffic: Minimal (can scale up quickly)

# Aurora Global Database (cross-region replication < 1 second)
aws rds create-global-cluster \\
    --global-cluster-identifier my-global \\
    --engine aurora-postgresql

# Add secondary region
aws rds create-db-cluster \\
    --db-cluster-identifier mydb-secondary \\
    --global-cluster-identifier my-global \\
    --engine aurora-postgresql
```

**Process:**
- Minimal environment running in DR region
- Data continuously replicated (near real-time)
- When disaster: Scale up services, switch traffic
- Downtime: Minutes
- Data loss: Seconds

**Pros:** Fast recovery, low data loss ($$$)  
**Cons:** Higher cost (always running)

**4. Multi-Site Active-Active (RTO: seconds, RPO: near zero):**

```yaml
# Full environment in multiple regions
# Both regions serving traffic
# Global load balancer routes to nearest healthy region

# Route 53 health checks and failover
resource \"aws_route53_health_check\" \"primary\" {
  fqdn              = \"api.example.com\"
  port              = 443
  type              = \"HTTPS\"
  resource_path     = \"/health\"
  failure_threshold = 3
}

resource \"aws_route53_record\" \"api\" {
  zone_id = aws_route53_zone.main.zone_id
  name    = \"api.example.com\"
  type    = \"A\"
  
  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}
```

**Process:**
- Both regions actively serving traffic
- Global load balancer routes to nearest
- If one region fails, traffic shifts to other
- Downtime: Seconds
- Data loss: Near zero

**Pros:** Best availability, zero downtime ($$$$)  
**Cons:** Most expensive, complex

**Comparison:**

| Strategy | RTO | RPO | Cost | Complexity | Use Case |
|----------|-----|-----|------|------------|----------|
| **Backup/Restore** | Hours | Hours | $ | Low | Non-critical |
| **Pilot Light** | 10-60 min | Minutes | $$ | Medium | Important |
| **Warm Standby** | Minutes | Seconds | $$$ | High | Critical |
| **Multi-Site** | Seconds | Near zero | $$$$ | Very High | Mission-critical |

**Key Concepts:**
- **RTO** (Recovery Time Objective): How long to recover
- **RPO** (Recovery Point Objective): How much data loss is acceptable
- Choose based on business requirements
- Test DR plan regularly
- Automate where possible
- Monitor DR health

**DR Plan Components:**

```python
# 1. Automated failover
# 2. Data replication
# 3. Health monitoring
# 4. Runbook documentation
# 5. Regular DR drills
# 6. Communication plan
# 7. Post-mortem process

# Example: Automated DR with Route 53
# - Health checks every 30 seconds
# - Failover to DR region after 3 failures
# - Automated database promotion
# - DNS update via Route 53
# - Notification via SNS
```

**Common Mistakes:**
- Not testing DR plan (untested plans fail)
- No documentation (can't execute under stress)
- Unclear RTO/RPO requirements
- Single region assumption
- Not considering data consistency
- No budget for DR

**Interview Tip:**
> "I design DR based on business requirements. For non-critical apps, I use backup/restore (daily backups to S3 with cross-region replication). For important apps, I use pilot light (RDS read replica in DR region, promote on disaster). For critical apps, I use warm standby (minimal environment running, near real-time replication). For mission-critical, I use multi-site active-active. The key is defining RTO/RPO requirements first, then choosing the appropriate strategy. Always test the DR plan with regular drills."

</details>

---

## Question 16: Container Security

**Difficulty:** Advanced  
**Category:** Learning

What are the key container security practices? How do you secure Docker and Kubernetes?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Image Security:**

**1. Use Official Base Images:**

```dockerfile
# Good: Official, minimal
FROM python:3.11-slim
# or
FROM python:3.11-alpine

# Bad: Unmaintained, large
FROM ubuntu:latest
```

**2. Scan Images for Vulnerabilities:**

```bash
# Trivy (most popular)
trivy image myapp:latest

# Scan in CI/CD
trivy image --exit-code 1 --severity HIGH,CRITICAL myapp:latest

# Snyk
snyk container test myapp:latest

# Docker Scout
docker scout cves myapp:latest
```

**3. Multi-Stage Builds (already covered):**

```dockerfile
# Build stage
FROM python:3.11-slim AS builder
# ... build dependencies

# Runtime stage (minimal)
FROM python:3.11-slim
COPY --from=builder /opt/venv /opt/venv
COPY . .
USER appuser  # Non-root
```

**4. Don't Store Secrets in Images:**

```dockerfile
# Bad: Secrets in Dockerfile
ENV OPENAI_API_KEY=sk-...
RUN echo \"password\" > /etc/secret.txt

# Good: Pass at runtime
# docker run -e OPENAI_API_KEY=sk-... myapp
# or use secrets management
```

**5. Sign Images:**

```bash
# Cosign (Sigstore)
cosign sign --key cosign.key myapp:latest

# Verify
cosign verify --key cosign.pub myapp:latest
```

**6. Use Distroless Images:**

```dockerfile
# No shell, no package manager (smaller attack surface)
FROM gcr.io/distroless/python3-debian11
COPY --from=builder /app /app
WORKDIR /app
CMD [\"main:app\"]
```

**Runtime Security:**

**1. Run as Non-Root User:**

```dockerfile
# Create non-root user
RUN useradd -m -u 1000 appuser && \\
    chown -R appuser:appuser /app
USER appuser

# In Kubernetes
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000
  readOnlyRootFilesystem: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

**2. Read-Only Filesystem:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
      - name: app
        securityContext:
          readOnlyRootFilesystem: true
        volumeMounts:
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: tmp
        emptyDir: {}
```

**3. Security Contexts in Kubernetes:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      # Pod-level security
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
        seccompProfile:
          type: RuntimeDefault
      
      containers:
      - name: app
        # Container-level security
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop:
              - ALL
            add:
              - NET_BIND_SERVICE  # Only if needed
          seccompProfile:
            type: RuntimeDefault
```

**4. Network Policies (already covered):**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

**5. Pod Security Standards:**

```yaml
# Enforce restricted policy
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

**6. Image Pull Secrets:**

```yaml
# Pull from private registry
apiVersion: v1
kind: Secret
metadata:
  name: regcred
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: <base64-encoded-config>

---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      imagePullSecrets:
      - name: regcred
```

**7. Resource Limits (Prevent DoS):**

```yaml
containers:
- name: app
  resources:
    requests:
      memory: \"256Mi\"
      cpu: \"250m\"
    limits:
      memory: \"512Mi\"
      cpu: \"500m\"
```

**Supply Chain Security:**

```bash
# 1. Verify base images
docker pull python:3.11-slim
docker inspect python:3.11-slim | grep -i digest

# 2. Scan for vulnerabilities
trivy image python:3.11-slim

# 3. Check for malware
clamav-scan /path/to/image

# 4. Use private registry
# AWS ECR, GCP Container Registry, Harbor

# 5. Enable image scanning in registry
aws ecr put-image-scanning-configuration \\
    --repository-name myapp \\
    --image-scanning-configuration scanOnPush=true
```

**Secret Management:**

```yaml
# Use External Secrets Operator (already covered)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secret
spec:
  secretStoreRef:
    name: aws-secrets-manager
    kind: ClusterSecretStore
  target:
    name: db-secret
  data:
  - secretKey: url
    remoteRef:
      key: myapp/database
      property: url
```

**Monitoring and Detection:**

```yaml
# Falco (runtime security monitoring)
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: falco
spec:
  template:
    spec:
      containers:
      - name: falco
        image: falcosecurity/falco:latest
        securityContext:
          privileged: true
        volumeMounts:
        - name: docker-socket
          mountPath: /var/run/docker.sock
      volumes:
      - name: docker-socket
        hostPath:
          path: /var/run/docker.sock
```

**Key Concepts:**
- Image security: Scan, sign, minimal base images
- Runtime security: Non-root, read-only FS, drop capabilities
- Network security: Network Policies, mTLS
- Supply chain: Verify dependencies, private registry
- Secrets: External Secrets Operator, not in env vars
- Monitoring: Falco, audit logs
- Defense in depth: Multiple layers

**Common Mistakes:**
- Running as root
- Not scanning images
- Storing secrets in images
- No resource limits
- Privileged containers
- No network policies
- Using latest tag (not reproducible)

**Interview Tip:**
> "Container security requires defense in depth: image security (scan with Trivy, use distroless, sign with Cosign), runtime security (run as non-root, read-only filesystem, drop capabilities), network security (Network Policies, deny-all default), and supply chain security (verify dependencies, private registry). I use Pod Security Standards (restricted), External Secrets Operator for secrets, and Falco for runtime monitoring. The key is never trusting defaults and assuming breach."

</details>

---

## Question 17: Message Queues and Event Streaming

**Difficulty:** Advanced  
**Category:** Learning

Compare message queues (RabbitMQ, SQS) and event streaming (Kafka). When would you use each?

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Comparison:**

| Feature | RabbitMQ | SQS | Kafka |
|---------|----------|-----|-------|
| **Type** | Message Queue | Message Queue | Event Streaming |
| **Model** | Queue (point-to-point) | Queue (point-to-point) | Log (pub-sub) |
| **Retention** | Until consumed | 4-14 days | Configurable (days-years) |
| **Throughput** | Thousands/sec | Thousands/sec | Millions/sec |
| **Ordering** | Per queue | Best-effort | Per partition (guaranteed) |
| **Replay** | No (after consumed) | No (after deleted) | Yes (replay from offset) |
| **Use case** | Task distribution | Decoupling | Event sourcing, streaming |

**RabbitMQ:**

```python
# Producer
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

# Declare queue
channel.queue_declare(queue='tasks', durable=True)

# Publish
channel.basic_publish(
    exchange='',
    routing_key='tasks',
    body='Process this task',
    properties=pika.BasicProperties(
        delivery_mode=2,  # Persistent
    )
)

connection.close()

# Consumer
import pika

connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()

channel.queue_declare(queue='tasks', durable=True)

def callback(ch, method, properties, body):
    print(f\"Received: {body}\")
    # Process task
    ch.basic_ack(delivery_tag=method.delivery_tag)

channel.basic_qos(prefetch_count=1)
channel.basic_consume(queue='tasks', on_message_callback=callback)

channel.start_consuming()
```

**AWS SQS:**

```python
import boto3

sqs = boto3.client('sqs')

# Create queue
response = sqs.create_queue(
    QueueName='my-tasks',
    Attributes={
        'DelaySeconds': '0',
        'VisibilityTimeout': '300',  # 5 minutes
        'MessageRetentionPeriod': '1209600'  # 14 days
    }
)

queue_url = response['QueueUrl']

# Send message
sqs.send_message(
    QueueUrl=queue_url,
    MessageBody='Process this task',
    MessageAttributes={
        'Priority': {'StringValue': 'high', 'DataType': 'String'}
    }
)

# Receive messages
response = sqs.receive_message(
    QueueUrl=queue_url,
    MaxNumberOfMessages=10,
    WaitTimeSeconds=20,  # Long polling
    VisibilityTimeout=300
)

for message in response.get('Messages', []):
    # Process message
    print(f\"Processing: {message['Body']}\")
    
    # Delete after processing
    sqs.delete_message(
        QueueUrl=queue_url,
        ReceiptHandle=message['ReceiptHandle']
    )
```

**Kafka:**

```python
from kafka import KafkaProducer, KafkaConsumer
import json

# Producer
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Send message
producer.send('user-events', {
    'user_id': 'user-123',
    'action': 'login',
    'timestamp': '2026-06-23T22:00:00Z'
})

producer.flush()

# Consumer
consumer = KafkaConsumer(
    'user-events',
    bootstrap_servers=['localhost:9092'],
    value_deserializer=lambda m: json.loads(m.decode('utf-8')),
    group_id='analytics-service',
    auto_offset_reset='earliest',  # Read from beginning
    enable_auto_commit=False
)

for message in consumer:
    event = message.value
    print(f\"Received: {event}\")
    
    # Process event
    process_event(event)
    
    # Manual commit
    consumer.commit()
```

**Decision Framework:**

```python
# Use RabbitMQ when:
# - Task distribution (work queues)
# - Complex routing (topic exchanges)
# - Need acknowledgments
# - Lower throughput (< 10K msg/sec)
# Examples: Email sending, image processing, background jobs

# Use SQS when:
# - AWS-native architecture
# - Simple queue needs
# - Auto-scaling workers
# - Pay per use
# Examples: Decoupling services, async processing

# Use Kafka when:
# - Event sourcing
# - Stream processing
# - High throughput (> 10K msg/sec)
# - Need to replay events
# - Multiple consumers
# Examples: Log aggregation, real-time analytics, CDC
```

**Kafka Advanced Features:**

```python
# Consumer groups for parallel processing
consumer = KafkaConsumer(
    'user-events',
    group_id='analytics-service',
    # Multiple consumers in same group share partitions
)

# Manual offset management
consumer = KafkaConsumer(
    'user-events',
    enable_auto_commit=False,
    auto_offset_reset='earliest'
)

for message in consumer:
    process(message)
    consumer.commit()  # Commit after processing

# Exactly-once semantics
producer = KafkaProducer(
    transactional_id='my-transactional-id',
    enable_idempotence=True
)

producer.begin_transaction()
producer.send('topic1', value=b'message1')
producer.send('topic2', value=b'message2')
producer.commit_transaction()

# Stream processing with Kafka Streams
from kafka import StreamsConfig, KafkaStreams

# Or use Faust (Python)
import faust

app = faust.App('my-app', broker='kafka://localhost:9092')

@app.agent('user-events')
async def process_events(events):
    async for event in events:
        # Process event
        yield event
```

**Key Concepts:**
- **Message Queue:** Point-to-point, message consumed once
- **Event Streaming:** Pub-sub, multiple consumers, replay
- **RabbitMQ:** Traditional MQ, complex routing
- **SQS:** AWS-native, managed, simple
- **Kafka:** High-throughput, event sourcing, replay
- **Retention:** Queue (until consumed) vs Log (time-based)

**Common Mistakes:**
- Using queue when you need streaming (can't replay)
- Using Kafka for simple task queues (overkill)
- Not handling message failures (poison messages)
- No dead letter queue
- Not setting proper timeouts
- Ignoring message ordering requirements

**Interview Tip:**
> "I choose the messaging system based on requirements. For task distribution (work queues), I use RabbitMQ or SQS. For event sourcing, stream processing, or when I need to replay events, I use Kafka. For AWS-native architectures, I use SQS (simple) or SNS+SQS (pub-sub). For high-throughput (> 10K msg/sec) with multiple consumers, I use Kafka. The key is understanding the difference: queues (message consumed once) vs logs (event stream, multiple consumers, replay)."

</details>

---

## Question 18: Logging and Observability

**Difficulty:** Intermediate  
**Category:** Learning

Design a comprehensive logging strategy for a microservices application.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Structured Logging:**

```python
import logging
import json
from pythonjsonlogger import jsonlogger
from datetime import datetime

# Configure
logHandler = logging.StreamHandler()
formatter = jsonlogger.JsonFormatter(
    '%(asctime)s %(name)s %(levelname)s %(message)s'
)
logHandler.setFormatter(formatter)
logger = logging.getLogger()
logger.addHandler(logHandler)
logger.setLevel(logging.INFO)

# Log with context
logger.info(
    'Request processed',
    extra={
        'request_id': 'req-123',
        'user_id': 'user-456',
        'method': 'POST',
        'path': '/api/users',
        'status': 200,
        'duration_ms': 245
    }
)
```

**Output:**

```json
{
  \"asctime\": \"2026-06-23 22:00:00,000\",
  \"name\": \"app\",
  \"levelname\": \"INFO\",
  \"message\": \"Request processed\",
  \"request_id\": \"req-123\",
  \"user_id\": \"user-456\",
  \"method\": \"POST\",
  \"path\": \"/api/users\",
  \"status\": 200,
  \"duration_ms\": 245
}
```

**Request ID Tracking:**

```python
import uuid
from contextvars import ContextVar

# Context variable for request ID
request_id_var: ContextVar[str] = ContextVar('request_id', default='')

# Middleware to set request ID
@app.middleware(\"http\")
async def add_request_id(request: Request, call_next):
    request_id = request.headers.get(\"X-Request-ID\", str(uuid.uuid4()))
    request_id_var.set(request_id)
    
    response = await call_next(request)
    response.headers[\"X-Request-ID\"] = request_id
    return response

# Custom log filter
class RequestIDFilter(logging.Filter):
    def filter(self, record):
        record.request_id = request_id_var.get()
        return True

# Add filter to all handlers
for handler in logger.handlers:
    handler.addFilter(RequestIDFilter())
```

**Centralized Logging with ELK:**

```yaml
# docker-compose.yml
version: '3.8'
services:
  app:
    build: .
    logging:
      driver: \"json-file\"
      options:
        max-size: \"10m\"
        max-file: \"3\"
  
  filebeat:
    image: docker.elastic.co/beats/filebeat:8.5.0
    user: root
    volumes:
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
    depends_on:
      - elasticsearch
  
  elasticsearch:
    image: elasticsearch:8.5.0
    environment:
      - discovery.type=single-node
    ports:
      - \"9200:9200\"
  
  kibana:
    image: kibana:8.5.0
    ports:
      - \"5601:5601\"
```

**Log Levels:**

```python
# Use appropriate log levels
logger.debug(\"Detailed debug information\")  # Development only
logger.info(\"Normal operation events\")       # Production default
logger.warning(\"Warning, but not error\")     # Potential issues
logger.error(\"Error occurred\")               # Errors that need attention
logger.critical(\"Critical error\")            # System-critical issues
```

**What to Log:**

```python
# DO log:
# - Request/response (with request ID)
# - Errors with stack traces
# - Authentication events
# - Business-critical operations
# - Performance metrics
# - Security events

# DON'T log:
# - Passwords, tokens, API keys
# - PII (use redaction)
# - Sensitive data (credit cards, SSN)
# - High-cardinality data

# Example: Logging with redaction
import re

def redact_sensitive(text: str) -> str:
    # Redact credit cards
    text = re.sub(r'\\b\\d{16}\\b', '****-****-****-****', text)
    # Redact SSN
    text = re.sub(r'\\b\\d{3}-\\d{2}-\\d{4}\\b', '***-**-****', text)
    # Redact emails (optional)
    text = re.sub(r'\\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\\.[A-Z|a-z]{2,}\\b', '***@***.com', text)
    return text

logger.info(f\"User data: {redact_sensitive(user_data)}\")
```

**Distributed Tracing:**

```python
from opentelemetry import trace
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.exporter.jaeger.thrift import JaegerExporter
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.trace import TracerProvider

# Setup tracing
trace.set_tracer_provider(TracerProvider())
jaeger_exporter = JaegerExporter(
    agent_host_name=\"jaeger\",
    agent_port=6831,
)
trace.get_tracer_provider().add_span_processor(
    BatchSpanProcessor(jaeger_exporter)
)

# Auto-instrument
FastAPIInstrumentor.instrument_app(app)
RequestsInstrumentor().instrument()
SQLAlchemyInstrumentor().instrument(engine=db.engine)

# Custom spans
tracer = trace.get_tracer(__name__)

@app.get(\"/users/{user_id}\")
async def get_user(user_id: str):
    with tracer.start_as_current_span(\"get_user\") as span:
        span.set_attribute(\"user.id\", user_id)
        
        # Database call (automatically traced)
        user = await db.get_user(user_id)
        
        # External API call (automatically traced)
        async with httpx.AsyncClient() as client:
            response = await client.get(f\"https://api.example.com/users/{user_id}\")
        
        return user
```

**Metrics Collection:**

```python
from prometheus_client import Counter, Histogram, Gauge

# Counters (monotonically increasing)
request_count = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

# Histograms (distributions)
request_duration = Histogram(
    'http_request_duration_seconds',
    'HTTP request duration',
    ['method', 'endpoint']
)

# Gauges (can go up or down)
active_connections = Gauge(
    'active_connections',
    'Number of active connections'
)

# Usage
@app.middleware(\"http\")
async def track_metrics(request, call_next):
    start = time.time()
    response = await call_next(request)
    duration = time.time() - start
    
    request_count.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()
    
    request_duration.labels(
        method=request.method,
        endpoint=request.url.path
    ).observe(duration)
    
    return response

# Expose metrics endpoint
from prometheus_client import generate_latest

@app.get(\"/metrics\")
async def metrics():
    return Response(generate_latest(), media_type=\"text/plain\")
```

**Key Concepts:**
- Structured logging (JSON) for queryability
- Request ID for tracing across services
- Centralized logging (ELK, Loki, CloudWatch)
- Distributed tracing (Jaeger, Zipkin)
- Metrics (Prometheus, StatsD)
- Log levels (debug, info, warning, error, critical)
- Redact sensitive data

**Common Mistakes:**
- Unstructured logs (hard to query)
- No request ID (can't trace requests)
- Logging sensitive data (PII, secrets)
- Too verbose logging (performance, cost)
- No centralized logging (can't search across services)
- No log retention policy (disk fills up)

**Interview Tip:**
> "I use structured JSON logging with request IDs for correlation across services. Logs are centralized in ELK stack (Elasticsearch, Logstash, Kibana) or cloud-native (CloudWatch, Stackdriver). I add distributed tracing with OpenTelemetry and Jaeger for request flow visibility. I collect metrics with Prometheus and visualize in Grafana. The key is structured logging, request correlation, and avoiding logging sensitive data. I always redact PII and use appropriate log levels."

</details>

---

## Question 19: Cost Optimization

**Difficulty:** Advanced  
**Category:** Learning

How do you optimize cloud costs? Provide specific strategies for AWS, GCP, and Kubernetes.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**Compute Optimization:**

**1. Right-Sizing:**

```bash
# AWS: Check CloudWatch metrics
aws cloudwatch get-metric-statistics \\
    --namespace AWS/EC2 \\
    --metric-name CPUUtilization \\
    --dimensions Name=InstanceId,Value=i-12345 \\
    --start-time 2026-06-01 \\
    --end-time 2026-06-23 \\
    --period 3600 \\
    --statistics Average

# If avg CPU < 30%, downsize instance
# If avg CPU > 70%, upsize instance
```

**2. Reserved Instances / Committed Use:**

```bash
# AWS Reserved Instance (1-3 year commitment, 40-60% savings)
aws ec2 purchase-reserved-instances-offering \\
    --reserved-instances-offering-id <id> \\
    --instance-count 10

# GCP Committed Use Discount
gcloud compute commitments create \\
    --region=us-central1 \\
    --plan=12-month \\
    --resources=vcpu=100,memory=400GB

# Azure Reserved VM Instances
az vm reservation create \\
    --resource-group mygroup \\
    --vm-name myvm \\
    --reserved-vm-name myreservation
```

**3. Spot/Preemptible Instances:**

```yaml
# Kubernetes with Spot instances (70-90% savings)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-job
spec:
  template:
    spec:
      nodeSelector:
        cloud.google.com/gke-preemptible: \"true\"  # GCP
        # node.kubernetes.io/instance-type: spot    # AWS
      tolerations:
      - key: cloud.google.com/gke-preemptible
        value: \"true\"
        effect: NoSchedule
      containers:
      - name: app
        image: myapp:latest
        resources:
          requests:
            memory: \"256Mi\"
            cpu: \"250m\"
```

**4. Auto-Scaling (Scale to Zero):**

```yaml
# Scale down during off-hours
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-app-hpa
spec:
  minReplicas: 0  # Scale to zero when idle
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

**Storage Optimization:**

**1. S3 Storage Classes:**

```bash
# Intelligent Tiering (automatic cost optimization)
aws s3api put-bucket-intelligent-tiering-configuration \\
    --bucket my-bucket \\
    --id EntireBucket \\
    --intelligent-tiering-configuration '{
        \"Tierings\": [
            {\"Days\": 90, \"AccessTier\": \"ARCHIVE_ACCESS\"},
            {\"Days\": 180, \"AccessTier\": \"DEEP_ARCHIVE_ACCESS\"}
        ]
    }'

# Lifecycle policy (move to cheaper storage)
aws s3api put-bucket-lifecycle-configuration \\
    --bucket my-bucket \\
    --lifecycle-configuration '{
        \"Rules\": [{
            \"Status\": \"Enabled\",
            \"Transitions\": [
                {\"Days\": 30, \"StorageClass\": \"STANDARD_IA\"},
                {\"Days\": 90, \"StorageClass\": \"GLACIER\"}
            ]
        }]
    }'
```

**2. Delete Unused Resources:**

```bash
# Find unattached EBS volumes
aws ec2 describe-volumes \\
    --filters Name=status,Values=available \\
    --query 'Volumes[].[VolumeId,Size,CreateTime]'

# Find unused Elastic IPs
aws ec2 describe-addresses \\
    --query 'Addresses[?AssociationId==`null`]'

# Find old snapshots
aws ec2 describe-snapshots \\
    --query 'Snapshots[?StartTime<=`2026-01-01`]'
```

**Database Optimization:**

**1. Use Serverless Databases:**

```bash
# AWS Aurora Serverless (auto-scales, pay per second)
aws rds create-db-cluster \\
    --engine aurora-postgresql \\
    --engine-version 13.7 \\
    --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=16

# GCP Cloud SQL with auto-scaling
gcloud sql instances create mydb \\
    --database-version=POSTGRES_15 \\
    --tier=db-custom-1-3840 \\
    --enable-autoscaling \\
    --min-cpu=1 \\
    --max-cpu=8
```

**2. Query Optimization:**

```sql
-- Add indexes for slow queries
CREATE INDEX idx_user_email ON users(email);

-- Analyze query performance
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'user@example.com';
```

**Network Optimization:**

**1. CloudFront/CDN:**

```yaml
# CloudFront distribution
Resources:
  Distribution:
    Type: AWS::CloudFront::Distribution
    Properties:
      DistributionConfig:
        Origins:
          - DomainName: my-bucket.s3.amazonaws.com
            Id: S3Origin
        Enabled: true
        DefaultCacheBehavior:
          TargetOriginId: S3Origin
          ViewerProtocolPolicy: redirect-to-https
          TTL: 86400  # Cache for 24 hours
        PriceClass: PriceClass_100  # Use only North America/Europe
```

**2. Data Transfer Optimization:**

```bash
# Use VPC endpoints (avoid internet gateway costs)
aws ec2 create-vpc-endpoint \\
    --vpc-id vpc-12345 \\
    --service-name com.amazonaws.us-east-1.s3

# Compress data
gzip -9 large-file.json
```

**Monitoring and Alerts:**

```yaml
# AWS Budget Alert
Resources:
  Budget:
    Type: AWS::Budgets::Budget
    Properties:
      Budget:
        BudgetLimit:
          Amount: 1000
          Unit: USD
        TimeUnit: MONTHLY
      NotificationsWithSubscribers:
      - Notification:
          ComparisonOperator: GREATER_THAN
          Threshold: 80
          ThresholdType: PERCENTAGE
        Subscribers:
        - SubscriptionType: EMAIL
          Address: ops@example.com
```

**Cost Optimization Checklist:**

```python
# 1. Right-size resources
# - Check CloudWatch metrics
# - Downsize underutilized instances
# - Remove idle resources

# 2. Use appropriate pricing models
# - Reserved/Committed for stable workloads
# - Spot/Preemptible for batch jobs
# - On-demand for variable workloads
# - Serverless for sporadic traffic

# 3. Optimize storage
# - S3 Intelligent Tiering
# - Lifecycle policies
# - Delete unused snapshots
# - Compress data

# 4. Database optimization
# - Use serverless databases
# - Optimize queries
# - Add indexes
# - Use read replicas wisely

# 5. Network optimization
# - Use CDN (CloudFront, Cloudflare)
# - VPC endpoints
# - Compress data
# - Cache aggressively

# 6. Auto-scaling
# - Scale to zero when idle
# - HPA for variable load
# - Cluster autoscaler

# 7. Monitoring
# - Set up billing alerts
# - Tag resources for cost allocation
# - Regular cost reviews
# - Use cost explorer

# 8. Governance
# - Resource quotas
# - Approval workflows
# - Tagging policies
# - Regular audits
```

**Key Concepts:**
- Right-sizing based on actual usage
- Reserved/Committed use for predictable workloads
- Spot/Preemptible for fault-tolerant workloads
- Storage lifecycle policies
- Database optimization (serverless, queries, indexes)
- CDN for static content
- Auto-scaling to match demand
- Continuous monitoring and optimization

**Common Mistakes:**
- Not monitoring costs (bill shock)
- Over-provisioning resources
- Not using reserved instances
- Keeping unused resources running
- No lifecycle policies for storage
- Ignoring data transfer costs
- Not tagging resources

**Interview Tip:**
> "I optimize costs through right-sizing (check CloudWatch metrics, downsize underutilized), reserved instances for stable workloads (40-60% savings), spot instances for fault-tolerant batch jobs (70-90% savings), S3 lifecycle policies (move to cheaper storage), serverless databases (auto-scale, pay per use), and CDN for static content. I set up billing alerts and tag resources for cost allocation. The key is continuous monitoring and optimization—costs grow with usage, so regular reviews are essential."

</details>

---

## Question 20: SRE and Site Reliability

**Difficulty:** Advanced  
**Category:** Learning

Explain SRE principles: SLIs, SLOs, error budgets, and incident response.

<details>
<summary><b>View Answer & Explanation</b></summary>

**Answer:**

**SLI (Service Level Indicator):**

Quantitative measure of service quality.

```python
# Common SLIs:

# 1. Availability
availability = successful_requests / total_requests

# 2. Latency
p50_latency = 50th percentile response time
p95_latency = 95th percentile response time
p99_latency = 99th percentile response time

# 3. Throughput
throughput = requests_per_second

# 4. Error rate
error_rate = failed_requests / total_requests

# 5. Durability
durability = successful_writes / total_writes
```

**SLO (Service Level Objective):**

Target value for SLI.

```yaml
# Example SLOs
slos:
  - name: \"API Availability\"
    sli: availability
    target: 99.9%  # Three nines = 8.76 hours downtime/year
    
  - name: \"API Latency\"
    sli: p95_latency
    target: 500ms
    
  - name: \"Error Rate\"
    sli: error_rate
    target: 0.1%  # 0.1% of requests can fail
```

**SLA (Service Level Agreement):**

Contract with customers, includes consequences.

```
SLA: 99.9% availability
- If we miss: 10% service credit
- Measured monthly
```

**Error Budget:**

Allowable failure based on SLO.

```python
# 99.9% availability SLO
# Monthly error budget = 0.1% = 43.2 minutes of downtime

monthly_minutes = 30 * 24 * 60  # 43,200 minutes
error_budget = monthly_minutes * 0.001  # 43.2 minutes

# If we use 20 minutes, we have 23.2 minutes left
# If we exceed budget, freeze non-critical changes
```

**Error Budget Policy:**

```yaml
# When error budget is exhausted:
# 1. Stop non-critical deployments
# 2. Focus on reliability work
# 3. Post-mortem on incidents
# 4. Increase testing

# When error budget is healthy:
# 1. Allow more risky changes
# 2. Feature work
# 3. Experiments
```

**Monitoring and Alerting:**

```yaml
# Prometheus alerting rules
groups:
- name: slo_alerts
  rules:
  # Burn rate alerting (Google SRE workbook)
  - alert: SLO_BurnRate_High
    expr: |
      (
        sum(rate(http_requests_total{status=~"5.."}[1h]))
        /
        sum(rate(http_requests_total[1h]))
      ) > (14.4 * 0.001)  # 14.4x burn rate for 1h
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "High SLO burn rate"
      description: "Burning error budget 14.4x faster than sustainable"
  
  - alert: SLO_BurnRate_Medium
    expr: |
      (
        sum(rate(http_requests_total{status=~"5.."}[6h]))
        /
        sum(rate(http_requests_total[6h]))
      ) > (6 * 0.001)  # 6x burn rate for 6h
    for: 5m
    labels:
      severity: warning
```

**Incident Response:**

**1. Severity Levels:**

```yaml
severity:
  SEV1: # Critical
    - Complete service outage
    - Data loss
    - Security breach
    - Response: Immediate, all hands
    
  SEV2: # Major
    - Major feature broken
    - Significant degradation
    - Response: 15 minutes, on-call
    
  SEV3: # Minor
    - Minor feature broken
    - Workaround available
    - Response: 1 hour, on-call
    
  SEV4: # Low
    - Cosmetic issues
    - Response: Next business day
```

**2. Incident Process:**

```python
# 1. Detect (alerting, monitoring)
# 2. Triage (severity, impact)
# 3. Mitigate (stop the bleeding)
# 4. Resolve (fix root cause)
# 5. Post-mortem (learn and improve)
```

**3. Incident Roles:**

```yaml
roles:
  Incident Commander:
    - Coordinates response
    - Makes decisions
    - Communicates with stakeholders
    - Does NOT debug
    
  Tech Lead:
    - Leads debugging
    - Coordinates engineering work
    - Proposes solutions
    
  Communications Lead:
    - Updates stakeholders
    - Manages status page
    - Customer communication
    
  Scribe:
    - Documents timeline
    - Records decisions
    - Captures context
```

**4. Post-Mortem:**

```markdown
# Post-Mortem: API Outage on 2026-06-23

## Summary
API was down for 45 minutes from 14:00 to 14:45 UTC.

## Impact
- 100% of API requests failed
- ~50,000 affected users
- Revenue loss: $15,000

## Timeline
- 14:00 - Alert triggered (error rate spike)
- 14:02 - On-call paged
- 14:05 - Incident declared (SEV1)
- 14:15 - Root cause identified (database connection pool exhausted)
- 14:20 - Mitigation applied (increased pool size)
- 14:45 - Full recovery
- 15:00 - Post-mortem scheduled

## Root Cause
Connection pool size was set to 10, but traffic spike required 50.
Misconfigured during recent deployment.

## Resolution
- Increased pool size to 100
- Added monitoring for pool utilization
- Set up auto-scaling for connection pool

## Action Items
- [ ] Add connection pool auto-scaling (Owner: Alice, Due: 2026-07-01)
- [ ] Improve load testing to catch this (Owner: Bob, Due: 2026-07-15)
- [ ] Review all connection pool configs (Owner: Charlie, Due: 2026-07-01)
- [ ] Add SLO for connection pool saturation (Owner: Alice, Due: 2026-07-15)

## Lessons Learned
- Need better load testing
- Connection pool sizing is critical
- Monitoring should catch this before users
```

**Toil Reduction:**

```python
# Identify toil (manual, repetitive, automatable work)

# To Do:
# 1. Automate repetitive tasks
# 2. Self-service tools for developers
# 3. Better monitoring and alerting
# 4. Runbooks for common issues
# 5. Chaos engineering

# SRE goal: Keep toil under 50% of work time
```

**Chaos Engineering:**

```python
# Use Chaos Mesh or AWS Fault Injection Service

# Example: Kill random pods
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - production
    labelSelectors:
      app: my-app
  duration: \"5m\"
  scheduler:
    cron: \"@every 2m\"
```

**Key Concepts:**
- SLI: What you measure
- SLO: Target for SLI
- SLA: Contract with consequences
- Error Budget: Allowable failure
- Incident Response: Process for handling outages
- Post-Mortem: Learn from incidents (blameless)
- Toil: Manual work to eliminate
- Chaos Engineering: Test failure scenarios

**Common Mistakes:**
- Too many SLOs (can't focus)
- No error budget policy
- Blameful post-mortems (people hide issues)
- No runbooks for common issues
- Alerting on symptoms, not causes
- Not learning from incidents
- Too much toil (burnout)

**Interview Tip:**
> "I define SLIs (availability, latency, error rate) and set SLOs based on user expectations (e.g., 99.9% availability, p95 < 500ms). The error budget gives us flexibility—if we have budget, we can take risks; if not, we focus on reliability. For incidents, I follow a structured process: detect, triage, mitigate, resolve, post-mortem. Post-mortems are blameless—we focus on systems, not people. The goal is continuous improvement through learning from incidents and reducing toil through automation."

</details>

---

## Summary Checklist

- [x] Docker multi-stage builds
- [x] Kubernetes Pods, Deployments, Services
- [x] CI/CD pipeline design
- [x] AWS Lambda deployment
- [x] GCP Cloud Run deployment
- [x] Kubernetes networking (DNS, Ingress)
- [x] Container orchestration comparison
- [x] Monitoring and logging
- [x] Terraform Infrastructure as Code
- [x] Secrets management
- [x] Auto-scaling strategies
- [x] Database scaling strategies
- [x] Zero-downtime deployments
- [x] API Gateway patterns
- [x] Disaster recovery
- [x] Container security
- [x] Message queues vs event streaming
- [x] Logging and observability
- [x] Cost optimization
- [x] SRE principles

**Total: 20 questions** covering cloud platforms, container orchestration, DevOps practices, and SRE principles.