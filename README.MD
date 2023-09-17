 Below is a step-by-step guide for starting with a free tier account in GCP, installing the GCP SDK, and setting up the environment to create a Kubernetes cluster using `kubeadm`:

---

## Set Up a GCP Free Tier Account & Install the GCP SDK

### Step 1: Sign up for GCP's Free Tier

1. Go to the Google Cloud [Free Tier page](https://cloud.google.com/free).
2. Click on `Get started for free`.
3. Log in with your Google account or create a new one.
4. Follow the prompts to select your country and agree to the terms of service.
5. Set up your billing information. **Note**: While you need to provide a credit card or billing method, the free tier has generous limits that will not be charged unless you exceed those limits.

### Step 2: Install Google Cloud SDK

#### For macOS:

```bash
# Using Homebrew
brew cask install google-cloud-sdk
```

#### For Linux:

```bash
# Download the SDK installer
curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-sdk-<VERSION>-linux-x86_64.tar.gz

# Extract the archive
tar zxvf google-cloud-sdk-<VERSION>-linux-x86_64.tar.gz

# Navigate to the SDK directory
cd google-cloud-sdk

# Run the install script
./install.sh
```

Replace `<VERSION>` with the appropriate SDK version from the [official page](https://cloud.google.com/sdk/docs/downloads-versioned-archives).

#### For Windows:

1. Download the interactive installer from the [official SDK downloads page](https://cloud.google.com/sdk/docs/downloads-interactive).
2. Launch the installer and follow the on-screen instructions.

### Step 3: Initialize the SDK

After installation, you'll need to authenticate and set some default properties:

```bash
gcloud init
```

1. When prompted, log in using the same Google account you used to sign up for GCP's free tier.
2. Select a project or create a new one.
3. Choose a default Compute Engine zone (e.g., `us-central1-a`).
4. Confirm the changes and finish the setup.

---

## Configuring the Google Cloud SDK and CLI

After installing the Google Cloud SDK, you need to configure it to interact with your Google Cloud account and resources. This involves authenticating, creating a new project, and setting that project as the default for your current session.

### Step 1: Authenticate with Google Cloud

The first step is to authenticate your SDK installation with your Google Cloud account. This ensures that commands you run locally have the necessary permissions to manage resources in your cloud account.

```bash
gcloud auth login
```

1. The command will open a new browser window prompting you to sign in to your Google account. 
2. After signing in, you'll be asked to grant the Google Cloud SDK permission to access your Google Cloud resources.
3. Confirm the permissions, and then return to the terminal. You should see a confirmation message indicating that you've successfully authenticated.

### Step 2: Create a New Project

You will now create a project named "kubeadm-cluster". Every resource in Google Cloud belongs to a project, which serves as an organizational unit.

```bash
gcloud projects create kubeadm-cluster --name="Kubeadm Cluster Project"
```

This command creates a new project with the ID `kubeadm-cluster` and a name "Kubeadm Cluster Project".

### Step 3: Set the Active Project

After creating the project, you need to set it as the active project for your current session. This ensures that resources you create using the Google Cloud SDK are associated with this project.

```bash
gcloud config set project kubeadm-cluster
```

With this command, all subsequent `gcloud` commands will be executed within the context of the "kubeadm-cluster" project unless otherwise specified.


---

## Setting up the Virtual Machines in GCP

Before deploying the Kubernetes cluster, you need to prepare the infrastructure. In this guide, we'll be creating 3 VMs: one for the master node and two for worker nodes. All VMs will be set up in the same region (US-West) and use Ubuntu as the base OS.

### VM Specifications:

| Node Name  | Type       | Machine Type | Image OS               | Network IP | Boot Disk Size | Boot Disk Type |
|------------|------------|--------------|------------------------|------------|----------------|----------------|
| master-1   | Master     | e2-medium    | ubuntu-2204-jammy-v20230114 | 10.240.0.200 | 30GB           | pd-standard    |
| worker-1   | Worker     | e2-medium    | ubuntu-2204-jammy-v20230114 | 10.240.0.205 | 30GB           | pd-standard    |
| worker-2   | Worker     | e2-medium    | ubuntu-2204-jammy-v20230114 | 10.240.0.206 | 30GB           | pd-standard    |

### VM Creation Scripts:

Here's a breakdown of the provided script with added comments for clarity:

```bash
#!/bin/bash

# Define constants for the GCP setup
REGION="us-west4"
ZONE="us-west4-b"
PROJECT_ID=$(gcloud config get-value project)  # Get the active project ID
MACHINE_TYPE="e2-medium"
SUBNET="kubeadm-nodes-subnet"
NODE_SUBNET_RANGE="10.240.0.0/24"
IMAGE_OS="projects/ubuntu-os-cloud/global/images/ubuntu-2204-jammy-v20230114"

# Create a custom VPC Network and define a subnet range for the nodes
gcloud compute networks create kubeadm-network --subnet-mode custom
gcloud compute networks subnets create ${SUBNET} --network=kubeadm-network \
    --range=${NODE_SUBNET_RANGE} --region=${REGION}

# Create the master node VM
INSTANCE_NAME="master-1"
NETWORK_IP="10.240.0.200"
gcloud compute instances create ${INSTANCE_NAME} --zone=${ZONE} \
    --machine-type=${MACHINE_TYPE} --subnet=${SUBNET} \
    --can-ip-forward \
    --tags=${INSTANCE_NAME},master \
    --image=${IMAGE_OS} \
    --private-network-ip ${NETWORK_IP} \
    --boot-disk-size=30GB \
    --boot-disk-type=pd-standard \
    --boot-disk-device-name=${INSTANCE_NAME}  \
    --preemptible

# Create the first worker node VM
INSTANCE_NAME="worker-1"
NETWORK_IP="10.240.0.205"
gcloud compute instances create ${INSTANCE_NAME} --zone=${ZONE} \
    --machine-type=${MACHINE_TYPE} --subnet=${SUBNET} \
    --can-ip-forward \
    --tags=${INSTANCE_NAME},worker \
    --image=${IMAGE_OS} \
    --private-network-ip ${NETWORK_IP} \
    --boot-disk-size=30GB \
    --boot-disk-type=pd-standard \
    --boot-disk-device-name=${INSTANCE_NAME}  \
    --preemptible

# Create the second worker node VM
INSTANCE_NAME="worker-2"
NETWORK_IP="10.240.0.206"
gcloud compute instances create ${INSTANCE_NAME} --zone=${ZONE} \
    --machine-type=${MACHINE_TYPE} --subnet=${SUBNET} \
    --can-ip-forward \
    --tags=${INSTANCE_NAME},worker \
    --image=${IMAGE_OS} \
    --private-network-ip ${NETWORK_IP} \
    --boot-disk-size=30GB \
    --boot-disk-type=pd-standard \
    --boot-disk-device-name=${INSTANCE_NAME}  \
    --preemptible

# Create a firewall rule to allow all traffic. (For learning purposes only, not recommended for production)
gcloud compute firewall-rules create kubeadm-allow-all --allow all --network kubeadm-network --source-ranges 0.0.0.0/0

```

Explanation:

1. **Define constants**: Constants like `REGION`, `ZONE`, `PROJECT_ID`, and others are defined for use in the script.
2. **Create a VPC Network**: A custom VPC network named `kubeadm-network` is created, and a subnet is defined with the range `10.240.0.0/24`.
3. **Create VMs**: Three virtual machines are created, each with its own specific properties such as machine type, image OS, and network IP. The machines are set to be preemptible, meaning they can be terminated if GCP needs the resources elsewhere (this is cost-saving but not ideal for long-running processes).
4. **Firewall Rule**: A firewall rule named `kubeadm-allow-all` is created to allow all incoming and outgoing traffic. This is for learning purposes and is not recommended for a production environment.

---

<div align="center">
  <a href="./next-page.md" style="padding: 10px 15px; background-color: #4CAF50; color: white; text-decoration: none; border-radius: 4px;">Next Page ➡️</a>
</div>


