
### Deployment kit for dataspace installation
tim-ds-kit provides the Helm-based deployment framework required to bootstrap and deploy a complete TIM Dataspace environment on Kubernetes using ArgoCD.
A Dataspace is a trusted digital ecosystem where multiple organizations can securely exchange data without losing ownership or control of their data.

A Dataspace is a secure environment for 
•	Publish data
•	Discover data
•	Request access to data
•	Negotiate legal agreements
•	Exchange data securely
while maintaining full ownership and control of data assets.

## TIM Dataspace Architecture:

The architecture consists of two major layers:
# Common Applications:
Shared infrastructure services used by the entire Dataspace:
•	Keycloak
•	PostgreSQL
•	OpenBao
•	OpenBao Init
•	Vault Webhook
•	Kafka
•	NGINX

# TIM Applications:
Dataspace-specific business services:

•	IdentityHub
•	IssuerService
•	TIM-EDC Control Plane
•	TIM-EDC Data Plane
•	DS-Catalog
•	Federated Catalog

# Repository Structure
tim-ds-kit
├── charts
│   └── templates
│       ├── application-vault-webhook.yaml
│       ├── application-tim-edc.yaml
│       ├── application-postgres.yaml
│       ├── application-openbao.yaml
│       ├── application-openbao-init.yaml
│       ├── application-nginx.yaml
│       ├── application-keycloak.yaml
│       ├── application-kafka.yaml
│       ├── application-issuerservice.yaml
│       ├── application-identityhub.yaml
│       ├── application-federated-catalog.yaml
│       ├── application-dscatalog.yaml
│       ├── 00-namespaces.yaml
│       └── 01-nginx-namespace.yaml
│
├── dataspace-bootstrap
│   └── bootstrap
│       ├── templates
│       │   ├── root-application.yaml
│       │   ├── git-secret.yaml
│       │   ├── application-repos.yaml
│       │   └── 00-argocd-namespace.yaml
│       └── Bootstrap_Install.sh
│
└── APP_Install.sh

## Deployment Overview
Deployment is performed in two phases:

# Phase 1 - Bootstrap Installation
The Bootstrap Helm chart performs:
•	ArgoCD namespace creation
•	Repository registration
•	Git credential configuration
•	Initial Dataspace bootstrap setup
Component	Purpose:

|--------------------------------------------------------------|
|00-argocd-namespace.yaml    |Creates ArgoCD namespace|
|git-secret.yaml	           |Stores Git repository credentials|
|application-repos.yaml	     |Registers repositories in ArgoCD|
|---------------------------------------------------------------|


# Phase 2 - Dataspace Application Installation
The Application Helm chart deploys all Dataspace services through ArgoCD Applications.
Application Components
Service	Description
Keycloak	Identity and Access Management
PostgreSQL	Database
OpenBao	Secrets Management
OpenBao Init	OpenBao Initialization
Vault Webhook	Secret Injection
Kafka	Messaging Platform
NGINX	Ingress Controller
IdentityHub	Identity Management
IssuerService	Credential Issuance
TIM EDC Control Plane	Connector Management
TIM EDC Data Plane	Secure Data Transfer
DS Catalog	Asset Discovery
Federated Catalog	Cross-participant Catalog

# Prerequisites:
Before starting the installation, ensure the following tools are available:
Kubernetes
Verify cluster access:
kubectl cluster-info
kubectl get nodes

Helm
Verify Installation:
helm version

Git
Verify Installation:
git --version

Required Access
•	Kubernetes Cluster Access
•	GitHub Username
•	GitHub Personal Access Token (PAT)
•	Permission to install resources in the cluster
•	Phase 1: Bootstrap Installation
•	The repository provides an automated installation script:

# Phase 1: Bootstrap Installation
The repository provides an automated installation script:
dataspace-bootstrap/bootstrap/Bootstrap_Install.sh
What the Script Does
The script automatically:
1.	Requests GitHub credentials.
2.	Clones the TIM DS Kit repository.
3.	Creates the argocd namespace if not present.
4.	Applies required CRDs.
5.	Installs the Bootstrap Helm chart.
6.	Configures Git repository access.
7.	Creates ArgoCD bootstrap applications.

Execute Bootstrap Installation
Navigate to:
cd dataspace-bootstrap/bootstrap
Provide execution permission:
chmod +x Bootstrap_Install.sh
Run the script:

./Bootstrap_Install.sh
Script Inputs
You will be prompted for:
Enter GitHub Username:
Enter GitHub Token:

