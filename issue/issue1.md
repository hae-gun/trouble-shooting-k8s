## k8s 설치 이슈

* 보통 스크립트를 통해 k8s 설치 진행을 많이 진행한다.
* 가장 편리한 방법중 하나 Vagrant 를 이용하여 설치하는 방법.


## 내가 사용했던 스크립트
```shell


Vagrant.configure("2") do |config|
    
  config.vm.box = "rockylinux/8"
  # Disk 확장설정 추가
  config.disksize.size = "50GB"

  #https://cafe.naver.com/kubeops/26
  config.vbguest.auto_update = false
  config.vm.synced_folder "./", "/vagrant", disabled: true

  config.vm.provision :shell, privileged: true, inline: $install_default
  config.vm.define "master-node" do |master|
    master.vm.hostname = "k8s-master"
    master.vm.network "private_network", ip: "192.168.56.30"
	master.vm.provider :virtualbox do |vb|
      vb.memory = 6144
      vb.cpus = 4
	  vb.customize ["modifyvm", :id, "--firmware", "efi"]
	end
    master.vm.provision :shell, privileged: true, inline: $install_master
  end

end

$install_default = <<-SHELL

echo '======== [4] Rocky Linux 기본 설정 ========'
echo '======== [4-1] 패키지 업데이트 ========'
yum -y update

echo '======== [4-2] 타임존 설정 ========'
timedatectl set-timezone Asia/Seoul

echo '======== [4-3] Disk 확장 / Bug: soft lockup 설정 추가========'
# https://cafe.naver.com/kubeops/25
yum install -y cloud-utils-growpart
growpart /dev/sda 4
xfs_growfs /dev/sda4
echo 0 > /proc/sys/kernel/hung_task_timeout_secs
echo "kernel.watchdog_thresh = 20" >> /etc/sysctl.conf

echo '======== [4-4] [WARNING FileExisting-tc]: tc not found in system path 로그 관련 업데이트 ========'
yum install -y yum-utils iproute-tc

echo '======= [4-4] hosts 설정 =========='
cat << EOF >> /etc/hosts
192.168.56.30 k8s-master
EOF

echo '======== [5] kubeadm 설치 전 사전작업 ========'
echo '======== [5] 방화벽 해제 ========'
systemctl stop firewalld && systemctl disable firewalld

echo '======== [5] Swap 비활성화 ========'
swapoff -a && sed -i '/ swap / s/^/#/' /etc/fstab


echo '======== [6] 컨테이너 런타임 설치 ========'
echo '======== [6-1] 컨테이너 런타임 설치 전 사전작업 ========'
echo '======== [6-1] iptable 세팅 ========'
cat <<EOF |tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

cat <<EOF |tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

echo '======== [6-2] 컨테이너 런타임 (containerd 설치) ========'
echo '======== [6-2-1] containerd 패키지 설치 (option2) ========'
echo '======== [6-2-1-1] docker engine 설치 ========'
echo '======== [6-2-1-1] repo 설정 ========'
yum install -y yum-utils
yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

echo '======== [6-2-1-1] containerd 설치 ========'
yum install -y containerd.io-1.6.21-3.1.el8
systemctl daemon-reload
systemctl enable --now containerd

echo '======== [6-3] 컨테이너 런타임 : cri 활성화 ========'
# defualt cgroupfs에서 systemd로 변경 (kubernetes default는 systemd)
containerd config default > /etc/containerd/config.toml
sed -i 's/ SystemdCgroup = false/ SystemdCgroup = true/' /etc/containerd/config.toml
systemctl restart containerd



echo '======== [7] kubeadm 설치 ========'
echo '======== [7] repo 설정 ========'
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/v1.27/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/v1.27/rpm/repodata/repomd.xml.key
exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni
EOF


echo '======== [7] SELinux 설정 ========'
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

echo '======== [7] kubelet, kubeadm, kubectl 패키지 설치 ========'
yum install -y kubelet-1.27.2-150500.1.1.x86_64 kubeadm-1.27.2-150500.1.1.x86_64 kubectl-1.27.2-150500.1.1.x86_64 --disableexcludes=kubernetes
systemctl enable --now kubelet

SHELL



$install_master = <<-SHELL

echo '======== [8] kubeadm으로 클러스터 생성  ========'
echo '======== [8-1] 클러스터 초기화 (Pod Network 세팅) ========'
kubeadm init --pod-network-cidr=20.96.0.0/12 --apiserver-advertise-address 192.168.56.30

echo '======== [8-2] kubectl 사용 설정 ========'
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config

echo '======== [8-3] Pod Network 설치 (calico) ========'
kubectl create -f https://raw.githubusercontent.com/k8s-1pro/install/main/ground/k8s-1.27/calico-3.25.1/calico.yaml
kubectl create -f https://raw.githubusercontent.com/k8s-1pro/install/main/ground/k8s-1.27/calico-3.25.1/calico-custom.yaml

echo '======== [8-4] Master에 Pod를 생성 할수 있도록 설정 ========'
kubectl taint nodes k8s-master node-role.kubernetes.io/control-plane-


echo '======== [9] 쿠버네티스 편의기능 설치 ========'
echo '======== [9-1] kubectl 자동완성 기능 ========'
echo "source <(kubectl completion bash)" >> ~/.bashrc
echo 'alias k=kubectl' >>~/.bashrc
echo 'complete -o default -F __start_kubectl k' >>~/.bashrc

echo '======== [9-2] Dashboard 설치 ========'
kubectl create -f https://raw.githubusercontent.com/k8s-1pro/install/main/ground/k8s-1.27/dashboard-2.7.0/dashboard.yaml

echo '======== [9-3] Metrics Server 설치 ========'
kubectl create -f https://raw.githubusercontent.com/k8s-1pro/install/main/ground/k8s-1.27/metrics-server-0.6.3/metrics-server.yaml
SHELL


```

