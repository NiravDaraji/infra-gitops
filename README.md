
### Deployment kit for dataspace installation
tim-ds-kit provides the Helm-based deployment framework required to bootstrap and deploy a complete TIM Dataspace environment on Kubernetes using ArgoCD..

A Dataspace is a trusted digital ecosystem where multiple participants can securely exchange data without losing ownership or control of their data.

A Dataspace is a secure environment for :
-   Publish data
-   Discover data
-   Request access to data
-   Exchange data securely

## TIM Dataspace Architecture:
**The architecture consists of two major layers:**
# Common Applications:
-   Keycloak
-   PostgreSQL
-   OpenBao
-   OpenBao Init
-   Vault Webhook
-   Kafka

# TIM Applications:
Dataspace-specific business services:

-   IdentityHub
-   IssuerService
-   TIM-EDC Control Plane
-   TIM-EDC Data Plane
-   DS-catalog
-   Federated Catalog

## Deployment Overview

Deployment is performed in two phases:

- Phase 1: Bootstrap Installation
- Phase 2: Dataspace Application Installation

# Phase 1: Bootstrap Installation
The Bootstrap Helm chart performs:
-   ArgoCD namespace creation
-   Repository registration
-   Git credential configuration
-   Initial Dataspace bootstrap setup
  
| Component | Purpose |
|-----------|---------|
| 00-argocd-namespace.yaml | Creates ArgoCD namespace |
| git-secret.yaml | Stores Git repository credentials |
| application-repos.yaml | Registers repositories in ArgoCD |

> [!IMPORTANT]
> **Bootstrap installation is a one-time activity per cluster.**
>
> The Bootstrap chart deploys and configures the foundational platform components required by the TIM Dataspace environment, including:
>
> - ArgoCD
> - PostgreSQL Operator
> - Kafka / Confluent Operator
> - Git Repository Integrations
> - ArgoCD Bootstrap Configuration
>
> **Install the Bootstrap chart only once per Kubernetes cluster.** After the bootstrap process is completed, any additional Dataspace environments within the same cluster can be deployed directly using the Dataspace application Helm chart without reinstalling Bootstrap.


# Phase 2: Dataspace Application Installation
The Application Helm chart deploys all Dataspace services through ArgoCD Applications.

| Service | Description |
|----------|-------------|
| Keycloak | Identity and Access Management |
| PostgreSQL | Database |
| OpenBao | Secrets Management |
| OpenBao-Init | OpenBao Initialization |
| Vault Webhook | Secret Injection |
| Kafka | Messaging Platform |
| NGINX | Ingress Controller |
| IdentityHub | Identity Management |
| IssuerService | Credential Issuance |
| TIM EDC Control Plane | Connector Management |
| TIM EDC Data Plane | Secure Data Transfer |
| DS Catalog | Asset Discovery |
| Federated Catalog | Cross-participant Catalog |

# Prerequisites:
Before starting the installation, ensure the following tools are available:
- Kubernetes
  
  Verify cluster access:
    ```bash
    kubectl cluster-info
    kubectl get nodes
    ```

- Helm
  
  Verify Helm Installation:
    ```bash
    helm version
    ```

- Git
  
  Verify Git Installation:
    ```bash
    git --version
    ```

Required Access
-   Kubernetes Cluster Access
-   GitHub Username
-   GitHub Personal Access Token (PAT)
-   Permission to install resources in the cluster
   
> [!IMPORTANT]
> **Note:** This deployment kit assumes that **ArgoCD is not** installed on the cluster in the `argocd` namespace.
>
> If ArgoCD is already deployed in the `argocd` namespace, the following components must be installed manually.
>
> # 1. Install Required Operators
>
> Navigate to:
>
> tim-ds-kit/dataspace-bootstrap/bootstrap/charts
>
> Install the following operators manually:
>
> - PostgreSQL Operator
> - Kafka / Confluent Operator
>
> # 2. Create Bootstrap ArgoCD Applications
>
> Navigate to:
>
> tim-ds-kit/dataspace-bootstrap/bootstrap/templates
>
> Apply the bootstrap application manifests manually after validating and updating the required configuration values in templates, values present in: tim-ds-kit/dataspace-bootstrap/bootstrap/values.yaml
>
> kubectl apply -f git-secret.yaml
> 
> kubectl apply -f application-repos.yaml


## Phase 1: Bootstrap Installation
There are two ways to install the Bootstrap components:
1. **Automated Installation** using the helper script (`Bootstrap_Install.sh`)
2. **Manual Installation** using Helm commands

# Option 1: Automated Installation using the helper script (Bootstrap_Install.sh)
The repository provides an automated installation script: tim-ds-kit/dataspace-bootstrap/bootstrap/Bootstrap_Install.sh

The script automatically:
-  Requests GitHub credentials.
-  Clones the tim-ds-kit repository.
-  Creates the argocd namespace if not present.
-  Applies required CRDs.
-  Installs the Bootstrap Helm chart.
-  Configures Git repository access.
-  Creates ArgoCD bootstrap applications.

Execute Bootstrap Installation

Download the `Bootstrap_Install.sh` script from:

`tim-ds-kit/dataspace-bootstrap/bootstrap/Bootstrap_Install.sh`

and execute it from the directory where you want the repository to be cloned and the bootstrap installation to be performed.

Provide execution permission:
```bash
chmod +x Bootstrap_Install.sh
```

Run the script:
```bash
./Bootstrap_Install.sh
```

Script Inputs

You will be prompted for:

    ```text
    Enter GitHub Username:
    Enter GitHub Token:
    ```

Successful Output

After successful execution you should see:
    ```text
    Bootstrap installation completed.
    ```

Verify Bootstrap Installation:

Verify Namespace and Pod status:

