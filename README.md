# Azure DTC Demo with Container Apps + Private Endpoints + Infoblox vNIOS

This repository demonstrates how to use **Azure Container Apps** deployed in separate subnets with **Private Endpoints**, and **Infoblox vNIOS** as an internal authoritative DNS with **DNS Traffic Control (DTC)** for failover between the endpoints based on health checks.

---

## 📁 Project Structure

```plaintext
azure-dtc-demo/
├── app/                    # Flask app with /health
│   ├── Dockerfile
│   └── app.py
├── terraform/
│   ├── main.tf            # Root Terraform entrypoint
│   ├── variables.tf
│   ├── network/           # VNet and subnet setup
│   ├── app/               # ACA + PE config using Docker Hub image
│   ├── vnios/             # vNIOS VM deployment with cloud-init
│   └── client/            # Windows VM for testing DTC
└── README.md              # Setup guide
```

---

## 🚀 Step-by-Step Instructions

### 1. ✅ Prerequisites
- Azure CLI
- Terraform >= 1.4
- Docker Hub account

---

### 2. 🐍 App Build and Push (Docker Hub)
```bash
# Build your image
cd app

docker build -t igorracic/dtc-demo:latest .
docker push igorracic/dtc-demo:latest
```
Ensure the image is public, or you'll need to supply credentials in Terraform.

---

### 3. 🧱 Infrastructure Deployment
```bash
cd terraform
terraform init
terraform apply -auto-approve
```
This will create:
- Resource group
- VNet with 4 subnets: `vnios`, `aca1`, `aca2`, `client`
- 2 Azure Container Apps (same image, different endpoints)
- 2 Private Endpoints
- vNIOS VM with static IP
- **Windows VM for RDP testing** (with public IP + NSG allowing port 3389)

---

### 4. ⚙️ DTC Configuration (Manual or WAPI)
On Infoblox UI or via WAPI:
- Zone: `app.internal`
- DTC Pool: `aca-shared`
  - Member 1: Private IP of ACA 1 PE
  - Member 2: Private IP of ACA 2 PE
- Health Monitor: TCP/HTTPS to `/health`
- Mode: Failover or Weighted Round Robin

---

### 5. 🔍 Testing
RDP into the deployed **Windows client VM** using the public IP output by Terraform. Then test DNS + HTTP:
```powershell
nslookup app.internal <vnios_ip>
curl http://app.internal/health
```
You can simulate failure by stopping one ACA revision.

---

## 📦 Module Highlights

### 🔹 `app/app.py`
```python
from flask import Flask
app = Flask(__name__)

@app.route("/")
def index():
    return "Hello from Container App!"

@app.route("/health")
def health():
    return "OK", 200
```

### 🔹 `app/Dockerfile`
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY app.py .
RUN pip install flask
CMD ["python", "app.py"]
```

### 🔹 Terraform Modules
Each of these is written with reusability and clarity in mind:
- `network/`: creates VNet, 4 subnets, NSGs
- `app/`: deploys ACA environments with Docker Hub image and Private Endpoints
- `vnios/`: deploys vNIOS with static IP and custom cloud-init config
- `client/`: deploys Windows VM with RDP access to validate DTC logic

---

## 🔚 Final Notes
- This setup simulates internal service routing and health-aware DNS-based failover.
- Great for secure API delivery, multiregion private apps, or split-brain DNS testing.
- Windows test VM is useful to demo internal-only resolution with browser + nslookup.

---

Let me know if you'd like WAPI scripts included for auto-configuring DTC pools and monitors.