* 설치중 발생한 오류
  * 전체적으로 실행은 완료 되었으나 kubectl 을 통해 pod 생성을 확인해 보았을때 tigera-operator pod 가 생성이 안되며 계속되는 CrashLoopBackOff 로 상태가 남음
  * 노드 확인시 생성한 k8s-master 노드의 상태가 NotReady 상태로 존재함.
  * 재기동 시에도 동일한 현상 발생
  * journalctl -u kubelet 확인시 아래와 같은 메시지 반복확인.
```shell
"command failed" err="failed to load kubelet config file, error: failed to load Kubelet config file /var/lib/kubelet/config.yaml, error failed to read kubelet config file \"/var/lib/kubelet/config.yaml\", error: open /var/lib/kubelet/config.yaml: no such file or directory, path: /var/lib/kubelet/config.yaml"

"Error syncing pod, skipping" err="failed to \"StartContainer\" for \"tigera-operator\" with CrashLoopBackOff: \"back-off 40s restarting failed container=tigera-operator pod=tigera-operator-549d4f9bdb-t2p6w_tigera-operator(446828e0-f98b-4047-8cdf-48886d3e19a0)\"" pod="tigera-operator/tigera-operator-549d4f9bdb-t2p6w" podUID=446828e0-f98b-4047-8cdf-48886d3e19a0

"Container runtime network not ready" networkReady="NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized"
```
 
* 원인파악
  * 계속되는 원인 파악중 스크립트에 존재하는 calico 관련 패키지가 pod 목록에서 존재하지 않는것을 발견함.
  * calico 는 CNI (Container Network Interface) 중 하나로 컨테이너간 네트워킹을 제어하는 어플리케이션임.
  * tigera-operator 는 calico 의 설치 라이프사이클을 관리해주는 패키지.
  * calico 가 설치가 안되어있으니 tigera-operator 는 설치되지 않은 calico를 관리하고자 하여 에러 발생.

* 해결방안
  * calico 패키지가 설치되지 않는 원인을 찾고자 하였으나 정확한 원인은 찾을수 없었음.
  * 기존 스크립트에 작성된 3.25.1 버전에서 3.26.1 버전으로 버전 업그레이드 진행.
  * 이후 재설치후 kubelet 재기동.