# ceph配置入门篇

## 准备工作

### 创建账户,并设置权限（所有ceph节点）

    sudo useradd ceph-node -m -s /bin/bash
    echo "ceph-node ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ceph-node
    sudo chmod 0440 /etc/sudoers.d/ceph-node

### 设置ssh免密登录（仅主控节点）

#### 生成密钥

    ssh-keygen
  
#### 设置主控节点通过ssh访问其他节点

1. 修改/etc/hosts文件，添加各个节点

2. 编辑.ssh/config文件，如下：

       Host node-0
         Hostname node-0
         User ceph-node  
       Host node-1
         Hostname node-1
         User ceph-node  
       Host node-2
         Hostname node-2
         User ceph-node

#### 拷贝密钥

    ssh-copy-id ceph-node@node-x

### 安装ntp服务（所有节点）

    sudo apt-get install ntp
    
修改ntp配置/etc/ntp.conf，在最上面添加一行：

> tinker step 0.5

重启ntp服务

    sudo service restart
    
### 添加ceph安装源（所有节点）

国内源

    wget -q -O- 'http://mirrors.163.com/ceph/keys/release.asc' | sudo apt-key add -
    echo deb http://mirrors.163.com/ceph/debian-jewel/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list
    
国外源

    wget -q -O- 'https://download.ceph.com/keys/release.asc' | sudo apt-key add -
    echo deb https://download.ceph.com/debian-jewel/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list

### 安装ceph-deploy部署工具（仅主控节点）

更新仓库，并安装ceph_deploy（如果使用国外源安装失败，请换用国内源）

    sudo apt-get update && sudo apt-get install ceph-deploy
    
## 启动ceph-mon

### 安装ceph

为了加快安装速度，建议使用国内源，命令如下：

    export CEPH_DEPLOY_REPO_URL=http://mirrors.163.com/ceph/debian-jewel 
    export CEPH_DEPLOY_GPG_URL=http://mirrors.163.com/ceph/keys/release.asc
    
### 创建部署目录（目录名不重要），目录内生成ceph配置文件

    mkdir my-cluster && cd my-cluster
    
### 配置新节点

    ceph-deploy new node-0,node-1,node-2, ...
    
mycluster目录下生成几个文件，如ceph.conf;ceph.mon.keyring等

在ceph.conf的末尾添加一行：public network=xxx.xxx.xxx.0/24　（结尾需要换行，在文件末尾有一个空行）

### 安装节点

    ceph-deploy install node-0,node-1,node-2, ...
    
### 配置并启动ceph-mon

    ceph-deploy mon create initial
然后运行sudo ceph -s就可以看到当前集群的状态，mon的数量就是上面的节点数，因为没有osd，所以健康状况是HEALTH_ERR，有１个默认的pool

## 添加ceph-osd节点

假如在node1上添加一个osd，使用磁盘sdb

### 安装ceph到osd节点(如果该节点也是上一步设置完成的mon节点，则跳过该步骤）

    ceph-deploy install node-list
    
### 准备磁盘（如果磁盘已挂载，请先备份数据并卸载，下面的命令会删除磁盘内所有内容）

    ceph-deploy disk zap node1:sdb
    
### 创建osd

    ceph-deploy osd prepare node1:sdb
    
### 启动osd

    ceph-deploy osd activate node1:sdb1
    
所有osd节点添加完成后，运行sudo ceph -s，确认osd节点全部加入

### 使集群处于健康状态

查看副本数

    sudo ceph osd dump | grep 'replicated size'

查看已经存在的pools

    sudo ceph osd lspools
    
查看rbd pool中的pg_num和pgp_num属性

    sudo ceph osd pool get rbd pg_num
    sudo ceph osd pool get rbd pgp_num
    
健康的pg_num和pgp_num计算方法

> 关于pgmap的数目，osd_num *100 / replica_num，向上取2的幂。比如15个osd，三备份，15 *100/3=500，得到pg_num = 512，线上重新设定这个数值时会引起数据迁移，且**只能增加，不能减少**，请谨慎处理。

## 设置ceph的rbd块存储设备

确保ceph存储集群处于active+clean的状态，这样才能使用块设备

### 建立主控机器到rbd所在设备的ssh免密登录关系

参见[准备工作](https://github.com/hekailing/ceph-doc/blob/master/install.md#设置主控节点通过ssh访问其他节点)

### 安装ceph环境，并授予权限

假设rbd所在机器的hostname为ceph-cli（ceph-cli可以是一台新主机，也可以是上面任意节点）

1.在主控节点上为ceph-cli安装环境

    ceph-deploy install ceph-cli
    
2.将ceph配置文件（ceph.conf）复制到ceph-cli
    
    ceph-deploy config push ceph-cli
    
3.客户机需要ceph密钥去访问ceph集群。ceph创建了一个默认用户 client.admin，它有足够的权限去访问ceph集群。不建议把client.admin共享到所有其他客户端节点。更好的做法是用分开的密钥创建一个新的ceph用户去访问特定的存储池。这里，创建一个ceph用户client.rbd，它拥有访问rbd存储池的权限

    sudo ceph auth get-or-create client.rbd mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=rbd'
    
4.为ceph-cli上的client.rbd用户添加密钥

    ceph auth get-or-create client.rbd | ssh ceph-cli 'sudo tee /etc/ceph/ceph.client.rbd.keyring'
    
5.登录ceph-cli，将密钥添加到/etc/ceph/keyring

    cat /etc/ceph/ceph.client.rbd.keyring >> /etc/ceph/keyring

然后就可以在ceph-cli上查询集群的状态了

    ceph -s --name client.rbd
    
### 创建ceph块设备

1.创建一个1000G的RADOS块设备，取名rbd1（设备大小视需求而定）

    sudo rbd create rbd1 --size 102400 --name client.rbd --image-feature layering
    # 14.04的内核只支持layering，增加不支持的特性会导致后续的map失败。
    
2.列出RBD镜像

    sudo rbd ls --name client.rbd
    
3.检查rbd镜像的细节
   
   sudo rbd --image rbd1 info --name client.rbd
   
### 映射块设备并初始化

1.映射块设备到 ceph-cli

    sudo rbd map --image rbd1 --name client.rbd
    > /dev/rbd0
    
2.检查被映射的块设备

    sudo rbd showmapped --name client.rbd
    > id pool image snap device    
    > 0  rbd  rbd1  -    /dev/rbd0
    
3.要使用这个块设备，我们需要创建并挂载一个文件系统

    sudo mkfs.ext3 /dev/rbd0
    sudo mount /dev/rdb0 /mountdir
