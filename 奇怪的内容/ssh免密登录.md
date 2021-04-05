# ssh免密登录

## 生产私钥、公钥

指令：然后一路回车就行了，期间要输入文件名、密码（这个就可以省了）  
生成了公钥和私钥，pub格式的时公钥

* 以管理员身份运行terminal

``` text
ssh-keygen -t rsa
```

## 公钥发送到服务器

将公钥添加到服务器

``` text
# 发送到服务器
scp C:\Users\sytem\.ssh\sshkey.pub ysh@192.168.199.128:/tmp

# 添加到文件的末尾， 如果没有自行在用户目录下创建
cat /tmp/sshkey.pub >> ~/.ssh/authorized_keys
```

## ssh连接

连接时带上私钥就可以了

``` text
ssh -i C:/Users/sytem/.ssh/sshkey ysh@192.168.199.128
```

## 更加详尽的连接

在.ssh文件夹中创建config文件，配置如下：  
`Host`是一个ssh连接的名称，`HostName`是服务器的地址，`User`是服务器的用户名，`IdentityFile`是本地保存的私钥  
可以在终端中使用`ssh vm` 进行连接

config中可以配置多个Host

``` text
Host vm
    HostName 192.168.199.128
    User ysh
    IdentityFile .ssh/sshkey
```
