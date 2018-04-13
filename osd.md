# OSD相关命令

## 添加OSD

    ceph-deploy osd prepare node1:sdb
    ceph-deploy osd activate node1:sdb1
    
## 查看OSD

    ceph osd tree

## 删除OSD

这里以删除osd.0为例

    ceph osd out osd.0
    service ceph stop osd.0
    ceph osd crush remove osd.0
    ceph auth del osd.0
    ceph osd rm 0
