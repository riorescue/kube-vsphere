# Kubernetes with vSphere Integration (Part 2)

## Configure Kubernetes Nodes (basics)
In our last tutorial [Part 1](build-part1.md) we created an Ubuntu based cloud template within vSphere to deploy four(4) virtual machines to be used by our Kubernetes cluster.

```shell
Name:           k8s-master
  IP address:   10.0.200.123
Name:           k8s-worker3
  IP address:   10.0.200.135
Name:           k8s-worker1
  IP address:   10.0.200.207
Name:           k8s-worker2
  IP address:   10.0.200.129
```

We will now configure each node with all the necessary components to create a Kubernetes cluster:
* `kubeadm`
* `kubectl`
* `kubelet`
* The container runtime Docker/Containerd

The following process will require executing the exact same commands across all or a subset of the 4 kubernetes nodes. Therefore, I highly recommend leveraging `tmux` to make the job much easier and less time consuming. If you are using a Debian/Ubuntu-based distribution you can install `tmux` using the following:

```shell
$ sudo apt install tmux -y
```

> The `tmux` project can also be found on github at https://github.com/tmux/tmux

### Create Kubernetes Configuration Files
#### vSphere Configuration
**On the control-plane node "k8s-master"** via SSH create a new file called `vsphere.conf`. This configuration file is used by Kubernetes control-plane to interact with vSphere. Fill out the information as appropriate for your vCenter environment.

```shell
$ ssh ubuntu@10.0.200.123
$ sudo mkdir /etc/kubernetes
```

```shell
$ sudo tee /etc/kubernetes/vsphere.conf >/dev/null <<EOF
[Global]
user = "administrator@vsphere.local"
password = "P@ssw0rd110"
port = "443"
insecure-flag = "1"

[VirtualCenter "vcenter.lab.vstable.com"]
datacenters = "Datacenter"

[Workspace]
server = "vcenter.lab.vstable.com"
datacenter = "Datacenter"
default-datastore = "SATA"
resourcepool-path = "Cluster/Resources"
folder = "k8s"

[Disk]
scsicontrollertype = pvscsi
EOF
```

#### Create the Kubeadm Config File (Kubernetes Version 1.17.3)
**On the control-plane node "k8s-master"** create a new file called `kubeadm.conf`. This file is like a roadmap for kubeadm to bootstrap the control-plane node. This file will reference the `vsphere.conf` file created above. Ensure you update the IP address information and DNS name to reflect your environment.

```shell
$ sudo tee /etc/kubernetes/kubeadm.conf >/dev/null <<EOF
---
apiServer:
  extraArgs:
    cloud-config: /etc/kubernetes/vsphere.conf
    cloud-provider: vsphere
    endpoint-reconciler-type: lease
  extraVolumes:
  - hostPath: /etc/kubernetes/vsphere.conf
    mountPath: /etc/kubernetes/vsphere.conf
    name: cloud
apiServerCertSANs:
- 10.0.200.123
- k8s-master.lab.vstable.com
apiServerExtraArgs:
  endpoint-reconciler-type: lease
apiVersion: kubeadm.k8s.io/v1beta1
controlPlaneEndpoint: k8s-master.lab.vstable.com
controllerManager:
  extraArgs:
    cloud-config: /etc/kubernetes/vsphere.conf
    cloud-provider: vsphere
  extraVolumes:
  - hostPath: /etc/kubernetes/vsphere.conf
    mountPath: /etc/kubernetes/vsphere.conf
    name: cloud
kind: ClusterConfiguration
kubernetesVersion: 1.17.3
networking:
  podSubnet: 172.17.0.0/16
EOF
```

### Use tmux to SSH into every node
```shell
tmux new\; split-window\; split-window\; split-window\; select-layout even-vertical
# Use ctrl b, then the arrow keys to cycle through the tmux panes and SSH to each box independently
ssh ubuntu@10.0.200.123
ssh ubuntu@10.0.200.207
ssh ubuntu@10.0.200.129
ssh ubuntu@10.0.200.135
```

