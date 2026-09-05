💬 Realtime Chat App — Production-Ready Kubernetes & Docker DeploymentA full-stack, 3-tier real-time chat application containerized with Docker and orchestrated locally on Kubernetes (KinD) using NGINX Ingress, PersistentVolumes, and native Secrets management.💡 Project Attribution & Scope:The underlying web application (frontend, backend API, and WebSocket engine) was developed by its original creator. This repository documents my hands-on DevOps, Containerization, and Cloud-Native Infrastructure practice—designing Kubernetes manifests from scratch, setting up internal service discovery, managing persistent storage lifecycles, configuring ingress routing, and debugging orchestration bottlenecks.📌 Table of ContentsWhat, Why, and HowArchitecture & Traffic FlowTech StackRepository StructurePrerequisitesStep-by-Step Deployment GuideNetworking & Access MethodsDevOps Challenges & Real-World FixesEssential Commands Cheat SheetCleanup🎯 What, Why, and HowWhat?A resilient, 3-tier microservices deployment comprising:Frontend: React + Vite Single Page Application served via a high-performance NGINX web server.Backend: Node.js/Express REST & WebSocket service handling real-time chat operations and user authentication.Database: A stateful MongoDB instance with decoupled persistent volume storage.Why?Running multi-container applications locally using plain Docker or Docker Compose often hides the real-world complexity of cloud deployments. Migrating this 3-tier app to Kubernetes provides hands-on experience in:Decoupling application configuration and sensitive data via ConfigMaps and Secrets.Resolving PVC binding lifecycles (WaitForFirstConsumer) and persistent data retention across pod restarts.Managing Layer-7 Ingress path routing (/ vs /api) without unintentionally stripping or corrupting application routes.Debugging network tunnels and container-in-container environments like KinD (Kubernetes in Docker).How?The infrastructure is organized into declarative YAML manifests inside k8s/, deployed into an isolated namespace (chat-app), fronted by an NGINX Ingress Controller, and exposed locally through host-bound port forwarding.🏗️ Architecture & Traffic FlowPlaintext                             [ Web Browser / Client ]
                                        │
                                        ▼ (Port 80 / 8080)
                     [ NGINX Ingress Controller (chat-app) ]
                                        │
                    ┌───────────────────┴───────────────────┐
                    │                                       │
     Path: "/"      ▼                        Path: "/api"   ▼
            [ frontend-service ]                     [ backend-service ]
             (ClusterIP: 80)                          (ClusterIP: 5001)
                    │                                       │
                    ▼                                       ▼
            [ Frontend Pod ]                         [ Backend Pod ]
             (React + Vite)                         (Node.js/Express)
                                                            │
                                                            ▼ (DNS: mongodb-service:27017)
                                                   [ mongodb-service ]
                                                    (ClusterIP: 27017)
                                                            │
                                                            ▼
                                                     [ MongoDB Pod ]
                                                            │
                                                            ▼
                                                   [ PersistentVolumeClaim ]
                                                            │
                                                            ▼
                                                   [ PersistentVolume ]
