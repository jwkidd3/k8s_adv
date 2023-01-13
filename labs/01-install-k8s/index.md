# Install Kubernetes on AWS
## Log into all servers 

Run the following commands in a terminal 

## Create Instances
Use the Ubuntu Server 18.04 LTS AMI for your region. Use small instance type: t2.small or t3.small
Use the k8s security group for all instances - make sure all ports are open for in/out traffic
Both Instances should be part of the same subnet (in the same VPC)

Create 3 instances: 1 master/2 worker - set their name label to your initials-master/your initials-worker1 (e.g jk-master/jk-worker1)

## Install Kubernetes on all servers

Following commands must be run as the root user. To become root run: 
```
sudo su - 
```
## Install Docker Runtime 
NOTE: This will have to be done on all servers - will have to be done as sudo
```
apt-get update
apt-get install -y \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```
```
add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```
```
apt-get update
apt-get install -y  docker-ce docker-ce-cli containerd.io
```

## Install packages required for Kubernetes on all servers as the root user
```
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
```

Create Kubernetes repository by running the following as one command.
```
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
```

Now that you've added the repository install the packages
```
apt-get update
apt-get install -y kubelet=1.17.0-00 kubeadm=1.17.0-00 kubectl
```
***Note: kubectl does not need to be installed on worker nodes

The kubelet is now restarting every few seconds, as it waits in a `crashloop` for `kubeadm` to tell it what to do.

### Initialize the Master 
Run the following command on the master node to initialize 
```
kubeadm init --kubernetes-version=1.17.0 --ignore-preflight-errors=all
```

If everything was successful output will contain 
````
Your Kubernetes master has initialized successfully!
````

Note the `kubeadm join...` command, it will be needed later on. Save this in a text editor.

Exit to ubuntu user 
```
exit
```

Now configure server so you can interact with Kubernetes as the unprivileged user. 
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Run following on the master to enable IP forwarding to IPTables.
```
sudo sysctl net.bridge.bridge-nf-call-iptables=1
```

### Pod overlay network
Install a Pod network on the master node
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

Wait until `coredns` pod is in a `running` state
```
kubectl get pods -n kube-system
```

### Join nodes to cluster 
Log into each of the worker nodes and run the join command from `kubeadm init` master output. 
```
sudo kubeadm join <command from kubeadm init output> --ignore-preflight-errors=all
```

To confirm nodes have joined successfully log back into master and run 
```
watch kubectl get nodes 
````

When they are in a `Ready` state the cluster is online and nodes have been joined. 

Setup kubectl in your environment (local or cloud 9)
scp -i u<your pep>.pem  ubuntu@<master IP>:~/.kube/config /home/ubuntu/.kube/
# Congrats! 