If you need to easily find the IP addresses of each of your four virtual machines you can use `govc` as shown below:

```shell
$ govc find / -type m -name 'k8s*' | xargs govc vm.info | grep 'Name:\|IP'
```

After you have successfully established an SSH session to each node you can enable tmux synchronization:

```
ctrl b, shift :, set synchronize-panes on
```

### Prepare Nodes for Kubernetes
It's exteremely important that you disable swap within Kubernetes nodes. This was completed in our templates in [Part 1](build-part1.md); however, if you do need to disable swap on Debian based machines:

```shell
$ sudo swapoff -a
$ sudo sed -i.bak -r 's/(.+ swap .+)/#\1/' /etc/fstab
```

**On ALL four(4) nodes perform the following:**

Install `Kubelet`, `Kubeadm`, and `Kubectl`

```shell
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
```

```shell
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list >/dev/null
```

```shell
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
```

```shell
sudo apt-mark hold kubelet kubeadm kubectl
```

Install `Docker` (We are using the latest Docker build here)

```shell
# Install Docker CE
# Update the apt package index
sudo apt update

## Install packages to allow apt to use a repository over HTTPS
sudo apt install ca-certificates software-properties-common apt-transport-https curl -y

## Add Docker’s official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

## Add docker apt repository.
sudo add-apt-repository \
"deb [arch=amd64] https://download.docker.com/linux/ubuntu \
$(lsb_release -cs) \
stable"

# Install docker ce (latest supported for K8s 1.17.3 is Docker version 19.03.6)
sudo apt update && sudo apt install docker-ce -y
```

Configure Docker daemon switching over to systemd cdgroup driver.

```shell
# Setup daemon parameters, like log rotation and cgroups
$ sudo tee /etc/docker/daemon.json >/dev/null <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
EOF

sudo mkdir -p /etc/systemd/system/docker.service.d

# Restart docker.
sudo systemctl daemon-reload
sudo systemctl restart docker
```

In this tutorial we will be using **Calico** CNI. If you were going to use **Flannel** you would want to configure bridged IPv4 traffic to pass to iptables chains. **Do NOT configure the below unless you are using Flannel**

```shell
sudo sysctl net.bridge.bridge-nf-call-iptables=1
```

**On the control-plane node "k8s-master" ONLY**
Update the configuration of the Kubelet so that it knows about the vSphere configuration/environment:

Edit the `/etc/systemd/system/kubelet.service.d/10-kubeadm.conf` and add two configuration options to the line starting with `ExecStart=/usr/bin/kubelet`:

* `--cloud-provider=vsphere`
* `--cloud-config=/etc/kubernetes/vsphere.conf`

```shell
$ sudo vi /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
```

This is an example of the edited 10-kubeadm.conf file:

```conf
# Note: This dropin only works with kubeadm and kubelet v1.11+
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf"
Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml"
# This is a file that "kubeadm init" and "kubeadm join" generates at runtime, populating the KUBELET_KUBEADM_ARGS variable dynamically
EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env
# This is a file that the user can use for overrides of the kubelet args as a last resort. Preferably, the user should use
# the .NodeRegistration.KubeletExtraArgs object in the configuration files instead. KUBELET_EXTRA_ARGS should be sourced from this file.
EnvironmentFile=-/etc/default/kubelet
ExecStart=
ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS --cloud-provider=vsphere --cloud-config=/etc/kubernetes/vsphere.conf
```

After editing the above file you will need to recycle the Kubelet service:

```shell
$ sudo systemctl daemon-reload
```

> Snapshot time! It's recommended to snapshot the four Kubernetes nodes prior to moving forward with bootstraping the control-plan server.

### Bootstrap the Kubernetes Control-Plane Node
It's now time to setup the Kubernetes cluster!

**On the control-plane node "k8s-master" ONLY**

