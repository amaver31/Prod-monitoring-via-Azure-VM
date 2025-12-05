# Prod-monitoring-via-Azure-VM
Self-Hosted Prometheus + Grafana on Azure VM for Private AKS on Azure VM


#FINAL PROD IMPLEMENTATION PLAN
#Self-Hosted Prometheus + Grafana on Azure VM for Private AKS
________________________________________
#🔹 PHASE 1 – PREREQUISITES (Before Any Work)
#1. Access Required
You / your manager must have:
•	✅ VM Contributor
•	✅ Network Contributor
•	✅ AKS Cluster User
•	✅ Reader on Resource Group
________________________________________
#2. Collect These Details from Azure Portal
From AKS → Networking / Properties:
•	AKS Name
•	Resource Group
•	VNet Name
•	Subnet Name
•	Region
•	Kubernetes Version
________________________________________
#3. Final Decision Confirmed
•	✔ Monitoring only for PROD
•	✔ AKS is PRIVATE
•	✔ Prometheus & Grafana will run on Azure VM
•	✔ Azure Managed Prometheus & Grafana will be disabled after validation
________________________________________
#🔹 PHASE 2 – CREATE MONITORING VM
#VM Configuration
•	OS: Ubuntu 22.04 LTS
•	Size: B2s
•	Disk: 64–128 GB Standard SSD
•	Network:
o	Same VNet as AKS
o	Same or peered Subnet
o	Public IP = Yes (only for Grafana UI)
________________________________________
#NSG Rules (Security)
Port	Allow From	Purpose
22	Office IP only	SSH
3000	Office IP only	Grafana
9090	❌ Blocked	Prometheus must be private
________________________________________
#🔹 PHASE 3 – CONNECT TO VM
ssh azureuser@<VM_PUBLIC_IP>
________________________________________
#🔹 PHASE 4 – INSTALL DOCKER
sudo apt update
sudo apt install -y docker.io docker-compose
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
logout
Login again.
________________________________________
#🔹 PHASE 5 – CONNECT VM TO PRIVATE AKS
#1. Install kubectl on VM
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
________________________________________
#2. Get kubeconfig from Azure
From a machine with Azure CLI:
az aks get-credentials \
  --resource-group <RG> \
  --name <AKS_NAME> \
  --overwrite-existing
Copy it to VM:
scp ~/.kube/config azureuser@<VM_IP>:/home/azureuser/.kube/config
Test from VM:
kubectl get nodes
✅ This must work before moving forward.
________________________________________
#🔹 PHASE 6 – DEPLOY METRICS COMPONENTS IN AKS
kubectl create namespace monitoring
Install kube-state-metrics
kubectl apply -f https://github.com/kubernetes/kube-state-metrics/releases/latest/download/kube-state-metrics.yaml
Install node-exporter
kubectl apply -f https://raw.githubusercontent.com/prometheus/node_exporter/master/examples/node-exporter-daemonset.yaml
________________________________________
#🔹 PHASE 7 – INSTALL PROMETHEUS ON VM
Create directories
mkdir -p ~/monitoring/prometheus
cd ~/monitoring
________________________________________
#Create prometheus.yml
global:
  scrape_interval: 15s

scrape_configs:

- job_name: 'kubernetes-nodes'
  kubernetes_sd_configs:
  - role: node
  scheme: https
  tls_config:
    insecure_skip_verify: true
  bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token

- job_name: 'kube-state-metrics'
  static_configs:
  - targets:
    - kube-state-metrics.kube-system.svc.cluster.local:8080
________________________________________
#Docker Compose for Prometheus
version: '3'
services:
  prometheus:
    image: prom/prometheus
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "127.0.0.1:9090:9090"
Start:
docker-compose up -d
Verify:
curl http://127.0.0.1:9090
________________________________________
#🔹 PHASE 8 – INSTALL GRAFANA
docker run -d \
  --name grafana \
  -p 3000:3000 \
  grafana/grafana
Open in browser:
http://<VM_PUBLIC_IP>:3000
Login:
admin / admin
________________________________________
#🔹 PHASE 9 – CONNECT GRAFANA TO PROMETHEUS
Grafana UI:
•	Settings → Data Sources → Add Data Source
•	Type: Prometheus
•	URL:
http://127.0.0.1:9090
•	Save & Test ✅
________________________________________
#🔹 PHASE 10 – IMPORT PROD DASHBOARDS
Import:
•	Kubernetes Cluster Overview
•	Node Metrics
•	Pod Metrics
•	Namespace Resource Usage
•	Application /metrics dashboards (if any)
________________________________________
#🔹 PHASE 11 – VALIDATION (CRITICAL FOR PROD)
Run both systems in parallel for 5–7 days:
✅ Check:
•	Node count
•	Pod count
•	CPU/Memory values
•	Alert firing
•	Query performance
•	Dashboard accuracy
✅ Only after this → proceed to cutover.
________________________________________
#🔹 PHASE 12 – CUTOVER (DISABLE AZURE MANAGED)
#After full sign-off:
•	Disable Azure Managed Prometheus
•	Disable Azure Managed Grafana
•	Disable Diagnostic Settings
•	Remove Azure Monitor Agent (if only used for metrics)

