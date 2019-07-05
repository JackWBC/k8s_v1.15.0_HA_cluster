# k8s_v1.15.0_HA_cluster
k8s_v1.15.0高可用集群基础环境搭建

# 虚拟机环境配置清单
- 操作系统 CentOS7 x86_64 mini  
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
