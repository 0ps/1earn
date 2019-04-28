# Power-Linux🎓
`Linux 下各种常见服务的搭建/配置指南`
[TOC]

**Todo**
- [ ] oracle 11e

`目前主要以安装搭建为主，更深一步的配置请自行研究`

---

# 系统配置
## Net
```vim
vi /etc/sysconfig/network-scripts/ifcfg-eth0
	DEVICE="enoXXXXXX"
	BOOTPROTO=static　　　　　　　#使用静态IP,而不是由DHCP分配IP
	IPADDR=172.16.102.61
	PREFIX=24
	GATEWAY=172.16.102.254
	HOSTNAME=dns1.abc.com

vi /etc/hosts
	127.0.0.1  test localhost  #修改localhost.localdomain为test,shutdown -r now重启使修改生效

	修改DNS
		vim /etc/resolv.conf
			nameserver 8.8.8.8
```
>service network restart

---

## 配置本地yum源,挂载,安装
挂载到/mnt/cdrom

>	mkdir /mnt/cdrom
>	mount /dev/cdrom /mnt/cdrom/

设置一下自动挂载
```vim
vim /etc/fstab
	/dev/cdrom /mnt/cdrom iso9660 defaults 0 0
```

进入 /etc/yum.repos.d 目录,将其中三个改名或者移走留下 CentOS-Base.repo
```bash
cd /etc/yum.repos.d
rm  CentOS-Media.repo    
rm  CentOS-Vault.repo     
```

编辑 CentOS-Base.repo
```vim
vi CentOS-Base.repo
	baseurl=file:///mnt/cdrom/  #这里为本地源路径
	gpgcheck=0	
	enabled=1    #开启本地源

yum list  #看一下包
```

---

## RAID
- 创建RAID1阵列,设备文件名为md0；
- 将新建的RAID1格式化为xfs文件系统,编辑/etc/fstab文件实现以UUID的形式开机自动挂载至/data/ftp_data目录。

**安装**
>yum remove mdadm	#建议先把原本的卸掉重装
>yum install mdadm

**分区**
```bash
fdisk /dev/sdb
n 创建
p 主分区
接下来一路回车选默认值
w 写入

fdisk /dev/sdc
n 创建
p 主分区
接下来一路回车选默认值
w 写入
```

**创建阵列**
- RAID1
	`mdadm -Cv /dev/md0 -a yes -l1 -n2 /dev/sd[b,c]1`
	- -Cv: 创建一个阵列并打印出详细信息。
	- /dev/md0: 阵列名称。
	-a　: 同意创建设备,如不加此参数时必须先使用mknod 命令来创建一个RAID设备,不过推荐使用-a yes参数一次性创建；
	- -l1 (l as in "level"): 指定阵列类型为 RAID-1 。
	- -n2: 指定我们将两个分区加入到阵列中去,分别为/dev/sdb1 和 /dev/sdc1

- RAID5
	`mdadm -Cv /dev/md0 -a yes -l5 -n3 /dev/sd[b,c,d]1`

	可以使用以下命令查看进度：
	`cat /proc/mdstat`

	另外一个获取阵列信息的方法是：
	`mdadm -D /dev/md0`

格式化为xfs
`mkfs.xfs /dev/md0`

以UUID的形式开机自动挂载
```vim
mkdir /data/ftp_data
blkid	/dev/md0 查UUID值

vi /etc/fstab
	UUID=XXXXXXXXXXXXXXXXXXXXXXXXXX    /data/ftp_data  xfs defaults 0 0

重启验证
shutdown -r now 
mount | grep '^/dev'
```

---

## Lvm物理卷
```bash
fdisk ‐l		查看磁盘情况
fdisk /dev/sdb	创建系统分区
	n
	p
	1
	后面都是默认,直接回车
		
	t	转换分区格式
	8e

	w	写入分区表
```

**卷组**
创建一个名为 datastore 的卷组,卷组的PE尺寸为 16MB；
>	pvcreate /dev/sdb1	创建物理卷
>	vgcreate ‐s 16M datastore /dev/sdb1	

**逻辑卷**
逻辑卷的名称为 database 所属卷组为 datastore,该逻辑卷由 50 个 PE 组成；
>	lvcreate ‐l 50 ‐n database datastore

逻辑卷的名称为database所属卷组为datastore,该逻辑卷大小为8GB；
>	lvcreate ‐L 8G ‐n database datastore
>	lvdisplay

**格式化**
将新建的逻辑卷格式化为 XFS 文件系统,要求在系统启动时能够自动挂在到 /mnt/database 目录。	
>	mkfs.xfs /dev/datastore/database
>	mkdir /mnt/database
```vim
vi /etc/fstab
	/dev/datastore/database /mnt/database/ xfs defaults 0 0
```

重启验证
>shutdown -r now 
>mount | grep '^/dev'

**扩容**
业务扩增,导致database逻辑卷空间不足,现需将database逻辑卷扩容至15GB空间大小,以满足业务需求。（注意扩容前后截图）
>lvextend -L 15G /dev/datastore/database
>lvs	#确认有足够空间
>resize2fs /dev/datastore/database
>lvdisplay

---

## Vim
常用配置
`sudo vim /etc/vim/vimrc`
最后面直接添加你想添加的配置,下面是一些常用的（不建议直接复制这个货网上的,要理解每个的含义及有什么用,根据自己需要来调整）
```vim
set number #显示行号
set nobackup #覆盖文件时不备份
set cursorline #突出显示当前行
set ruler #在右下角显示光标位置的状态行
set shiftwidth=4 #设定 > 命令移动时的宽度为 4
set softtabstop=4 #使得按退格键时可以一次删掉 4 个空格
set tabstop=4 #设定 tab 长度为 4(可以改）
set smartindent #开启新行时使用智能自动缩进
set ignorecase smartcase #搜索时忽略大小写,但在有一个或以上大写字母时仍 保持对大小写敏感
下面这个没觉得很有用,在代码多的时候会比较好
#set showmatch #插入括号时,短暂地跳转到匹配的对应括号
#set matchtime=2 #短暂跳转到匹配括号的时间
```

---

# 网络服务
## [Chrony](https://chrony.tuxfamily.org/)
它由两个程序组成：chronyd和chronyc。
chronyd是一个后台运行的守护进程,用于调整内核中运行的系统时钟和时钟服务器同步。它确定计算机增减时间的比率,并对此进行补偿。
chronyc是用来监控chronyd性能和配置其参数程序

**安装**
```bash
yum install chrony
```

**配置文件**
```vim
vim /etc/chrony.conf
  server time1.aliyun.com iburst  
  server time2.aliyun.com iburst 
  server time3.aliyun.com iburst 
  server time4.aliyun.com iburst 
  server time5.aliyun.com iburst 
  server time6.aliyun.com iburst 
  server time7.aliyun.com iburst 
  或
  server time1.google.com iburst 
  server time2.google.com iburst 
  server time3.google.com iburst 
  server time4.google.com iburst
```

**启服务**
```vim
systemctl stop ntpd
systemctl disable ntpd

systemctl enable chronyd.service
systemctl start chronyd.service
```

**查看同步状态**
```bash
chronyc sourcestats #检查ntp源服务器状态
chronyc sources -v  #检查ntp详细同步状态

chronyc #进入交互模式
  activity
```

---

## DHCP
>yum install dhcp

复制一份示例
>cp /usr/share/doc/dhcp-4.1.1/dhcpd.conf.sample /etc/dhcp/dhcpd.conf 

```vim
vim /etc/dhcp/dhcpd.conf
	ddns-update-style interim;      # 设置DNS的动态更新方式为interim
	option domain-name "abc.edu";
	option domain-name-servers  8.8.8.8;           # 指定DNS服务器地址
	default-lease-time  43200;                          # 指定默认租约的时间长度,单位为秒
	max-lease-time  86400;  # 指定最大租约的时间长度
```

以下为某区域的 IP 地址范围
```bash
subnet 192.168.1.0 netmask 255.255.255.0 {         # 定义DHCP作用域
	range  192.168.1.20 192.168.1.100;                # 指定可分配的IP地址范围
	option routers  192.168.1.254;                       # 指定该网段的默认网关
}
```

>dhcpd -t    #检测语法有无错误
>service dhcpd start    #开启 dhcp 服务

记得防火墙放行

查看租约文件,了解租用情况
>cat /var/lib/dhcpd/dhcpd.leases

---

## DNS
**安装**
>yum install bind*

**主配置文件**
```vim
vim /etc/named.conf
options {
    listen-on port 53 { any; };
    listen-on-v6 port 53 { any; };
    allow-query     { any; };
}
```

**区域配置文件**
```vim
vim /etc/named.rfc1912.zones
zone "abc.com" IN { 
        type master;
        file "abc.localhost";
};

zone "1.1.1.in-addr.arpa" IN { 
        type master;
        file "abc.loopback";
};

zone "2.1.1.in-addr.arpa" IN { 
        type master;
        file "www.loopback";
};
```

**创建区域数据文件**
>cd /var/named/
cp named.localhost abc.localhost
cp named.loopback abc.loopback
cp named.loopback www.loopback

>chown named abc.localhost 
chown named abc.loopback
chown named www.loopback