Successful Output
After successful execution you should see:
Bootstrap installation completed.

Verify Bootstrap Installation
Verify Namespace
kubectl get ns argocd
``
Expected:
NAME STATUS AGE
argocd Active
Verify Pods
Verify ArgoCD Applications
kubectl get applications -n argocd
Verify Helm Release
helm ls -n argocd
Expected:
NAME NAMESPACE STATUS
bootstrap argocd deployed

Phase 2: Dataspace Application Installation
After Bootstrap deployment completes successfully, install the Dataspace applications.
The repository provides:
APP_Install.sh
What the Script Does
The script:
1.	Validates user inputs.
2.	Validates chart path.
3.	Validates values file.
4.	Deploys the application Helm chart.
5.	Creates all ArgoCD applications.
6.	Starts synchronization of Dataspace services.
Execute Application Installation
Provide execution permission:
chmod +x APP_Install.sh
Run the installer:
./APP_Install.sh
Script Inputs
Example:
Enter Release Name (example: dataspace1): dataspace1
Enter Chart Path (example: ./charts): ./charts
Enter Values File (example: values-governance.yaml): values-governance.yaml
Installation Summary Example
Release Name : dataspace1
Chart Path : ./charts
Values File : values-governance.yaml
Namespace : argocd
Verify Application Installation
Check Helm Release
helm ls -n argocd
Example:
NAME NAMESPACE REVISION STATUS
bootstrap argocd 1 deployed
dataspace1 argocd 1 deployed

Verify ArgoCD Applications
Expected applications:
Verify Synchronization Status
kubectl get applications -n argocd
``
Ensure all applications show:
SYNC STATUS : Synced
HEALTH STATUS : Healthy
Useful Commands:
View ArgoCD Applications
kubectl get applications -n argocd
``
View Application Details

kubectl describe application <application-name> -n argocd

View Application Details
Troubleshooting
Bootstrap Installation Failed
kubectl get pods -n <namespace>




Bootstrap Installation
There are two ways to install the Bootstrap components:
1.	Automated Installation (Recommended)
2.	Manual Installation
________________________________________
Option 1: Automated Installation (Recommended)
Navigate to the bootstrap directory:
cd dataspace-bootstrap/bootstrap
Option 2: Manual Installation
Step 1: Clone Repository
Replace the values below with your GitHub username and personal access token
git clone https://<GITHUB_USERNAME>:<GITHUB_TOKEN>@github.com/ipcei-tim-t3/tim-ds-kit.git
cd tim-ds-kit
Step 2: Create ArgoCD Namespace
kubectl apply -f - <<EOF
apiVersion: v1
kind: Namespace
metadata:
name: argocd
labels:
pod-security.kubernetes.io/enforce: privileged
pod-security.kubernetes.io/audit: privileged
pod-security.kubernetes.io/warn: privileged
EOF
Verify:
kubectl get ns argocd

Step 3: Apply Bootstrap CRDs
Navigate to Bootstrap chart:
cd dataspace-bootstrap/bootstrap
Apply CRD:
kubectl apply -f crds/

Step 4: Install Bootstrap Helm Chart
Replace the placeholders with your GitHub credentials.
helm upgrade --install bootstrap . \
-n argocd \
--create-namespace \
--set argocd.createNamespace=false \
--set argocd.installRepositories=true \
--set git.username="<GITHUB_USERNAME>" \
--set git.password="<GITHUB_TOKEN>"
Step 5: Verify Bootstrap Installation
helm ls -n argocd
Verify ArgoCD applications:
kubectl get applications -n argocd

Dataspace Application Installation
After Bootstrap installation is completed successfully, install the Dataspace applications.
There are two deployment options:
1.	Automated Installation (Recommended)
2.	Manual Installation
Option 2: Manual Installation
Step 1: Navigate to Repository
Go to the location where:
•	tim-ds-kit repository is cloned
•	Values file is available

cd ~/tim-ds-kit/charts

Step 2: Install Dataspace Helm Chart
Generic format:
helm upgrade --install <RELEASE_NAME> <CHART_PATH> \
-f <VALUES_FILE> \
-n argocd \
--create-namespace

Example:

helm upgrade --install dataspace1 . \
-f values.yaml \
-n argocd \
--create-namespace

Step 3: Verify Installation
Check Helm releases:
helm ls -n argocd
Expected Output:
NAME NAMESPACE REVISION STATUS
bootstrap argocd 3 deployed
dataspace1 argocd 1 deployed

Step 4: Verify ArgoCD Applications
kubectl get applications -n argocd


