This script prepares all the hosts, including the control plane and nodes, to install `kubeadm`. Here's a breakdown of the script with added comments:

---

```bash
############# PREPARE ALL HOSTS INCLUDING CONTROL PLANE AND NODES TO INSTALL KUBEADM #############
# Official Kubeadm Installation Guide: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

# Update the package lists for upgrades and new package installations.
echo "Update OS......"
sudo apt-get update

# 1. Disabling Swap: Kubernetes requires swap to be disabled to work properly.
echo "Disable Linux Swap......"
sudo swapoff -a
# Update /etc/fstab so that swap is permanently disabled after reboots.
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

# 2. Enabling Kernel Modules: These modules are needed for the functioning of container networking and CNI plugins.
echo "Enable Required Kernel Modules and Parameters......"
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter

# Set sysctl parameters to ensure proper network operation of Kubernetes. They persist across reboots.
cat <<EOF | sudo tee /etc/sysctl.d/containerd.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1 
EOF
# Apply sysctl params without needing to reboot.
sudo sysctl --system

# 3. Install Containerd: A lightweight container runtime.
CONTAINERD_VERSION="1.6.20"
wget https://github.com/containerd/containerd/releases/download/v${CONTAINERD_VERSION}/containerd-${CONTAINERD_VERSION}-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-${CONTAINERD_VERSION}-linux-amd64.tar.gz

# 4. Setup Containerd Service
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mv  containerd.service /lib/systemd/system
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

# 5. Install runc: It's the runtime spec for containers and needs to be installed separately for containerd.
RUNC_VERSION="1.1.7"
wget https://github.com/opencontainers/runc/releases/download/v${RUNC_VERSION}/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc

# 6. Configure Containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

# Enable Systemd Control Group for Containerd.
sudo sed -i -r 's/SystemdCgroup =.*/SystemdCgroup = true/' /etc/containerd/config.toml

# Configure the systemd cgroup driver.
echo "Restarting containerd service......"
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

### How to run this script on all VMs:

1. **Transfer Script**: Firstly, ensure that this script is available on all VMs. You can use tools like `scp` to transfer the script to each VM.

   ```bash
   scp your_script.sh username@your_VM_IP:/path/to/directory/
   ```

2. **Execute the Script**: Once the script is available on each VM, you can run it with:

   ```bash
   ssh username@your_VM_IP 'bash -s' < /path/to/your_script.sh
   ```

3. Repeat the above steps for each VM.

Ensure that you have password-less SSH set up, or you will be prompted for a password for each VM. Also, replace `username`, `your_VM_IP`, and `/path/to/directory/` with the appropriate values for your setup.


---

## Summary and Rationale of the VM Preparation

In our quest to set up a Kubernetes cluster, we've just executed a series of steps to prime our VMs for Kubernetes. Let's reflect on the significance of what we've accomplished:

- **Operating System Update:** Keeping the OS updated ensures we are working with the latest patches and security updates. This step is crucial for the overall health and security of our infrastructure.

- **Linux Swap Disablement:** Kubernetes works best when the swap is disabled. Swap can affect the performance and proper functioning of pods within a Kubernetes node. Disabling it aligns with the Kubernetes best practices.

- **Kernel Modules Configuration:** Overlay and bridge network filters (br_netfilter) are vital for containers' networking functionalities. Enabling these modules makes sure that Kubernetes pods can communicate seamlessly.

- **Sysctl Parameter Setting:** By setting certain network parameters at the kernel level, we are ensuring a consistent and stable networking environment for our containers. This aids in the proper functioning of the Cluster Network Interface (CNI) plugins used by Kubernetes.

- **Containerd Installation:** Containerd is a lightweight container runtime. By having this in our VMs, we ensure that our Kubernetes setup can spin up and manage containers efficiently. The choice of container runtime can influence performance, security, and compatibility.

- **Runc Installation:** Runc is the CLI tool for spawning and running containers according to the OCI specification. It's pivotal for the container runtime (like containerd) to manage the complete container lifecycle.

- **Containerd Configuration and Service Setup:** By configuring and setting up containerd as a service, we're making sure that the container runtime is always available. This guarantees that Kubernetes can consistently and reliably start containers on demand.

- **Systemd Control Group Enablement for Containerd:** Systemd cgroup driver is recommended, especially if you use cgroup v2. This allows Kubernetes to better manage and monitor system resources, ensuring optimal resource allocation and utilization.


---
