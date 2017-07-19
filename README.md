# kubernetes-macos

Vagrant config to run a full local Kubernetes cluster using the source directory from your Mac.

## Getting started
You must have the following installed:

* Virtualbox >= 5.1.22

  Download and install from https://www.virtualbox.org/.

* Vagrant >= 1.9.7

  Download and install from https://www.vagrantup.com/.

* vagrant-vbguest Vagrant plugin
  automatically installs the host's VirtualBox Guest Additions on the guest system.

  Install by running: 

    vagrant plugin install vagrant-vbguest

    vagrant vbguest --do install

* update Vagrant box

  Install by running: 
    
    vagrant box update
    
* run Virtual machine (VM)

  Install by running: 
  
    vagrant up

## Using kubeadm to create a cluster

Kubernetes is hard to install without using third party tools. kubeadm is an official tool for simple deployment. 

* Before you begin
	1.	One or more virtual machines running Ubuntu 16.04+
	1.	1GB or more of RAM per machine (any less will leave little room for your apps)
	1.	Full network connectivity between all machines in the cluster

* Objectives
	* Install a secure Kubernetes cluster on your machines
	* Install a pod network on the cluster so that application components (pods) can talk to each other
	* Install a sample Golang application on the cluster

Everything is done manually for a better understanding of the process. Here is Vagrantfile I used to run 2 VMs:

```javascript
Vagrant.configure("2") do |config|
    config.vm.box = "ubuntu/xenial64"
 
    config.vm.provider "virtualbox" do |v|
      v.memory = 2048
      v.cpus = 1
    end
 
    config.vm.define "master" do |node|
      node.vm.hostname = "master"
      node.vm.network :private_network, ip: "192.168.33.10"
      node.vm.provision :shell, inline: "sed 's/127\.0\.0\.1.*master.*/192\.168\.33\.10 master/' -i /etc/hosts"
    end
 
    config.vm.define "worker" do |node|
      node.vm.hostname = "worker"
      node.vm.network :private_network, ip: "192.168.33.20"
      node.vm.provision :shell, inline: "sed 's/127\.0\.0\.1.*worker.*/192\.168\.33\.20 worker/' -i /etc/hosts"
    end
 
    config.vm.define "worker2" do |node|
      node.vm.hostname = "worker2"
      node.vm.network :private_network, ip: "192.168.33.30"
      node.vm.provision :shell, inline: "sed 's/127\.0\.0\.1.*worker2.*/192\.168\.33\.30 worker2/' -i /etc/hosts"
    end
end
```

After all VMs are up and running the first step is to add official Kubernetes repo and to install all required packages:

```bash
vagrant ssh master
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && sudo apt-get install -y docker-engine kubelet kubeadm kubectl kubernetes-cni
```

Repeat above step on the workers:
```bash
vagrant ssh worker
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && sudo apt-get install -y docker-engine kubelet kubeadm kubectl kubernetes-cni
```

```bash
vagrant ssh worker2
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && sudo apt-get install -y docker-engine kubelet kubeadm kubectl kubernetes-cni
```

When installed we can start cluster initialization on the master node:
When using flannel as the pod network (described in next step), specify --pod-network-cidr=10.244.0.0/16. 
```bash
vagrant ssh master
sudo kubeadm init --apiserver-advertise-address 192.168.33.10 --pod-network-cidr 10.244.0.0/16 --token 8c2350.f55343444a6ffc46
```

To start using your cluster, you need to run (as a regular user):

```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

You should now deploy a pod network to the cluster.
Flannel RBAC:
```bash
 curl -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel-rbac.yml
kubectl apply -f kube-flannel-rbac.yml
```

Flannel config:
```bash
curl -O https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml
```



