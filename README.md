Hii :)) , I am Shrook Ibrahim, Cloud and DevOps Engineer, and this is a role I created to help you start building your Kubernetes cluster using your own machine. All you need is to create two EC2 instances on your AWS account.

# Kubernetes Cluster Setup with Ansible (Control Plane + Worker)

## ℹ️ Info
This Ansible role automates the deployment of a **Kubernetes cluster** on AWS (or any VMs), including:
- Initializing the **control plane** with `kubeadm`.
- Joining worker nodes to the cluster.
- Configuring networking (CNI).
- Generating kubeconfig so you can manage the cluster remotely (from your local machine or Ansible server).

---

## ✅ Prerequisites
- **Ubuntu 20.04** (tested, required for kubeadm).
- At least **2 EC2 instances (t3.medium recommended)** on AWS:
  - **1 control-plane**
  - **1 worker**
- Security group open for:
  - `6443/tcp` (Kubernetes API server)
  - `22/tcp` (SSH)
  - `10250/tcp` (kubelet)
- Ansible installed on your local or management server.
- SSH access to your EC2 nodes (with key).
---

## 🚀 Steps to Use


1. Download the role
     ```bash
     ansible-galaxy role install shrookibrahem.k8s_setup

2. Clone the repository:
   ```bash
   git clone https://github.com/shrookibrahem/k8s-setup-playbook.git
   cd k8s-ansible-role

3. Edit the inventory file

4. Run the playbook

5. When finished, copy kubeconfig from the controller node:

    ```bash
    scp -i ~/.ssh/mykey.pem ubuntu@<controller_public_ip>:/home/ubuntu/.kube/config ./kubeconfig

6. 
Verify the cluster:
    ```bash
    kubectl get nodes

## 🚧 The Problem We Faced
By default, when you run:
```bash
kubeadm init
```
The **API Server certificate** is generated with only **internal/private IPs** (e.g., `10.0.x.x`).

So, if you copy the kubeconfig to a machine outside that private subnet (like your **Ansible server** or local laptop), and try:
```bash
kubectl get nodes
```
You’ll get TLS errors like:
```
certificate is valid for 10.96.0.1, 10.0.1.188, not <your-public-ip>
```

This happens because the certificate doesn’t include the **public IP/DNS**, and clients can’t verify the connection.

---

## ✅ How To Fix
We regenerated the API Server certificate with the public IP added as a **SAN (Subject Alternative Name)**:

```bash
# On control plane node
sudo mv /etc/kubernetes/pki/apiserver.crt /etc/kubernetes/pki/apiserver.crt.bak
sudo mv /etc/kubernetes/pki/apiserver.key /etc/kubernetes/pki/apiserver.key.bak

sudo kubeadm init phase certs apiserver --apiserver-cert-extra-sans=<public-ip>

sudo systemctl restart kubelet
```

This regenerates the API Server cert with both the **private** and **public IPs**.  
Now when you copy `~/.kube/config` to your local/Ansible machine, `kubectl` will connect successfully.

---

## 🚀 Testing Your Cluster
1. From your Ansible/local machine:
```bash
export KUBECONFIG=./kubeconfig
kubectl get nodes
```
✅ You should see both the control-plane and worker nodes as `Ready`.

2. To test scheduling a Pod and accessing it:
```bash
kubectl run nginx --image=nginx --port=80
kubectl expose pod nginx --type=NodePort --port=80
kubectl get svc nginx
```
Use the **worker node public IP + NodePort** to access Nginx from outside.

---
