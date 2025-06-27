# Runbook: Deploying GKE Cluster using Terraform on GCP

## Prerequisites

  - GCP project with billing enabled
  - Required IAM roles: `container.admin`, `compute.admin`, `iam.serviceAccountUser`
  - Installed and configured Google Cloud SDK
  - Installed Terraform CLI.

### Terraform Installation for Debian/Ubuntu

```bash
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform
```

*For other operating systems, please refer to the official [Terraform Downloads](https://www.terraform.io/downloads.html) page.*

  - Familiarity with a command-line terminal and a basic Linux text editor (e.g., `nano`, `vim`).

-----

## Deployment Procedure

⚠️ **Note on File Creation**: Steps involving file creation (like `main.tf`, `variables.tf`, etc.) must be executed using a Linux text editor.

> **How to use `nano` text editor:**
>
> 1.  To create or edit a file, type:
>     ```sh
>     nano filename.tf
>     ```
> 2.  Type or paste the required content into the editor.
> 3.  Press `Ctrl + O`, then `Enter` to save the file.
> 4.  Press `Ctrl + X` to exit the editor.

-----

### Step 1: Create GCP Project and Enable APIs

Execute the following commands in your terminal or Cloud Shell:

```bash
gcloud projects create my-gke-proj
gcloud config set project my-gke-proj
gcloud services enable container.googleapis.com compute.googleapis.com
```

### Step 2: Create a Service Account for Terraform

Execute the following command in your terminal or Cloud Shell:

```bash
gcloud iam service-accounts create terraform --display-name "Terraform Admin"
```

#### Step 2.1: Assign IAM Roles to the Service Account

Execute the following commands in your terminal or Cloud Shell:

```bash
# Role: Container Admin
gcloud projects add-iam-policy-binding my-gke-proj \
--member="serviceAccount:terraform@my-gke-proj.iam.gserviceaccount.com" \
--role="roles/container.admin"

# Role: Compute Admin
gcloud projects add-iam-policy-binding my-gke-proj \
--member="serviceAccount:terraform@my-gke-proj.iam.gserviceaccount.com" \
--role="roles/compute.admin"

# Role: Service Account User
gcloud projects add-iam-policy-binding my-gke-proj \
--member="serviceAccount:terraform@my-gke-proj.iam.gserviceaccount.com" \
--role="roles/iam.serviceAccountUser"
```

#### Step 2.2: Generate and Download Service Account Key

Execute the following command in your terminal or Cloud Shell:

```bash
gcloud iam service-accounts keys create terraform-key.json \
--iam-account=terraform@my-gke-proj.iam.gserviceaccount.com
```

### Step 3: Create Terraform Project Directory and Files

Execute the following commands in your terminal or Cloud Shell:

```bash
mkdir gke-tf-cluster && cd gke-tf-cluster
mv ~/terraform-key.json .
```

#### Step 3.1: Create `main.tf` File

Create a file named `main.tf` and paste the following content:

```hcl
provider "google" {
  credentials = file("terraform-key.json")
  project     = var.project_id
  region      = var.region
  zone        = var.zone
}

resource "google_container_cluster" "primary" {
  name                     = "my-gke-tf-cluster"
  location                 = var.region
  remove_default_node_pool = true
  initial_node_count       = 1
  network                  = "default"
  subnetwork               = "default"
}

resource "google_container_node_pool" "primary_nodes" {
  name       = "node-pool"
  location   = var.region
  cluster    = google_container_cluster.primary.name
  node_count = 2

  node_config {
    machine_type = "e2-medium"
    disk_size_gb = 50
    oauth_scopes = [
      "https://www.googleapis.com/auth/cloud-platform"
    ]
  }
}
```

#### Step 3.2: Create `variables.tf` File

Create a file named `variables.tf` and paste the following content:

```hcl
variable "project_id" {}
variable "region" { default = "us-central1" }
variable "zone" { default = "us-central1-a" }
```

#### Step 3.3: Create `terraform.tfvars` File

Create a file named `terraform.tfvars` and paste the following content, replacing `"my-gke-proj"` with your actual GCP Project ID.

```hcl
project_id = "my-gke-proj"
```

#### Step 3.4: Create `outputs.tf` File

Create a file named `outputs.tf` and paste the following content:

```hcl
output "kubernetes_cluster_name" {
  value = google_container_cluster.primary.name
}
```

### Step 4: Run Terraform to Deploy

Execute the following commands in your terminal or Cloud Shell:

```bash
# Initialize Terraform
terraform init

# (Optional) Plan the deployment
terraform plan

# Apply the configuration to create resources
terraform apply
```

### Step 5: Get GKE Credentials and Verify

Execute the following commands in your terminal or Cloud Shell:

```bash
# Get credentials for your new cluster
gcloud container clusters get-credentials my-gke-tf-cluster --region us-central1 --project my-gke-proj

# Verify that you can connect to the cluster and see the nodes
kubectl get nodes
```

### Step 6: Deploy a Sample NGINX Application

Execute the following commands in your terminal or Cloud Shell:

```bash
# Create an NGINX deployment
kubectl create deployment nginx --image=nginx

# Expose the deployment as a service with a LoadBalancer
kubectl expose deployment nginx --port=80 --target-port=80 --type=LoadBalancer

# Watch the service until an External IP is assigned
kubectl get service nginx --watch
```

Once you see an **EXTERNAL-IP** assigned to the `nginx` service, copy that IP address and paste it into your web browser. You should see the default "Welcome to nginx\!" page.

-----

## Troubleshooting & Validation

Common issues and their resolutions:

  - **Issue:** Quota Exceeded (e.g., `SSD_TOTAL_GB` exceeded).

      - **Resolution:** Reduce the node pool disk size in `main.tf` by setting `disk_size_gb = 50` (or lower), or request a quota increase from the GCP Console.

  - **Issue:** External IP for the LoadBalancer service remains in a `<pending>` state.

      - **Resolution:** Wait for up to 5 minutes. If it is still pending, check that your VPC firewall rules allow ingress traffic on `TCP:80` and that the cluster nodes have external IP addresses.

  - **Issue:** `terraform apply` fails with IAM permission errors.

      - **Resolution:** Ensure your Service Account has all the required roles (`container.admin`, `compute.admin`, `iam.serviceAccountUser`). Re-run the `gcloud projects add-iam-policy-binding` commands, ensuring the project and service account names are correct.

  - **Issue:** `kubectl` is not connecting to the cluster.

      - **Resolution:** Re-run the `gcloud container clusters get-credentials ...` command, verifying that the project ID, region, and cluster name are correct.

-----

## Helpful References

  - [Terraform GCP Provider Documentation](https://registry.terraform.io/providers/hashicorp/google/latest/docs)
  - [Google Kubernetes Engine (GKE) Documentation](https://cloud.google.com/kubernetes-engine/docs)
  - [gcloud CLI Reference](https://cloud.google.com/sdk/gcloud/reference)
  - [GCP IAM Roles Overview](https://cloud.google.com/iam/docs/understanding-roles)
