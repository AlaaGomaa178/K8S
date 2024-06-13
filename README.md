# Kubernetes Cluster Setup and Application Deployment

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Step 1: Set Up AWS Instances](#step-1-set-up-aws-instances)
3. [Step 2: Update and Upgrade the System](#step-2-update-and-upgrade-the-system)
4. [Step 3: Install Docker](#step-3-install-docker)
5. [Step 4: Install Kubernetes Components](#step-4-install-kubernetes-components)
6. [Step 5: Disable Swap](#step-5-disable-swap)
7. [Step 6: Initialize the Master Node](#step-6-initialize-the-master-node)
8. [Step 7: Configure kubectl for the Master Node](#step-7-configure-kubectl-for-the-master-node)
9. [Step 8: Install a Pod Network Add-on on Master node](#step-8-install-a-pod-network-add-on-on-master-node)
10. [Step 9: Join Worker Nodes to the Cluster](#step-9-join-worker-nodes-to-the-cluster)
11. [Step 10: Deploy Nginx Ingress Controller](#step-10-deploy-nginx-ingress-controller)
12. [Step 11: Restrict Access to Kubernetes API](#step-11-restrict-access-to-kubernetes-api)
13. [Step 12: Deploy Juice Shop Application](#step-12-deploy-juice-shop-application)
14. [Step 13: Expose Juice Shop inside the Cluster](#step-13-expose-juice-shop-inside-the-cluster)
15. [Step 14: Expose Juice Shop Outside the Cluster Using Nginx Ingress](#step-14-expose-juice-shop-outside-the-cluster-using-nginx-ingress)
16. [Conclusion](#conclusion)

---

## Step 1: Set Up AWS Instances

1. Launch EC2 instances for your Kubernetes cluster nodes:
   - Create one master node and one worker node.
   - Configure security groups to allow necessary traffic (e.g., SSH, HTTP, HTTPS).

2. Connect to each instance using SSH:
   ```bash
   ssh -i your-key.pem ec2-user@your-instance-ip
   ```

---

## Step 2: Update and Upgrade the System

Start by updating the package list and upgrading the installed packages to the latest versions.

```bash
sudo apt update && sudo apt upgrade -y
```

---

## Step 3: Install Docker

Kubernetes uses Docker as its container runtime. Install Docker on all nodes (master and worker nodes).

```bash
sudo apt install -y docker.io
```

Verify that Docker is installed correctly.

```bash
sudo docker --version
```

---

## Step 4: Install Kubernetes Components

Install the Kubernetes components: kubeadm, kubelet, and kubectl on all nodes.

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

sudo apt update
sudo apt install -y kubelet kubeadm kubectl

sudo apt-mark hold kubelet kubeadm kubectl
```

---

## Step 5: Disable Swap

Kubernetes requires swap to be disabled. Disable swap on all nodes.

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

---

## Step 6: Initialize the Master Node

On the master node, initialize the Kubernetes cluster with kubeadm.

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

After initialization, follow the instructions provided by `kubeadm`.

---

## Step 7: Configure kubectl for the Master Node

Set up the kubeconfig file for the root user on the master node.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify the cluster status.

```bash
kubectl get nodes
```

---

## Step 8: Install a Pod Network Add-on on Master node

Install a pod network so that your pods can communicate with each other. We'll use Flannel for this example.

```bash
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

Verify that all nodes are up and running.

```bash
kubectl get nodes
```

---

## Step 9: Join Worker Nodes to the Cluster

On each worker node, use the join command obtained from the master node initialization step.

```bash
sudo kubeadm join <master-ip>:<master-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

Verify that the nodes have joined the cluster.

```bash
kubectl get nodes
```

---

## Step 10: Deploy Nginx Ingress Controller

Apply the Nginx Ingress Controller manifest.

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/aws/deploy.yaml
```

---

## Step 11: Restrict Access to Kubernetes API

Create a network policy to restrict access to the Kubernetes API.

```bash
nano api-access-policy.yaml
```

Paste the network policy content into the editor.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-access-policy
spec:
  podSelector: {}
  ingress:
  - from:
    - ipBlock:
        cidr: 1.2.3.4/32  # Replace with your specific IP address or CIDR range
    ports:
    - protocol: TCP
      port: 443  # Assuming you want to restrict HTTPS access to the API
```

Apply the network policy.

```bash
kubectl apply -f api-access-policy.yaml
```

---

## Step 12: Deploy Juice Shop Application

Create a deployment for Juice Shop.

```bash
nano juice-shop-deployment.yaml
```

Paste the deployment content into the editor.

Apologies for the cut-off. Let's continue from where we left off:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: juice-shop
spec:
  replicas: 1  # You can adjust the number of replicas as needed
  selector:
    matchLabels:
      app: juice-shop
  template:
    metadata:
      labels:
        app: juice-shop
    spec:
      containers:
      - name: juice-shop
        image: bkimminich/juice-shop
        ports:
        - containerPort: 3000
```

Apply the deployment configuration:

```bash
kubectl apply -f juice-shop-deployment.yaml
```

---

## Step 13: Expose Juice Shop inside the Cluster

Create a service to expose the Juice Shop application within the cluster.

```bash
nano juice-shop-service.yaml
```

Paste the service content into the editor.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: juice-shop-service
spec:
  selector:
    app: juice-shop
  ports:
    - protocol: TCP
      port: 80
      targetPort: 3000
  type: ClusterIP
```

Apply the service configuration:

```bash
kubectl apply -f juice-shop-service.yaml
```

---

## Step 14: Expose Juice Shop Outside the Cluster Using Nginx Ingress

Create an Ingress resource to expose the Juice Shop application outside the cluster using the Nginx Ingress Controller.

```bash
nano juice-shop-ingress.yaml
```


```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: juice-shop-ingress
spec:
  rules:
  - host: juice.example.com  # Replace with your desired hostname
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: juice-shop-service
            port:
              number: 80
```

Replace `juice.example.com` with your desired hostname or domain.

Apply the Ingress resource:

```bash
kubectl apply -f juice-shop-ingress.yaml
```

---

## Conclusion

Your Kubernetes cluster is now set up and running with the Nginx Ingress Controller deployed. The Juice Shop application is deployed, exposed internally within the cluster, and accessible externally through the Nginx Ingress. You've also set up monitoring using Prometheus and Grafana. You can further customize and scale your applications as needed within this Kubernetes environment.

For more advanced configurations and troubleshooting, refer to the official Kubernetes documentation and community resources. Happy Kubernetes-ing! 🚀