**域名正向反向解析配置文件**
```vim
vim /var/named/abc.localhost
	$TTL 1D
	@      IN SOA  @ rname.invalid. (
                                        	0      ; serial
                                        	1D      ; refresh
                                        	1H      ; retry
                                        	1W      ; expire
                                        	3H )    ; minimum
	       	NS     @
       		A      127.0.0.1
	    	AAAA   ::1
	ftp    	A      1.1.1.1
	www     A      1.1.2.1

vim /var/named/abc.loopback 
	$TTL 1D
	@	IN SOA  @ rname.invalid. (
    	                                    0 ; serial
                                        	1D ; refresh
                                        	1H ; retry
                                        	1W ; expire
                                        	3H ) ; minimum
        	NS 		@
        	A 		127.0.0.1
        	AAAA	::1
        	PTR 	localhost.
	1 PTR ftp.abc.com.

vim /var/named/www.loopback 
	$TTL 1D
	@ 		IN SOA  @ rname.invalid. (
    	                                    0 ; serial
                                        	1D ; refresh
                                        	1H ; retry
                                        	1W ; expire
                                        	3H ) ; minimum
        	NS 		@
        	A 		127.0.0.1
        	AAAA	::1
        	PTR 	localhost.
	1 PTR www.abc.com.
```

**启服务**
```bash
named-checkconf
named-checkzone abc.com abc.localhost
named-checkzone abc.com abc.loopback 
named-checkzone abc.com www.loopback 
service named restart

setenforce 0
firewall-cmd --zone=public --add-service=dns --permanent
firewall-cmd --reload
```

---

## [SSH🔑](https://www.ssh.com)
一般主机安装完毕后 SSH 是默认开启的
使用`/etc/init.d/ssh status`查看主机SSH状态

**Kali/Manjaro**
安装完毕后会自动启动,但是没有配置配置文件会无法登陆,修改下配置文件
```vim
vim /etc/ssh/sshd_config
	PasswordAuthentication yes
	PermitRootLogin yes

service ssh restart
systemctl enable ssh
```
若在使用工具登录时,当输完用户名密码后提示SSH服务器拒绝了密码,请再试一遍。
这时请不要着急,只需要在Kali控制端口重新生成两个秘钥即可。
```bash
ssh-keygen -t dsa -f /etc/ssh/ssh_host_dsa_key
ssh-keygen -t dsa -f /etc/ssh/ssh_host_rsa_key
```

**Ubuntu**
如果没有就装一下
如果你只是想登陆别的机器的SSH只需要安装openssh-client（ubuntu有默认安装,如果没有则sudo
apt-get install openssh-client）,如果要使本机开放SSH服务就需要安装openssh-server
```bash
apt install openssh-client=1:7.2p2-4ubuntu2.8
apt install openssh-server=1:7.2p2-4ubuntu2.8
apt install ssh
```
然后重启SSH服务
`service ssh restart`

---

# web服务
## [Apache](https://www.apache.org/)
**安装**
```bash
yum install httpd
yum install mod_ssl
```

**配置文件**
```vim
vim /etc/httpd/conf/httpd.conf
		DocumentRoot "/var/www/html" 
		ServerName  xx.xx.xx.xx:80   ////设置Web服务器的主机名和监听端口
```

**启服务**
```vim
vim var/www/html/index.html 
	Hello World!

service httpd restart
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --reload
```

**虚拟主机**
```vim
配置虚拟主机文件
vim /etc/httpd/conf.d/virthost.conf
<VirtualHost 192.168.1xx.22:80>
	ServerName  www.abc.com     ////设置Web服务器的主机名和监听端口
	DocumentRoot "/data/web_data" 
	<Directory "/data/web_data">
		Require all granted
	</Directory>
</VirtualHost>

Listen 192.168.1XX.33:443 
<VirtualHost 192.168.1xx.22:443>
	ServerName  www.abc.com     ////设置Web服务器的主机名和监听端口
	DocumentRoot "/data/web_data" 
	
	SSLEngine on
	SSLCertificateFile /etc/httpd/ssl/httpd.crt
	SSLCertificateKeyFile /etc/httpd/ssl/httpd.key

	<Directory "/data/web_data">
		Require all granted
	</Directory>
</VirtualHost>

mkdir -p /data/web_data
vim /data/web_data/index.html 
	Hello World!	

service httpd restart
firewall-cmd --zone=public --add-service=http --permanent
firewall-cmd --reload
```

**mod_ssl**
- 为linux提供web证书
```bash
>cd /etc/pki/CA/private
>openssl genrsa 2048 > cakey.pem 
>openssl req -new -x509 -key cakey.pem > /etc/pki/CA/cacert.pem

cd /etc/pki/CA
touch index.txt  #索引问文件
touch serial    #给客户发证编号存放文件
echo 01 > serial

mkdir /etc/httpd/ssl
cd /etc/httpd/ssl
openssl genrsa 1024 > httpd.key 
openssl req -new -key httpd.key > httpd.csr
openssl ca -days 365 -in httpd.csr > httpd.crt 

使用cat /etc/pki/CA/index.txt查看openssl证书数据库文件
cat /etc/pki/CA/index.txt
```

- 为windows提供web证书
```bash
>cd /etc/pki/CA/private
>openssl genrsa 2048 > cakey.pem 
>openssl req -new -x509 -key cakey.pem > /etc/pki/CA/cacert.pem

cd /etc/pki/CA
touch index.txt  #索引问文件
touch serial    #给客户发证编号存放文件
echo 01 > serial

cd 
openssl genrsa 1024 > httpd.key 
openssl req -new -key httpd.key > httpd.csr
openssl ca -days 365 -in httpd.csr > httpd.crt 

openssl pkcs12 -export -out server.pfx -inkey httpd.key -in httpd.crt
自己想办法把server.pfx导出给windows2008主机
```

- 向 windows CA 服务器申请证书
`Openssl genrsa 2048 > httpd.key`
`openssl req -new -key httpd.key -out httpd.csr`
通过这个csr文件在内部的windows CA服务器上申请证书

**ab**
安装
`sudo apt install apache2-utils`
`yum install httpd-tools`

---

## [Rpm](https://rpm.org/)&[Node✔](https://nodejs.org)
**包管理器方式**
`apt-get install nodejs npm`	讲道理apt不好用

`yum install epel-release`
`yum install nodejs npm`

**源文件方式安装**
首先下载NodeJS的二进制文件,http://nodejs.org/download/ 。在 Linux Binaries (.tar.gz)行处根据自己系统的位数选择
```bash
#解压到当前文件夹下运行
tar zxvf node-v0.10.26-linux-x64.tar.gz

进入解压后的目录bin目录下,执行ls会看到两个文件node,npm. 然后执行./node -v ,如果显示出 版本号说明我们下载的程序包是没有问题的。 依次运行如下三条命令
cd node-v0.10.26-linux-x64/bin
ls
./node -v
```
因为 /home/kun/mysofltware/node-v0.10.26-linux-x64/bin这个目录是不在环境变量中的,所以只能到该目录下才能node的程序。如果在其他的目录下执行node命令的话 ,必须通过绝对路径访问才可以的

如果要在任意目录可以访问的话,需要将node 所在的目录,添加PATH环境变量里面,或者通过软连接的形式将node和npm链接到系统默认的PATH目录下的一个
在终端执行echo $PATH可以获取PATH变量包含的内容,系统默认的PATH环境变量包括/usr/local/bin:/usr/bin:/bin:/usr/local/sbin:/usr/sbin:/sbin: ,冒号为分隔符。所以我们可以将node和npm链接到/usr/local/bin 目录下如下执行

```bash
ln -s /home/kun/mysofltware/node-v0.10.26-linux-x64/bin/node /usr/local/bin/node
ln -s /home/kun/mysofltware/node-v0.10.26-linux-x64/bin/npm /usr/local/bin/npm
```

---

## [PHP](https://www.php.net/)
```bash
若之前安装过其他版本PHP,先删除
yum remove php*

rpm安装PHP7相应的yum源
rpm -Uvh https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm

yum install php70w

php -v

service php-fpm start #要运行PHP网页,要启动php-fpm解释器
```

---

## [Nginx](https://nginx.org/)
**安装**
```bash
yum install nginx
systemctl start nginx.service
firewall-cmd --permanent --zone=public --add-service=http 
firewall-cmd --reload
```

**虚拟主机**
在/etc/nginx/conf.d/目录下新建一个站点的配置文件,列如：test.com.conf
```vim
vim /etc/nginx/conf.d/test.com.conf
  server {
          listen 80;
          server_name www.test.com test.com;
          root /usr/share/nginx/test.com;
          index index.html;

          location / {
          }
  }

nginx -t  #检测文件是否有误  
```

```bash
mkdir /usr/share/nginx/test.com
echo "hello world!" > /usr/share/nginx/test.com/index.html
firewall-cmd --permanent --zone=public --add-service=http 
firewall-cmd --reload
systemctl start nginx.service
```

如果服务器网址没有注册,那么应该在本机电脑的/etc/hosts添加设置：
`192.168.1.112   www.test.com test.com`
`curl www.test.com`

