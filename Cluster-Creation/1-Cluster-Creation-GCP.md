🛠️ Phase 0: Organization Policy Adjustment
Before beginning the installation, you must ensure that your GCP project allows the creation of Service Account keys. Many organizations have a default policy that restricts this.

Disable the Key Creation Restriction:
# Set your project ID
export GCP_PROJECT_ID=<ENTER_YOUR_PROJECT_ID_HERE>

# Disable the enforcement of the "Disable Service Account Key Creation" policy
gcloud resource-manager org-policies disable-enforce iam.disableServiceAccountKeyCreation --project=$GCP_PROJECT_ID

2: Wait for Propagation:
⏳ Important: Please wait 3–5 minutes after running the command above. GCP organization policies take a few moments to propagate across the infrastructure. Moving too fast may cause the Service Account key generation in Phase 3 to fail.


# This is the complete, step-by-step technical guide to deploy a **Red Hat OpenShift Container Platform** cluster on **Google Cloud Platform (GCP)** using the **Installer Provisioned Infrastructure (IPI)** method.

-----

##  Phase 1: Control Node & User Setup

**Goal:** Create a stable, persistent environment for the installation process to avoid session timeouts.

### 1\. Provision the Installer VM

In your GCP Console, create a VM with these specifications:

  * **Name:** `ocp-installer-node`
  * **Machine Type:** `e2-medium` (2 vCPU, 4GB RAM)
  * **OS:** Ubuntu 22.04 LTS
  * **Disk:** 20GB Standard Persistent Disk

### 2\. Configure the Student User (`siva`)

SSH into your VM and run these commands to create a dedicated user and enable password-based access for remote SSH.

```bash
# 1. Create the user 'siva'
sudo adduser siva

# 2. Grant sudo (administrator) privileges
sudo usermod -aG sudo siva

# 3. Enable Password Authentication for SSH access
sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
sudo sed -i 's/KbdInteractiveAuthentication no/KbdInteractiveAuthentication yes/' /etc/ssh/sshd_config

# 4. Restart the SSH service to apply changes
sudo systemctl restart ssh
```

> **Note:** Log out of the root session and log back in as **siva** to ensure all subsequent files are owned by the correct user.

-----

## 🌐 Phase 2: Red Hat Account & Pull Secret

**Goal:** Authenticate your cluster with Red Hat to allow it to download protected container images.

