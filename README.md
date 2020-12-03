# hanaquickstart

本文档描述了如何在AWS上使用SUSE Linux Enterprise Server for SAP 搭建双机热备SAP HANA数据库

## HANA on AWS 架构

SAP HANA on AWS 支持四种部署方式
* 性能模式（双机热备）
* 性价比模式（备机上起测试，QA）
* 多租户模式
* 水平扩展模式（Scale-out）

架构如下图所示：  
![arch](https://github.com/xjq-SUSE/hanaquickstart/blob/main/pic/arch.png)  

在开始部署前需要确定好系统参数，如下图所示：  
![table](https://github.com/xjq-SUSE/hanaquickstart/blob/main/pic/table.png)

## 部署AWS资源

推荐使用CloudFormation部署AWS资源，需要的资源如下：
* 访问实例所需的key-pair
* 两个实例和其他资源（包括一个新的VPC）
* 配置IAM权限

### 创建key-pair

进入AWS EC2控制台，创建一个key-pair并下载pem文件保存在本地。

### 部署AWS资源

进入AWS CloudFormation控制台，导入模板susehana.template，该模板预置了HANA所需的资源，如下图所示：  
![network](https://github.com/xjq-SUSE/hanaquickstart/blob/main/pic/network.png)

创建stack。等待stack创建完成无报错。检查以下资源已被成功创建：
* 两个实例
* 每个实例外挂4个volume
* 两个Elastic IP
* 一个安全组
* 一个VPC
* 两个子网
* 一个Internetgateway
* 一张路由表
* 默认路由
* 一个浮动IP

### 为资源配置IAM权限

为了使用AWS的STONITH机制和Overlay-IP需要为资源配置权限，打开AWS IAM控制台，创建一个新的Role，并导入iam.json，注意将脚本中的instanceid改为对应的id。将这个role赋予两个实例。

## 安装SAP HANA数据库

1. 将四块外挂硬盘格式化并挂载

```
# mkfs.xfs /dev/sdb1
# mkfs.xfs /dev/sdc1
# mkfs.xfs /dev/sdd1
# mkfs.xfs /dev/sde1
# mount /dev/sdb1 /hana/shared
# mount /dev/sdb1 /hana/shared
# mount /dev/sdb1 /hana/shared
# mount /dev/sdb1 /hana/shared
```

2. 将HANA安装文件上传到指定目录，并给予执行权限

```
# chmod +x /hana/shared*
```

3. 安装数据库，按照数据表填入参数

```
# ./hdblcm --ignore=check_signature_file
# reboot
```

4. 修改sidadm用户权限

```
# sp1adm ALL=(ALL) NOPASSWD: /usr/sbin/crm_attribute -n hana_ha1_site_srHook_*
```

5. 验证数据库安装成功

```
# su sidadm
# HDB info
```

### 配置HANA System Replication 

HANA SR用于在主备节点间进行数据同步

1. 将以下密钥从主用节点复制到备用节点相同位置

```
# /usr/sap/SP1/SYS/global/security/rsecssfs/data/SSFS_SP1.DAT
# /usr/sap/SP1/SYS/global/security/rsecssfs/key/SSFS_SP1.KEY
```

2. 在主用节点上备份数据库

```
# hdbsql -u system -i 10 "BACKUP DATA USING FILE ('backup')"
# hdbsql -u system -d systemdb -i 10 "BACKUP DATA USING FILE ('sysbackup')"
```

3. 在主用节点激活SR并查看状态

```
# hdbnsutil -sr_enable --name=AWS
# hdbnsutil -sr_state 
```

4. 在备用节点激活SR并查看状态

```
# HDB stop
# hdbnsutil -sr_register --name=SWA \
 --remoteHost=node1 --remoteInstance=10 \
 --replicationMode=sync --operationMode=logreplay
# HDB start
# hdbnsutil -sr_state
```

## 配置HA

SUSE Linux Enterprise Server for SAP 自带 High Availability Extension高可用组件，可以帮助用户快速完成高可用集群搭建。这里使用AWS STONITH机制和Overlay-IP做为集群的浮动IP。

### 系统配置

1.将系统注册到SCC，以获取最新的更新和安全补丁，如果没有注册码可以在SUSE官网申请60天试用

```
# SUSEConnect -r your_reg_code
```

2. 创建aws-cli配置文件 /root/.aws/config

```
[default]
region = region-name
[profile cluster]
region = region-name
output = text
```

### 初始化HA集群

1. 在主节点上生成密钥并拷贝至备用节点

```
# corosync-keygen
# ls -l /etc/corosync/authkey
```

2. 创建配置文件 /etc/corosync/corosync.conf

```
# Read the corosync.conf.5 manual page
totem {
 version: 2
 rrp_mode: passive
 token: 5000
 consensus: 7500
 token_retransmits_before_loss_const: 6
 secauth: on
 crypto_hash: sha1
 crypto_cipher: aes256
 clear_node_high_bit: yes
 interface {
 ringnumber: 0
 bindnetaddr: ip-local-node
 mcastport: 5405
 ttl: 1
 }
 transport: udpu
}
logging {
 fileline: off
 to_logfile: yes
 to_syslog: yes
 logfile: /var/log/cluster/corosync.log
 debug: off
 timestamp: on
 logger_subsys {
 subsys: QUORUM
 debug: off
 }
}
nodelist {
 node {
 ring0_addr: ip-node-1-a ring1_addr: ip-node-1-b # redundant ring
 nodeid: 1
 }
 node {
 ring0_addr: ip-node-2-a
 ring1_addr: ip-node-2-b # redundant ring
 nodeid: 2
 }
}
quorum {
# Enable and configure quorum subsystem (default: off)
# see also corosync.conf.5 and votequorum.5
 provider: corosync_votequorum
 expected_votes: 2
 two_node: 1
}
```

3. 在两个节点上启动集群

```
# systemctl start pacemaker
# systemctl enable pacemaker
# crm_mon -r
```

4. 通过配置文件创建集群资源

```
# crm configure load update crm-hana.txt
# crm status
```

# 参考文档

[SAP HANA on the AWS Cloud: Quick Start Reference Deployment](https://docs.aws.amazon.com/quickstart/latest/sap-hana/)  
[SAP HANA High Availability Cluster for the AWS Cloud - Setup Guide (v12)](https://documentation.suse.com/sbp/all/html/SLES4SAP-hana-sr-guide-PerfOpt-12_AWS/index.html)  