**https**
```vim
openssl req -new -x509 -nodes -days 365 -newkey rsa:1024  -out httpd.crt -keyout httpd.key #生成自签名证书,信息不要瞎填,Common Name一定要输你的网址

mv httpd.crt /etc/nginx
mv httpd.key /etc/nginx

vim /etc/nginx/conf.d/test.com.conf
  server {
          listen       443 ssl http2;
          server_name  www.test.com test.com;
          root         /usr/share/nginx/test.com;
          index index.html;

          ssl_certificate "/etc/nginx/httpd.crt";
          ssl_certificate_key "/etc/nginx/httpd.key";
          location / {
          }

          error_page 404 /404.html;
              location = /40x.html {
          }

          error_page 500 502 503 504 /50x.html;
              location = /50x.html {
          }
      }

systemctl restart nginx
```

**添加PHP/PHP-FPM环境支持**
```vim
# 安装PHP源
rpm -ivh https://mirror.webtatic.com/yum/el7/epel-release.rpm
rpm -ivh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm

# 安装 PHP7.0
yum install php70w php70w-fpm php70w-mysql php70w-mysqlnd

systemctl start php-fpm.service
netstat -tnlp #检查php-fpm默认监听端口：9000

添加配置
vim /etc/nginx/conf.d/test.com.conf
          # php-fpm  (新增)
          location ~\.php$ {
                  fastcgi_pass 127.0.0.1:9000;
                  fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                  fastcgi_param PATH_INFO $fastcgi_script_name;
                  include fastcgi_params;
          }
  }

systemctl restart nginx
systemctl restart php-fpm

测试
vim /usr/share/nginx/test.com/info.php
  <?php 
      phpinfo(); 
  ?>

curl http://www.test.com/info.php
```

---

## [phpMyAdmin](https://www.phpmyadmin.net/)
**建议搭配上面的nginx+php扩展**

**创建数据库和一个用户**
```bash
yum install mariadb mariadb-server
systemctl start mariadb
systemctl enable mariadb
mysql_secure_installation

mysql -u root -p

创建一个专给WordPress存数据的数据库
MariaDB [(none)]> create database idiota_info;  ##最后的"idiota_info"为数据库名

创建用于WordPress对应用户
MariaDB [(none)]> create user idiota@localhost identified by 'password';   ##"idiota"对应创建的用户,"password"内填写用户的密码

分别配置本地登录和远程登录权限
MariaDB [(none)]> grant all privileges on idiota_info.* to idiota@'localhost' identified by 'password';
MariaDB [(none)]> grant all privileges on idiota_info.* to idiota@'%' identified by 'password';

刷新权限
MariaDB [(none)]> flush privileges;
```

**下载**
```bash
wget https://files.phpmyadmin.net/phpMyAdmin/4.8.5/phpMyAdmin-4.8.5-all-languages.zip
unzip phpMyAdmin-4.8.5-all-languages.zip
mv phpMyAdmin-4.8.5-all-languages phpMyAdmin
cp phpMyAdmin /usr/share/nginx/test.com/
cd /usr/share/nginx/test.com/phpMyAdmin

cp config.sample.inc.php config.inc.php

systemctl restart nginx
```

访问 `https://www.test.com/phpMyAdmin/index.php`

---

## [Caddy](https://caddyserver.com/)
- 安装Caddy
```bash
curl https://getcaddy.com | bash -s personal
或
wget -N --no-check-certificate https://raw.githubusercontent.com/ToyoDAdoubiBackup/doubi/master/caddy_install.sh && chmod +x caddy_install.sh && bash caddy_install.sh
或
https://caddyserver.com/download 进入到 caddy 官网的下载界面,选择平台和插件
下载后用 cp 命令放到 /usr/local/bin/caddy ,解压
```

- 运行
`caddy`然后打开浏览器输入： http://ip:2015 ,得到了一个404页面,Caddy 已经成功运行了

在无配置文件的情况下,Caddy 默认是映射当前程序执行的目录所有文件(即/usr/local/bin),因此可以创建一个文件
`echo "<h1>Hello Caddy</h1>" >> index.html`

`caddy -port 80`改为运行在80端口

- 配置文件
```bash
chown -R root:www-data /usr/local/bin     #设置目录数据权限
vim /usr/local/bin/Caddyfile	#注.一般来说caddy路径都是这个,个别安装脚本可能有不同路径

echo -e ":80 {
	gzip	
	root /usr/local/bin/www
}" > /usr/local/bin/Caddyfile

echo "<h1>first</h1>" >> /usr/local/bin/www/index.html

caddy
# 如果启动失败可以看Caddy日志： tail -f /tmp/caddy.log
```

- 反向代理
做一个ip跳转
```bash
echo ":80 {
	gzip
	proxy / http://www.baidu.com
}" > /usr/local/bin/Caddyfile

caddy
```

- HTTPS
为已经绑定域名的服务器自动从 Let’s Encrypt 生成和下载 HTTPS 证书,支持 HTTPS 协议访问,你只需要将绑定的 IP 换成 域名 即可
```bash
echo -e "xxx.com {
	gzip
    root /usr/local/bin/www
	tls xxxx@xxx.com  #你的邮箱
}" > /usr/local/bin/Caddyfile
```

---

## [Wordpress](https://wordpress.org/)
**下载WordPress安装包并解压**
```bash
wget https://wordpress.org/latest.tar.gz

tar -xzvf latest.tar.gz 
```

**创建WordPress数据库和一个用户**
```bash
yum install mariadb mariadb-server
systemctl start mariadb
systemctl enable mariadb
mysql_secure_installation

mysql -u root -p

创建一个专给WordPress存数据的数据库
MariaDB [(none)]> create database idiota_info;  ##最后的"idiota_info"为数据库名

创建用于WordPress对应用户
MariaDB [(none)]> create user idiota@localhost identified by 'password';   ##"idiota"对应创建的用户,"password"内填写用户的密码

分别配置本地登录和远程登录权限
MariaDB [(none)]> grant all privileges on idiota_info.* to idiota@'localhost' identified by 'password';
MariaDB [(none)]> grant all privileges on idiota_info.* to idiota@'%' identified by 'password';

刷新权限
MariaDB [(none)]> flush privileges;
```

**配置 PHP**
```bash
# 安装PHP源
rpm -ivh https://mirror.webtatic.com/yum/el7/epel-release.rpm
rpm -ivh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm

# 安装 PHP7.0
yum install php70w
yum install php70w-mysql
yum install httpd

# 重启Apache
systemctl restart httpd

# 查看PHP版本
php -v
```

**设置wp-config.php文件**
```bash
cd wordpress
vim wp-config-sample.php
```

在标有 
> // ** MySQL settings - You can get this info from your web host ** //

下输入你的数据库相关信息
```php
DB_NAME 
    在第二步中为WordPress创建的数据库名称
DB_USER 
    在第二步中创建的WordPress用户名
DB_PASSWORD 
    第二步中为WordPress用户名设定的密码
DB_HOST 
    第二步中设定的hostname（通常是localhost,但总有例外；参见编辑wp-config.php文件中的"可能的DB_HOST值）。
DB_CHARSET 
    数据库字符串,通常不可更改（参见zh-cn:编辑wp-config.php）。
DB_COLLATE 
    留为空白的数据库排序（参见zh-cn:编辑wp-config.php）。
```

在标有
>* Authentication Unique Keys.

的版块下输入密钥的值,保存wp-config.php文件,也可以不管这个

**上传文件**
接下来需要决定将博客放在网站的什么位置上：
    网站根目录下（如：http://example.com/）
    网站子目录下（如：http://example.com/blog/

根目录
如果需要将文件上传到web服务器,可用FTP客户端将wordpress目录下所有内容（无需上传目录本身）上传至网站根目录
如果文件已经在web服务器中且希望通过shell访问来安装wordpress,可将wordpress目录下所有内容（无需转移目录本身）转移到网站根目录

子目录
如果需要将文件上传到web服务器,需将wordpress目录重命名,之后用FTP客户端将重命名后的目录上传到网站根目录下某一位置
如果文件已经在web服务器中且希望通过shell访问来安装wordpress,可将wordpress目录转移到网站根目录下某一位置,之后重命名 wordpress目录

```bash
mv wordpress/* /var/www/html

setenforce 0
service httpd start
service firewalld stop
```

**运行安装脚本**
在常用的web浏览器中运行安装脚本。
将WordPress文件放在根目录下的用户请访问：http://example.com/wp-admin/install.php
将WordPress文件放在子目录（假设子目录名为blog）下的用户请访问：http://example.com/blog/wp-admin/install.php

访问`http://xxx.xxx.xxx.xxx/wp-admin/setup-config.php`
下面就略了,自己照着页面上显示的来

---

## [Mijisou](https://mijisou.com/)
基于开源项目 Searx 二次开发的操作引擎
项目地址:https://github.com/entropage/mijisou

**依赖**
自行安装python3 pip redis

**安装**
```bash
systemctl start redis
systemctl enable redis
git clone https://github.com/entropage/mijisou.git
cd mijisou && pip install -r requirements.txt
```

