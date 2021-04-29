---
title: Elasticsearch 分布式部署
date: 2021-04-29 10:34:08
categories:
- Elasticsearch
tags:
- Elasticsearch
- 分布式
- 部署
---

# 环境说明

1. 操作系统 CentOS 7，软件版本 Elasticsearch 7.12.2。

2. 三台服务器，详情如下：

| ip | 节点名 |
|:--:|:-----:|
| 192.168.3.100 | node-01 |
| 192.168.3.101 | node-02 |
| 192.168.3.102 | node-03 |

3. 以下步骤如无特殊说明，均为三台服务器上同样执行。

# 准备工作

### 创建 Elasticsearch 用户

因为 Elasticsearch 不允许通过 `root` 用户运行，需要首先创建用于运行 Elasticsearch 的用户。

```shell
# 创建用户组 elastic
groupadd elastic
# 创建用户
adduser -g elastic elastic
# 设置密码
passwd es
# 赋予 sudo 权限
chmod +w /etc/sudoers
vi /etc/sudoers
# 在打开的文件中添加如下行
elastic ALL=(ALL) ALL
# 修改完成后恢复 /etc/sudoers 文件的原始权限
chmod -w /etc/sudoers
```

### 创建 Elasticsearch 相关目录

为了方便管理，可以提前建好 Elasticsearch 相关目录。

```shell
# data 目录
mkdir /opt/elastic/data
# logs 目录
mkdir /opt/elastic/logs
# 更改所有者
chmod -R elastic:elastic /opt/elastic
```

### 防火墙相关

保证端口开放可以访问即可，具体操作可查看各发行版防火墙文档。

# 安装 Elasticsearch

### 安装

因为 Elasticsearch 解压即可运行，无需安装，为了方便管理，我们把 Elasticsearch 放到 `/opt` 目录下。

```shell
cd /opt
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.12.2-linux-x86_64.tar.gz
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.12.2-linux-x86_64.tar.gz.sha512
shasum -a 512 -c elasticsearch-7.12.2-linux-x86_64.tar.gz.sha512
tar -xzf elasticsearch-7.12.2-linux-x86_64.tar.gz
chmod -R elastic:elastic /opt/elasticsearch-7.12.2
```

### 配置

```shell
cd /opt/elasticsearch-7.12.2
vi config/elasticsearch.yml
# Cluster 段
cluster.name: my_elastic_cluster
# Node 段，三台服务器根据自己的节点名字配置
node.name: node-01
# Path 段，配置相关目录，用到准备工作提前建好的目录
path.data: /opt/elastic/data
path.logs: /opt/elastic/logs
# Network 段，三台服务器根据自己的网络进行配置
network.host: 192.168.3.100
# http 协议端口
http.port: 9200
# tcp 端口，可用于节点间交互
transport.tcp.port: 9300
# Discovery 段
discovery.seed_hosts: ["192.168.3.100:9300", "192.168.3.101:9300", "192.168.3.102:9300"]
cluster.initial_master_nodes: ["node-01"]
```

# 启动与停止

```shell
# 切换到 elastic 用户
su elastic
cd /opt/elasticsearch-7.12.2
# 后台运行并将进程 id 保存到 /opt/elasticsearch-7.12.2/pid 文件
./bin/elasticsearch -d -p pid
# 停止
pkill -F pid
```

启动后，即可访问 `http://192.168.3.100:9200` 查看运行状态信息。

# 开启基础用户认证

### 证书的创建与配置

以下所有操作都是在 elastic 用户下。

##### 192.168.3.100

1. 创建一个放证书的目录，统一管理。

```shell
cd /opt/elasticsearch-7.12.2
mkdir config/certs
```

2. 生成 ca 证书。

```shell
./bin/elasticsearch-certutil ca --out config/certs/my-elastic-ca.p12
```

3. 为本节点生成证书。

```shell
./bin/elasticsearch-certutil --ca config/certs/my-elastic-ca.p12 --ip 192.168.3.100 --out config/certs/node-01.p12
```

4. 配置证书，在配置文件 `config/elasticsearch.yml` 中添加如下配置：

```
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.transport.ssl.keystore.path: certs/${node.name}.p12
xpack.security.transport.ssl.truststore.path: certs/${node.name}.p12
```

`${node.name}` 变量为当前节点的名称。如果你的证书并没有这样命名，这里修改成实际的证书文件名。

5. 为其它节点生成证书，该操作会在当前目录下生成一个压缩文件 `certificate-bundle.zip`。

```shell
./bin/elasticsearch-certutil cert --ca config/certs/my-elastic-ca.p12 --multiple
```

6. 解压文件会得到以其它节点名称命名的文件夹，把各个节点的证书放到各个节点服务器上待处理。

```shell
unzip certificate-bundle.zip
scp node-02 root@192.168.3.101:/
scp node-03 root@192.168.3.102:/
```

##### 192.168.3.101 及 192.168.3.102

1. 创建一个证书文件目录，参考主节点上操作，并将传送过来的证书文件放在此处。

2. 配置证书，参考主节点上的证书配置。

### 为 Elasticsearch 内部用户设置密码

```shell
./bin/elasticsearch-setup-passwords interactive
```

服务重启后，再次访问 Elasticsearch 将会弹出用户认证界面。
