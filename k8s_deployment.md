#Ubuntu 20.0 虚拟机上 K8s配置过程
##1基础环境（docker，kubernetes)
**环境准备：**

	配置阿里源 # 后面要拉取镜像
	apt-get update && apt-get install -y apt-transport-https
	curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg --output apt-key.gpg
	cat <<EOF >/etc/apt/sources.list.d/kubernetes.list 
	deb [arch=amd64] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
	EOF
	apt-key adv --recv-keys --keyserver keyserver.ubuntu.com 8B57C5C2836F4BEB（把key加到服务器公钥）
	apt-get update
	#安装三件套
	apt-get install -y kubelet kubeadm kubectl
	
	ufw disable
	# setenforce 0 # 设置Selinux为permissive模式
	# setenforce 1 # 设置SELinux 成为enforcing模式
	vim /etc/fstab # 注释掉最后一行的swap
	swapoff -a
	# 桥接的ipv4流量转到iptables：
	cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
	overlay
	br_netfilter
	EOF

	sudo modprobe overlay
	sudo modprobe br_netfilter
	# 设置所需的 sysctl 参数，参数在重新启动后保持不变
	cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
	net.bridge.bridge-nf-call-iptables  = 1
	net.bridge.bridge-nf-call-ip6tables = 1
	net.ipv4.ip_forward                 = 1
	EOF
	# 应用 sysctl 参数而不重新启动
	sudo sysctl --system
	# 安装containerd：
	apt install containerd
	systemctl start containerd
	mkdir -p /etc/containerd/
	containerd config default > /etc/containerd/config.toml
	sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
	systemctl restart containerd

	date -R
	sudo ntpdate ntp.aliyun.com
	# chronyc sources
	sudo timedatectl set-timezone Asia/Shanghai
	sudo dpkg-reconfigure tzdata
	sudo timedatectl set-local-rtc 0
	timedatectl 作者：WeiyiGeek https://www.bilibili.com/read/cv16293585 出处：bilibili
	

**配置kubeadm init：**
查看kubernetes的版本需要哪些版本镜像：

	kubeadm config images list --kubernetes-version v1.24.2

拉取docker镜像：（注意在kubernetes1.20版本之后docker被弃用）

 	# cat dockerPull.sh 
	docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.23.0
	docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.23.0
	docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.23.0
	docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.23.0
	docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.8.6
	docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.3-0
	docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.7
 
 	重新打标签：

	# cat dockerTag.sh 
	docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-controller-manager:v1.23.0 k8s.gcr.io/kube-controller-manager:v1.23.0
	docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-proxy:v1.23.0 k8s.gcr.io/kube-proxy:v1.23.0
	docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-apiserver:v1.23.0 k8s.gcr.io/kube-apiserver:v1.23.0
	docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler:v1.23.0 k8s.gcr.io/kube-scheduler:v1.23.0
	docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/coredns:1.8.6 k8s.gcr.io/coredns/coredns:v1.8.6
	docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/etcd:3.5.3-0 k8s.gcr.io/etcd:3.5.3-0
	docker tag registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.7 k8s.gcr.io/pause:3.7

或者拉取ctr镜像：

	#拉取镜像：
	kubeadm config images pull --kubernetes-version=v1.24.2 --image-repository=registry.aliyuncs.com/google_containers
	ctr -n k8s.io image tag registry.aliyuncs.com/google_containers/coredns:v1.8.6 k8s.gcr.io/coredns/coredns:v1.8.6
	ctr -n k8s.io image tag registry.aliyuncs.com/google_containers/etcd:3.5.3-0  k8s.gcr.io/etcd:3.5.3-0
	ctr -n k8s.io image tag registry.aliyuncs.com/google_containers/kube-apiserver:v1.24.2 k8s.gcr.io/kube-apiserver:v1.24.2
	ctr -n k8s.io image tag registry.aliyuncs.com/google_containers/kube-controller-manager:v1.24.2 k8s.gcr.io/kube-controller-manager:v1.24.2
	ctr -n k8s.io image tag registry.aliyuncs.com/google_containers/kube-proxy:v1.24.2 k8s.gcr.io/kube-proxy:v1.24.2
	ctr -n k8s.io image tag registry.aliyuncs.com/google_containers/kube-scheduler:v1.24.2 k8s.gcr.io/kube-scheduler:v1.24.2
	ctr -n k8s.io image tag registry.aliyuncs.com/google_containers/pause:3.7 k8s.gcr.io/pause:3.7

kubeadm init:
 
	kubeadm init --kubernetes-version=v1.24.2 --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=192.168.36.131 --image-repository=registry.aliyuncs.com/google_containers
kubeadm reset:

	kubeadm reset



##2网络配置（cni和flanneld）

