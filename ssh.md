本地测试，连接的是本地虚拟机
虚拟机中linux安装 openssh
关闭防火墙
```shell
systemctl stop firewalld
systemctl disable firewalld
```

在虚拟机中将端口转发出来
在宿主机终端检测端口是否能用
登录虚拟机
```shell
ssh username@127.0.0.1 -p 2222
```
本地生成id
```shell
ssh-keygen
```

然后将该id拷贝到服务器(虚拟机)中
```shell
ssh-copy-id username@127.0.0.1 -p 2222
```

在本机主目录下建议配置文件
```shell
touch ~/.ssh/config
```
打开该文件并输出以下内容
```vim
Host username
    HostName 127.0.0.1
    Port 2222
    User username
    ServerAliveInterval 90
    TCPKeepAlive yes
```
保存退出
然后就可以在宿主机使用下面命令登入服务器了
```shell
ssh username
```