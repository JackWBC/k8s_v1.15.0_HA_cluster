# k8s_v1.15.0_HA_cluster
k8s_v1.15.0高可用集群基础环境搭建

# 虚拟机环境配置清单
- 操作系统 CentOS7 x86_64 mini  
- 网卡 ens33
- 3个master, 3个node, 域名与IP如下  

| 角色      | 域名     | IP     |
| ---------- | :-----------:  | :-----------: |
| master     | master1.k8s  | 192.168.250.141  |
| master     | master2.k8s  | 192.168.250.142  |
| master     | master3.k8s  | 192.168.250.143  |
| node     | node1.k8s  | 192.168.250.144  |
| node     | node2.k8s  | 192.168.250.145  |
| node     | node3.k8s  | 192.168.250.146  |
| 虚拟IP     | --  | 192.168.250.99  |

- kube-proxy ipvs模式  
- 堆叠式etcd集群  
- calico网络组件  

# 虚拟机基础配置
在所有master与node上操作  

#### 解决 setLocale 问题  
```  
cat <<EOF >  /etc/environment
LANG=en_US.UTF-8
LC_ALL=C
EOF
```  
#### 停止iptables
```
systemctl stop firewalld.service && systemctl disable  firewalld.service 
```
#### 设置 SELinux 为 disabled 模式
```
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=disabled/' /etc/selinux/config
```
#### 禁用交换分区
```
swapoff -a && sed -i '/ swap / s/^/#/' /etc/fstab
```
#### 设置sysctl
```
cat <<EOF > /etc/sysctl.conf
fs.file-max=1000000
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
vm.swappiness = 0
net.ipv4.ip_forward = 1
net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.tcp_sack = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_rmem = 4096 87380 4194304
net.ipv4.tcp_wmem = 4096 16384 4194304
net.ipv4.tcp_max_syn_backlog = 16384
net.core.netdev_max_backlog = 32768
net.core.somaxconn = 32768
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_timestamps = 1
net.ipv4.tcp_fin_timeout = 20
net.ipv4.tcp_synack_retries = 2
net.ipv4.tcp_syn_retries = 2
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_mem = 94500000 915000000 927000000
net.ipv4.tcp_max_orphans = 3276800
net.ipv4.ip_local_port_range = 1024 65000
net.nf_conntrack_max = 6553500
net.netfilter.nf_conntrack_max = 6553500
net.netfilter.nf_conntrack_tcp_timeout_close_wait = 60
net.netfilter.nf_conntrack_tcp_timeout_fin_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_time_wait = 120
net.netfilter.nf_conntrack_tcp_timeout_established = 3600
EOF
```
#### 加载ipvs
```
cat << EOF | tee /etc/sysconfig/modules/ipvs.modules
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF
chmod 755 /etc/sysconfig/modules/ipvs.modules && bash /etc/sysconfig/modules/ipvs.modules && lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```
#### 修改yum repo, 提升下载速度
```
mkdir /etc/yum.repos.d/bak
mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/bak/
```
###### CentOS-Base.repo
```
vi /etc/yum.repos.d/CentOS-Base.repo

# CentOS-Base.repo
#
# The mirror system uses the connecting IP address of the client and the
# update status of each mirror to pick mirrors that are updated to and
# geographically close to the client.  You should use this for CentOS updates
# unless you are manually picking other mirrors.
#
# If the mirrorlist= does not work for you, as a fall back you can try the
# remarked out baseurl= line instead.
#
#
[os]
name=Qcloud centos os - $basearch
baseurl=http://mirrors.cloud.tencent.com/centos/$releasever/os/$basearch/
enabled=1
gpgcheck=1
gpgkey=http://mirrors.cloud.tencent.com/centos/RPM-GPG-KEY-CentOS-7

[updates]
name=Qcloud centos updates - $basearch
baseurl=http://mirrors.cloud.tencent.com/centos/$releasever/updates/$basearch/
enabled=1
gpgcheck=1
gpgkey=http://mirrors.cloud.tencent.com/centos/RPM-GPG-KEY-CentOS-7

[centosplus]
name=Qcloud centosplus - $basearch
baseurl=http://mirrors.cloud.tencent.com/centos/$releasever/centosplus/$basearch/
enabled=0
gpgcheck=1
gpgkey=http://mirrors.cloud.tencent.com/centos/RPM-GPG-KEY-CentOS-7

[cloud]
name=Qcloud centos contrib - $basearch
baseurl=http://mirrors.cloud.tencent.com/centos/$releasever/cloud/$basearch/openstack-kilo/
enabled=0
gpgcheck=1
gpgkey=http://mirrors.cloud.tencent.com/centos/RPM-GPG-KEY-CentOS-7

[cr]
name=Qcloud centos cr - $basearch
baseurl=http://mirrors.cloud.tencent.com/centos/$releasever/cr/$basearch/
enabled=0
gpgcheck=1
gpgkey=http://mirrors.cloud.tencent.com/centos/RPM-GPG-KEY-CentOS-7

[extras]
name=Qcloud centos extras - $basearch
baseurl=http://mirrors.cloud.tencent.com/centos/$releasever/extras/$basearch/
enabled=1
gpgcheck=1
gpgkey=http://mirrors.cloud.tencent.com/centos/RPM-GPG-KEY-CentOS-7

[fasttrack]
name=Qcloud centos fasttrack - $basearch
baseurl=http://mirrors.cloud.tencent.com/centos/$releasever/fasttrack/$basearch/
enabled=0
gpgcheck=1
gpgkey=http://mirrors.cloud.tencent.com/centos/RPM-GPG-KEY-CentOS-7

```
###### kubernetes.repo
```
vi /etc/yum.repos.d/kubernetes.repo

[kubernetes]
name=Kubernetes
baseurl=http://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=http://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg http://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg

```
###### docker-ce.repo
```
vi /etc/yum.repos.d/docker-ce.repo

[docker-ce-stable]
name=Docker CE Stable - $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/stable
enabled=1
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-stable-debuginfo]
name=Docker CE Stable - Debuginfo $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/debug-$basearch/stable
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-stable-source]
name=Docker CE Stable - Sources
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/source/stable
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-edge]
name=Docker CE Edge - $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/edge
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-edge-debuginfo]
name=Docker CE Edge - Debuginfo $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/debug-$basearch/edge
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-edge-source]
name=Docker CE Edge - Sources
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/source/edge
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-test]
name=Docker CE Test - $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/test
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-test-debuginfo]
name=Docker CE Test - Debuginfo $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/debug-$basearch/test
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-test-source]
name=Docker CE Test - Sources
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/source/test
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-nightly]
name=Docker CE Nightly - $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/$basearch/nightly
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-nightly-debuginfo]
name=Docker CE Nightly - Debuginfo $basearch
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/debug-$basearch/nightly
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

[docker-ce-nightly-source]
name=Docker CE Nightly - Sources
baseurl=https://mirrors.aliyun.com/docker-ce/linux/centos/7/source/nightly
enabled=0
gpgcheck=1
gpgkey=https://mirrors.aliyun.com/docker-ce/linux/centos/gpg

```
###### 清空yum缓存并重新加载
```
yum clean all
yum makecache
```
#### 安装相关插件
```
yum install ipset -y

yum install ipvsadm -y

yum install -y docker-ce-18.09.7-3.el7
mkdir /etc/docker
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF
mkdir -p /etc/systemd/system/docker.service.d
systemctl daemon-reload && systemctl restart docker
systemctl enable docker && systemctl start docker

yum install -y kubelet kubeadm kubectl
systemctl enable kubelet && systemctl start kubelet
systemctl daemon-reload && systemctl restart kubelet

```
# 安装haproxy + keepalived, 实现HA
- 在所有的master节点上配置haproxy代理和keepalived
```
mkdir /etc/haproxy
cat >/etc/haproxy/haproxy.cfg<<EOF
global
  log 127.0.0.1 local0 err
  maxconn 4096
  uid 99
  gid 99
  #daemon
  nbproc 1
  pidfile haproxy.pid

defaults
  mode http
  log 127.0.0.1 local0 err
  maxconn 4096
  retries 3
  timeout connect 5s
  timeout client 30s
  timeout server 30s
  timeout check 2s

listen admin_stats
  mode http
  bind 0.0.0.0:1080
  log 127.0.0.1 local0 err
  stats refresh 30s
  stats uri     /haproxy-status
  stats realm   Haproxy\ Statistics
  stats auth    baicheng:baicheng
  stats hide-version
  stats admin if TRUE

frontend k8s-https
  bind 0.0.0.0:8443
  mode tcp
  #maxconn 4096
  default_backend k8s-https

backend k8s-https
  mode tcp
  balance roundrobin
  server master1.k8s.wisebeta.sidri.com 10.190.194.159:6443 weight 1 maxconn 1000 check inter 2000 rise 2 fall 3
  server master2.k8s.wisebeta.sidri.com 10.190.194.160:6443 weight 1 maxconn 1000 check inter 2000 rise 2 fall 3
  server master3.k8s.wisebeta.sidri.com 10.190.194.161:6443 weight 1 maxconn 1000 check inter 2000 rise 2 fall 3
EOF

docker run -d --name my-haproxy \
-v /etc/haproxy:/usr/local/etc/haproxy:ro \
-p 8443:8443 \
-p 1080:1080 \
--restart always \
registry.cn-shanghai.aliyuncs.com/baicheng_dev/haproxy:2.0.0

docker run --net=host --cap-add=NET_ADMIN -d \
-e KEEPALIVED_INTERFACE=ens33 \
-e KEEPALIVED_VIRTUAL_IPS="#PYTHON2BASH:['192.168.250.99']" \
-e KEEPALIVED_UNICAST_PEERS="#PYTHON2BASH:['192.168.250.141','192.168.250.142','192.168.250.143']" \
-e KEEPALIVED_PASSWORD=baicheng \
--name k8s-keepalived \
--restart always \
registry.cn-shanghai.aliyuncs.com/baicheng_dev/keepalived:2.0.16

```
