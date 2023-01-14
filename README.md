# Use v2ray with protection by Apache & Web & SSL
出于隐私需求和工作需要，我自建了一个海外v2ray服务器。但是这个节点在高敏感时期经常被墙，所以我给它隐藏在web服务器后面来混淆。

### 首先明确的是，我之前完全没有在Debian环境下安装或者配置Apache的经验（其实Windows环境的经验也少之又少）
#### 这次的任务是 
  1. 基于WSS协议的，直接转发到127.0.0.1:（v2ray端口）
  2.	基于HTTPS协议的，如果域名是指定的域名，转发到127.0.0.1:80（正常网页），返回域名对应的SSL证书
#### 环境介绍
  1. 云端测试 Windows Server 2022云服务器一台
  2. 本地测试 Kali虚拟机 by Debian
  3. 实际上线 Debian x64 without GUI
>我的实际上线环境是已经有一个独立的HTTP服务端的，如果你没有的话需要自己在Apache里面做一个

>证书申请啥的不再提了，不会的建议去看看v2ray的官方文档

>为了方便我自己，这个文档将直接解释最后的成果，而不是把中间我早上4点做到晚上9点的过程遇到的所有麻烦放出来。
#### 任务开始
#### 在Debian下安装Apache2.4
  首先我习惯性的认为这是个要手动编译的东西，去apache.org各种找源码。结果翻 https://httpd.apache.org/docs/2.4/install.html 看到句“Installing on Ubuntu/Debian”
```
sudo apt install apache2
sudo service apache2 start 或 systemctl start apache2
```
  就这么简单，没别的了，这破事花了我半个多小时。
  
#### 配置文件
Debian下的Apache配置文件跟Windows有很大不同。\
配置文件全部存储在在默认位置/etc/apache2下，其中apache2.conf我没管，默认就行。但是要注意apache2.conf的末尾
  ```
  IncludeOptional sites-enabled/*.conf
  ```
  这一行是否正确\
在/etc/apache2/sites-enabled下有一个000-defalt.conf，先备份一下防止出问题。
```
cp 000-defalt.conf 000-defalt.bk
```
然后重命名现有的000-defalt.conf为你看着舒服的文件名，为了演示将其命名为001-web+v2.conf
```
mv 000-defalt.conf 001-web+v2.conf
```
然后打开这个文件
```
vim 001-web+v2.conf
```
依照Apache的配置规则写一个配置，这里展示一下我自己的最终配置，地址、端口等信息已经抹除。（该配置适用于已经有独立的Web服务的情况下，比如cloudreve）
```conf
<VirtualHost *:443>
	SSLEngine on
	SSLCertificateFile "（证书文件（最好是完整证书链））"
	SSLCertificateKeyFile "（私钥文件）"
	#ServerAdmin （电子邮件地址，可加可不加，也可以直接注释掉）
#这里开始是反代一个虚拟路径到v2ray服务端
	ProxyPass "（你想要的ws路径）" "ws://127.0.0.1:[v2ray服务端口]/[v2ray服务端的ws路径]" 
	ProxyRequests off
	ProxyAddHeaders Off
	ProxyPreserveHost On
	RequestHeader append X-Forwarded-For %{REMOTE_ADDR}s
	AllowEncodedSlashes On
#反代到v2ray结束
#反代到你的web服务开始
	<Proxy />
		Order deny,allow
		Allow from all
	</Proxy>
	ProxyPass / "http://127.0.0.1:80/" nocanno
	ProxyPassReverse / "http://127.0.0.1:80/"  nocanno
#反代到你web服务结束
	ErrorLog ${APACHE_LOG_DIR}/error.log #错误日志路径
		CustomLog ${APACHE_LOG_DIR}/access.log combined #日志路径
</VirtualHost>
```
保存后，终端
```
systemctl restart apache2
```
>在这期间无数个caikun233不幸累死\
当然，如果你没有独立的Web服务端，可以用Apache自己建一个。我就不教了因为我也不会写HTML。但是Apache的大概配置可以提供。
```
<VirtualHost 127.0.0.1:80>
  Include "${INSTALL_DIR}/alias/*.conf"
  ServerName localhost
  ServerAlias localhost
  DocumentRoot "${INSTALL_DIR}/www"
  <Directory "${INSTALL_DIR}/www/">
    Options +Indexes +Includes +FollowSymLinks +MultiViews
    AllowOverride All
    Require local
  </Directory>
</VirtualHost>
```
上面这一块代码中，/www/放你的网页文件。我的建议是做一个类似私有云盘的东西，这样可以很好的解释长时间大流量访问操作。

