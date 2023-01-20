# Kubernetes-cluster-expansion
 Spinup new worker-node and add it to the existing Kubernetes cluster.

Agenda:- Spinup new worker-node and add it to the existing cluster.

Pre-requisite:- 
> we should have an existing cluster with alteast one master node.
> We will require a one vm with below specification.
  ubuntu 20.04 os , 4gb ram, alteast 8 gb disk space 
> New node should have below network connectivity.
  port 22,80,443,6443 port should be open

Steps:- Here we will devide this steps in two parts. Few steps will be run on existing master node and few steps will be run on new worker node.

Master node steps:-

Befoe starting below steps we will run below command to check existing status of cluster
````
sudo su -
kubectl get nodes
````
Step 1) Setup /etc/hosts file by adding new worker node entry.
````
vi /etc/hosts
new-worker-node-ip worker-node2

Step 2) Get join Token

kubeadm token list

ref :- ge7cif.4bajb8lyjhbvgdrt
````
Step 3) Get Discovery Token CA cert Hash
````
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'

ref :- 45217d9065415e2e0ef662b2966960fbaa48eb97faa0900e878cf8f7d63dbf0a
Step 4) Get API Server Advertise address

kubectl cluster-info

ref :- master-node:6443
````
Step 5) Creating a join command for worker node by replacing variable from above command.
````
kubeadm join \
  master-node:6443 \
  --token ge7cif.4bajb8lyjhbvgdrt \
  --discovery-token-ca-cert-hash sha256:45217d9065415e2e0ef662b2966960fbaa48eb97faa0900e878cf8f7d63dbf0a --v=5
````

Worker node steps:-

Step 1) update and upgrade the machine then setup hostname and /etc/hosts file.
````
sudo su -
sudo apt update
sudo apt -y full-upgrade
hostnamectl set-hostname worker-node2
vi /etc/hosts
master-node-ip master-node
worker-node2-ip worker-node2
````
Step 2) Install kubelet, kubeadm and kubectl
````
sudo apt -y install curl apt-transport-https
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
````
Step 3) Install required packages.
````
sudo apt update
sudo apt -y install vim git curl wget kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
kubectl version --client && kubeadm version         //verify installed package
````
Step 4) Disable Swap
````
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a
sudo mount -a
free -h
````
Step 5) Enable kernel modules and configure sysctl.
````
# Enable kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Add some settings to sysctl
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload sysctl
sudo sysctl --system
````
Step 6) Install Container runtime (Installing CRI-O:)
````
# Ensure you load modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Set up required sysctl params
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

# Reload sysctl
sudo sysctl --system

# Add Cri-o repo

OS="xUbuntu_20.04"
VERSION=1.25
echo "deb https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable.list
echo "deb http://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable:/cri-o:/$VERSION/$OS/ /" > /etc/apt/sources.list.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.list
curl -L https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/Release.key | apt-key add -
curl -L https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/Release.key | apt-key add -

# Install CRI-O
sudo apt update
sudo apt install cri-o cri-o-runc

# Update CRI-O CIDR subnet
sudo sed -i 's/10.85.0.0/192.168.0.0/g' /etc/cni/net.d/100-crio-bridge.conf

# Start and enable Service
sudo systemctl daemon-reload
sudo systemctl restart crio
sudo systemctl enable crio
sudo systemctl status crio
````
Step 7) Enable kubelet
````
lsmod | grep br_netfilter
sudo systemctl enable kubelet
````
Step 8) Need to run join command on worker node with was generated in previous section.


Step 9) Login into the master server and run below command to check cluster condition.
````
kubectl get nodes
 
````


