# Local Kubernetes cluster with Vagrant and Ansible
A local kubernetes cluster with a single master and multiple worker nodes, provisioned with Vagrant and Ansible.


## Requirements
- [Install Vagrant](https://developer.hashicorp.com/vagrant/docs/installation)
- [Install Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
- [Install a Vagrant provider](https://developer.hashicorp.com/vagrant/docs/providers) (e.g. virtualbox, vmware, ...)


## Setup

![](setup.png)

- x1 Master Node
  - hostname: `k8s-master`
  - memory: `2GiB`
  - cpus: `2`
  - ip: `192.168.50.10`

- x2 Worker Nodes (can be changed by modifing `NUM_WORKERS` in the `Vagrantfile`)
  - hostnames: `node-1, node-2`
  - memory: `2GiB`
  - cpus: `2`
  - ips: `192.168.50.11, 192.168.50.12`

- Network: `192.168.50.0/24`

- Pod Cluster Network: `10.88.0.0/16`

## Versions

- Kubernetes `v1.30` (Latest version: https://kubernetes.io/releases)
- Containerd `v1.7.16` (Latest version: https://github.com/containerd/containerd/releases)
- Runc `v1.1.12` (Latest version: https://github.com/opencontainers/runc/releases)
- CNI Plugins `v1.4.1` (Latest version: https://github.com/containernetworking/plugins/releases)
- Calico `v3.28.0` (Latest version: https://github.com/projectcalico/calico/releases)
- Ubuntu `23.10` - `ubuntu/mantic64` (Latest version: https://app.vagrantup.com/ubuntu)

All versions are parameterized. You can edit them in the `Vagrantfile`.


## Building the Cluster

To build the whole cluster:

1. `cd` into the repository
2. Run: `vagrant up` (this will also execute the Ansible playbooks)
3. Wait for the whole process to finish
4. Run: `vagrant ssh k8s-master` to ssh into the master node
5. Run: `kubectl get nodes` to see if all the nodes were registered and `Ready` (it may take up to a minute)
6. Run: `kubectl get pods --all-namespaces` and check if all pods are running


## Context

The Ansible playbooks follow the official Kubernetes documentation for creating a cluster with `kubeadm`.
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/

The hardware requirements for `kubeadm` are set in the `Vagrantfile`.

Containerd and its dependencies are installed from source, because the official Ubuntu repo version is broken.
- If your pods hang on deletion or similar, it might be due to the Ubuntu version.

After building the cluster, the `kubernetes-setup` folder will contain the `kubectl` config file named `kube-config`, if you want to copy it to your host machine.
Otherwise, you can run `kubectl` from the master node, where it is already configured.

To `ssh` into the master, use:
```bash
vagrant ssh k8s-master
```


## Deploying Pods

After successful completion of the `Vagrantfile` you can deploy pods.
Here's an example of x2 nginx Pods with a Service/Nodeport loadbalancing between them.

```bash
# Deploy the pods
kubectl run --image=nginx --port=80 nginx01
kubectl run --image=nginx --port=80 nginx02

# Check that the pods are running and note on which node they are deployed
kubectl get pods -o wide

# Tag them with some common labels (e.g. name=myapp)
kubectl label pod/nginx01 name=myapp
kubectl label pod/nginx02 name=myapp

# Create a Node Port Service that will listen on port 35321
kubectl create service nodeport myapp --tcp=35321:80

# Specify the selector to be name=myapp so that it loadbalances between the nginx pods
kubectl set selector svc/myapp name=myapp

# See the node port it listens on
# At the line "NodePort:" see the second value with format <PORT>/TCP
kubectl describe svc/myapp
```

At this point, we have a Node Port Service listening on `<PORT>`.
We can now access the nginx pods from our browser using any of the node's IP addresses.

For example, in your browser type: `192.168.50.11:<PORT>`

But to see that the Service actually loadbalances between the two pods, we can edit the nginx `index.html` file.

```bash
# Enter into pod/nginx01's shell
kubectl exec -it nginx01 -- bash

# Change nginx to nginx01
sed -i 's/nginx\!/nginx01\!/' /usr/share/nginx/html/index.html

# Enter into pod/nginx02's shell
kubectl exec -it nginx02 -- bash

# Change nginx to nginx01
sed -i 's/nginx\!/nginx02\!/' /usr/share/nginx/html/index.html
```

Now go back to the browser and (hard) reload the page.
You should see the page switch between `Welcome to nginx01!` and `Welcome to nginx02!`.

Hurray! We have a working cluster :)


## Troubleshooting Vagrant and VirtualBox

You may encounter problems with VirtualBox after the installation.
Newer versions set some restrictions on the IP range for host-only networks.

To overcome this, complete the following check list:

- [x] Make sure you have install the latest version of Vagrant and VirtualBox
- [x] Make sure you have installed the VirtualBox: host modules for your kernel version, guest iso and guest utils
- [x] Check if `VBoxManage --version` returns the version without errors
- [x] Check if your user is in the VirtualBox groups: `groups` should show `vboxsf` and `vboxusers`
  - To add yourself, run: `sudo usermod -a -G vboxsf <user>` `sudo usermod -a -G vboxusers <user>`
  - Reboot your machine after this
- [x] Make sure the `vboxdrv` module is loaded: `sudo modprobe vboxdrv`
- [x] Make sure the `/etc/vbox` folder is present: `mkdir -p /etc/vbox`
- [x] Execute the following line to allow more ranges for the host-only network: `printf "* 192.168.0.0/16\n* 10.0.0.0/8" | sudo tee -a /etc/vbox/networks.conf`