Execute kubeadm to initialize the control-plane server. This may take a few minutes while the process downloads the api-server, etcd, controller-manager container images and also sets up the necessary SSL certificates in `/etc/kubernetes/pki`.

```shell
$ sudo kubeadm init --config /etc/kubernetes/kubeadm.conf --upload-certs
```

Once kubeadm completes you should have output similar to:

```shell
W0217 19:48:08.901284   13530 common.go:77] your configuration file uses a deprecated API spec: "kubeadm.k8s.io/v1beta1". Please use 'kubeadm config migrate --old-config old.yaml --new-config new.yaml', which will write the new, similar spec using a newer API version.
W0217 19:48:08.901882   13530 strict.go:54] error unmarshaling configuration schema.GroupVersionKind{Group:"kubeadm.k8s.io", Version:"v1beta1", Kind:"ClusterConfiguration"}: error unmarshaling JSON: while decoding JSON: json: unknown field "apiServerCertSANs"
W0217 19:48:08.903690   13530 validation.go:28] Cannot validate kube-proxy config - no validator is available
W0217 19:48:08.903716   13530 validation.go:28] Cannot validate kubelet config - no validator is available
[init] Using Kubernetes version: v1.17.3
[preflight] Running pre-flight checks
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [k8s-master kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local k8s-master.lab.vstable.com] and IPs [10.96.0.1 10.0.200.123]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [k8s-master localhost] and IPs [10.0.200.123 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [k8s-master localhost] and IPs [10.0.200.123 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[controlplane] Adding extra host path mount "cloud" to "kube-apiserver"
[controlplane] Adding extra host path mount "cloud" to "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[controlplane] Adding extra host path mount "cloud" to "kube-apiserver"
[controlplane] Adding extra host path mount "cloud" to "kube-controller-manager"
W0217 19:48:48.121634   13530 manifests.go:214] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[controlplane] Adding extra host path mount "cloud" to "kube-apiserver"
[controlplane] Adding extra host path mount "cloud" to "kube-controller-manager"
W0217 19:48:48.123151   13530 manifests.go:214] the default kube-apiserver authorization-mode is "Node,RBAC"; using "Node,RBAC"
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[kubelet-check] Initial timeout of 40s passed.
[apiclient] All control plane components are healthy after 42.503632 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config-1.17" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
2f3ca04ebc34ab5e97b2748407b1685586cbfc5cf6b082606664a7ea7a5997bb
[mark-control-plane] Marking the node k8s-master as control-plane by adding the label "node-role.kubernetes.io/master=''"
[mark-control-plane] Marking the node k8s-master as control-plane by adding the taints [node-role.kubernetes.io/master:NoSchedule]
[bootstrap-token] Using token: v8tfi1.ee4xtvf7k4xy5a3l
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

You can now join any number of the control-plane node running the following command on each as root:

  kubeadm join k8s-master.lab.vstable.com:6443 --token v8tfi1.ee4xtvf7k4xy5a3l \
    --discovery-token-ca-cert-hash sha256:bf818c4a1c82f9fc9b1e557028345f40f63a71020408224bc275975eb5e57830 \
    --control-plane --certificate-key 2f3ca04ebc34ab5e97b2748407b1685586cbfc5cf6b082606664a7ea7a5997bb

Please note that the certificate-key gives access to cluster sensitive data, keep it secret!
As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use
"kubeadm init phase upload-certs --upload-certs" to reload certs afterward.

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join k8s-master.lab.vstable.com:6443 --token v8tfi1.ee4xtvf7k4xy5a3l \
    --discovery-token-ca-cert-hash sha256:bf818c4a1c82f9fc9b1e557028345f40f63a71020408224bc275975eb5e57830
```

### Optional (Setup additional Control-Plane Nodes)
In this tutorial we will not setup additional control-place nodes. If you did you would need to modify the kubeadm.conf file to leverage a load-balancer IP and hostname. However, the below process can be used to add additional control-place nodes.

