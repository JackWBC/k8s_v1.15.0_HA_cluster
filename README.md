# k8s_v1.15.0_HA_cluster
k8s_v1.15.0高可用集群基础环境搭建

# 虚拟机环境配置清单
操作系统 CentOS7 x86_64 mini
3个master, 3个node, 域名与IP如下

'192.168.250.141       master1.k8s'
'192.168.250.142       master2.k8s'

192.168.250.143       master3.k8s

192.168.250.144       node1.k8s

192.168.250.145       node2.k8s

192.168.250.146       node3.k8s

192.168.250.99        虚拟IP
'''
角色  | 域名  | IP
 ---- | ----- | ------  
 master  | master1.k8s | 192.168.250.141 
 master  | master2.k8s | 192.168.250.142

