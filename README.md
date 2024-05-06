# Kubeadm Cluster Setup

## Part A - Controller and Worker Nodes ( on 3 servers )

- Configure Network Prerequisites

### Container Runtimes

#### Forwarding IPv4 and letting iptables see bridged traffic
- Execute the below mentioned instructions:
```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```
```
sudo modprobe overlay
sudo modprobe br_netfilter
```
#### sysctl params required by setup, params persist across reboots
```
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```
#### Apply sysctl params without reboot
```
sudo sysctl --system
```
- Verify that the br_netfilter, overlay modules are loaded by running the following commands:
```
lsmod | grep br_netfilter
lsmod | grep overlay
```
- Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:
```
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```
### Configure Container Runtime

- Set up Docker's apt repository.

#### Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

#### Add the repository to Apt sources:
```
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
```
### Install the Docker packages.
```
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
### add sudo permission to docker 
```
sudo usermod -aG docker ubuntu
newgrp docker
docker images
```
### Make daemon file for docker to aviod service errors later
```
sudo mkdir /etc/docker
cat <<EOF | sudo tee /etc/docker/daemon.json
{
"exec-opts": ["native.cgroupdriver=systemd"],
"log-driver": "json-file",
"log-opts": {
"max-size": "100m"
},
"storage-driver": "overlay2"
}
EOF
```
```
sudo systemctl enable docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```
### Switch-off the swap in all the nodes
```
sudo swapoff -a
sudo vi /etc/fstab
```
### Configure the containerd runtime environment.

### Login as root:
```
sudo su -
```
### Create a default containerd configuration file:
- containerd config default > /etc/containerd/config.toml
- Open config.toml in a text editor:
```
vi /etc/containerd/config.toml
```
- Change the value of SystemdCgroup from false to true (it should be visible around line number 125 in config.toml):
**SystemdCgroup = true**

### Restart containerd
```
systemctl restart containerd
```
### Exit the sudo mode:
exit

### Installing kubeadm, kubelet and kubectl

*kubeadm:* the command to bootstrap the cluster.

*kubelet:* the component that runs on all of the machines in your cluster and does things like starting pods and containers.

*kubectl:* the command line util to talk to your cluster.

```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

sudo apt-get install -y kubelet kubeadm kubectl
```
- kubeadm version
- kubelet --version
- kubectl version
  
```
sudo apt-mark hold kubelet kubeadm kubectl
```
----------------------------------------------------------------------
# Part B - Controller Node ONLY ( Master Node )

## (RUN AS ROOT) Initiate API server:
```
sudo su -
kubeadm init --apiserver-advertise-address=*<ControllerVM-PrivateIP>* --pod-network-cidr=10.244.0.0/16 
exit
```
## (RUN AS NORMAL USER) Add a user for kube config:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
## (RUN AS NORMAL USER) Deploy Weave network:
```
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```
## (RUN AS ROOT) Create cluster join command:
```
sudo su -
kubeadm token create --print-join-command

kubeadm join <**172.31.8.221:6443**> --token 9ep3pf.ilxyyzjqku0q2il5 --discovery-token-ca-cert-hash sha256:9c482bb9d478c4ec10ce419bc27948be3eb9b38fb5c6595f135f419385fbb13e

kubectl get nodes
```

-------------------------------------------------------------------------
# Part C - Worker Nodes ONLY ( In Worker Node )

## connecting nodes to master plane
- Copy the output of the cluster join command from the previous step and run on the VMs designated as the worker nodes.
```
sudo su -
kubeadm join 172.31.8.221:6443 --token 9ep3pf.ilxyyzjqku0q2il5 --discovery-token-ca-cert-hash sha256:9c482bb9d478c4ec10ce419bc27948be3eb9b38fb5c6595f135f419385fbb13e
```
### Note:we need to run the above command in our nodes to join them in cluster



