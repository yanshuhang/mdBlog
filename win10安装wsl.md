# 安装wsl

## 开启功能

在启用和关闭windows功能中勾选适用于`linux的windows的子系统`

## 商店下载

在win10商店中搜索ubuntu下载安装即可

## 启动

## 切换下载源

默认的下载源速度很慢，切换到阿里云的镜像速度就快很多了  
命令打开文件  
`sudo vim /etc/apt/sources.list`  
然后替换：  
`:%s/security.ubuntu/mirrors.aliyun/g`  
`:%s/archive.ubuntu/mirrors.aliyun/g`  
更新：  
`sudo apt update`
