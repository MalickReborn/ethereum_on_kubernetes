# ethereum_on_kubernetes
a test project to deploy an ethereum based blockchain within a kubernetes blockchain

We've set a challenge to deploy a blockchain network on a kubernetes cluster.

we've made the choice to go with a 2 nodes cluster (which will be widen), with a master node and a worker node deployed using kubeadm automation tool.
we will then deployed the blockchain upon this kubernetes cluster with the blockchain nodes deployed as pods in the cluster


i've first provisioned virtual machines with Virtualbox with 4GB of RAM and  2 CPU
- one called ubuntu_k8s_master
- the other ubuntu_k8s_worker

to build our cluster with kubeadm 

We will go with a local kubernetes 1.28 version

 first we need to install a container runtime on every machines opf the cluster
 	Forwarding IPv4 and letting iptables see bridged traffic
	Execute the below mentioned instructions:
	  `cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
	overlay
	br_netfilter
	EOF

	sudo modprobe overlay
	sudo modprobe br_netfilter

	# sysctl params required by setup, params persist across reboots
	cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
	net.bridge.bridge-nf-call-iptables  = 1
	net.bridge.bridge-nf-call-ip6tables = 1
	net.ipv4.ip_forward                 = 1
	EOF

	# Apply sysctl params without reboot
	sudo sysctl --system`
 Verify that the br_netfilter, overlay modules are loaded by running the following commands:
 	`lsmod | grep br_netfilter
	 lsmod | grep overlay`
	 
 Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and 	 	net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:
 	`sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables 	net.ipv4.ip_forward
`
 
 Now we can install our Container runtime. We have picked containerd:
 Since we use ubuntu as OS, let's follow these steps:
 1.  Run the following command to uninstall all conflicting packages:
 `for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done`
 2. Set up Docker's apt repository:
 `# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update`
 3. To install the latest version, run:
 `sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`
 4. check the status now:
 `systemctl status containerd`
 
 Since containerd is installed we will configure group drivers for containerd:
  the cgroup has to be the same between 'cgroupfs & systemd' as options for both kubelet and   	containerd
  to check the right option let's try:
  `ps -p 1`
  my output is systemd so i'll configure systemd for both kubelet and containerd
  
  first generate the default configuration:
  ` sudo mkdir /etc/containerd/config.toml
    sudo containerd config default | sudo tee /etc/containerd/config.toml`
  
  To use the systemd cgroup driver in /etc/containerd/config.toml with runc, set

	`[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
	  ...
	  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
	    SystemdCgroup = true`
   then restart containerd:
   
  `sudo systemctl restart containerd`
  
  We can now install kubernetes 1.287 with kubeadm on our machines:
  1. Check required ports 
  `nc 127.0.0.1 6443`
	  Update the apt package index and install packages needed to use the Kubernetes apt repository:
	`sudo apt-get update
	# apt-transport-https may be a dummy package; if so, you can skip that package
	sudo apt-get install -y apt-transport-https ca-certificates curl gpg`
	
	Download the public signing key for the Kubernetes package repositories. The same signing key is used for all repositories so you can disregard the version in the URL:
	`curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg`
	
	Add the appropriate Kubernetes apt repository. 
	`# This overwrites any existing configuration in /etc/apt/sources.list.d/kubernetes.list
	echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list`
	
	Update the apt package index, install kubelet, kubeadm and kubectl, and pin their version:
	`sudo apt-get update
	sudo apt-get install -y kubelet kubeadm kubectl
	sudo apt-mark hold kubelet kubeadm kubectl`
	
  In v1.22 and later, if the user does not set the cgroupDriver field under KubeletConfiguration, kubeadm defaults it to systemd so we don't need to make any adjustment.
  
  We can therefore create our cluster:
  
  first Check required ports
