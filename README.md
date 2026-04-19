# migrosone-2048case

2048 on Kubernetes (KIND)
Play the classic 2048 game running inside a local Kubernetes cluster, accessible at http://2048.local.

What's Inside
2048-k8s/
├── Dockerfile            # nginx:alpine serving the 2048 static app
├── nginx.conf            # minimal nginx config
├── app/                  # 2048 source code (gabrielecirulli/2048)
└── k8s/
    ├── kind-config.yaml  # KIND cluster config with port mapping
    ├── deployment.yaml   # Kubernetes Deployment
    ├── svc.yaml          # Kubernetes Service (ClusterIP)
    └── ingress.yaml      # Kubernetes Ingress (2048.local → pod)

Prerequisites
Install these before you start:
ToolInstallCheckDocker Desktopdocker.com/products/docker-desktopdocker --versionKINDbrew install kindkind --versionkubectlbrew install kubectlkubectl version --clientGitbrew install gitgit --version

macOS only: This guide uses Homebrew. Install it from brew.sh if you don't have it.


Step-by-Step Setup
1. Clone this repo
bashgit clone https://github.com/YOUR_USERNAME/2048-k8s.git
cd 2048-k8s
2. Get the 2048 source code
bashgit clone https://github.com/gabrielecirulli/2048.git app
rm -rf app/.git
3. Build the Docker image
bashdocker build -t 2048:latest .
4. Create the KIND cluster
bashkind create cluster --config k8s/kind-config.yaml --name 2048-cluster
This creates a local Kubernetes cluster with port 80 mapped to your machine.
5. Load the image into the cluster
bashkind load docker-image 2048:latest --name 2048-cluster

Without this step, Kubernetes can't find the image — it would try to pull from Docker Hub and fail.

6. Install NGINX Ingress Controller
bashkubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
Wait for it to be ready:
bashkubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s
7. Deploy the app
bashkubectl create namespace 2048
kubectl apply -f k8s/deployment.yaml -n 2048
kubectl apply -f k8s/svc.yaml -n 2048
kubectl apply -f k8s/ingress.yaml -n 2048
8. Add the local domain to /etc/hosts
bashecho "127.0.0.1  2048.local" | sudo tee -a /etc/hosts
9. Open the game 🎮
Open your browser and go to:
http://2048.local

Verify Everything is Running
bashkubectl get pods -n 2048      # STATUS should be Running
kubectl get svc -n 2048       # game-2048-svc should appear
kubectl get ingress -n 2048   # ADDRESS should be localhost

Stopping & Starting
Stop (delete the cluster)
bashkind delete cluster --name 2048-cluster
This removes the cluster but keeps your files and Docker image intact.
Start again
bashkind create cluster --config k8s/kind-config.yaml --name 2048-cluster
kind load docker-image 2048:latest --name 2048-cluster
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl create namespace 2048
kubectl apply -f k8s/deployment.yaml -n 2048
kubectl apply -f k8s/svc.yaml -n 2048
kubectl apply -f k8s/ingress.yaml -n 2048
Remove the local domain entry (optional)
bashsudo sed -i '' '/2048.local/d' /etc/hosts

How It Works
Browser (http://2048.local)
        │
        ▼
  /etc/hosts  →  127.0.0.1
        │
        ▼
  KIND cluster (port 80)
        │
        ▼
  NGINX Ingress Controller
        │
        ▼
  Service (ClusterIP)
        │
        ▼
  Pod (nginx:alpine + 2048 app)

Dockerfile — packages the 2048 static files into an nginx container
KIND — runs a full Kubernetes cluster locally inside Docker
Ingress — routes 2048.local traffic to the correct pod
/etc/hosts — makes your Mac resolve 2048.local to 127.0.0.1