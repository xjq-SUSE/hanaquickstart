# hanaquickstart

本文档描述了如何在AWS上使用SUSE Linux Enterprise Server for SAP 搭建双机热备SAP HANA数据库

## HANA on AWS 架构

SAP HANA on AWS 支持四种部署方式
* 性能模式（双机热备）
* 性价比模式（备机上起测试，QA）
* 多租户模式
* 水平扩展模式（Scale-out）

架构如下图所示：  
在开始部署前需要确定好系统参数，如下图所示：  

## 部署AWS资源

推荐使用CloudFormation部署AWS资源，需要的资源如下：
* 访问实例所需的key-pair
* 两个实例和其他资源（包括一个新的VPC）
* 配置IAM权限

### 创建key-pair

进入AWS EC2控制台，创建一个key-pair并下载pem文件保存在本地。

### 部署AWS资源

进入AWS CloudFormation控制台，导入模板susehana.template，该模板预置了HANA所需的资源，如下图所示：  

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





