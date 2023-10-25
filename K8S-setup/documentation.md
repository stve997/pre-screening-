Approach
Setting up a highly available K8S cluster would require the configuration of 3 master nodes, 2 worker nodes, 2 load balancers and a monitoring & logging system. 
Three master nodes are recommended to archive the high availability as opposed to 2.
This is because in a case where there are 2 master nodes and one of them fails, the cluster will not have quorum and can’t elect a new master.
Three dedicated master nodes provide two backup quorums to elect a new master. Having two load balancers also increases the availability of the cluster, where one acts as a backup of the other incase of a failure.
For monitoring, I will deploy Prometheus in my K8S cluster and Grafana to the centralized server. After which I will create dashboards for monitoring k8s cluster components & workloads. 
Lastly, I will deploy Elastic Stack as the centralized log server to collect and store logs from all cluster nodes then configure Kubernetes to send container logs to the centralized log server for analysis and troubleshooting.

Implementation
Deployment of Virtual Machines
I will deploy 8 VMs with Ubuntu 22.04 as the base image. The VMs will be named as below:
•	Master node1
•	Master node2
•	Master node3
•	Worker node1
•	Worker node2
•	Load balancer1
•	Load balancer2
•	Logging & Monitoring server
Make an entry of each host in /etc/hosts file for name resolution on all nodes.

Security Configuration Setup:
Use approved and secure base images. Do not use images from unknown sources. It is recommended to use minimal base images such as distroless images because package managers add unnecessary dependencies that could contain unknown vulnerabilities. Also ensure that images are up to date
Harden the base image by disabling unnecessary services, applying security patches, and configuring a firewall.
Mount secrets/credentials into read-only volumes in the containers instead of exposing them as environment variables
Implement RBAC to limit user permissions.

Load Balancer Setup
Install Keepalived and HAproxy on Load balancer 1 & 2.
HAproxy’s configuration on both Load balancers is identical.
Edit the file /etc/haproxy/haproxy.cfg and add frontend & backend kube-apiserver details.
Save the file, restart HAproxy and make it persist through reboots.
To configure keepalived, create the file /etc/keepalived/keepalived.conf and add the configuration.
The configuration of Load balancer 1 and 2 only differ at the values of unicast_src_ip with unicast_peer.
Save the file, restart Keepalived & make it persist through reboots.

K8S Setup:
Disable swap on all K8S nodes and make it permanent by commenting out the swap entry in /etc/fstab .
Allow systemctl to pass bridged traffic of IPv4 and IPv6 to iptables chains for Kubernetes networking.
Update the Ubuntu repositories and install dependencies (ca-certificates, apt-transport-https & gnupg)
Once installation is completed, enable and install docker on all nodes.
Next, add K8S signing key and K8S repository on all nodes.
Use apt-get command to install kubeadm, kubelet and kubectl packages with specific version.
Check whether kubelet service is running 
Initializing and setting up K8S master node
While working only on the first K8S master node, use "kubeadm" command to initialize the kubernetes cluster along with "--control-plane-endpoint", "apiserver-advertise-address" and "--pod-network-cidr" options. 
Once Kubernetes cluster initialization is completed, copy the join token to join any number of the control-plane node.
Copy the other join token to any number of worker nodes.
Make sure you store the tokens in a secure location as you may need it later.

Post K8S cluster setup
To start using your cluster, you need to run the following command as ROOT:
	export KUBECONFIG=/etc/kubernetes/admin.conf
For a normal user the command is as follows:
	  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
Install Network add-on to enable the communication between the pods using the command:
	kubectl create -f https://docs.projectcalico.org/v3.15/manifests/calico.yaml

Monitoring & Logging
Install and setup Prometheus and Grafana on the logging & monitoring server.
Install Prometheus node exporter in all K8S cluster node and setup the node exporter service.
Configure Prometheus on the monitoring server to fetch scrapped metrics from the K8S cluster.
Create a dashboard on Grafana for visualizing the data and notification alerts.
To set up the Elastic Stack for analyzing logs, install and configure Elasticsearch, optimizing its settings for log data.
Deploy Logstash for log data processing, creating pipelines to parse and standardize logs from K8S nodes, containers, and applications.
Then, set up Kibana for log analysis and visualization. 
Ingest K8S logs using Fluentd deploying it as DaemonSet within the cluster, and configure these shippers to forward logs either to Logstash or directly to Elasticsearch.
