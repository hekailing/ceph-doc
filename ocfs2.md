## 验证ceph-client可以正常连接到集群

    sudo rbd ls
  
## 安装ocfs2支持

    sudo apt-get install ocfs2-tools
    
添加或者修改配置文件/etc/ocfs2/cluster.conf

    node:
        ip_port = 7777
        ip_address = 192.168.0.13
        number = 0
        name = node-0
        cluster = ocfs2
    node:
        ip_port = 7777
        ip_address = 192.168.0.14
        number = 1
        name = node-1
        cluster = ocfs2
    node:
        ip_port = 7777
        ip_address = 192.168.0.15
        number = 2
        name = node-2
        cluster = ocfs2
    cluster:
        node_count = 3
        name = ocfs2
    
## 启动ocfs服务

    sudo service o2cb enable
    sudo service o2cb status
    sudo service ocfs2 start
    
## 创建块设备（如果已创建可跳过该步骤）

    sudo rbd create testocfs2 --size 10240
    
## 映射块设备

    sudo rbd map --name client.rbd
    
## 格式化块设备并挂载

    sudo mkfs.ocfs2 /dev/rbd0
    sudo mkdir /mount_point
    sudo mount -t ocfs2 /dev/rbd0 /mount_point