1.  **Register:** Create an account at [**console.redhat.com**](https://www.google.com/search?q=https://console.redhat.com/).
2.  **Navigate:** Search for **Red Hat OpenShift** \> **Create Cluster**.
3.  **Select Infrastructure:** Choose **Google Cloud Platform (GCP)**.
4.  **Select Method:** Choose **Installer-provisioned infrastructure (IPI)**.
5.  **Get Pull Secret:** Click **Download pull secret** or **Copy pull secret**. Keep this JSON string ready in a notepad.

-----

## ☁️ Phase 3: GCP Infrastructure & Identity Setup

**Goal:** Prepare the Google Cloud "landing zone" and create the identity the installer will use.

### 1\. Project Initialization

```bash
# Set your project ID (Replace with your actual ID)
export GCP_PROJECT_ID=p**************

# authenticate to gcp using your gmail account
gcloud auth login 


gcloud config set project $GCP_PROJECT_ID

# Set default region and zone
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a
```

### 2\. Enable Required Google APIs

Enable the APIs required for OpenShift to manage VPCs, DNS, and Compute instances.

```bash
gcloud services enable compute.googleapis.com \
    cloudapis.googleapis.com \
    cloudresourcemanager.googleapis.com \
    dns.googleapis.com \
    iamcredentials.googleapis.com \
    iam.googleapis.com \
    servicemanagement.googleapis.com \
    serviceusage.googleapis.com \
    storage-api.googleapis.com \
    storage-component.googleapis.com \
    deploymentmanager.googleapis.com \
    file.googleapis.com --project $GCP_PROJECT_ID
```

### 3\. DNS Configuration (Managed Zone)

```bash
export GCP_DOMAIN=i27openshift.com
gcloud dns managed-zones create i27openshift-dns --dns-name=$GCP_DOMAIN. --description="DNS for Openshift"

# Retrieve Name Servers for your Domain Registrar
gcloud dns managed-zones describe i27openshift-dns
```

> **Critical Note:** Copy the 4 Name Servers provided and update them in **GoDaddy**. Remove the trailing dot (`.`) when pasting. Verify propagation with `nslookup -q=ns i27openshift.com`.

### 4\. Create Installer Service Account

```bash
export GCP_SA=i27-gcp-sa-ocp
# Create SA 
gcloud iam service-accounts create $GCP_SA --display-name="i27 Openshift cluster sa"
#  Grant 'Owner' Permissions for the service account
gcloud projects add-iam-policy-binding $GCP_PROJECT_ID \
    --member="serviceAccount:$GCP_SA@$GCP_PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/owner"
# Create directory and generate the JSON key file
sudo -i
mkdir -p ~/ocp
gcloud iam service-accounts keys create ~/ocp/i27-gcp-sa-ocp-key.json \
    --iam-account=$GCP_SA@$GCP_PROJECT_ID.iam.gserviceaccount.com
```

### 5\. Open Firewall for Testing

```bash
gcloud compute --project=$GCP_PROJECT_ID firewall-rules create allow-all-for-i27-openshift \
    --direction=INGRESS --priority=1000 --network=default --action=ALLOW --rules=all --source-ranges=0.0.0.0/0
```

-----

## 📦 Phase 4: Downloading & Preparing Binaries

**Goal:** Install the tools required to build and manage the cluster.

### 1\. Download & Extract

```bash
cd ~/ocp

# Download Installer & Client
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-install-linux.tar.gz
wget https://mirror.openshift.com/pub/openshift-v4/x86_64/clients/ocp/stable/openshift-client-linux.tar.gz

# Extract and Move to System Path
tar -xvzf openshift-install-linux.tar.gz
tar -xvzf openshift-client-linux.tar.gz
sudo mv openshift-install oc kubectl /usr/local/bin/
```

### 2\. Generate SSH Keys

```bash
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_rsa
# Starts SSH agent to manage keys. This loads the key into memory . 
# Now system can: Use SSH automatically and Authenticate to servers
eval "$(ssh-agent -s)"
# Add Key to Agent
ssh-add ~/.ssh/id_rsa
```

-----

## 🛠️ Phase 5: Credentials & Config Generation

**Goal:** Connect the installer to your GCP identity and define the cluster blueprint.

### 1\. Export Credentials

**Important:** The installer will not automatically find your key unless you set this path.

```bash
export GOOGLE_APPLICATION_CREDENTIALS=/root/ocp/i27-gcp-sa-ocp-key.json
export OPENSHIFT_INSTALL_GCP_CREDENTIALS_MODE=manual
```

### 2\. Generate Cluster Configuration

This command creates the `install-config.yaml` file.

```bash
mkdir -p ~/ocp/i27-cluster
openshift-install create install-config --dir=~/ocp/i27-cluster
```

**Prompts:** Select **ssh key**, **gcp**, Project ID, **us-central1**, base domain **i27openshift.com**, cluster name **i27-cluster**, and paste your **Pull Secret**.

-----

## 🚀 Phase 6: Launching & Verifying the Cluster

**Goal:** Execute the build and verify the cluster health.

### 1\. Trigger the Cluster Creation

This takes 30–45 minutes.

```bash
./openshift-install create cluster --dir=/root/ocp/i27-cluster --log-level debug
```

### 2\. Accessing the Cluster

```bash
# Set Kubeconfig for CLI access
export KUBECONFIG=/root/ocp/i27-cluster/auth/kubeconfig

# Get login credentials
cat /root/ocp/i27-cluster/auth/kubeadmin-password

# Get Web Console URL
oc get route console -n openshift-console
```

### 3\. Verification Commands

```bash
oc get nodes
oc get pods -n openshift-ingress
```

### ⚠️ Cleanup

Always destroy the cluster when finished to avoid charges:

```bash
./openshift-install destroy cluster --dir=/root/ocp/i27-cluster --log-level debug
```

-----