#### 装载Mod和遇到的一些问题
在Debian Apache下，装载module的方式不是写在配置文件里。而是使用命令行。
```
a2enmod [MOD]
```
列举一下需要手动load的mod
```
proxy proxy_html proxy_http proxy_http2 ssl
```
这些modules可以用下面这个命令一键load
```
a2enmod proxy proxy_html proxy_http proxy_http2 ssl
```

##### 问题1：虚拟路径里的/
最初的测试是在Windows环境下做的，当时遇到了一个ws路径的问题。\
测试的部分conf是这样的
```
<Location "/qsdsas/">
		ProxyPass "ws://[IP&port]/qsdsas" upgrade=WebSocket
		ProxyAddHeaders Off
		ProxyPreserveHost On
		RequestHeader append X-Forwarded-For %{REMOTE_ADDR}s
</Location>
```
测试环境下v2ray的ws路径是/qsdsas，但是Apache配置的是/qsdsas/，并且认为不会造成影响。正常情况下应该会返回一个HTTP101然后切换协议到WebSocket，结果浏览器进去/qsdsas这路径居然回了个404。\
也就是说，Apache和v2ray对“/”这个东西都是敏感的，别乱加。
##### 问题2：python版本
在kali测试环境下，预装python3.10。在python3下是可以像上面<location>那样多行写的。\
但是上线环境是python2，只能用文档开始的ProxyPass方法一行写完，不然报错无法启动。把upgrade=WebSocket删掉后Apache可以启动，但是不代理。
##### 问题3：URL字符编码
因为我是已有的Web服务，所以Apache在转发的时候会把我想要访问Web服务的URL编码一次。可是我的浏览器已经编码过了，所以我们需要在Apache配置中禁止它再次编译
#### v2ray服务配置
在反代web服务之前添加一行
```
AllowEncodedSlashes On
```
在反代到web服务那一块，ProxyPass和ProxyPassReverse这两个选项最后面加nocanno\
如果你把我上面的配置复制走了，那就不用加，我已经帮你加好了。
由于Apache已经代理了TLS，并且指定了ws协议，**所以v2ray不能再启用TLS且协议必须是ws**。\
上文配置中，把ws协议反代到了127.0.0.1:[v2ray服务端口]\
**相应的v2ray也必须监听127.0.0.1:[v2ray服务端口]，不能是0.0.0.0或其他地址**
#### v2ray客户端配置
上述配置中提到，Apache代理了一个虚拟ws路径。也就是说，v2ray客户端需要连接到Apache所提供的路径，Apache才能正常将数据代理到v2ray服务端。
```
ProxyPass "（你想要的ws路径）" "ws://127.0.0.1:[v2ray服务端口]/[v2ray服务端的ws路径]" 
```
就是这一行，v2ray客户端需要连接到“你想要的ws路径”才可以。端口是443。
#### 如果你之前已经有过独立的Web服务
与v2ray类似，Apache已经代理了TLS，需要**将你服务占用的443 SSL端口移开，并且启用HTTP页面**，不再使用HTTPS。
#### 使其更安全
安全等级什么的我是去https://myssl.com 查的
1. 开启HSTS\
在配置文件开头,SSLEngine on那一行前面添加下面这一块可以开启HSTS
```
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
```

2. 关闭TLSv1.0、TLS1.1、SSLv3\
在上述HSTS配置后添加
```
SSLProtocol all -SSLv3 -TLSv1 -TLSv1.1
SSLProxyProtocol all -SSLv3 -TLSv1 -TLSv1.1
```
这些配置将仅允许TLSv1.2及以上通讯。

3. 加密方式白名单\
MD5等加密方式已经不再安全，我们可以指定一些加密方式，仅允许这些加密方式通讯。在上述TLS协议版本后添加
```
SSLCipherSuite ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384
```
这样可以开启可以使用加密方式白名单

##### 任务完成
我是真的喜欢钱，所以喜欢的大伙可以往下面这个USDT-TRC20钱包地址转点USDT（不支持DEX、BSC、HECO的转币）\
TJnbVK1pGZPULNd2JsrwaArbYhFd286UXH
