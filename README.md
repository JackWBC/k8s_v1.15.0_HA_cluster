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