Using the output above from kubeadm, copy and paste the text below (yours will be similar but not the same) to each of your control-plane nodes (nodes 2 and 3):

```shell
$ sudo kubeadm join k8s-master.lab.vstable.com:6443 --token v8tfi1.ee4xtvf7k4xy5a3l \
    --discovery-token-ca-cert-hash sha256:bf818c4a1c82f9fc9b1e557028345f40f63a71020408224bc275975eb5e57830 \
    --control-plane --certificate-key 2f3ca04ebc34ab5e97b2748407b1685586cbfc5cf6b082606664a7ea7a5997bb
```

> Before we add any additional control plan nodes you will need to copy the contents of the pki directory to the other control plane nodes. This is needed because those certificates are for authentication purposes with the existing control plane node.

### Join Worker Nodes to the Cluster (we will do this)
Using the output above from kubeadm, copy and paste the text below (yours will be similar but not the same) to each of your worker nodes:

```shell
$ sudo kubeadm join k8s-master.lab.vstable.com:6443 --token v8tfi1.ee4xtvf7k4xy5a3l \
    --discovery-token-ca-cert-hash sha256:bf818c4a1c82f9fc9b1e557028345f40f63a71020408224bc275975eb5e57830
```

Output from this process should be similar to:

```shell
[preflight] FYI: You can look at this config file with 'kubectl -n kube-system get cm kubeadm-config -oyaml'
[kubelet-start] Downloading configuration for the kubelet from the "kubelet-config-1.17" ConfigMap in the kube-system namespace
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Starting the kubelet
[kubelet-start] Waiting for the kubelet to perform the TLS Bootstrap...

This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure connection details.

Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

### Setup Kubeconfig
**On the control-plane node "k8s-master" ONLY**

As a regular user execute the following:

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> You may optionally set the KUBECONFIG env variable using `export KUBECONFIG=/etc/kubernetes/admin.conf`

### Verification Checks
**On the control-plane node "k8s-master" ONLY**

Check the status of your cluster nodes. It's normal that the status is reported as `NotReady` for each node because we haven't installed and enabled a CNI (Calico) yet.

```shell
$ kubectl get nodes -o wide
NAME          STATUS     ROLES    AGE     VERSION   INTERNAL-IP    EXTERNAL-IP    OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-master    NotReady   master   17m     v1.17.3   10.0.200.123   10.0.200.123   Ubuntu 18.04.4 LTS   4.15.0-88-generic   docker://19.3.6
k8s-worker1   NotReady   <none>   9m49s   v1.17.3   10.0.200.207   <none>         Ubuntu 18.04.4 LTS   4.15.0-88-generic   docker://19.3.6
k8s-worker2   NotReady   <none>   9m      v1.17.3   10.0.200.129   <none>         Ubuntu 18.04.4 LTS   4.15.0-88-generic   docker://19.3.6
k8s-worker3   NotReady   <none>   8m25s   v1.17.3   10.0.200.135   <none>         Ubuntu 18.04.4 LTS   4.15.0-88-generic   docker://19.3.6

