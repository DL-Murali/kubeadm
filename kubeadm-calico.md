# Solution
Log in to the control plane server
Install Packages
## Log in to the control plane & Worker node.

**Note**: The following steps must be performed on all three nodes.

- Create the configuration file for containerd:
```
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
```
Load the modules:
```
sudo modprobe overlay
sudo modprobe br_netfilter
```
## Set the system configurations for Kubernetes networking:
```
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
```
### Apply the new settings:
```
sudo sysctl --system
```
## Install containerd:
```
sudo apt-get update && sudo apt-get install -y containerd.io
```
### Create the default configuration file for containerd:
```
sudo mkdir -p /etc/containerd
```
### Generate the default containerd configuration, and save it to the newly created default file:
```
sudo containerd config default | sudo tee /etc/containerd/config.toml
```
#### Restart containerd to ensure the new configuration file is used:
```
sudo systemctl restart containerd
```
**Verify that containerd is running:**
```
sudo systemctl status containerd
```
**Disable swap:**
```
sudo swapoff -a

```
**Install the dependency packages:**
```
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
```
## Download and add the GPG key:

```
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.27/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```
## Add Kubernetes to the repository list:
```
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.27/deb/ /
EOF
```
**Update the package listings:**
```
sudo apt-get update
```
## Install Kubernetes packages:

Note: If you get a dpkg lock message, just wait a minute or two before trying the command again.
```
sudo apt-get install -y kubelet kubeadm kubectl
```
**Turn off automatic updates:**
```
sudo apt-mark hold kubelet kubeadm kubectl
```
### Log in to both worker nodes to perform the previous steps.

# Initialize the Cluster
## Initialize the Kubernetes cluster on the control plane node using kubeadm:
```
sudo kubeadm init --pod-network-cidr 192.168.0.0/16 --kubernetes-version 1.27.11
```
### Set kubectl access:
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
### Test access to the cluster:
```
kubectl get nodes
```
## Install the Calico Network Add-On
**On the control plane node, install Calico Networking:**
```
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.25.0/manifests/calico.yaml
```
**Check the status of the control plane node:**
```
kubectl get nodes
```
## Join the Worker Nodes to the Cluster
**In the control plane node, create the token and copy the kubeadm join command:**
```
kubeadm token create --print-join-command
```
**Note**: This output will be used as the next command for the worker nodes.

- Copy the full output from the previous command used in the control plane node. This command starts with kubeadm join.

- In both worker nodes, paste the full kubeadm join command to join the cluster. Use sudo to run it as root:
```
sudo kubeadm join...
```
**In the control plane node, view the cluster status:**
```
kubectl get nodes
```
**Note**: You may have to wait a few moments to allow all nodes to become ready.

### Conclusion
- Congratulations — you've completed this hands-on lab!
