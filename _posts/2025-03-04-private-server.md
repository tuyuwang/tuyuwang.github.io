---
title: 树莓派私有云
author: tuyw
date: 2025-03-04 09:37:00 +0800
categories: [后端]
tags: [云服务器, 树莓派]
---

## 前言

为了让吃灰派能有所用途,发挥它的价值. 所以耗费了些许时间研究如何在电信家庭网络环境下,开启一个能在外网环境进行访问使用的云服务.

相关设备:
- Mac
- 树莓派3B+
- 小米路由器

这篇文章将详细的讲述实现步骤:
1. 树莓派3B+刷ubuntu系统
2. 部署webdav服务
3. 设置小米路由器DMZ服务进行内网穿透
4. 申请阿里域名,使用DDNS动态域名解析
5. 支持https访问服务


## 1. 树莓派3B+刷ubuntu系统

1.进入[树莓派官网](https://www.raspberrypi.com/software/). 下载Raspberry Pi Imager

![pi-imager](/assets/img/raspberrypi/pi-imager.png)

2.启动Pi Imager, 然后根据自己情况配置. 我的树莓派设备是3B+, 刷的ubuntu server 是20.04.5 64-Bit. 然后选择已经插入的SD卡. 点击Next

![pi-imager-prepare](/assets/img/raspberrypi/pi-imager-prepare.png)

3.选择编辑设置(可以不需要连接显示器登陆树莓派)

![pi-imager-config](/assets/img/raspberrypi/pi-imager-config.png)

<span id="username"></span>
4.配置登陆账户名、wifi和密码

![pi-imager-config-general](/assets/img/raspberrypi/pi-imager-config-general.png)

- 主机名: 给树莓派服务起个名, 可以任意设置, 这里为了区分设置pi3
- Username: 登陆的账户名
- 配置WiFI时: 热点不能是隐藏的wifi, 会连不上. 

5.配置ssh服务

这里可选任意一种方式. 一种需要设置登陆密码, 一种直接以当前电脑进行ssh进行免密登陆.
为了方便,我这里选择免密登陆. (后续也可以自己更改)


![pi-imager-config-services](/assets/img/raspberrypi/pi-imager-config-services.png)

6.点击保存, 一路选择是, 开始烧录ubuntu server到SD卡中

> 由于设置了ssh免密登陆, Pi Imager需要访问电脑公钥, 所以需要输入密码授权同意

<div align="center">
	<img src="/assets/img/raspberrypi/pi-imager-start-0.png" width="30%" />
	<img src="/assets/img/raspberrypi/pi-imager-start-1.png" width="30%" />
	<img src="/assets/img/raspberrypi/pi-imager-start-2.png" width="30%" />
</div>

7.烧录成功后将SD卡插入树莓派中,并启动树莓派 \n
如果一直烧录失败,可以在通过其他渠道进行手动下载对应版本的server. 然后在Pi Imager里选择`Use custom`来选择本地下载好的系统镜像文件进行烧录

![ubuntu-custom](/assets/img/raspberrypi/ubuntu-custom.png)


## 2. 无屏登陆树莓派

1.找到树莓派在局域网的ip地址

这里提供两种方式说明: 

- 使用命令行工具arp

打开终端,在终端输入命令进行查看设备ip.(电脑wifi与树莓派连的wifi要处于同一个局域网内)
> arp -a

![arp](/assets/img/raspberrypi/arp.png)

如果不知道哪个是新增的ip, 可以尝试对每个ip进行ssh登陆(`ssh ubuntu@192.168.31.16`)操作. 或者让树莓派关机重启来观察这个命令的输出结果, 对比重启后的输出结果来找到树莓派的ip地址


- 使用手机上的小米wifi app查看路由器(其他路由器类似操作)新连入的设备,在设备信息中找到ip地址

<div align="center">
	<img src="/assets/img/raspberrypi/xiaomi-0.png" width="30%" />
	<img src="/assets/img/raspberrypi/xiaomi-1.png" width="30%" />
	<img src="/assets/img/raspberrypi/xiaomi-2.png" width="30%" />
</div>

2.登陆树莓派

我这里的找到ip地址为: 192.168.31.16, 按自己找到的地址进行替换登陆

~~~bash
ssh ubuntu@192.168.31.16
~~~

ubuntu是烧录系统时设置的[Username](#username). 登陆成功之后如下图:

![ubuntu-login](/assets/img/raspberrypi/ubuntu-login.png)

系统版本为20.04.1

3.修改apt国内镜像源. 避免无法安装软件包

这里使用的是[清华ubuntu镜像源](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/)

~~~
sudo vim /etc/apt/sources.list
~~~

根据自己安装的unbuntu版本(v20.04.1)和格式来修改这个文件. 

![ubuntu-apt-source-qh](/assets/img/raspberrypi/ubuntu-apt-source-qh.png)

这里以v20.04.1 对应的focal版本为例:先将原始配置注释掉. 然后在文件末尾添加以下配置(非focal版本则需要从[清华ubuntu镜像源](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/)找到对应版本的配置进行复制)

![ubuntu-apt-source](/assets/img/raspberrypi/ubuntu-apt-source.png)

~~~
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-updates main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-updates main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-backports main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-backports main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-security main restricted universe multiverse
# deb-src https://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-security main restricted universe multiverse
deb http://mirrors.tuna.tsinghua.edu.cn/ubuntu-ports/ focal-proposed main restricted universe multiverse
~~~



4.更新apt

```
sudo apt update
```


## 2. 部署webdav服务

这里使用nginx和apache来实现webdav服务, 包含设置开机挂载SD卡目录、设置指定目录需要密码访问的步骤

1. 安装nginx和apache2

```
sudo apt install nginx apache2
```

2.使用apache配置一个webdav的服务

创建webdav.conf文件, 并将配置复制到文件中

```
sudo vim /etc/apache2/sites-available/webdav.conf

<VirtualHost *:888>
        ServerAdmin webmaster@localhost
        ServerName dav.tuyw.cn 

        DocumentRoot /var/www/webdav
        <Directory />
                Options FollowSymLinks
                AllowOverride None
        </Directory>
        <Directory /var/www/webdav/>
                Options Indexes FollowSymLinks MultiViews
                AllowOverride None
                Order allow,deny
                allow from all
        </Directory>
	
	Alias /data /var/www/webdav/data
	<Location /data>
   		DAV On
	</Location>
	

	Alias /secret /var/www/webdav/secret
	<Location /secret>
    		DAV On
        	AuthType Basic
        	AuthName "webdav"
        	AuthUserFile /usr/local/apache2/webdav.passwords
       		Require valid-user
	</Location>
</VirtualHost>

```

让apache2监听1999端口, 将`Listen 1999`添加到ports.conf文件中

```
sudo vim /etc/apache2/ports.conf 
```
![webdav-port](/assets/img/raspberrypi/webdav-port.png)


---


以下是配置说明:

```apache
<VirtualHost *:1999> 
    ServerAdmin webmaster@localhost 
    ServerName dav.tuyw.cn 
    DocumentRoot /var/www/webdav
</VirtualHost>
```
- 监听端口：1999
- 管理员邮箱：webmaster@localhost
- 服务器域名：dav.tuyw.cn, 服务域名为tuyw.cn的二级域名dav.tuyw.cn (当有多个服务共用一个端口号888时, 可以直接使用dav.tuyw.cn:1999域名访问到webdav服务)
- 网站根目录：/var/www/webdav

---

```apache
<Directory /> 
    Options FollowSymLinks 
    AllowOverride None 
</Directory> 
<Directory /var/www/webdav/> 
    Options Indexes FollowSymLinks MultiViews 
    AllowOverride None 
    Order allow,deny 
    allow from all 
</Directory>
```
- 根目录权限：
- - 允许符号链接
- - 禁止.htaccess覆盖
- WebDAV目录权限：
- - 允许目录列表
- - 允许符号链接
- - 允许多视图: 允许服务器根据客户端请求自动选择最合适的资源版本
- - 允许所有客户端访问

---

```apache
Alias /data /var/www/webdav/data 
<Location /data> 
    DAV On 
</Location>
```
- 路径映射：/data → /var/www/webdav/data
- 功能：启用WebDAV（无需认证）

---

```apache
Alias /secret /var/www/webdav/secret 
<Location /secret> 
    DAV On 
    AuthType Basic 
    AuthName "webdav" 
    AuthUserFile /usr/local/apache2/webdav.passwords 
    Require valid-user 
</Location>

```
- 路径映射：/secret → /var/www/webdav/secret
- 认证方式：基本认证
- 认证文件：/usr/local/apache2/webdav.passwords
- 访问控制：仅限有效用户

---

以上配置可以实现:
1. 公开访问的WebDAV共享（/data）
2. 需要密码验证的WebDAV共享（/secret）
3. 普通HTTP访问的网站根目录

启动该配置:
```
sudo mkdir /var/www/webdav
sudo chown www-data.www-data /var/www/webdav
sudo a2ensite webdav.conf 
sudo sh -c 'echo "Welcome from WebDAV.local" > /var/www/webdav/index.html'
sudo service apache2 reload
```
然后在浏览器上打开树莓派的ip地址+端口号(`http://192.168.31.16:1999`), 成功的话就可以看到下图内容

![webdav-index](/assets/img/raspberrypi/webdav-index.png)


3.设置开机挂载SD卡

通过上述的操作,在路径`/var/www/webdav`目录下只有一个index.html文件,现在给树莓派USB口插入一个SD卡, 然后将这个SD卡的内容开机自动挂载到`/var/www/webdav/data`下


先创建data和secret文件夹
```
sudo mkdir -p /var/www/wedav/data
```

找到挂载的sd卡名称,执行命令
```
sudo fdisk -l
```
找到SD卡名称为`/dev/sda1`
![webdav-fdisk](/assets/img/raspberrypi/webdav-fdisk.png)


根据找到的SD卡名称`/dev/sda1`来找出uuid,执行命令:
```
sudo blkid /dev/sda1
```
![webdav-uuid](/assets/img/raspberrypi/webdav-uuid.png)

然后将找到的uuid: `66F8-074E` 和SD卡类型: `exfat` 写入`/etc/fstab`文件中
```
sudo vim /etc/fstab

#UUID 挂载到的路径 SD卡格式类型 默认挂载参数 不备份该文件系统 不磁盘检查顺序
UUID=66F8-074E /var/www/webdav/data exfat defaults 0 0
```

![webdav-fstab](/assets/img/raspberrypi/webdav-fstab.png)

修改完成后,执行重启命令
```
sudo reboot
```
查看是否挂载成功
```
sudo df -lh
```
![webdav-mount](/assets/img/raspberrypi/webdav-mount.png)

浏览器打开树莓派+端口ip地址+/data(`http://192.168.31.16:1999/data`), 如果成功可以在浏览器上看到SD卡的内容

![webdav-data](/assets/img/raspberrypi/webdav-data.png)

4.webdav密码访问

在编辑webdav.conf时, 已经设置目录`/var/www/webdav/secret`为加密目录,访问需要密码,且密码文件指定在` /usr/local/apache2/webdav.passwords`

使用htpasswd命令创建密码文件,用于授权用户(admin)访问secret目录
```
sudo htpasswd -c /usr/local/apache2/webdav.passwords admin
```
![webdav-passwd](/assets/img/raspberrypi/webdav-passwd.png)
- 创建了一个可以访问secret目录的用户为: admin
- 输入密码
- 再次输入密码

浏览器打开树莓派+端口ip地址+/data(`http://192.168.31.16:1999/secret`),如果成功可以在浏览器上看到输入账户密码弹窗

![webdav-user](/assets/img/raspberrypi/webdav-user.png)


3.使用nginx代理webdav服务


```
#创建webdav文件
sudo touch /etc/nginx/sites-available/webdav

#将下面代码写入webdav文件中
server {
    server_name dav.tuyw.cn;
    listen 666

    location / {
        proxy_pass http://127.0.0.1:1999/;  # webdav: apache2已开启的端口1999 
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }


    access_log /var/log/nginx/dav.tuyw.cn.access.log;
    error_log /var/log/nginx/dav.tuyw.cn.error.log;
}

```

4.重启nginx
```
sudo systemctl restart nginx
```

5.验证

浏览器打开树莓派+端口ip地址+/data(`http://192.168.31.16:666/data`), 如果成功可以在浏览器上看到SD卡的内容,说明转发成功. 即通过访问nginx监听的666端口,nginx将其转发给了apache监听的1999端口, 对apache提供的webdav服务进行了访问.

![webdav-data](/assets/img/raspberrypi/webdav-data.png)

到了这一步,内网的webdav服务已经准备就绪, 接下来是实现外网访问这个webdav服务.

## 3. 设置小米路由器DMZ服务进行内网穿透

这里说明一下: 为什么不使用光猫做DMZ转发, 首先光猫性能比不过路由器,其次设置光猫的DMZ需要超级管理员帐号密码,这个账户密码电信是不会给个人用户的,也尝试过网上的破解账户密码和淘宝上有售卖.都不靠谱,就转换思路,尝试联系电信客服来将光猫改成桥接模式,让小米路由器来进行拨号,这样可以自由设置DMZ服务.

1.将电信光猫的连接方式改成桥接模式

因为之前安装宽带时有添加电信师傅的微信.所以联系师傅直接在后台帮忙转成桥接模式. 也可以拨打电信客服帮忙联系电信师傅, 并且向客服要到宽带的帐号和密码,会以短信的方式发送. `重点: 都是免费的,不需要人工费用`

2.登陆小米路由器后台`192.168.1.1`. 设置拨号上网

<div align="center">
	<img src="/assets/img/raspberrypi/xiaomi-network-1.png" width="49%" />
	<img src="/assets/img/raspberrypi/xiaomi-network-2.png" width="49%" />
</div>

这样小米路由器就会拥有一个动态的公网ip地址

3.固定树莓派的ip地址

![xiaomi-static-ip](/assets/img/raspberrypi/xiaomi-static-ip.png)

这一步的作用是将树莓派的内网ip地址设置为静态的. 这样当树莓派重启时,就不会变更ip地址.避免后续转发请求异常

4.设置DMZ转发树莓派ip地址

<div align="center">
	<img src="/assets/img/raspberrypi/xiaomi-dmz-1.png" width="49%" />
	<img src="/assets/img/raspberrypi/xiaomi-dmz-2.png" width="49%" />
</div>


这一步的作用是将所有访问小米路由器的请求都转发给树莓派,这样就可以将公网的请求转发到树莓派上, 也就是说可以在外网访问到树莓派上的服务内容.

5.验证

找到小米路由器上显示的公网ip地址. 然后在浏览器上输入公网ip地址, 然后填入树莓派上开放的webdav服务端口1999, 成功则出现如下页面
<div align="center">
	<img src="/assets/img/raspberrypi/xiaomi-ip-1.png" width="49%" />
	<img src="/assets/img/raspberrypi/xiaomi-ip-2.png" width="49%" />
</div>


## 4. 申请阿里域名,使用DDNS动态域名解析

由于现在的家庭网络申请不到固定的公网ip地址(ipv4), 所以小米路由器当前拥有的是一个动态的ip地址,会不定期由于运营商重启服务而导致变更.并且直接通过ip地址访问十分不便.
所以我这里申请了一个阿里云的域名,然后通过在树莓派上运行DDNS服务来实现动态域名解析.

最终可以实现仅通过域名+端口来访问到树莓派上的webdav服务

1.在阿里云申请一个域名

[阿里云](https://www.aliyun.com)
注册一个想要的域名. 我这里假设注册的域名是tuyw.cn

<div align="center">
	<img src="/assets/img/raspberrypi/aliyun-1.png" width="49%" />
	<img src="/assets/img/raspberrypi/aliyun-2.png" width="49%" />
</div>



2.配置域名解析
购买域名付费成功后.找到域名解析页面,将域名解析到路由器的ip地址上

<div align="center">
	<img src="/assets/img/raspberrypi/aliyun-dns-1.png" width="49%" />
	<img src="/assets/img/raspberrypi/aliyun-dns-2.png" width="49%" />
</div>

---

<div align="center">
	<img src="/assets/img/raspberrypi/aliyun-dns-3.png" width="49%" />
	<img src="/assets/img/raspberrypi/aliyun-dns-4.png" width="49%" />
</div>

---

<div align="center">
	<img src="/assets/img/raspberrypi/aliyun-dns-5.png" width="49%" />
	<img src="/assets/img/raspberrypi/aliyun-dns-6.png" width="49%" />
</div>

配置完成后,点击生效检测. 当你所在区域的地点的检测结果显示路由器的ip地址时,就可以通过域名+端口访问到webdav服务了

## 5. 支持https访问

1.设置防火墙端口

防火墙放行666与1999端口
```
ufw allow 666
ufw allow 1999
ufw enable
```

2.申请免费证书

安装certbot
```
apt install certbot python3-certbot-nginx
apt install certbot python3-certbot-apache
```

使用certbot自动为域名申请证书
```
certbot --nginx -d tuyw.cn -d dav.tuyw.cn
或
certbot --apache -d tuyw.cn -d dav.tuyw.cn
```

由于certbot自动申请证书流程会通过80和443端口进行校验,而家庭网络的80和443端口已经被运营商禁用.所以只能使用certbot手动为域名申请证书
```
#80/443被占用,手动操作
certbot certonly --manual --preferred-challenge dns -d tuyw.cn -d dav.tuyw.cn
```
![ssl](/assets/img/raspberrypi/ssl.png)

输入命令后,并按提示操场后,会给出一些信息要求你在阿里云域名解析配置添加一条TXT记录, 用于certbot验证该域名是属于你的.

使用以上信息在阿里云域名解析中新增一条TXT记录

![ssl-txt](/assets/img/raspberrypi/ssl-txt.png)

点击确认后,等待解析生效.生效后再回到终端敲击`Enter`进行下一步

![ssl-cert](/assets/img/raspberrypi/ssl-cert.png)

在申请成功后,会给出两个pem的文件路径, 将这两个文件路径写入到nginx的配置中即可生效
```
sudo vim /etc/nginx/sites-available/webdav
```
修改文件内容:
![ssl-config](/assets/img/raspberrypi/ssl-config.png)

保存后重启生效
```
sudo systemctl restart nginx
```

输入域名+端口号即可访问
![ssl-end](/assets/img/raspberrypi/ssl-end.png)


## 总结


本文记录了如何通过家庭网络的服务公开到公网中访问.在实现这个的过程中,还做过许多其他的尝试,比如白嫖甲骨文的永久云服务器、使用ipv6进行公网访问等操作.最后选择了DDNS动态域名解析的方式实现该功能,对于家庭网络而言,80和443端口被屏蔽是该方案比较痛的一个点,并且还会导致cert免费证书过期之后需要手动更新,比较繁琐.不过对于租赁云服务器而言具有低成本的优势,勉强够个人使用.

本篇文章是下篇文章《Photoprism家庭相册私有云服务实现》的前置文章,在《Photoprism家庭相册私有云服务实现》中将会介绍如何使用开源框架Photoprism实现一个公网可以访问的家庭相册服务,将上百G只能躺在硬盘中的照片、视频可以通过服务器自由访问, 并且可以生成公开链接供家庭、朋友们在公网访问.



## 参考
- [无80端口情况下使用 CertBot 申请SSL证书 并实现自动续期](https://blog.csdn.net/conghua19/article/details/81433716)
- [中国电信光猫TEWA-1000E获取超级管理员密码/超管密码](https://www.bilibili.com/opus/1018282423178231814)