```

Check the status of all pods in all namespaces:

```shell
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                 READY   STATUS    RESTARTS   AGE
kube-system   coredns-6955765f44-p6h47             0/1     Pending   0          16m
kube-system   coredns-6955765f44-tc5t6             0/1     Pending   0          16m
kube-system   etcd-k8s-master                      1/1     Running   0          16m
kube-system   kube-apiserver-k8s-master            1/1     Running   0          16m
kube-system   kube-controller-manager-k8s-master   1/1     Running   0          16m
kube-system   kube-proxy-5zhpg                     1/1     Running   0          7m14s
kube-system   kube-proxy-7frd8                     1/1     Running   0          7m49s
kube-system   kube-proxy-lvbwd                     1/1     Running   0          16m
kube-system   kube-proxy-r2d7w                     1/1     Running   0          8m38s
kube-system   kube-scheduler-k8s-master            1/1     Running   0          16m
```

Verify the vSphere `providerID` on each node:

```shell
$ kubectl describe nodes | grep "ProviderID"
ProviderID:                   vsphere://4238008c-f862-0ebd-823f-39b6ae3df964
```

> Need to verify if each node should be present in this output. If so, might need to tweak --cloud-provider=vsphere, etc. on each worker node?

### Deploy Calico CNI
**On the control-plane node "k8s-master" ONLY**

There are many CNIs available (Flannel for example) other than Calico. Calico is a popular option that we will use for this tutorial. Apply the below manifest to the cluster to enable Calico networking:

```shell
$ kubectl apply -f https://docs.projectcalico.org/v3.9/manifests/calico.yaml
```

You should expect output similar to:

```shell
configmap/calico-config created
customresourcedefinition.apiextensions.k8s.io/felixconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamblocks.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/blockaffinities.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamhandles.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ipamconfigs.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgppeers.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/bgpconfigurations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/ippools.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/hostendpoints.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/clusterinformations.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/globalnetworksets.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networkpolicies.crd.projectcalico.org created
customresourcedefinition.apiextensions.k8s.io/networksets.crd.projectcalico.org created
clusterrole.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrolebinding.rbac.authorization.k8s.io/calico-kube-controllers created
clusterrole.rbac.authorization.k8s.io/calico-node created
clusterrolebinding.rbac.authorization.k8s.io/calico-node created
daemonset.apps/calico-node created
serviceaccount/calico-node created
deployment.apps/calico-kube-controllers created
serviceaccount/calico-kube-controllers created
```

Verify cluster status. You should now see all nodes in the `Ready` state.

```shell
$ kubectl get nodes -o wide
NAME          STATUS   ROLES    AGE   VERSION   INTERNAL-IP    EXTERNAL-IP    OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
k8s-master    Ready    master   25m   v1.17.3   10.0.200.123   10.0.200.123   Ubuntu 18.04.4 LTS   4.15.0-88-generic   docker://19.3.6
k8s-worker1   Ready    <none>   17m   v1.17.3   10.0.200.207   <none>         Ubuntu 18.04.4 LTS   4.15.0-88-generic   docker://19.3.6
k8s-worker2   Ready    <none>   16m   v1.17.3   10.0.200.129   <none>         Ubuntu 18.04.4 LTS   4.15.0-88-generic   docker://19.3.6
k8s-worker3   Ready    <none>   16m   v1.17.3   10.0.200.135   <none>         Ubuntu 18.04.4 LTS   4.15.0-88-generic   docker://19.3.6
```

All pods should be reporting ready (may take some time for Calico containers to fire up and get into running state):

```shell
$ kubectl get pods --all-namespaces
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-6b9d4c8765-jm2p8   1/1     Running   0          10m
kube-system   calico-node-khs58                          1/1     Running   0          10m
kube-system   calico-node-m7j6n                          1/1     Running   0          10m
kube-system   calico-node-tgbvs                          1/1     Running   0          10m
kube-system   calico-node-x2krg                          1/1     Running   0          10m
kube-system   coredns-6955765f44-p6h47                   1/1     Running   0          33m
kube-system   coredns-6955765f44-tc5t6                   1/1     Running   0          33m
kube-system   etcd-k8s-master                            1/1     Running   0          33m
kube-system   kube-apiserver-k8s-master                  1/1     Running   0          33m
kube-system   kube-controller-manager-k8s-master         1/1     Running   3          33m
kube-system   kube-proxy-5zhpg                           1/1     Running   0          24m
kube-system   kube-proxy-7frd8                           1/1     Running   0          24m
kube-system   kube-proxy-lvbwd                           1/1     Running   0          33m
kube-system   kube-proxy-r2d7w                           1/1     Running   0          25m
kube-system   kube-scheduler-k8s-master                  1/1     Running   3          33m
```

### Continue to Part 3
We are now ready to install the Kubernetes Dashboard
[Continue to part 3](build-part3.md)