🛠️ Tech StackDomainTechnologiesContainerization & OrchestrationDocker, Kubernetes, KinD (Kubernetes in Docker)Ingress & NetworkingNGINX Ingress Controller, Kubernetes ClusterIP ServicesStorage & SecurityPersistentVolumes (PV), PersistentVolumeClaims (PVC), Kubernetes SecretsApplication LayerReact.js, Vite, Node.js, Express, Socket.IO / WebSockets, MongoDBEnvironmentLinux / Ubuntu (WSL2), kubectl, curl📂 Repository StructurePlaintext.
├── k8s/
│   ├── namespace.yml            # Isolated Kubernetes namespace (chat-app)
│   ├── secrets.yml              # Base64-encoded JWT secret & database credentials
│   ├── mongodb-pv.yml           # HostPath/Local PersistentVolume definition
│   ├── mongo-pvc.yml            # PersistentVolumeClaim requested by MongoDB
│   ├── mongodb-service.yml      # ClusterIP service routing port 27017
│   ├── deployment-mongo.yml     # Stateful MongoDB Deployment referencing PVC
│   ├── backend-service.yml      # ClusterIP service exposing backend on 5001
│   ├── deployment-backend.yml   # Node.js backend Deployment consuming secrets
│   ├── frontend-service.yml     # ClusterIP service exposing frontend on port 80
│   ├── deployment-frontend.yml  # Frontend Deployment running NGINX web container
│   └── ingress.yml              # Path-based Ingress routing (/ and /api)
└── README.md
📋 PrerequisitesEnsure you have the following tools installed and running:Docker EnginekubectlKinD or an equivalent local Kubernetes setup (e.g., Minikube)Verify your cluster connectivity:Bashkubectl cluster-info
🚀 Step-by-Step Deployment Guide1. Install NGINX Ingress ControllerFor KinD clusters, deploy the ingress controller using the official provider manifest:Bashkubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait until the Ingress Controller pod is in 'Running' state
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=120s
2. Provision Namespace & SecretsCreate the dedicated environment and inject security tokens:Bashkubectl apply -f k8s/namespace.yml
kubectl apply -f k8s/secrets.yml
Verify that the secret was created:Bashkubectl get secret chatapp-secrets -n chat-app
3. Deploy the Database Layer (MongoDB + Persistent Storage)Apply the storage definitions, the service discovery endpoint, and the database container:Bashkubectl apply -f k8s/mongodb-pv.yml
kubectl apply -f k8s/mongo-pvc.yml
kubectl apply -f k8s/mongodb-service.yml
kubectl apply -f k8s/deployment-mongo.yml
Verify that the PVC binds to the PV:Bashkubectl get pvc,pv -n chat-app
4. Deploy the Backend APIStart the backend service and pod instances:Bashkubectl apply -f k8s/backend-service.yml
kubectl apply -f k8s/deployment-backend.yml
Check the startup status and database connection logs:Bashkubectl logs -l tier=backend -n chat-app --tail=30
5. Deploy Frontend and Ingress RoutingDeploy the client interface and the routing layer:Bashkubectl apply -f k8s/frontend-service.yml
kubectl apply -f k8s/deployment-frontend.yml
kubectl apply -f k8s/ingress.yml
6. Verify Full Cluster StateRun this command to ensure every tier is operational:Bashkubectl get all,pvc,ingress -n chat-app
Expected status:All pods (deployment-frontend, deployment-backend, deployment-mongo) report Running.The mongo-pvc status is Bound.Ingress has backend endpoints mapped.🌐 Networking & Access MethodsOption A: Local Port-Forward via Ingress (Standard)Forward incoming traffic through the NGINX Ingress Controller service:Bash# Bind to all interfaces (0.0.0.0) so host machine traffic reaches WSL2:
kubectl port-forward --address 0.0.0.0 -n ingress-nginx svc/ingress-nginx-controller 8080:80
Open your browser and navigate to:Plaintexthttp://localhost:8080/
Option B: Custom Host Domain Setup (chat-aj.com)If your ingress.yml specifies host: chat-aj.com:Add the loopback mapping to your hosts file:Windows: C:\Windows\System32\drivers\etc\hostsLinux / macOS: /etc/hostsPlaintext127.0.0.1 chat-aj.com
Start the ingress port-forward:Bashkubectl port-forward --address 0.0.0.0 -n ingress-nginx svc/ingress-nginx-controller 8080:80
Open [http://chat-aj.com:8080](http://chat-aj.com:8080) in your browser.🔧 DevOps Challenges & Real-World FixesIssue EncounteredRoot CauseFix ImplementedCreateContainerConfigErrorDeployment expected key jwt under chatapp-secrets, but the secret manifest defined jwd.Corrected key spelling in secrets.yml and verified via kubectl describe pod.404 Not Found via IngressThe Ingress Controller received Host: localhost while ingress.yml strictly enforced - host: chat-aj.com.Relaxed host matching rules for local testing and verified routing via explicit curl -H "Host: chat-aj.com" checks.Pending PVC on MongoDBDefault storage class behavior waiting for initial pod scheduling (WaitForFirstConsumer).Defined a dedicated local/hostPath PersistentVolume matching access modes and storage specifications.WSL2 to Windows Host Connection Dropskubectl port-forward bound strictly to loopback interface 127.0.0.1.Appended --address 0.0.0.0 to allow the Windows host network adapter to bridge into the WSL2 VM.📜 Essential Commands Cheat SheetPod & Service InspectionBash# Check status of pods across all namespaces
kubectl get pods -A

# Follow live logs for backend or frontend
kubectl logs -f -l tier=backend -n chat-app
kubectl logs -f -l tier=frontend -n chat-app

# Inspect internal routing endpoints
kubectl get endpoints -n chat-app
Testing Ingress via CLIBash# Direct test with host header override
curl -i -H "Host: chat-aj.com" http://localhost:8080/

# Test backend health route
curl -i http://localhost:8080/api
Managing Port ConflictsBash# Terminate lingering port-forward sessions
sudo pkill -f "kubectl port-forward"

# Identify and kill any process blocking a port
sudo fuser -k 8080/tcp
🧹 CleanupTo delete all deployed resources and namespaces cleanly:Bashkubectl delete -f k8s/
