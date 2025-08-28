# Kubernetes Installation Guide (Ubuntu, kubeadm) in Ubuntu 22.04 LTS By Admin Bui Minh Thanh.

This guide helps you install Kubernetes (single-node cluster) on Ubuntu using `kubeadm`.

---

## 1. Update system and install dependencies

```bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl
```

---

## 2. Install container runtime (containerd)

```bash
sudo apt install -y containerd
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl enable containerd
```

---

## 3. Install kubeadm, kubelet, kubectl

```bash
sudo curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

---

## 4. Initialize Kubernetes master node

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

If successful, it will print instructions like:

```text
Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Run those commands:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 5. Install Pod Network (Flannel)

```bash
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

---

## 6. Verify cluster

```bash
kubectl get nodes
kubectl get pods -A
```

---

✅ Done! You have a single-node Kubernetes cluster running on Ubuntu.
