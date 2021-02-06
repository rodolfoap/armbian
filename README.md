# OdroidMC1/Kubernetes/Armbian cluster

The Odroid-MC1: https://www.hardkernel.com/shop/odroid-mc1-my-cluster-one-with-32-cpu-cores-and-8gb-dram/

![OcroidMC1](https://github.com/rodolfoap/odroid-k8s-armbian/blob/master/mc1.jpg)

This is a setup reference to install and run a small Odroid-MC1+Armbian+Kubernetes cluster.

## Odroid setup

* Look for Armbian versions here:

	* https://archive.armbian.com/odroidxu4/archive/
	* https://mirrors.netix.net/armbian/dl/odroidxu4/archive/

* Download armbian on this directory.
```
wget https://mirrors.netix.net/armbian/dl/odroidxu4/archive/Armbian_21.02.1_Odroidxu4_buster_legacy_4.14.212.img.xz
```

* Put an SD card to some slot. Find the device it is referred on (e.g. `/dev/sdc1`).
* `./burn` will setup the SD card with Armbian/Buster.
* `./save` will save install data to the card. Normally, private data. If you don't have access to the `/dat` directory, just continue, this document shows how to configure the system.
* Insert the SD card on the MC1. Boot. Find the DHCP lease and connect to it via ssh.
* Follow this instructions attentively:
```
$ ssh root@192.168.1.26
...
root@192.168.1.26's password: 1234
...
New root password: ************
Repeat password: ************
...
Please provide a username (eg. your forename): rodolfoap
Create password: ************
Repeat password: ************
...
```

* Change the password!
* If required, change the locale to en_US.UTF8: `dpkg-reconfigure locales`

## Simple Configuration

* `./apply.config`
* `armbian-config` # System -> Other (Y) -> Choose the last kernel, e.g. `linux-image-dev-odroidxu4/20/08/01, 5.7.15-odroidxu4`
* Jump to the **Kubernetes** section.

## Detailed Configuration

* Modify the hostname: `vi -o /etc/hosts /etc/hostname`
* Get the ethernet card name using `ifconfig`.
* `./gen.etc-network-interfaces` can be used to generate the `/etc/network/interfaces` file.
```
source /etc/network/interfaces.d/*
auto lo
iface lo inet loopback

auto enx001e05372bb1
allow-hotplug enx001e05372bb1
iface enx001e05372bb1 inet static
address 192.168.1.11
netmask 255.255.255.0
gateway 192.168.1.1
dns-nameservers 192.168.1.1
```

* Run the following commands to disable NetworkManager:
```
rm -v /etc/systemd/system/dbus-org.freedesktop.nm-dispatcher.service
rm -v /etc/systemd/system/multi-user.target.wants/NetworkManager.service
```

* Update, install whatever you need
```
apt update
apt full-upgrade
apt install mc
```

* Add `/etc/docker/daemon.json`:
```
cat << 'EOF' > /etc/docker/daemon.json
{
	"exec-opts": ["native.cgroupdriver=systemd"],
	"log-driver": "json-file",
	"log-opts": {
		"max-size": "100m"
	},
	"storage-driver": "overlay2"
}
EOF
```

* Enable IP forwarding and use classic iptables:
```
sed -i '/net.ipv4.ip_forward/s/#//' /etc/sysctl.conf
update-alternatives --set iptables /usr/sbin/iptables-legacy
```

* Disable IPv6:
```
$ tail /etc/sysctl.conf
...
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
```

* Disable swap (didn't found another way):
```
cat << 'EOF' > /etc/rc.local
#!/bin/sh

/usr/sbin/swapoff -a
exit 0
EOF

```

* Using `armbian-config` -> `System` -> `Other`, switch to a kernel 5.7 or higher, for example `linux-image-dev-odroidxu4/20/08/01, 5.7.15-odroidxu4`.
* Reboot

## Kubernetes

* https://wiki.learnlinux.tv/index.php/How_to_build_your_own_Raspberry_Pi_Kubernetes_Cluster
* https://www.youtube.com/watch?v=B2wAJ5FLOYw

```
swapoff -a # Just in case, check /proc/swaps being empty

curl -sL get.docker.com|sh
usermod -aG docker rodolfoap

curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" > /etc/apt/sources.list.d/kubernetes.list
apt-get update
apt-get install -y kubeadm kubectl # Master
apt-get install -y kubeadm kubelet # Workers
```

* You can check _cgroups_ validity:
```
wget https://raw.githubusercontent.com/docker/docker/master/contrib/check-config.sh -O cgroups_check && chmod +x cgroups_check
./cgroups_check
```

## Master Node

```
kubeadm config images pull

# In case of requiring a different CIDR,
#kubeadm init --apiserver-advertise-address=0.0.0.0 --pod-network-cidr 10.10.0.0/16 --service-cidr 10.11.0.0/16

# Or the common one, ready for Flannel to run,
kubeadm init --apiserver-advertise-address=0.0.0.0 --pod-network-cidr=10.244.0.0/16 # Common one
```

Which will output something like...

```
	Your Kubernetes control-plane has initialized successfully!

	To start using your cluster, you need to run the following as a regular user:

		mkdir -p $HOME/.kube
		sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
		sudo chown $(id -u):$(id -g) $HOME/.kube/config

	You should now deploy a pod network to the cluster.
	Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
		https://kubernetes.io/docs/concepts/cluster-administration/addons/

	Then you can join any number of worker nodes by running the following on each as root:

	kubeadm join 192.168.1.91:6443 --token o255i5.bian2b1m6hcd3yvn --discovery-token-ca-cert-hash sha256:134e4bef7cee2b5548e5ceb04bbf3c5a0a7ca3b7dda66f9e14588a685141edbc
```

* **Save** the last output on the MASTER node, in the file `/root/kubejoin`.
* Verify that the last output was **saved** on the MASTER node, in the file `/root/kubejoin`!!!
* Verify that `/var/lib/kubelet/config.yaml` includes:
```
...
cgroupDriver: systemd
failSwapOn: false

```

* As a regular user, run `./kubejoin.pre`, or :

```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
export KUBECONFIG=$HOME/.kube/config
echo "export KUBECONFIG=$HOME/.kube/config" | tee -a ~/.bashrc

# Required to have a proper flannel configuration, when using the 10.10.0.0/16 network
# Or, in case of a different CIDR...
# curl -s https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml|sed 's/10\.244\.0\.0/10.10.0.0/' > kube-flannel.yml
wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
kubectl apply -f kube-flannel.yml

kubectl get nodes,all,pv,pvc,ep
kubectl get pods --all-namespaces
sudo journalctl -u service-name.service
```

### Make the master a worker

```
kubectl taint nodes --all node-role.kubernetes.io/master-
```

**The installation has finished.**

## Worker Nodes

* On each node, as root, run the `kubeadm join` command saved in the MASTER node, in the file `/root/kubejoin`.

```
kubeadm join 192.168.1.91:6443 --token o255i5.bian2b1m6hcd3yvn --discovery-token-ca-cert-hash sha256:134e4bef7cee2b5548e5ceb04bbf3c5a0a7ca3b7dda66f9e14588a685141edbc
```

**The installation has finished.**

## Some personal helpers...

```
for a in mc{1..4}; do ssh.$a apt install -y mc & done

# Example single files
for a in mc{1..4}; do scp root/.bashrc $a:/root/; done

# Example complete directories
for a in mc{1..4}; do scp -pr home/.docker/ $a:/home/docker/.docker/; done
...
```
## Tmux Broadcast application

* Just launch the app:
```
./ssh.all
```

Shortcuts:

* `F5` activate broadcast
* `F6` deactivate broadcast
* `F7` switch to the previous pane
* `F8` switch to the next pane

## Bad reinstalls

* It is difficult to pin the precise error, but after reinstalling `Kubernetes` multiple times, a weird behavior occurred (https://superuser.com/questions/1613126/).

Apparently the solution was to:

	* [DISCARDED] Include `--apiserver-advertise-address=0.0.0.0` in `kubeadm init`;
	* [DISCARDED] include `cgroupDriver: systemd` in `/var/lib/kubelet/config.yaml` before installing Flannel;
	* [DISCARDED] include `failSwapOn: false` in `/var/lib/kubelet/config.yaml` before installing Flannel.
	* Delete the configuration "--port=0" in `/etc/kubernetes/manifests/controller-manager.conf` and `/etc/kubernetes/manifests/scheduler.conf` (verifiable with `kc get cs`).
