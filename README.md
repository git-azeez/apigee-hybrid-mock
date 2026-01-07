```markdown
# Apigee Hybrid CI/CD with Argo Workflows

This repository contains the source code for Apigee Hybrid API proxies and a reusable CI/CD pipeline template using **Argo Workflows**. It is designed to deploy proxies to Apigee Hybrid on Google Cloud using **Workload Identity** (keyless authentication) for security and a **WorkflowTemplate** architecture for reusability.

## 📂 Project Structure

```text
.
├── apigee-workflow-template.yaml   # The reusable Argo Workflow Template
└── src
    └── gateway
        ├── hello-world-mock    # The API Proxy source code
        │   ├── apiproxy        # Standard Apigee proxy structure
        │   └── pom.xml         # Child POM (inherits config from parent)
        └── parent-pom          # Central Maven configuration
            └── pom.xml         # Defines plugins, auth settings, and profiles

```

## 🚀 Prerequisites

Before running the pipeline, ensure you have the following installed on your GKE cluster:

1. **Apigee Hybrid**: Installed and running in your Kubernetes cluster.
2. **Argo Workflows**: Installed in the `argo` namespace.
```bash
kubectl create namespace argo
kubectl apply -n argo -f [https://github.com/argoproj/argo-workflows/releases/download/v3.5.5/install.yaml](https://github.com/argoproj/argo-workflows/releases/download/v3.5.5/install.yaml)

```


3. **Argo CLI**: (Optional, for easier management)
```bash
curl -sLO [https://github.com/argoproj/argo-workflows/releases/latest/download/argo-linux-amd64.gz](https://github.com/argoproj/argo-workflows/releases/latest/download/argo-linux-amd64.gz)
gunzip argo-linux-amd64.gz && chmod +x argo-linux-amd64 && sudo mv argo-linux-amd64 /usr/local/bin/argo

```



---

## 🔐 Setup: Authentication

This project uses two forms of authentication: **GitHub Credentials** (to clone code) and **Workload Identity** (to deploy to Google Cloud).

### 1. Create GitHub Secret (Required)

Create a Kubernetes secret in the `argo` namespace containing your Git username and Personal Access Token (PAT). This allows the workflow to clone private repositories.

```bash
kubectl create secret generic github-creds -n argo \
  --from-literal=username=YOUR_GITHUB_USER \
  --from-literal=token=YOUR_PERSONAL_ACCESS_TOKEN

```

### 2. Configure Workload Identity (GCP)

Follow these steps to allow the Argo pipeline to deploy without JSON keys.

**A. Create Google Service Account (GSA)**

```bash
gcloud iam service-accounts create apigee-deployer
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
    --member "serviceAccount:apigee-deployer@YOUR_PROJECT_ID.iam.gserviceaccount.com" \
    --role "roles/apigee.environmentAdmin"

```

**B. Create Kubernetes Service Account (KSA)**

```bash
kubectl create serviceaccount argo-workflow -n argo

```

**C. Bind GSA to KSA**

```bash
# Allow KSA to impersonate GSA
gcloud iam service-accounts add-iam-policy-binding apigee-deployer@YOUR_PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:YOUR_PROJECT_ID.svc.id.goog[argo/argo-workflow]"

# Annotate KSA
kubectl annotate serviceaccount argo-workflow -n argo \
    iam.gke.io/gcp-service-account=apigee-deployer@YOUR_PROJECT_ID.iam.gserviceaccount.com

```

---

## 🏃‍♂️ How to Run the Pipeline

This pipeline uses an **Argo Workflow Template**. You must first apply the template to the cluster, and then submit a workflow referencing it.

### 1. Install the Template

```bash
kubectl apply -f apigee-workflow-template.yaml -n argo

```

### 2. Submit a Deployment

Run the following command to deploy a proxy. Replace the parameters as needed.

```bash
argo submit --from wftmpl/apigee-proxy-deployer \
  -p repo_url="[https://github.com/git-azeez/apigee-hybrid-mock.git](https://github.com/git-azeez/apigee-hybrid-mock.git)" \
  -p proxy_path="src/gateway/hello-world-mock" \
  -p apigee_env="walmart" \
  --namespace argo \
  --watch

```

### Parameters

| Parameter | Description | Example |
| --- | --- | --- |
| `repo_url` | Full HTTPS URL of the Git repository. | `https://github.com/user/repo.git` |
| `proxy_path` | Relative path to the proxy folder (where `pom.xml` is located). | `src/gateway/hello-world-mock` |
| `apigee_env` | The target Apigee environment for deployment. | `walmart` (or `dev`, `prod`, etc.) |

---

## 🛠 Local Development

You can also deploy directly from your local machine using Maven, provided you have `gcloud` installed and authenticated.

```bash
cd src/gateway/hello-world-mock

mvn clean install -Pwalmart \
  -Dbearer="$(gcloud auth print-access-token)" \
  -Dapigee.options=override \
  -Dapigee.config.options=none

```

---

## 📄 Argo Workflow Details

The workflow is defined in `apigee-workflow-template.yaml` and executes as a DAG (Directed Acyclic Graph):

1. **`git-clone`**:
* Uses `alpine/git`.
* Mounts the `github-creds` secret to securely clone the repository.
* Saves code to a shared PVC (`workdir`).


2. **`apigee-deploy`**:
* Uses `maven:3.8.5-openjdk-11`.
* Mounts the shared PVC.
* Generates a Google Cloud Access Token using Workload Identity metadata.
* Executes `mvn clean install` to deploy the bundle to Apigee.


