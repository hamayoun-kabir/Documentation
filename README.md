# Documentation

**Step 1:**

**Containerd Install prerequisites**

Before installing containerd, set the following kernel parameters on all the nodes.

Using this command edit this file

```bash
nano /etc/modules-load.d/containerd.conf
```

Add on this file 

```bash
overlay
br_netfilter
```

Load modules

```bash
sudo modprobe overlay
sudo modprobe br_netfilter
```

```bash
nano /etc/sysctl.d/99-kubernetes-cri.conf
```

Add this command in **99-kubernetes-cri.conf** file

```bash
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
```

To make above changes into the effect, run

```bash
sudo sysctl --system
```

**Step 2:**

**Install Containerd**

```bash
sudo apt update
sudo apt install containerd -y
```

**configuration file for Containerd**

```bash
sudo mkdir -p /etc/containerd
```

Default Containerd configuration generated and saved to the this file

```bash
containerd config default | sudo tee /etc/containerd/config.toml
```

Edit this file

```bash
nano /etc/containerd/config.toml
```

Find the following section in **config.toml** file

```bash
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
```

And set the value of `SystemdCgroup` to  `false` → `true` 

**Restart Containerd**

```bash
sudo systemctl restart containerd
```

**Step 3:**

**Install Kubernetes**

```bash
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add
```

**Add Kube repository on both nodes**

```bash
nano /etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
```

**Update both your systems, and install all the Kubernetes modules**

```bash
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

**Set hostnames On the master node**

```bash
sudo hostnamectl set-hostname "master-node"
```

**Set hostnames On the worker node**

```bash
sudo hostnamectl set-hostname "w-node1"
sudo hostnamectl set-hostname "w-node2"
```

Set the hostnames in the **/etc/hosts** file on all the nodes

```bash
sudo nano /etc/hosts #Edit host file and add below command in hosts file
192.168.10.234 master-node
192.168.10.233 w-node1
192.168.10.232 w-node2
```

**Disable Swap on All Nodes**

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab #turn off automatically mounted at system startup.
```

**Deploy the Kubernetes cluster (Now Only for Master Node Step)**

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

**Join the Worker Nodes to the Cluster.** when Kubernetes control plane initialized successfully it gives us token for join workers something like below

```bash
kubeadm join 160.119.248.60:6443 --token l5hlte.jvsh7jdrp278lqlr \
--discovery-token-ca-cert-hash sha256:07a3716ea4082fe158dce5943c7152df332376b39ea5e470e52664a54644e00a
```

Copy the **kubeadm join** from the end of the above output. We will be using this command to add worker nodes to our cluster.

**Configure kubectl**

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

**Step 4:**

**Install Calico**

```bash
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/tigera-operator.yaml
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.26.1/manifests/custom-resources.yaml
watch kubectl get pods -n calico-system
```

Remove the taints on the control plane **(If)**

```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
kubectl taint nodes --all node-role.kubernetes.io/master-
```

For worker node **variable path missing or some error show** then use this 

```bash
export PATH=$PATH:/sbin:/usr/sbin
```

**Step 5:**

**NGINX Ingress Controller**

```bash
git clone https://github.com/nginxinc/kubernetes-ingress.git --branch v3.3.0
cd kubernetes-ingress/deployments
```

**Configure RBAC**

```bash
kubectl apply -f common/ns-and-sa.yaml #namespace and a service account
kubectl apply -f rbac/rbac.yaml #Create a cluster role and cluster role binding for the service account:
kubectl apply -f rbac/ap-rbac.yaml #App Protect role and role binding
kubectl apply -f rbac/apdos-rbac.yaml #App Protect DoS role and role binding
```

**Create Common Resources**

```bash
kubectl apply -f ../examples/shared-examples/default-server-secret/default-server-secret.yaml
kubectl apply -f common/nginx-config.yaml
kubectl apply -f common/ingress-class.yaml
```

**Create Custom Resources**

```bash
kubectl apply -f common/crds/k8s.nginx.org_virtualservers.yaml
kubectl apply -f common/crds/k8s.nginx.org_virtualserverroutes.yaml
kubectl apply -f common/crds/k8s.nginx.org_transportservers.yaml
kubectl apply -f common/crds/k8s.nginx.org_policies.yaml
kubectl apply -f common/crds/k8s.nginx.org_globalconfigurations.yaml
kubectl apply -f common/crds/appprotect.f5.com_aplogconfs.yaml
kubectl apply -f common/crds/appprotect.f5.com_appolicies.yaml
kubectl apply -f common/crds/appprotect.f5.com_apusersigs.yaml
kubectl apply -f common/crds/appprotectdos.f5.com_apdoslogconfs.yaml
kubectl apply -f common/crds/appprotectdos.f5.com_apdospolicy.yaml
kubectl apply -f common/crds/appprotectdos.f5.com_dosprotectedresources.yaml
```

**Deploying NGINX Ingress Controller**

```bash
kubectl apply -f deployment/appprotect-dos-arb.yaml
kubectl apply -f service/appprotect-dos-arb-svc.yaml
```

**Running NGINX Ingress Controller**

```bash
kubectl apply -f deployment/nginx-ingress.yaml
kubectl apply -f daemon-set/nginx-ingress.yaml
kubectl get pods --namespace=nginx-ingress
```

**Create a Service for the NGINX Ingress Controller Pods**

```bash
kubectl create -f service/nodeport.yaml
```

**Cert-manager Install**

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.1/cert-manager.yaml
```

**Step 6:**

**Kubernetes Dashboard Install**

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0/aio/deploy/recommended.yaml
cd /opt
```

**Creating Admin user**

```bash
mkdir ~/dashboard && cd ~/dashboard
nano dashboard-admin.yaml
```

Create the following configuration and save it as dashboard-admin.yaml file.

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kubernetes-dashboard
```

Now deploy the admin user role with the next command.

```bash
kubectl apply -f dashboard-admin.yaml
```

We should see a service account and a cluster role binding created.

```bash
serviceaccount/admin-user created
clusterrolebinding.rbac.authorization.k8s.io/admin-user created
```

Now Get the admin token using the command below using this commad

```bash
kubectl get secret -n kubernetes-dashboard $(kubectl get serviceaccount admin-user -n kubernetes-dashboard -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode
```

**Creating Read-Only user**

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: read-only-user
  namespace: kubernetes-dashboard
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
  name: read-only-clusterrole
  namespace: default
rules:
- apiGroups:
  - ""
  resources: ["*"]
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - extensions
  resources: ["*"]
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - apps
  resources: ["*"]
  verbs:
  - get
  - list
  - watch
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-only-binding
roleRef:
  kind: ClusterRole
  name: read-only-clusterrole
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: read-only-user
  namespace: kubernetes-dashboard
```

Now deploy the read-only user account with the next command.

```bash
kubectl apply -f dashboard-read-only.yaml
```

To allow users to log in via the read-only account, you’ll need to provide a token which can be fetched using the next command.

```bash
kubectl get secret -n kubernetes-dashboard $(kubectl get serviceaccount read-only-user -n kubernetes-dashboard -o jsonpath="{.secrets[0].name}") -o jsonpath="{.data.token}" | base64 --decode
```