These required ports need to be open in order for Kubernetes components to communicate with each other. You can use tools like netcat to check if a port is open. For example:

  `nc 127.0.0.1 6443`
  
  we will initialiwe our control plane node:
  `kubeadm init --pod-network-cidr=10.244.0.0 --apiserver-advertise-address=$HOSTNAME_IP`
  For the options you can check [here](https://v1-28.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#initializing-your-control-plane-node)
  [![init.png](https://i.postimg.cc/YqmPP8Kw/init.png)](https://postimg.cc/mtBVzYgd)
  
 follow the set of instructions given to set up your kubeconfig file , then make the other nodes connect join the cluster and address the network file settings.
 
 if all of this done you would be able to have an output when quering the cluster nodes status:
 `kubectl get nodes`
 [![getnodes.png](https://i.postimg.cc/nrzcYgC7/getnodes.png)](https://postimg.cc/JDfLRT34)
 
 We went for a 2 nodes cluster but we even add another node considering the workload of a blockchain ne
work:
 the command is to apply `kubeadm token create --print-join-command`.
 you can then use the output to make the new node join the cluster.
 
 Our cluster is ready let's create our blockhain network
 
 Prerequisites
Clone the [Quorum-Kubernetes repository](https://github.com/ConsenSys/quorum-kubernetes)
Install Kubectl
Install Helm3
 
 Deploy charts
 
 Provisioning with Helm charts
Helm is a method of packaging a collection of objects into a chart which can then be deployed to the cluster. After you have cloned the Quorum-Kubernetes repository, change the directory to helm for the rest of this tutorial.
	`cd helm`

2. Create the namespace
We have top isolate groups of resources (for example, StatefulSets and Services) within a single cluster.
	`kubectl create ns quorum`
3. Deploy the monitoring chart
This chart deploys Prometheus and Grafana to monitor the metrics of the cluster, nodes and state of the network.
Update the admin username and password in the monitoring values file. Configure alerts to the receiver of your choice (for example, email or Slack), then deploy the chart using:
`helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install monitoring prometheus-community/kube-prometheus-stack --version 34.10.0 --namespace=quorum --values ./values/monitoring.yml --wait
kubectl --namespace quorum apply -f  ./values/monitoring/`
[![monitoring.png](https://i.postimg.cc/gjxrPj3r/monitoring.png)](https://postimg.cc/rRLq18nL)

To connect to Kibana or Grafana, we also need to deploy an ingress so you can access your monitoring endpoints publicly. We use nginx as our ingress here, and you are free to configure any ingress per your requirements.

`helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install quorum-monitoring-ingress ingress-nginx/ingress-nginx \
    --namespace quorum \
    --set controller.ingressClassResource.name="monitoring-nginx" \
    --set controller.ingressClassResource.controllerValue="k8s.io/monitoring-ingress-nginx" \
    --set controller.replicaCount=1 \
    --set controller.nodeSelector."kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.admissionWebhooks.patch.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.service.externalTrafficPolicy=Local

kubectl apply -f ../ingress/ingress-rules-monitoring.yml`

You can view the Grafana dashboard by going to:

`http://<INGRESS_IP>/d/a1lVy7ycin9Yv/goquorum-overview?orgId=1&refresh=10s`

You can view the Kibana dashboard (if deployed) by going to:

`http://<INGRESS_IP>/kibana`

4. Deploy the genesis chart
The genesis chart creates the genesis file and keys for the validators.
`cd helm
helm install genesis ./charts/goquorum-genesis --namespace quorum --create-namespace --values ./values/genesis-goquorum.yml`

5. Deploy the validators
This is the first set of nodes that we will deploy. 
Deploy the validators like so:

`helm install validator-1 ./charts/goquorum-node --namespace quorum --values ./values/validator.yml
helm install validator-2 ./charts/goquorum-node --namespace quorum --values ./values/validator.yml
helm install validator-3 ./charts/goquorum-node --namespace quorum --values ./values/validator.yml
helm install validator-4 ./charts/goquorum-node --namespace quorum --values ./values/validator.yml`

[![validators.png](https://i.postimg.cc/W1DJg9yx/validators.png)](https://postimg.cc/QF3trqhq)

7. Deploy RPC or Transaction nodes
An RPC node is simply a node that can be used to make public transactions or perform read heavy operations such as when connected to a chain explorer like Blockscout
To deploy an RPC node:

`helm install rpc-1 ./charts/quorum-node --namespace quorum --values ./values/reader.yml`

A Transaction or Member node in turn is one which has an accompanying Private Transaction Manager, such as Tessera; which allow you to make private transactions between nodes.
To deploy a Transaction or Member node:
`helm install member-1 ./charts/quorum-node --namespace quorum --values ./values/txnode.yml`

you can now check the deployment of your blockchain network components as pods in kubernetes following this:
`kubectl -n quorum get pods`
[![Capture-d-cran-du-2024-01-29-14-12-26.png](https://i.postimg.cc/9QPNCYpH/Capture-d-cran-du-2024-01-29-14-12-26.png)](https://postimg.cc/CR144DXc)

8. Connecting to the node from your local machine via an Ingress
In order to view the Grafana dashboards or connect to the nodes to make transactions from your local machine you can deploy an ingress controller with rules. We use the ingress-nginx ingress controller which can be deployed as follows:
'helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install quorum-network-ingress ingress-nginx/ingress-nginx \
    --namespace quorum \
    --set controller.ingressClassResource.name="network-nginx" \
    --set controller.ingressClassResource.controllerValue="k8s.io/network-ingress-nginx" \
    --set controller.replicaCount=1 \
    --set controller.nodeSelector."kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.admissionWebhooks.patch.nodeSelector."kubernetes\.io/os"=linux \
    --set controller.service.externalTrafficPolicy=Local'
Edit the rules file so the service names match your release name. In the example, we deployed a transaction node with the release name member-1 so the corresponding service is called quorum-node-member-1. Once you have settings that match your deployments, deploy the rules like so:

'kubectl apply -f ../ingress/ingress-rules-quorum.yml'


We could improve the access of metrics with additional tools implemented but the task was mainly about testing the possibility of using kubernetes as a fitting plateform to deploy a blockchain network.
