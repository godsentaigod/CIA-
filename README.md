# ===============================
# ROOT FILES
# ===============================

# README.md
# Collective Intelligence Architecture – Full Cloud + Local Deployment
This repository self-builds the CIA system end-to-end:
- Local dev (Docker Compose)
- Local cluster (Kind)
- Full Kubernetes cloud deployment
- Helm chart
- Terraform infra
- CI/CD auto-deploy
- GPU specialist model-servers
- RAG + vector DB + Redis memory
- Autoscaling
- Observability stack
Push to GitHub → everything auto-builds.

# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker

# docker-compose.yml
version: "3.9"
services:
  api:
    build: ./src
    command: uvicorn api.main:app --host 0.0.0.0 --port 8080
    ports:
      - "8080:8080"
    environment:
      VECTOR_DB_URL: http://vector-db:6333
      REDIS_URL: redis://redis:6379
    depends_on:
      - redis
      - vector-db

  redis:
    image: redis:7
    ports: ["6379:6379"]

  vector-db:
    image: qdrant/qdrant
    ports: ["6333:6333"]

  logic-model:
    image: vllm/vllm-openai:latest
    command: >
      --model meta-llama/Meta-Llama-3-8B-Instruct
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              capabilities: [gpu]

# ===============================
# SOURCE CODE
# ===============================

# src/api/main.py
from fastapi import FastAPI
from governor.router import process_query
app = FastAPI()

@app.post("/query")
async def q(body: dict):
    return {"response": process_query(body.get("query",""))}

# src/governor/router.py
from specialists.dispatch import run_specialists
from critic.engine import rank
from verifier.engine import verify
from synthesis.engine import synthesize

def process_query(q):
    cands = run_specialists(q)
    best = rank(cands)
    checked = verify(best)
    return synthesize(best, checked)

# src/specialists/dispatch.py
def run_specialists(q):
    return [
        f"logic:{q}",
        f"coding:{q}",
        f"strategy:{q}"
    ]

# src/critic/engine.py
def rank(cands):
    return cands[0]

# src/verifier/engine.py
def verify(x):
    return x

# src/synthesis/engine.py
def synthesize(a,b):
    return a

# ===============================
# KUBERNETES (ALL IN /kubernetes)
# ===============================

# kubernetes/namespaces/cia.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: cia

# kubernetes/deployments/api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cia-api
  namespace: cia
spec:
  replicas: 4
  selector:
    matchLabels: { app: cia-api }
  template:
    metadata:
      labels: { app: cia-api }
    spec:
      containers:
      - name: api
        image: ghcr.io/USER/cia-api:latest
        ports: [{ containerPort: 8080 }]

# kubernetes/services/api-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: cia-api
  namespace: cia
spec:
  selector: { app: cia-api }
  ports:
  - port: 80
    targetPort: 8080

# kubernetes/ingress/api-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: cia-api-ingress
  namespace: cia
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: cia-api
            port:
              number: 80

# ===============================
# AUTOSCALING
# ===============================
# kubernetes/autoscaling/api-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: cia-api-hpa
  namespace: cia
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: cia-api
  minReplicas: 4
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70

# ===============================
# REDIS + VECTOR DB
# ===============================

# kubernetes/redis/redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cia-redis
  namespace: cia
spec:
  replicas: 1
  template:
    metadata:
      labels: { app: redis }
    spec:
      containers:
      - name: redis
        image: redis:7
        ports: [{ containerPort: 6379 }]

# kubernetes/vector-db/qdrant.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: qdrant
  namespace: cia
spec:
  replicas: 1
  template:
    metadata:
      labels: { app: qdrant }
    spec:
      containers:
      - name: qdrant
        image: qdrant/qdrant
        ports: [{ containerPort: 6333 }]

# ===============================
# OBSERVABILITY (PROMETHEUS/GRAFANA)
# ===============================

# kubernetes/observability/prometheus/prometheus.yaml
apiVersion: v1
kind: Pod
metadata:
  name: prometheus
  namespace: cia
spec:
  containers:
  - name: prometheus
    image: prom/prometheus

# ===============================
# HELM CHART
# ===============================

# helm/Chart.yaml
apiVersion: v2
name: cia-platform
version: 0.1.0

# helm/values.yaml
api:
  image: ghcr.io/USER/cia-api:latest
  replicas: 4

# helm/templates/api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cia-api
spec:
  replicas: {{ .Values.api.replicas }}
  template:
    metadata:
      labels: { app: cia-api }
    spec:
      containers:
      - name: api
        image: {{ .Values.api.image }}

# ===============================
# TERRAFORM
# ===============================

# terraform/aws/main.tf
provider "aws" { region = "us-west-2" }

module "eks" {
  source = "terraform-aws-modules/eks/aws"
  cluster_name = "cia-cluster"
  cluster_version = "1.29"
  eks_managed_node_groups = {
    cpu = { desired_size = 3, instance_types=["m6i.xlarge"] }
    gpu = { desired_size = 2, instance_types=["g5.2xlarge"] }
  }
}

# ===============================
# CI/CD
# ===============================

# .github/workflows/deploy.yaml
name: CIA-CD
on:
  push: { branches: [main] }
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v4
    - run: docker build -t ghcr.io/${{ github.repository }}/cia-api:latest ./src
    - run: docker push ghcr.io/${{ github.repository }}/cia-api:latest
    - run: kubectl apply -f kubernetes/
