IMAGE_NAME = "ubuntu/mantic64"
NUM_WORKERS = 2
OS = "linux"
ARCH = "amd64"
CONTAINERD_VERSION = "1.7.16" # https://github.com/containerd/containerd/releases
RUNC_VERSION = "1.1.12" # https://github.com/opencontainers/runc/releases
CNI_PLUGINS_VERSION = "1.4.1" # https://github.com/containernetworking/plugins/releases
KUBERNETES_VERSION = "1.30" # https://github.com/kubernetes/kubernetes/releases
CALICO_VERSION = "3.28.0" # https://github.com/projectcalico/calico/releases

Vagrant.configure("2") do |config|
    config.ssh.insert_key = false

    config.vm.provider "virtualbox" do |v|
        v.memory = 2048 # 2GiB RAM
        v.cpus = 2
    end
      
    config.vm.define "k8s-master" do |master|
        master.vm.box = IMAGE_NAME
        master.vm.network "private_network", ip: "192.168.50.10"
        master.vm.hostname = "k8s-master"
        master.vm.provision "ansible" do |ansible|
            ansible.playbook = "ansible-playbooks/master-playbook.yml"
            ansible.extra_vars = {
                node_ip: "192.168.50.10",
                os: OS,
                arch: ARCH,
                containerd_version: CONTAINERD_VERSION,
                runc_version: RUNC_VERSION,
                cni_plugins_version: CNI_PLUGINS_VERSION,
                kubernetes_version: KUBERNETES_VERSION,
                calico_version: CALICO_VERSION,
                pod_network_cidr: "10.88.0.0/16"
            }
        end
    end

   (1..NUM_WORKERS).each do |i|
       config.vm.define "node-#{i}" do |node|
           node.vm.box = IMAGE_NAME
           node.vm.network "private_network", ip: "192.168.50.#{i + 10}"
           node.vm.hostname = "node-#{i}"
           node.vm.provision "ansible" do |ansible|
               ansible.playbook = "ansible-playbooks/node-playbook.yml"
               ansible.extra_vars = {
                   node_ip: "192.168.50.#{i + 10}",
                   os: OS,
                   arch: ARCH,
                   containerd_version: CONTAINERD_VERSION,
                   runc_version: RUNC_VERSION,
                   cni_plugins_version: CNI_PLUGINS_VERSION,
                   kubernetes_version: KUBERNETES_VERSION,
                   calico_version: CALICO_VERSION,
               }
           end
       end
   end
end