**配置**
```yml
vim searx/settings_et_dev.yml
general:
    debug : False # Debug mode, only for development
    instance_name : "123搜索" # displayed name

search:
    safe_search : 0 # Filter results. 0: None, 1: Moderate, 2: Strict
    autocomplete : "" # Existing autocomplete backends: "baidu", "dbpedia", "duckduckgo", "google", "startpage", "wikipedia" - leave blank to turn it off by default
    language : "zh-CN"
    ban_time_on_fail : 5 # ban time in seconds after engine errors
    max_ban_time_on_fail : 120 # max ban time in seconds after engine errors

server:
    port : 8888
    bind_address : "0.0.0.0" # address to listen on
    secret_key : "123" # change this!
    base_url : False # Set custom base_url. Possible values: False or "https://your.custom.host/location/"
    image_proxy : False # Proxying image results through searx
    http_protocol_version : "1.0"  # 1.0 and 1.1 are supported

cache:
    cache_server : "127.0.0.1" # redis cache server ip address
    cache_port : 6379 # redis cache server port
    cache_time : 86400 # cache 1 day
    cache_type : "redis" # cache type
    cache_db : 0 # we use db 0 in dev env

ui:
    static_path : "" # Custom static path - leave it blank if you didn't change
    templates_path : "" # Custom templates path - leave it blank if you didn't change
    default_theme : entropage # ui theme
    default_locale : "" # Default interface locale - leave blank to detect from browser information or use codes from the 'locales' config section
    theme_args :
        oscar_style : logicodev # default style of oscar

# searx supports result proxification using an external service: https://github.com/asciimoo/morty
# uncomment below section if you have running morty proxy
result_proxy:
    url : ""  #morty proxy service
    key : Your_result_proxy_key
    server_name : ""

outgoing: # communication with search engines
    request_timeout : 2.0 # seconds
    useragent_suffix : "" # suffix of searx_useragent, could contain informations like an email address to the administrator
    pool_connections : 100 # Number of different hosts
    pool_maxsize : 10 # Number of simultaneous requests by host
# uncomment below section if you want to use a proxy
# see http://docs.python-requests.org/en/latest/user/advanced/#proxies
# SOCKS proxies are also supported: see http://docs.python-requests.org/en/master/user/advanced/#socks
#    proxies :
#        http : http://192.168.199.5:24000
#        http : http://192.168.199.5:3128
#        https: http://127.0.0.1:8080
# uncomment below section only if you have more than one network interface
# which can be the source of outgoing search requests
#    source_ips:
#        - 1.1.1.1
#        - 1.1.1.2
    haipproxy_redis:
      #host: 192.168.199.5
      #port: 6379
      #password: kckdkkdkdkddk
      #db: 0

engines:
  - name : google
    engine : google
    shortcut : go

  - name : google images
    engine : google_images
    shortcut : goi

  - name : baidu
    engine : baidu
    shortcut : bd
  
  - name : baidu images
    engine : baidu_images
    shortcut : bdi

  - name : baidu videos
    engine : baidu_videos
    shortcut : bdv

  - name : sogou images
    engine : sogou_images
    shortcut : sgi

  - name : sogou videos
    engine : sogou_videos
    shortcut : sgv

  - name : 360 images
    engine : so_images
    shortcut : 360i

  - name : bing
    engine : bing
    shortcut : bi

  - name : bing images
    engine : bing_images
    shortcut : bii

  - name : bing videos
    engine : bing_videos
    shortcut : biv

  - name : bitbucket
    engine : xpath
    paging : True
    search_url : https://bitbucket.org/repo/all/{pageno}?name={query}
    url_xpath : //article[@class="repo-summary"]//a[@class="repo-link"]/@href
    title_xpath : //article[@class="repo-summary"]//a[@class="repo-link"]
    content_xpath : //article[@class="repo-summary"]/p
    categories : it
    timeout : 4.0
    shortcut : bb

  - name : free software directory
    engine : mediawiki
    shortcut : fsd
    categories : it
    base_url : https://directory.fsf.org/
    number_of_results : 5
# what part of a page matches the query string: title, text, nearmatch
# title - query matches title, text - query matches the text of page, nearmatch - nearmatch in title
    search_type : title
    timeout : 5.0

  - name : gentoo
    engine : gentoo
    shortcut : ge

  - name : gitlab
    engine : json_engine
    paging : True
    search_url : https://gitlab.com/api/v4/projects?search={query}&page={pageno}
    url_query : web_url
    title_query : name_with_namespace
    content_query : description
    page_size : 20
    categories : it
    shortcut : gl
    timeout : 10.0

  - name : github
    engine : github
    shortcut : gh

  - name : stackoverflow
    engine : stackoverflow
    shortcut : st

  - name : wikipedia
    engine : wikipedia
    shortcut : wp
    base_url : 'https://en.wikipedia.org/'

locales:
    en : English
    ar : العَرَبِيَّة (Arabic)
    bg : Български (Bulgarian)
    cs : Čeština (Czech)
    da : Dansk (Danish)
    de : Deutsch (German)
    el_GR : Ελληνικά (Greek_Greece)
    eo : Esperanto (Esperanto)
    es : Español (Spanish)
    fi : Suomi (Finnish)
    fil : Wikang Filipino (Filipino)
    fr : Français (French)
    he : עברית (Hebrew)
    hr : Hrvatski (Croatian)
    hu : Magyar (Hungarian)
    it : Italiano (Italian)
    ja : 日本語 (Japanese)
    nl : Nederlands (Dutch)
    pl : Polski (Polish)
    pt : Português (Portuguese)
    pt_BR : Português (Portuguese_Brazil)
    ro : Română (Romanian)
    ru : Русский (Russian)
    sk : Slovenčina (Slovak)
    sl : Slovenski (Slovene)
    sr : српски (Serbian)
    sv : Svenska (Swedish)
    tr : Türkçe (Turkish)
    uk : українська мова (Ukrainian)
    zh : 简体中文 (Chinese, Simplified)
    zh_TW : 繁體中文 (Chinese, Traditional)

doi_resolvers :
  oadoi.org : 'https://oadoi.org/'
  doi.org : 'https://doi.org/'
  doai.io  : 'http://doai.io/'
  sci-hub.tw : 'http://sci-hub.tw/'

default_doi_resolver : 'oadoi.org'

sentry:
  dsn: https://xkdkkdkdkdkdkdkdk@sentry.xxx.com/2
```

**运行+caddy反代**
```bash
mv searx/settings_et_dev.yml searx/settings.yml
gunicorn searx.webapp:app -b 127.0.0.1:8888 -D	#一定要在mijisou目录下运行

wget -N --no-check-certificate https://raw.githubusercontent.com/ToyoDAdoubiBackup/doubi/master/caddy_install.sh && chmod +x caddy_install.sh && bash caddy_install.sh

echo "www.你的域名.net {
 gzip
 tls xxxx@xxx.com
 proxy / 127.0.0.1:8888
}" >> /usr/local/caddy/Caddyfile

/etc/init.d/caddy start
# 如果启动失败可以看Caddy日志： tail -f /tmp/caddy.log
```

**opensearch**
```xml
vim /root/mijisou/searx/templates/__common__/opensearch.xml
  <?xml version="1.0" encoding="utf-8"?>
  <OpenSearchDescription xmlns="http://a9.com/-/spec/opensearch/1.1/">
    <ShortName>{{ instance_name }}</ShortName>
    <Description>a privacy-respecting, hackable metasearch engine</Description>
    <InputEncoding>UTF-8</InputEncoding>
    <Image>{{ urljoin(host, url_for('static', filename='img/favicon.png')) }}</Image>
    <LongName>searx metasearch</LongName>
    {% if opensearch_method == 'get' %}
      <Url type="text/html" method="get" template="https://www.你的域名.net/?q={searchTerms}"/>
      {% if autocomplete %}
      <Url type="application/x-suggestions+json" method="get" template="{{ host }}autocompleter">
          <Param name="format" value="x-suggestions" />
          <Param name="q" value="{searchTerms}" />
      </Url>
      {% endif %}
    {% else %}
      <Url type="text/html" method="post" template="{{ host }}">
          <Param name="q" value="{searchTerms}" />
      </Url>
      {% if autocomplete %}
      <!-- TODO, POST REQUEST doesn't work -->
      <Url type="application/x-suggestions+json" method="get" template="{{ host }}autocompleter">
          <Param name="format" value="x-suggestions" />
          <Param name="q" value="{searchTerms}" />
      </Url>
      {% endif %}
    {% endif %}
  </OpenSearchDescription>
```

**管理**
```bash
ps -aux
看一下哪个是gunicorn进程
kill 杀掉
gunicorn searx.webapp:app -b 127.0.0.1:8888 -D
```

**后话**
`秘迹®️是熵加网络科技（北京）有限公司所持有的注册商标,任何组织或个人在使用代码前请去除任何和秘迹相关字段,去除秘迹搜索的UI设计,否则熵加网络科技（北京）有限公司保留追究法律责任的权利。`
配置文件中改下名字
`mijisou/searx/static/themes/entropage/img`中的logo图标自己换一下

