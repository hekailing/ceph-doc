# ceph配置入门篇

## 创建账户,并设置权限（所有ceph节点）

    sudo useradd ceph-node -m -b /bin/bash
    echo "ceph-node ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ceph-node
    sudo chmod 0440 /etc/sudoers.d/ceph-node

## 设置ssh免密登录（仅主控节点）

### 生成密钥

    ssh-keygen
  
### 设置主控节点通过ssh访问其他节点

1. 修改/etc/hosts文件，添加各个节点

2. 编辑.ssh/config文件，如下：

> Host node-0
>  Hostname node-0
>  User ceph-node
> Host node-1
>  Hostname node-1
>  User ceph-node
> Host node-2
>  Hostname node-2
>  User ceph-node

### 拷贝密钥

    ssh-copy-id ceph-node@node-x
