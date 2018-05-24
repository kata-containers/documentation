# How to use Kata Containers and CRI (containerd plugin) with Kubernetes

This document describes how to set up a single-machine Kubernetes cluster. The Kubernetes cluster will use the [CRI containerd plugin](https://github.com/containerd/cri) and [Kata Containers](https://katacontainers.io) to launch untrusted workloads.

## Requirements

- Kubernetes, kubelet, kubeadm
- cri-containerd
- Kata Containers

Note|
----------------- |
|For information about the supported versions of these components, see the  Kata Containers [versions.yaml](https://github.com/kata-containers/runtime/blob/master/versions.yaml) file. |


## Install containerd(with CRI plugin enabled)

Follow the instructions from [CRI installation guide](http://github.com/containerd/cri/blob/master/docs/installation.md)

```bash
# Check if containerd is installed
$ command -v containerd
```

## Install Kata Containers

Follow the instructions to [install Kata](https://github.com/kata-containers/documentation/blob/master/install/README.md).

```bash
# Check if kata-runtime is installed
$ command -v kata-runtime
# Check kata is well configured
$ kata-runtime kata-env
```

## Install Kubernetes
Install Kubernetes in your host. See [kubeadm installation](https://kubernetes.io/docs/setup/independent/install-kubeadm/)

```bash
# Check if kubadm is installed
$ command -v kubeadm
```

### Configure containerd to use Kata Containers

The CRI containerd plugin supports configuration for two runtime types.

- **Default runtime:** A runtime that is used by default to run workloads.
- **Untrusted workload runtime:** A runtime that will be used to run untrusted workloads.

#### Define the Kata runtime as `untrusted_workload_runtime`

Configure the Kata runtime for untrusted workloads with the [config option](https://github.com/containerd/cri/blob/v1.0.0-rc.0/docs/config.md) `plugins.cri.containerd.untrusted_workload_runtime`.

Unless configured otherwise, the default runtime is set to `runc`.

```bash
# Configure containerd to use Kata as untrusted_workload_runtime
$ sudo mkdir -p /etc/containerd/
```
```bash
$ cat << EOT | sudo tee /etc/containerd/config.toml
[plugins]
    [plugins.cri.containerd]
      [plugins.cri.containerd.untrusted_workload_runtime]
        runtime_type = "io.containerd.runtime.v1.linux"
        runtime_engine = "/usr/bin/kata-runtime"
EOT
```

### Configure Kubelet to use containerd






In order to allow kubelet to use containerd (using CRI interface), configure the service to point to the `containerd` socket.

- Configure k8s to use containerd

```bash
$ sudo mkdir -p  /etc/systemd/system/kubelet.service.d/
```
```bash
$ cat << EOF | sudo tee  /etc/systemd/system/kubelet.service.d/0-containerd.conf
[Service]                                                 
Environment="KUBELET_EXTRA_ARGS=--container-runtime=remote --runtime-request-timeout=15m --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
EOF
```
```bash
$ sudo systemctl daemon-reload
```

### Optional: Configure proxy


If you are behind a proxy, use the following script to configure your proxy for docker, kubelet, and containerd:


```bash
# Set proxys
$ services=(
'kubelet'
'containerd'
'docker'
)

$ for s in "${services[@]}"; do 

	service_dir="/etc/systemd/system/${s}.service.d/"
	sudo mkdir -p ${service_dir}

	cat << EOT | sudo tee "${service_dir}/proxy.conf"
[Service]
Environment="HTTP_PROXY=${http_proxy}"
Environment="HTTPS_PROXY=${https_proxy}"
Environment="NO_PROXY=${no_proxy}"
EOT
done
```
```bash
$ sudo systemctl daemon-reload
```

###  Start Kubernetes with `kubeadm`


- Make sure containerd is up and running

```bash
$ sudo systemctl restart containerd
$ sudo systemctl status containerd
```

- Prevent conflicts of docker iptables rules & k8s pod communication

```bash
$ sudo iptables -P FORWARD ACCEPT
```

-  Start cluster using `kubeadm`

```bash
$ sudo kubeadm init --skip-preflight-checks \
--cri-socket /run/containerd/containerd.sock --pod-network-cidr=10.244.0.0/16

$ export KUBECONFIG=/etc/kubernetes/admin.conf

$ sudo -E kubectl get nodes
$ sudo -E kubectl get pods
```

### Install a Pod Network

A pod network plugin is needed to allow pods to communicate with each other.

Install the `flannel` plugin by following the [Using kubeadm to Create a Cluster](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/#instructions) guide, starting from the **Installing a pod network** section.


```bash
# Install a pod network using flannel
# There is not a programmatic way to know last what flannel commit use
# See https://github.com/coreos/flannel/issues/995
$ sudo -E kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```

```bash
# wait for pod network
$ timeout_dns=0
$ until [ "$timeout_dns" -eq "420" ]; do
	if sudo -E kubectl get pods --all-namespaces | grep dns | grep Running; then
		break
	fi
	sleep 1s
	((timeout_dns+=1))
 done

# check pod network is running
$ sudo -E kubectl get pods --all-namespaces | grep dns | grep Running && echo "OK" || ( echo "FAIL" && false )
```

### Allow run pods in master node

By default, the cluster will not schedule pods in the master node. To enable master node scheduling, run:

```bash
# allow master node to run pods
$ sudo -E kubectl taint nodes --all node-role.kubernetes.io/master-
```


### Create an unstrusted pod using Kata Containers

By default, all pods are created with the default runtime configured in CRI containerd plugin. If a pod has the `io.kubernetes.cri.untrusted-workload` annotation set to `"true"`, the CRI plugin runs the pod with the [Kata Containers runtime](https://github.com/kata-containers/runtime/blob/master/README.md).


```bash
# Create untrusted pod configuration
$ cat << EOT | tee nginx-untrusted.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-untrusted
  annotations:
    io.kubernetes.cri.untrusted-workload: "true"
spec:
  containers:
  - name: nginx
    image: nginx
    
EOT
```

```bash
# Create untrusted pod
$ sudo -E kubectl apply -f nginx-untrusted.yaml
```
```bash
# Check pod is running
$ sudo -E kubectl get pods
```

```bash
# Check qemu is running
$ ps aux | grep qemu
```

```bash
### Delete created pod
# Delete pod
$ sudo -E kubectl delete -f  nginx-untrusted.yaml
```
