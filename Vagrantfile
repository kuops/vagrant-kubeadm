# -*- mode: ruby -*-
# vi: set ft=ruby :

file_to_disk = ".external-disk.vmdk"

Vagrant.configure("2") do |config|
  config.vm.box = 'centos/7'
  config.vm.box_version = "1811.02"
  config.vm.box_check_update = false
  config.ssh.insert_key = false
  config.ssh.private_key_path = [ "/mnt/d/vagrant-home/.vagrant.d/insecure_private_key","~/.ssh/vagrant_id_rsa" ]
  config.vm.network "private_network", ip: "10.0.7.100", netmask: "255.255.255.0", dhcp_enabled: "true"
  config.vm.hostname = "vagrant-kubernetes"
  config.hostmanager.enabled = true
  config.hostmanager.manage_host = true
  config.hostmanager.manage_guest = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline = true
  config.vm.synced_folder '.', '/vagrant', disabled: "true"
  config.vm.provider "virtualbox" do |vb|
    vb.memory = "8192"
    vb.cpus = "4"
    vb.name = "vagrant-kubernetes"
    unless File.exist?(file_to_disk)
      vb.customize [ 'createmedium', 'disk', '--filename', file_to_disk, '--size', 500 * 1024, '--format', 'VMDK' ]
    end
    vb.customize [ 'storageattach', :id, '--storagectl', 'IDE', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', file_to_disk ]
  end
  config.vm.provision "file", source: "~/.ssh/vagrant_id_rsa.pub", destination: "/home/vagrant/.ssh/vagrant.pub"
  config.vm.provision "shell", inline: <<-SHELL
	#!/usr/bin/env bash
	set -ex

	PATH=$PATH:/usr/local/bin

	# ssh public key insert to authorized_keys
	cat /home/vagrant/.ssh/vagrant.pub > /home/vagrant/.ssh/authorized_keys

	## replace yum repo to mirrors.aliyun.com
	curl -Lo /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
	curl -Lo /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
	
	# disable selinux
	sed  -i 's@^SELINUX=.*@SELINUX=disabled@g' /etc/selinux/config
	setenforce 1
	
	# set dns use self manage
	echo -e "[main]\ndns=none" > /etc/NetworkManager/conf.d/dns.conf
	systemctl restart NetworkManager
	echo "nameserver 114.114.114.114" > /etc/resolv.conf	

	# common soft
	yum -y install wget pv git htop httpd-tools mariadb tree bash-completion zip unzip software-properties-common lrzsz bind-utils lftp net-tools curl

	# add sdb storage
	mkdir -p /data/docker
	parted /dev/sdb mklabel gpt
	parted /dev/sdb mkpart primary 1 500G
	kpartx -u /dev/sdb1
	sleep 1
	mkfs.xfs /dev/sdb1
	mount /dev/sdb1 /data
	echo "/dev/sdb1 /data  xfs defaults 0 0" >> /etc/fstab

	# enable bridge nf call
	cat > /etc/sysctl.d/kubernetes.conf <<-EOF
	net.ipv4.ip_forward=1
	net.bridge.bridge-nf-call-ip6tables = 1
	net.bridge.bridge-nf-call-iptables = 1
	EOF
	sysctl --system

	# install docker-ce 18.06
	curl -Lo /etc/yum.repos.d/docker-ce.repo https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
	sed  -i 's@download-stage.docker.com@mirrors.aliyun.com/docker-ce@g' /etc/yum.repos.d/docker-ce.repo
	yum -y install docker-ce-18.06.1.ce
	mkdir -p /etc/docker/
	cat > /etc/docker/daemon.json <<-EOF
	{
	  "insecure-registries":["0.0.0.0/0"],
	  "data-root": "/data/docker",
	  "registry-mirrors": ["http://f1361db2.m.daocloud.io"],
	  "log-driver": "json-file",
	  "log-opts": {
	    "max-size": "100m",
	    "max-file": "3"
	  },
	  "storage-driver": "overlay2",
	  "storage-opts": [
	    "overlay2.override_kernel_check=true"
	  ]
	}
	EOF
	mkdir -p /etc/systemd/system/docker.service.d
	usermod -aG docker vagrant
	systemctl daemon-reload
	systemctl restart docker
        systemctl enable docker
	
	# install ntpd
	sudo yum -y install ntp
	sudo systemctl disable chronyd.service
	sudo systemctl stop chronyd.service
	sudo sed  -i 's@^server.*@#&@g;25aserver ntp1.aliyun.com' /etc/ntp.conf
        sudo systemctl start  ntpd
	sudo systemctl enable  ntpd
	sudo timedatectl set-timezone  Asia/Shanghai

	# install kubeadm kubectl and kubelet
	cat > /etc/yum.repos.d/kubernetes.repo <<-EOF
	[kubernetes]
	name=Kubernetes
	baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
	enabled=1
	gpgcheck=1
	repo_gpgcheck=1
	gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
	EOF
	# yum --showduplicates list <package>
	yum install -y kubelet-1.13.1 kubeadm-1.13.1 kubectl-1.13.1
        sed -i 's@KUBELET_EXTRA_ARGS=.*@KUBELET_EXTRA_ARGS="--pod-infra-container-image=kuops/pause:3.1 --cgroup-driver=cgroupfs"@g' /etc/sysconfig/kubelet
        systemctl daemon-reload
	systemctl enable kubelet && systemctl start kubelet
	
	# disable swap partition
	swapoff -a
	sed -i '/swap/d' /etc/fstab
	
	# kubeadm init cluster 
	curl https://raw.githubusercontent.com/kuops/vagrant-kubeadm/master/init-config/config.yaml > kubeadm-conf.yaml
	sudo kubeadm config images pull --config kubeadm-conf.yaml
	sudo kubeadm init --config kubeadm-conf.yaml

	# set kubectl config
	sudo mkdir -p /home/vagrant/.kube
	sudo cp -rpf /etc/kubernetes/admin.conf /home/vagrant/.kube/config
	sudo chown vagrant:vagrant /home/vagrant/.kube/config
	sudo mkdir -p /root/.kube
	sudo cp -rpf /etc/kubernetes/admin.conf /root/.kube/config
	sudo chown vagrant:vagrant /root/.kube/config

	# taint the master node
	sudo kubectl taint nodes --all node-role.kubernetes.io/master-

	# bash completion
	echo "source <(kubectl completion bash)" >> /etc/profile.d/kubernetes-kubectl.sh
	source /etc/profile.d/kubernetes-kubectl.sh

	# flannel network
	sudo kubectl apply -f https://raw.githubusercontent.com/kuops/vagrant-kubeadm/master/kube-flannel/kube-flannel.yaml
  SHELL
end