**Thank**
- [asciimoo/searx](https://github.com/asciimoo/searx)
- [entropage/mijisou: Privacy-respecting metasearch engine](https://github.com/entropage/mijisou)
- [一个可以保护个人隐私的网络搜索服务：秘迹搜索搭建教程 - Rat's Blog](https://www.moerats.com/archives/922/)
- [OpenSearch description format | MDN](https://developer.mozilla.org/en-US/docs/Web/OpenSearch)
- [Add or remove a search engine in Firefox | Firefox Help](https://support.mozilla.org/en-US/kb/add-or-remove-search-engine-firefox)

---

## Haproxy
原来写的太烂了，准备重写

---

# 数据库
## [Mariadb](https://mariadb.org/)
**安装**
>yum install mariadb mariadb-server

**数据库初始化**
>systemctl start mariadb
>mysql_secure_installation

|配置流程 	|说明 |操作
------------ | ------------- | ------------
Enter current password for root (enter for none) |	输入 root 密码 	| 初次运行直接回车
Set root password? [Y/n] |	是设置 root 密码 |	可以 y 或者 回车
New password |	输入新密码 	
Re-enter new password |	再次输入新密码
Remove anonymous users? [Y/n] |	是否删除匿名用户 | 可以 y 或者回车 本题y
Disallow root login remotely? [Y/n]  |	是否禁止 root 远程登录 |  可以 y 或者回车 本题n
Remove test database and access to it? [Y/n]  |	是否删除 test 数据库 | y 或者回车 本题y
Reload privilege tables now? [Y/n] | 是否重新加载权限表 | y 或者回车 本题y

**配置远程访问**
Mariadb数据库授权root用户能够远程访问
```sql
systemctl start mariadb
mysql -u root -p <password>
select User, host from mysql.user;
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%'IDENTIFIED BY 'toor' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```

```bash
firewall-cmd --permanent --add-service=mysql
firewall-cmd --reload

systemctl enable mariadb
```

---

## [MongoDB🍃](https://www.mongodb.com/)
**安装**
```vim
vim /etc/yum.repos.d/mongodb-org-4.0.repo
	[mongodb-org-4.0]
	name=MongoDB Repository
	baseurl=https://repo.mongodb.org/yum/redhat/$releasever/mongodb-org/4.0/x86_64/
	gpgcheck=1
	enabled=1
	gpgkey=https://www.mongodb.org/static/pgp/server-4.0.asc

yum install -y mongodb-org
```

**配置远程访问**
```vim
vim /etc/mongod.conf
	# Listen to all ip address
	bind_ip = 0.0.0.0
```

`service mongod start`

**创建管理员用户**
```sql
mongo
>use admin
 db.createUser(
  {
    user: "myUserAdmin",
    pwd: "abc123",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" }, "readWriteAnyDatabase" ]
  }
 )

> show dbs;	# 查看数据库
> db.version();	# 查看数据库版本
```

**启用权限管理**
```vim
vim /etc/mongod.conf
	#security 
	security:
	authorization: enabled

service mongod restart	
```

---

## [MySQL📦](https://www.mysql.com)
和 Mariadb 差不多,看 Mariadb 的就行了
```bash
sudo apt install mysql-server mysql-clien
sudo service mysql start
```

---

## [Oracle](https://www.oracle.com/technetwork/database/enterprise-edition/downloads/index.html)
鸽

---
## [Postgresql🐘](https://www.postgresql.org)
**安装**
```bash
yum install postgresql-server
postgresql-setup initdb #初始化数据库
service postgresql start #启动服务
```

PostgreSQL 安装完成后,会建立一下‘postgres’用户,用于执行PostgreSQL,数据库中也会建立一个'postgres'用户,默认密码为自动生成,需要在系统中改一下。

**修改用户密码**
```sql
 sudo -u postgres psql postgres
\l #查看当前的数据库列表  
\password postgres  #给postgres用户设置密码
\q  #退出数据库
```

**开启远程访问**
```vim
vim /var/lib/pgsql/data/postgresql.conf
	listen_addresses='*'

vim /var/lib/pgsql/data/pg_hba.conf
  # IPv4 local connections:
  host    all             all             127.0.0.1/32            md5
  host    all             all             0.0.0.0/0               md5       

其中0.0.0.0/0表示运行任意ip地址访问。
若设置为 192.168.1.0/24 则表示允许来自ip为192.168.1.0 ~ 192.168.1.255之间的访问。

service postgresql restart
防火墙记得放行
```

---

## [Redis🔺🔴⭐](https://redis.io/)
**安装**
- **包管理器方式**
  在CentOS和Red Hat系统中,首先添加EPEL仓库,然后更新yum源:
  `yum install epel-release`
  `yum install redis`
  安装好后启动Redis服务即可
  `systemctl start redis`

- **源代码编译方式安装**
  在官网下载tar.gz的安装包,或者通过wget的方式下载　　
  `wget http://download.redis.io/releases/redis-4.0.1.tar.gz`

  安装
  ```bash
  tar -zxvf redis-4.0.1.tar.gz
  cd redis-4.0.1
  make
  make test
  make install
  ```
  ```bash
  ./usr/local/bin/redis-server
  ctrl+z
  bg
  redis-cli
  ```
  
使用redis-cli进入Redis命令行模式操作
```bash
redis-cli
127.0.0.1:6379> ping
PONG
127.0.0.1:6379> exit
```

**开启远程访问**
为了可以使Redis能被远程连接,需要修改配置文件,路径为/etc/redis.conf
```vim
vim /etc/redis.conf
	#bind 127.0.0.1
	requirepass 密码	#设置redis密码

service redis restart
当然还要记得开防火墙  

redis-cli -h <ip> -p 6379 -a <PASSWORD>
```

---

## [Memcached](https://memcached.org/)
**安装**
- **软件包安装**
  ```bash
  yum -y install memcached
  cat /etc/sysconfig/memcached
  ```

- **源代码编译方式安装**
  在官网下载tar.gz的安装包,或者通过wget的方式下载　　
  `wget http://memcached.org/latest`

  安装
  ```bash
  tar -zxvf memcached-1.x.x.tar.gz
  cd memcached-1.x.x
  ./configure --prefix=/usr/local/memcached
  make && make test
  make install
  ```

**运行**
```bash
systemctl start memcached 
systemctl enable memcached

firewall-cmd --add-port=11211/tcp --permanent
firewall-cmd --reload
```

---

# 文件服务
## [Vsftp](https://security.appspot.com/vsftpd.html)
**匿名访问**
|参数|作用|
| :------------- | :------------- |
|anonymous_enable=YES |	允许匿名访问模式 |
|anon_umask=022 |	匿名用户上传文件的umask值|
|anon_upload_enable=YES |	允许匿名用户上传文件|
|anon_mkdir_write_enable=YES |	允许匿名用户创建目录|
|anon_other_write_enable=YES |	允许匿名用户修改目录名称或删除目录|

```vim
vim /etc/vsftpd/vsftpd.conf
1 anonymous_enable=YES
2 anon_umask=022
3 anon_upload_enable=YES
4 anon_mkdir_write_enable=YES
5 anon_other_write_enable=YES
```
```bash
setenforce 0
firewall-cmd --permanent --zone=public --add-service=ftp
firewall-cmd --reload
systemctl restart vsftpd
systemctl enable vsftpd
```

现在就可以在客户端执行ftp命令连接到远程的 FTP 服务器了。
在 vsftpd 服务程序的匿名开放认证模式下,其账户统一为 anonymous,密码为空。而且在连接到 FTP 服务器后,默认访问的是 /var/ftp 目录。
我们可以切换到该目录下的 pub 目录中,然后尝试创建一个新的目录文件,以检验是否拥有写入权限：
```bash
[root@linuxprobe ~]# ftp 192.168.10.10
Connected to 192.168.10.10 (192.168.10.10).
220 (vsFTPd 3.0.2)
Name (192.168.10.10:root): anonymous
331 Please specify the password.
Password:此处敲击回车即可
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.

ftp> cd pub
250 Directory successfully changed.

ftp> mkdir files
257 "/pub/files" created

ftp> rename files database
350 Ready for RNTO.
250 Rename successful.

ftp> rmdir database
250 Remove directory operation successful.

ftp> exit
221 Goodbye.
```

---

**本地用户**
|参数 |	作用|
| :------------- | :------------- |
|anonymous_enable=NO 	|禁止匿名访问模式|
|local_enable=YES |	允许本地用户模式|
|write_enable=YES |	设置可写权限|
|local_umask=022 |	本地用户模式创建文件的umask值|
|userlist_deny=YES 	|启用"禁止用户名单",名单文件为ftpusers和user_list|
|userlist_enable=YES |	开启用户作用名单文件功能|

```vim
vim /etc/vsftpd/vsftpd.conf
1 anonymous_enable=NO
2 local_enable=YES
3 write_enable=YES
4 local_umask=022
```
```bash
setenforce 0
firewall-cmd --permanent --zone=public --add-service=ftp
firewall-cmd --reload
systemctl restart vsftpd
systemctl enable vsftpd
```
按理来讲,现在已经完全可以本地用户的身份登录FTP服务器了。但是在使用root管理员登录后,系统提示如下的错误信息：
```bash
[root@linuxprobe ~]# ftp 192.168.10.10
Connected to 192.168.10.10 (192.168.10.10).
220 (vsFTPd 3.0.2)
Name (192.168.10.10:root): root
530 Permission denied.
Login failed.
ftp>
```
可见,在我们输入root管理员的密码之前,就已经被系统拒绝访问了。这是因为vsftpd服务程序所在的目录中默认存放着两个名为"用户名单"的文件（ftpusers和user_list）。只要里面写有某位用户的名字,就不再允许这位用户登录到FTP服务器上。
```bash
[root@linuxprobe ~]# cat /etc/vsftpd/user_list 

[root@linuxprobe ~]# cat /etc/vsftpd/ftpusers 
```
如果你确认在生产环境中使用 root 管理员不会对系统安全产生影响,只需按照上面的提示删除掉 root 用户名即可。我们也可以选择 ftpusers 和 user_list 文件中没有的一个普通用户尝试登录FTP服务器
在采用本地用户模式登录FTP服务器后,默认访问的是该用户的家目录,也就是说,访问的是/home/username目录。而且该目录的默认所有者、所属组都是该用户自己,因此不存在写入权限不足的情况。

---

**虚拟用户**
安装
`yum install vsftpd`

认证
创建虚拟用户文件,把这些用户名和密码存放在一个文件中。该文件内容格式是：用户名占用一行,密码占一行。
```vim
cd /etc/vsftp
vim login.list
	Ftpuser1
	123456
	Ftpuser2
	123456
	Ftpadmin
	123456
```

使用 db_load 命令生成 db 口令login数据库文件
>db_load -T -t hash -f login.list login.db

通过修改指定的配置文件,调整对该程序的认证方式
```vim
vim /etc/vsftpd/vsftpd.conf
	pam_service_name=vsftpd.vu  #设置PAM使用的名称,该名称就是/etc/pam.d/目录下vsfptd文件的文件名

cp /etc/pam.d/vsftpd /etc/pam.d/vsftpd.vu

vim /etc/pam.d/vsftpd.vu
	auth       required     pam_userdb.so db=/etc/vsftpd/login
	account    required     pam_userdb.so db=/etc/vsftpd/login
#注意：格式是db=/etc/vsftpd/login这样的,一定不要去掉源文件的.db后缀
```

配置文件
```vim
vim /etc/vsftpd/vsftpd.conf
1 anonymous_enable=NO
2 local_enable=YES
3 guest_enable=YES
4 guest_username=virtual
5 pam_service_name=vsftpd.vu
6 allow_writeable_chroot=YES
```

|参数 |	作用|
| :------------- | :------------- |
|anonymous_enable=NO 	|禁止匿名开放模式|
|local_enable=YES |	允许本地用户模式|
|guest_enable=YES |	开启虚拟用户模式|
|guest_username=virtual |	指定虚拟用户账户|
|pam_service_name=vsftpd.vu |	指定PAM文件|
|allow_writeable_chroot=YES |	允许对禁锢的FTP根目录执行写入操作,而且不拒绝用户的登录请求|

用户配置权限文件
所有用户主目录为 /home/ftp 宿主为 virtual 用户；
>useradd -d /home/ftp -s /sbin/nologin virtual  
>chmod -Rf 755 /home/ftp/
>cd /home/ftp/
>touch testfile

```vim
vim /etc/vsftpd/vsftpd.conf  
	guest_enable=YES      #表示是否开启vsftpd虚拟用户的功能,yes表示开启,no表示不开启。
	guest_username=virtual       # 指定虚拟用户的宿主用户  
	user_config_dir=/etc/vsftpd/user_conf     # 设定虚拟用户个人vsftpd服务文件存放路径
	allow_writeable_chroot=YES
```

编辑用户权限配置文件
```vim
vim Ftpadmin
	anon_upload_enable=YES
	anon_mkdir_wirte_enable=YES
	anon_other_wirte_enable=YES
	anon_umask=022
	虚拟用户具有写权限（上传、下载、删除、重命名）

	#umask = 022 时,新建的目录 权限是755,文件的权限是 644
	#umask = 077 时,新建的目录 权限是700,文件的权限时 600
	#vsftpd的local_umask和anon_umask借鉴了它
	#默认情况下vsftp上传之后文件的权限是600,目录权限是700
	#想要修改上传之后文件的权限,有两种情况
	#如果使用vsftp的是本地用户
	#则要修改配置文件中的 local_umask 的值
	#如果使用vsftp的是虚拟用户
	#则要修改配置文件中的 anon_umask 的值
```

启服务
```bash
setenforce 0
firewall-cmd --zone=public --add-service=ftp
firewall-cmd --reload
systemctl restart vsftpd
systemctl enable vsftpd
```

---

## [Samba](https://www.samba.org)
**服务端**
安装
>yum install samba 

修改配置文件
```vim	
vim /etc/samba/smb.conf
[smbshare]
	path = /smbshare	#共享目录
	public = yes
	writeable=yes
	hosts allow = 192.168.1xx.33/32	#允许主机
	hosts deny = all
	create mask = 0770	#创建文件的权限为0770；
```

验证配置文件有没有错误
>	testparm

添加用户,设置密码
>	useradd smb1
>	smbpasswd ‐a smb1(密码：smb123456)

将用户添加到 samba 服务器中,并设置密码
>	pdbedit ‐a smb1(密码：smb123456)

查看 samba 数据库用户
>	pdbedit ‐L

创建共享目录,设置所有者和所属组
>	mkdir /smbshare
>	chown smb1:smb1 /smbshare

关闭 selinux（需要重启）
```vim
vim /etc/selinux/config
SELINUX=disabled

firewall-cmd --zone=public --add-service=samba --permanent
firewall-cmd --reload

systemctl restart smb
```

**客户端**
```bash
yum install samba 

mkdir /data/web_data
mount -t cifs -o username=smb1,password='smb123456' //192.168.xx+1.xx/webdata 
/data/web_data
```

---

## NFS
**服务端**
安装
```bash
yum ‐y install nfs‐utils
```

修改配置文件
```bash
vim /etc/exports
	/public 192.168.xxx.xxx(ro)
```

启服务
```bash
mkdir /public

vi /etc/selinux/config
	SELINUX=disabled
		
firewall-cmd --zone=public --add-service=rpc-bind --permanent
firewall-cmd --zone=public --add-service=mountd --permanent
firewall-cmd --zone=public --add-port=2049/tcp --permanent
firewall-cmd --zone=public --add-port=2049/udp --permanent
firewall-cmd --reload 

service rpcbind start
service nfs start	
```

**客户端**
安装,创建用户
```bash
yum ‐y install nfs‐utils
mkdir /mnt/nfsfiles

useradd nfsuser1
passwd nfsuser1
```

验证共享是否成功
>showmount ‐e 192.168.xxx.xxx

挂载共享目录
```vim
vim /etc/fstab
	192.168.xxx.xxx:/public /mnt/nfsfiles/	nfs defaults 0 0
```

>su ‐l nfsuser1

**验证**
服务器
```bash
[root@localhost ~]# cd /public/
[root@localhost public]# echo "hello" > hello.txt
```
客户端
```bash
[nfsuser1@localhost ~]$ cd /mnt/nfsfiles/
[nfsuser1@localhost nfsfiles]$ cat hello.txt
```

---

# 编程语言
## C
```vim
vim world.c
	#include <stdio.h>
	int main(void){
					printf("Hello World");
					return 0;
	}

gcc helloworld.c -o execFile
./execFlie
```

---

## [Go🐹](https://golang.org/)
**源文件方式安装**
```bash
wget -c https://storage.googleapis.com/golang/go1.8.3.linux-amd64.tar.gz
tar -C /usr/local/ -zxvf go1.8.3.linux-amd64.tar.gz

PATH=$PATH:/usr/local/go/bin/
source ~/.bash_profile	
go version
```

---
## [JDK☕](https://www.oracle.com/technetwork/java/javase/downloads/)
**rpm 包方式安装**
下载
https://www.oracle.com/technetwork/java/javase/downloads/
```bash
chmod +x jdk-****.rpm
yum localinstall jdk-****.rpm
也可以
rpm -ivh jdk-****.rpm
```

**使用ppa/源方式安装**
1. 添加ppa
`sudo add-apt-repository ppa:webupd8team/java`
`sudo apt-get update`

2. 安装oracle-java-installer
	jdk7
	`sudo apt-get install oracle-java7-installer`

	jdk8
	`sudo apt-get install oracle-java8-installer`

---

## [Python3🐍](https://www.python.org/)
**yum 安装**
```bash
yum install epel-release
或
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
yum -y install python36 python36-devel

ln -s /usr/bin/python3.6 /usr/bin/python3 #配置Python3软链接
wget https://bootstrap.pypa.io/get-pip.py	#安装pip3
python3 get-pip.py
```

**源代码编译方式安装**
安装依赖环境
```bash
yum -y install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel
```

下载Python3
`wget https://www.python.org/ftp/python/3.6.1/Python-3.6.1.tgz`

安装python3
```bash
mkdir -p /usr/local/python3
tar zxvf Python-3.6.1.tgz
cd Python-3.6.1
./configure --prefix=/usr/local/python3
make
make install    或者 make && make install
```

添加到环境变量
```bash
ln -s /usr/local/python3/bin/python3 /usr/bin/python3

vim ~/.bash_profile #永久修改变量
	PATH=$PATH:/usr/local/python3/bin/
source ~/.bash_profile	

```

检查Python3及pip3是否正常可用
```bash
python3 -V
pip3 -V
```
---

## [Ruby💎](https://www.ruby-lang.org)
**安装**
注:在Ubuntu下有点问题,不建议用Ubuntu做运维环境
下载ruby安装包,并进行编译安装
```bash
wget https://cache.ruby-lang.org/pub/ruby/2.6/ruby-2.6.2.tar.gz
tar xvfvz ruby-2.6.2.tar.gz
cd ruby-2.6.2
./configure
make
make install
```

将ruby添加到环境变量,ruby安装在/usr/local/bin/目录下,因此编辑 ~/.bash_profile文件,添加一下内容:
```bash
vim ~/.bash_profile
export PATH=$PATH:/usr/local/bin/

不要忘了生效一下:
source ~/.bash_profile
```

---

# 监控服务
## [Zabbix](https://www.zabbix.com/)
**安装依赖**
```bash
yum install mysql
yum install httpd
yum install php
yum install php-mysqlnd php-gd libjpeg* php-snmp php-ldap php-odbc php-pear php-xml php-xmlrpc php-mbstring php-bcmath php-mhash php-common php-ctype php-xml php-xmlreader php-xmlwriter php-session php-mbstring php-gettext php-ldap php-mysqli --skip-broken
yum install wget telnet net-tools python-paramiko gcc gcc-c++ dejavu-sans-fonts python-setuptools python-devel sendmail mailx net-snmp net-snmp-devel net-snmp-utils freetype-devel libpng-devel perl unbound libtasn1-devel p11-kit-devel OpenIPMI unixODBC
```

**设置 mysql**
```vim
vim /etc/my.cnf
  innodb_file_per_table = 1
  innodb_status_file = 1
  innodb_buffer_pool_size = 6G
  innodb_flush_log_at_trx_commit = 2
  innodb_log_buffer_size = 16M
  innodb_log_file_size = 64M
  innodb_support_xa = 0
  default-storage-engine = innodb
  bulk_insert_buffer_size = 8M
  join_buffer_size = 16M
  max_heap_table_size = 32M
  tmp_table_size = 32M
  max_tmp_tables = 48
  read_buffer_size = 32M
  read_rnd_buffer_size = 16M
  key_buffer_size = 32M
  thread_cache_size = 32
  innodb_thread_concurrency = 8
  innodb_flush_method = O_DIRECT
  innodb_rollback_on_timeout = 1
  query_cache_size = 16M
  query_cache_limit = 16M
  collation_server = utf8_bin
  character_set_server = utf8
```
原则上 innodb_buffer_pool_size 需要设置为主机内存的 80%，如果主机内存不是 8GB，以上参数可依据相应比例进行调整，例如主机内存为 16GB，则 innodb_buffer_pool_size 建议设置为 12GB，innodb_log_buffer_size 建议设置为 32M，innodb_log_file_size 建议设置为 128M，以此类推。请注意innodb_buffer_pool_size的值必须是整数，例如主机内存是4G，那么innodb_buffer_pool_size可以设置为3G，而不能设置为3.2G  
```bash
systemctl enable mysqld && systemctl start mysqld  
grep 'temporary password' /var/log/mysqld.log #获取 MySQL 的 root 初始密码 
mysql_secure_installation #初始化，改下密码
systemctl restart mysqld
mysql -u root -p
  create database zabbix character set utf8;
  exit;
mysql_upgrade -u root -p
mysql -u root -p
  create user zabbix@'%' identified by '{mysql_zabbix_password}';
  grant all privileges on zabbix.* to zabbix@'%';
  flush privileges;
  exit;
```

**安装 zabbix**
```bash
rpm -ivh https://repo.zabbix.com/zabbix/4.2/rhel/7/x86_64/zabbix-release-4.2-1.el7.noarch.rpm
yum -y install zabbix-server-mysql zabbix-web-mysql zabbix-java-gateway zabbix-web
cd /usr/share/doc/zabbix-server-mysql-4.2.1
zcat create.sql.gz | mysql -uroot zabbix -p 
```

- 配置 zabbix 参数
  ```vim
  vim /etc/zabbix/zabbix_server.conf
    DBPassword={mysql_zabbix_password}
    CacheSize=512M
    HistoryCacheSize=128M
    HistoryIndexCacheSize=128M
    TrendCacheSize=128M
    ValueCacheSize=256M
    Timeout=30
  ```
  如果需要监控VMware虚拟机，则还需要设置以下选项参数：
  ```vim
    StartVMwareCollectors=2
    VMwareCacheSize=256M
    VMwareTimeout=300
  ```

**配置 Apache 中的 PHP 参数**
```vim
vim /etc/httpd/conf.d/zabbix.conf
  php_value max_execution_time 600
  php_value memory_limit 256M
  php_value post_max_size 32M
  php_value upload_max_filesize 32M
  php_value max_input_time 600
  php_value always_populate_raw_post_data -1
  date.timezone Asia/Shanghai
```

**配置 PHP 参数**
```vim
vim /etc/php.ini
  php_value post_max_size 32M
  max_execution_time 300
  max_input_time 300
  date.timezone Asia/Shanghai
```

**重启&起服务**
```bash
systemctl stop mysqld && reboot
systemctl start httpd && systemctl start zabbix-server
systemctl stop firewalld
setenforce 0
```
访问`http://{ip地址}/zabbix/setup.php`

**Reference**
- [CentOS 7安装Zabbix 3.4](https://www.centos.bz/2017/11/centos-7%E5%AE%89%E8%A3%85zabbix-3-4/)

---

# 虚拟化
## [Docker🐋](https://www.docker.com)
**centos**
`curl -sSL https://get.docker.com/ | sh`

or

Step 1 — Install Docker
```bash
Install needed packages:
$ sudo yum install -y yum-utils device-mapper-persistent-data lvm2

Configure the docker-ce repo:
$ sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

Install docker-ce:
$ sudo yum install docker-ce

Add your user to the docker group with the following command.
$ sudo usermod -aG docker $(whoami)

Set Docker to start automatically at boot time:
$ sudo systemctl enable docker.service

Finally, start the Docker service:
$ sudo systemctl start docker.service
```

Step 2 — Install Docker Compose
```bash
Install Extra Packages for Enterprise Linux
$ sudo yum install epel-release

Install python-pip
$ sudo yum install -y python-pip

Then install Docker Compose:
$ sudo pip install docker-compose

You will also need to upgrade your Python packages on CentOS 7 to get docker-compose to run successfully:
$ sudo yum upgrade python*

To verify a successful Docker Compose installation, run:
$ docker-compose version

docker login
```

**debian**
```bash
sudo apt update
sudo apt install docker.io
docker login	#讲道理,按官方文档说法并不需要账户并且登录,但实际上还是需要你登陆
```

---

# CI
## [Jenkins🤵🏻](https://jenkins.io/)
注,Jenkins需要jdk环境
**rpm包方式安装**
添加Jenkins源:
```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo http://jenkins-ci.org/redhat/jenkins.repo
sudo rpm --import http://pkg.jenkins-ci.org/redhat/jenkins-ci.org.key
```

使用yum命令安装Jenkins:
`yum install jenkins`

**使用ppa/源方式安装**
```bash
wget -q -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -

sed -i "1ideb https://pkg.jenkins.io/debian binary/" /etc/apt/sources.list

sudo apt-get update
sudo apt-get install jenkins
```

安装后默认服务是启动的,默认是8080端口,在浏览器输入:http://127.0.0.1:8080/即可打开主页

查看密码
`cat /var/lib/jenkins/secrets/initialAdminPassword`

---

# 堡垒机
## [Jumpserver](http://www.jumpserver.org/)
[官方文档](http://docs.jumpserver.org/zh/docs/setup_by_centos.html)写的很详细了,在此我只把重点记录

`注:鉴于国内环境,下面步骤运行中还是会出现docker pull镜像超时的问题,你懂的,不要问我怎么解决`
```bash
echo -e "\033[31m 1. 防火墙 Selinux 设置 \033[0m" \
  && if [ "$(systemctl status firewalld | grep running)" != "" ]; then firewall-cmd --zone=public --add-port=80/tcp --permanent; firewall-cmd --zone=public --add-port=2222/tcp --permanent; firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="172.17.0.0/16" port protocol="tcp" port="8080" accept"; firewall-cmd --reload; fi \
  && if [ "$(getenforce)" != "Disabled" ]; then setsebool -P httpd_can_network_connect 1; fi
```
```bash
echo -e "\033[31m 2. 部署环境 \033[0m" \
  && yum update -y \
  && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
  && yum -y install kde-l10n-Chinese \
  && localedef -c -f UTF-8 -i zh_CN zh_CN.UTF-8 \
  && export LC_ALL=zh_CN.UTF-8 \
  && echo 'LANG="zh_CN.UTF-8"' > /etc/locale.conf \
  && yum -y install wget gcc epel-release git \
  && yum install -y yum-utils device-mapper-persistent-data lvm2 \
  && yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo \
  && yum makecache fast \
  && rpm --import https://mirrors.aliyun.com/docker-ce/linux/centos/gpg \
  && echo -e "[nginx-stable]\nname=nginx stable repo\nbaseurl=http://nginx.org/packages/centos/\$releasever/\$basearch/\ngpgcheck=1\nenabled=1\ngpgkey=https://nginx.org/keys/nginx_signing.key" > /etc/yum.repos.d/nginx.repo \
  && rpm --import https://nginx.org/keys/nginx_signing.key \
  && yum -y install redis mariadb mariadb-devel mariadb-server nginx docker-ce \
  && systemctl enable redis mariadb nginx docker \
  && systemctl start redis mariadb \
  && yum -y install python36 python36-devel \
  && python3.6 -m venv /opt/py3
```
```bash
echo -e "\033[31m 3. 下载组件 \033[0m" \
  && cd /opt \
  && if [ ! -d "/opt/jumpserver" ]; then git clone --depth=1 https://github.com/jumpserver/jumpserver.git; fi \
  && if [ ! -f "/opt/luna.tar.gz" ]; then wget https://demo.jumpserver.org/download/luna/1.4.9/luna.tar.gz; tar xf luna.tar.gz; chown -R root:root luna; fi \
  && yum -y install $(cat /opt/jumpserver/requirements/rpm_requirements.txt) \
  && source /opt/py3/bin/activate \
  && pip install --upgrade pip setuptools -i https://mirrors.aliyun.com/pypi/simple/ \
  && pip install -r /opt/jumpserver/requirements/requirements.txt -i https://mirrors.aliyun.com/pypi/simple/ \
  && curl -sSL https://get.daocloud.io/daotools/set_mirror.sh | sh -s http://f1361db2.m.daocloud.io \
  && systemctl restart docker \
  && docker pull jumpserver/jms_coco:1.4.9 \
  && docker pull jumpserver/jms_guacamole:1.4.9 \
  && rm -rf /etc/nginx/conf.d/default.conf \
  && curl -o /etc/nginx/conf.d/jumpserver.conf https://demo.jumpserver.org/download/nginx/conf.d/jumpserver.conf
```
```bash
echo -e "\033[31m 4. 处理配置文件 \033[0m" \
  && if [ "$DB_PASSWORD" = "" ]; then DB_PASSWORD=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 24`; fi \
  && if [ "$SECRET_KEY" = "" ]; then SECRET_KEY=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 50`; echo "SECRET_KEY=$SECRET_KEY" >> ~/.bashrc; fi \
  && if [ "$BOOTSTRAP_TOKEN" = "" ]; then BOOTSTRAP_TOKEN=`cat /dev/urandom | tr -dc A-Za-z0-9 | head -c 16`; echo "BOOTSTRAP_TOKEN=$BOOTSTRAP_TOKEN" >> ~/.bashrc; fi \
  && if [ "$Server_IP" = "" ]; then Server_IP=`ip addr | grep inet | egrep -v '(127.0.0.1|inet6|docker)' | awk '{print $2}' | tr -d "addr:" | head -n 1 | cut -d / -f1`; fi \
  && if [ ! -d "/var/lib/mysql/jumpserver" ]; then mysql -uroot -e "create database jumpserver default charset 'utf8';grant all on jumpserver.* to 'jumpserver'@'127.0.0.1' identified by '$DB_PASSWORD';flush privileges;"; fi \
  && if [ ! -f "/opt/jumpserver/config.yml" ]; then cp /opt/jumpserver/config_example.yml /opt/jumpserver/config.yml; sed -i "s/SECRET_KEY:/SECRET_KEY: $SECRET_KEY/g" /opt/jumpserver/config.yml; sed -i "s/BOOTSTRAP_TOKEN:/BOOTSTRAP_TOKEN: $BOOTSTRAP_TOKEN/g" /opt/jumpserver/config.yml; sed -i "s/# DEBUG: true/DEBUG: false/g" /opt/jumpserver/config.yml; sed -i "s/# LOG_LEVEL: DEBUG/LOG_LEVEL: ERROR/g" /opt/jumpserver/config.yml; sed -i "s/# SESSION_EXPIRE_AT_BROWSER_CLOSE: false/SESSION_EXPIRE_AT_BROWSER_CLOSE: true/g" /opt/jumpserver/config.yml; sed -i "s/DB_PASSWORD: /DB_PASSWORD: $DB_PASSWORD/g" /opt/jumpserver/config.yml; fi
```
```bash
echo -e "\033[31m 5. 启动 Jumpserver \033[0m" \
  && systemctl start nginx \
  && cd /opt/jumpserver \
  && ./jms start all -d \
  && docker run --name jms_coco -d -p 2222:2222 -p 5000:5000 -e CORE_HOST=http://$Server_IP:8080 -e BOOTSTRAP_TOKEN=$BOOTSTRAP_TOKEN jumpserver/jms_coco:1.4.9 \
  && docker run --name jms_guacamole -d -p 8081:8081 -e JUMPSERVER_SERVER=http://$Server_IP:8080 -e BOOTSTRAP_TOKEN=$BOOTSTRAP_TOKEN jumpserver/jms_guacamole:1.4.9 \
  && echo -e "\033[31m 你的数据库密码是 $DB_PASSWORD \033[0m" \
  && echo -e "\033[31m 你的SECRET_KEY是 $SECRET_KEY \033[0m" \
  && echo -e "\033[31m 你的BOOTSTRAP_TOKEN是 $BOOTSTRAP_TOKEN \033[0m" \
  && echo -e "\033[31m 你的服务器IP是 $Server_IP \033[0m" \
  && echo -e "\033[31m 请打开浏览器访问 http://$Server_IP 用户名:admin 密码:admin \033[0m"
```

# 杀毒
## [ClamAV](https://www.clamav.net)
**安装**
```bash
yum -y install epel-release
yum -y install clamav-server clamav-data clamav-update clamav-filesystem clamav clamav-scanner-systemd clamav-devel clamav-lib clamav-server-systemd

#在两个配置文件/etc/freshclam.conf和/etc/clamd.d/scan.conf中移除“Example”字符
cp /etc/freshclam.conf /etc/freshclam.conf.bak
sed -i -e "s/^Example/#Example/" /etc/freshclam.conf

cp /etc/clamd.d/scan.conf /etc/clamd.d/scan.conf.bak
sed -i -e "s/^Example/#Example/" /etc/clamd.d/scan.conf
```

**病毒库操作**
关闭自动更新
```bash
freshclam命令通过文件/etc/cron.d/clamav-update来自动运行
vim /etc/cron.d/clamav-update

但默认情况下是禁止了自动更新功能，需要移除文件/etc/sysconfig/freshclam最后一行的配置才能启用
vim /etc/cron.d/clamav-update
  # FRESHCLAM_DELAY=

定义服务器类型（本地或者TCP），在这里定义为使用本地socket，将文件/etc/clam.d/scan.conf中的这一行前面的注释符号去掉：
vim /etc/clamd.d/scan.conf
  LocalSocket /var/run/clamd.scan/clamd.sock
```

下载病毒库
https://www.clamav.net/downloads
将main.cvd\daily.cvd\bytecode.cvd三个文件下载后上传到/var/lib/clamav目录下
```bash
vim /etc/freshclam.conf
  DatabaseDirectory /var/lib/clamav

systemctl enable clamd@scan.service
ln -s '/usr/lib/systemd/system/clamd@scan.service' '/etc/systemd/system/multi-user.target.wants/clamd@scan.service'
```

更新病毒库
```bash
vim /usr/lib/systemd/system/clam-freshclam.service
  # Run the freshclam as daemon 
  [Unit] 
  Description = freshclam scanner 
  After = network.target 

  [Service] 
  Type = forking 
  ExecStart = /usr/bin/freshclam -d -c 4 
  Restart = on-failure 
  PrivateTmp = true 

  [Install] 
  WantedBy=multi-user.target

systemctl start clam-freshclam.service
systemctl status clam-freshclam.service
freshclam
systemctl enable clam-freshclam.service
cp /usr/share/clamav/template/clamd.conf /etc/clamd.conf

vim /etc/clamd.conf
  TCPSocket 3310
  TCPAddr 127.0.0.1

/usr/sbin/clamd restart  
clamdscan -V

systemctl start clamd@scan.service
systemctl status clamd@scan.service
```

查杀病毒
```bash
clamscan -r /home #扫描所有用户的主目录就使用
clamscan -r --bell -i / #扫描所有文件并且显示有问题的文件的扫描结果
clamscan -r --remove  #查杀当前目录并删除感染的文件
```

**Reference**
- [Centos7安装和使用ClamAV杀毒软件](https://blog.51cto.com/11199460/2083697)

---

`"朋友的疏远大致分为两种。天各一方的两个人,慢慢的失掉了联系,彼此不再知道近况,多年之后再聚首往往就只是相对无言了。另一种就令人唏嘘的多了,两个朝夕得见的人,彼此的境遇竟因着造化相去渐远,这时心里也许会慢慢生出一种无力感来,因为无论怎么说怎么做也只能感觉心的距离越来越远了。——吴念真《这些人,那些事》`