**配置cni：**

	cd /opt/cni/bin
	wget https://github.com/containernetworking/plugins/releases/download/v0.8.7/cni-plugins-linux-amd64-v0.8.7.tgz
	tar -zxvf cni-plugins-linux-amd64-v0.8.7.tgz



**配置网络（flannel）：**
	
	#有可能访问不到flannel
	vim /etc/hosts
	#填写下面这行，然后保存
	185.199.109.133    raw.githubusercontent.com
	
	kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
**配置calio：**

	wget https://docs.projectcalico.org/v3.18/manifests/calico.yaml 






	kubectl create -f https://projectcalico.docs.tigera.io/manifests/tigera-operator.yaml
	kubectl create -f https://projectcalico.docs.tigera.io/manifests/custom-resources.yaml



##3节点加入
**其它node加入此cluster**

	kubeadm join 192.168.36.131:6443 --token 89hi9c.upm3k5ugxplms1n0 \
		--discovery-token-ca-cert-hash sha256:f72fb5390146bdb23192513871d806ecb7821e5e7fb275d04cba5fa359743c68
	kubeadm join 192.168.36.132:6443 --token k0dau6.qyknut9glx1wi8u6 \
	--discovery-token-ca-cert-hash sha256:02c55a1dfe4f57941ae958b44c135f61690055b8e26195a16795afaf7c83c8a6
##4配置结果
**节点配置成功：**

	root@node1:/home/zhu/Desktop# kubectl get nodes
	NAME    STATUS   ROLES           AGE   VERSION
	node1   Ready    control-plane   24h   v1.24.2



##5报错
1. node可能有污点：  


		#查看污点
		kubectl describe nodes node1 |grep Taints
		# node-role.kubernetes.io/master:NoSchedule
		# 最后的-号就是删除污点，没有-号就是增加污点
		kubectl taint nodes node1 node-role.kubernetes.io/master:NoSchedule-
		kubectl taint nodes node1 node-role.kubernetes.io/control-plane:NoSchedule-
		kubectl taint nodes ubuntu node-role.kubernetes.io/master:NoSchedule-
		kubectl taint nodes ubuntu node-role.kubernetes.io/control-plane:NoSchedule-
2. 引擎列表问题，导致查看不到ctr的images，也无法通过kubeadm拉取镜像，Error while dialing dial unix /var/run/cri-dockerd.sock: connect: no such file or directory\""

		vim /etc/crictl.yaml
		runtime-endpoint: unix:///run/containerd/containerd.sock
		image-endpoint: unix:///run/containerd/containerd.sock
3. error execution phase preflight: [preflight] Some fatal errors occurred:

		[ERROR Port-2379]: Port 2379 is in use
		[ERROR Port-2380]: Port 2380 is in use

4. docker和contain切换<https://www.cnblogs.com/scajy/p/15577903.html>

5. 证书过期：

		# 删除节点kubelet.kubeconfig及证书生成目录下所有文件(其中包含kubelet.crt)
		$ systemctl restart kubelet # 重新生成证书及配置
		$ reboot # 重启机器
		# 此时还需要重新审批node证书申请，重新加入集群
		$ kubectl get csr # 获取csr-id
		$ kubectl certificate approve [csr-id]
		# 稍等片刻后验证状态是否成功，Pod是否能正常调度。

		kubeadm certs renew all

6. Container没有启动：

	**Q1: Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady**

	**Q2: Unable to register node with API server**

		kubeadm reset
		kubeadm init
		swapoff -a
		cat > /etc/containerd/config.toml <<EOF
		[plugins."io.containerd.grpc.v1.cri"]
	  	systemd_cgroup = true
		EOF
		systemctl restart containerd
		rm -rf /etc/cni/net.d  # 通信不成功可能是这里没删除，还使用的旧文件


	如果还有问题，应该是pause的版本有问题，虽然kubeadm list(1.24.22版本)列出来了pause3.7，但是有可能是3.5或3.6版本才可以，所以要拉取其他版本镜像。

		ctr -n k8s.io image pull registry.aliyuncs.com/google_containers/pause:3.7 
		ctr -n k8s.io image tag registry.aliyuncs.com/google_containers/pause:3.7 k8s.gcr.io/pause:3.7



##6卸载相关
卸载docker

	sudo apt-get autoremove docker docker-ce docker-engine  	docker.io  containerd runc
	dpkg -l | grep docker
	sudo apt-get autoremove docker-ce-*
	sudo rm -rf /etc/systemd/system/docker.service.d
	sudo rm -rf /var/lib/docker
##7重启相关
已经启动了cluster，并且成功运行了，如果使用reset删除了cluster：**需要kubeadm init 和 重新部署calico或flannel,如果是master节点还需要去除污点**
##8使用nginx测试节点

	kubectl create deployment nginx --image=nginx
	kubectl expose deployment nginx --port=80 --target-port=80 --type=NodePort


#markdown语法
<https://zhuanlan.zhihu.com/p/108984311>