```bash
kubectl get ns argocd
kubectl get pod -n argocd
```

**Expected Output**

```console
kubectl get ns argocd
NAME      STATUS   AGE
argocd    Active   X

kubectl get pod -n argocd
NAME                                               READY   STATUS 
bootstrap-argocd-application-controller-0          1/1     Running
bootstrap-argocd-applicationset-controller-xxxxx   1/1     Running
bootstrap-argocd-dex-server-xxxxx                  1/1     Running
bootstrap-argocd-notifications-controller-xxxxx    1/1     Running
bootstrap-argocd-redis-xxxxx                       1/1     Running
bootstrap-argocd-repo-server-xxxxx                 1/1     Running
bootstrap-argocd-server-xxxxx                      1/1     Running
bootstrap-confluent-for-kubernetes-xxxxx           1/1     Running
bootstrap-postgres-operator-xxxxx                  1/1     Running
```

Verify ArgoCD Applications
```bash
kubectl get applications -n argocd
```

**Expected Output**

```console
NAME 		         SYNC STATUS    HEALTH STATUS
argocd-repositories  Synced         Healthy
```

Verify Helm Release
```bash
helm ls -n argocd
```
**Expected Output**

```console
NAME 		NAMESPACE 	STATUS
bootstrap 	argocd 		deployed
```

# Option 2:  Manual Installation

**Step 1: Clone repository**
Replace the values below with your GitHub username and personal access token

```bash
git clone https://<GITHUB_USERNAME>:<GITHUB_TOKEN>@github.com/ipcei-tim-t3/tim-ds-kit.git

cd tim-ds-kit
```
**Step 2: ArgoCD namespace creation**
```bash
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
```
Verify:

```bash
kubectl get ns argocd
```

**Step 3: Apply Bootstrap CRDs**

Navigate to Bootstrap chart:
```bash
cd dataspace-bootstrap/bootstrap
```
Apply CRD:
```bash
kubectl apply -f crds/
```

**Step 4: Install Bootstrap Helm Chart**
Replace the placeholders with your GitHub credentials.

```bash
helm upgrade --install bootstrap . \
-n argocd \
--create-namespace \
--set argocd.createNamespace=false \
--set argocd.installRepositories=true \
--set git.username="<GITHUB_USERNAME>" \
--set git.password="<GITHUB_TOKEN>"
```
**Step 5: Verify Bootstrap Installation**

Verify Helm chart:

```bash
helm ls -n argocd
```
Verify ArgoCD applications:

```bash
kubectl get applications -n argocd
```

## Phase 2: Dataspace Application Installation
After the Bootstrap installation is completed successfully, deploy the Dataspace applications using a Helm values file.

The deployment supports any valid values file generated for your environment, for example:

- `values.yaml`
- `values-smko.yaml`
- `values-dataspace.yaml`
- `values-governance.yaml`

You can install the Dataspace environment using any custom values file that **contains the required configuration for your deployment.**

There are two deployment options:
1. **Automated Installation** using the helper script (`APP_Install.sh`)
2. **Manual Installation** using Helm commands

# Option 1: Automated Installation using the helper script (APP_Install.sh)
The repository provides:

APP_Install.sh script does

-  Validates user inputs.
-  Validates chart path.
-  Validates values file.
-  Deploys the application Helm chart.
-  Creates all ArgoCD applications.
-  Starts synchronization of Dataspace services.

Provide execution permission:
```bash
chmod +x APP_install.sh
```
Run the installer:
```bash
./APP_install.sh
```

Script Inputs

```text
Example:
Enter Release Name (example: dataspace): dataspace
Enter Chart Path (example: ./charts): /home/ds/tim-ds-kit/charts
Enter Values File (example: values-governance.yaml): /home/ds/tim-ds-kit/charts/values-governance.yaml

Installation Summary Example:

Release Name : dataspace
Chart Path : /home/ds/tim-ds-kit/charts
Values File : /home/ds/tim-ds-kit/charts/values-governance.yaml
Namespace : argocd
```

Verify Application Installation:

Check Helm Release

```bash
helm ls -n argocd
```
Example:

```console
NAME 		NAMESPACE 		REVISION	STATUS
bootstrap 	argocd 			1 		    deployed
dataspace1	argocd 		    1 		    deployed
```

Verify ArgoCD Applications:

```bash
Kubectl get application -n argocd
```
Ensure all applications show:

```console
SYNC STATUS :   Synced
HEALTH STATUS : Healthy
```

# Option 2: Manual Installation

**Step 1: Navigate to repository**
Go to the location where:

-	tim-ds-kit repository is cloned
-	Values file is available

cd ~/tim-ds-kit/charts

**Step 2: Install Dataspace Helm Chart**

Generic format:

```bash
helm upgrade --install <RELEASE_NAME> <CHART_PATH> \
-f <VALUES_FILE> \
-n argocd \
--create-namespace
```

Example:

```bash
helm upgrade --install dataspace1 . \
-f values.yaml \
-n argocd \
--create-namespace
```

**Step 3: Verify Installation**

Check Helm releases:

```bash
helm ls -n argocd
```
Expected Output:

```console
NAME 		NAMESPACE 		REVISION 	STATUS
bootstrap 	argocd 			1 		    deployed
dataspace	argocd 			1 		    deployed
```

**Step 4: Verify ArgoCD applications**

```bash
kubectl get applications -n argocd
```

# Useful Commands:

View ArgoCD Applications

```bash
kubectl get applications -n argocd
```
View Application Details

```bash
kubectl describe application <application-name> -n argocd
```
# Troubleshooting:

```bash
kubectl get pod-n <namespace>
kubectl describe pod <podname> -n <namespace>
kubectl logs <podname> -n <namespace>
```
