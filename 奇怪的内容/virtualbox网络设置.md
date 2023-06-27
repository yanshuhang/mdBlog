# vitrualbox网络设置

## 配置网卡

配置两张网卡：连接方式分别为仅主机(Host-Only)和网络地址转换(NAT)模式

## 虚拟机配置文件

修改文件/etc/netplan/00-installer-config.yaml

```yml
network:
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: true
  version: 2
```