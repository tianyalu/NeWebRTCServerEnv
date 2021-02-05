# NeWebRTCServerEnv Webrtc服务器搭建

[TOC]

## 一、理论

### 1.1 整体架构

服务器中各种服务的关系图：  

![image](https://github.com/tianyalu/NeWebRTCServerEnv/raw/master/show/structure.png)  

### 1.2 参考文献


> Webrtc服务器搭建后台项目地址
>
> java项目:https://github.com/androidtencent/WebRtcJavaWeb
>
> NodeJs项目 : https://github.com/ddssingsong/webrtc_server

## 二、实践

本搭建是基于centos  7.6 64位系统，系统恢复原始状态，重新装系统，确保人人都能搭建成功**

如果系统安装了基础软件  如git  gcc++  可以跳该步骤

```bash
yum update
yum install git
yum install make
yum install gcc-c++
```

### 2.1 搭建`Node`环境

下载官网最新nodejs：https://nodejs.org/en/download

```bash
mkdir webrtc
cd webrtc
wget https://nodejs.org/dist/v10.16.0/node-v10.16.0-linux-x64.tar.xz
```

```bash
# 解压
tar -xvf node-v10.16.0-linux-x64.tar.xz
# 改名
mv node-v10.16.0-linux-x64 nodejs
# 进入目录
cd nodejs/

# 确认一下nodejs下bin目录是否有node 和npm文件，如果有就可以执行软连接
# 看清楚，这个路径是你自己创建的路径，我的路径是/home/dds/webrtc/nodejs
sudo ln -s /root/webrtc/nodejs/bin/npm /usr/local/bin/
sudo ln -s /root/webrtc/nodejs/bin/node /usr/local/bin/

#查看是否安装
node -v   # v10.16.0
npm -v   #6.9.0

# 注意，ubuntu 有的是需要sudo,如果不想sudo,可以
sudo ln -s /root/webrtc/nodejs/bin/node /usr/bin/
```

### 2.2 搭建`turn`服务器环境

#### 2.1.1 安装`turn`服务器的环境准备

```bash
cd ..
# yum install openssl openssl-libs libevent2 libevent-devel
yum install openssl openssl-libs libevent libevent-devel  # libevent2报错，改为libevent
yum install openssl-devel
yum install sqlite
yum install sqlite-devel
yum install postgresql-devel
yum install postgresql-server
yum install mysql-devel
yum install mysql-server
# yum install hiredis
# yum install hiredis-devel
```

直接安装`hiredis`报错，改用如下方式：

```bash
git clone https://github.com/redis/hiredis.git
cd hiredis
make
make install
sudo ldconfig # 更新动态库配置文件
cd hiredis/examples
gcc example.c -o example -I /usr/local/include/hiredis -lhiredis
# Connection error: Connection refused
```

#### 2.2.2 开始安装`turn`服务器

```bash
git clone https://github.com/coturn/coturn 
cd coturn 
./configure 
make 
sudo make install
# 查看是否安装成功
which turnserver  #/usr/local/bin/turnserver
```

生成用户名和密码

```bash
# -u 用户名  
# -p 密码
turnadmin -k -u tianyalu -r north.gov -p 123456
# 生成如下内容：
0x0213168be6952197883c73d971432450
0: log file opened: /var/log/turn_1791_2019-07-31.log
0: SQLite connection was closed.
```

安全访问秘钥  **0x0213168be6952197883c73d971432450**

接下来配置`turnserver ` 的配置文件，配置文件存放在`/usr/local/etc/turnserver.conf`文件下（这个文件本身是不存在的，需要我们自己创建），创建内容：

```bash
verbose
fingerprint
lt-cred-mech
realm=test
user=tianyalu:0x0213168be6952197883c73d971432450
user=tianyalu:123456
stale-nonce
no-loopback-peers
no-multicast-peers
mobility
no-cli
```

user="是你本机生成的随机ID  不要全部直接复制了"

启动服务：

```bash
/usr/local/bin/turnserver --syslog -a -f --min-port=32355 --max-port=65535 --user=dds:123456 -r dds --cert=/cert/cert.pem --pkey==/cert/cert.pem --log-file=stdout -v
```

看到以下内容代表启动成功：

```bash
0: : =====================================================
0: : NO EXPLICIT RELAY ADDRESS(ES) ARE CONFIGURED
0: : ===========Discovering relay addresses: =============
0: : Relay address to use: 172.18.119.188
0: : =====================================================
0: : Total: 1 relay addresses discovered
0: : =====================================================
0: : pid file created: /var/run/turnserver.pid
0: : IO method (main listener thread): epoll (with changelist)
0: : Wait for relay ports initialization...
0: :   relay 172.18.119.188 initialization...
0: :   relay 172.18.119.188 initialization done
0: : Relay ports initialization done
0: : IO method (general relay thread): epoll (with changelist)
0: : turn server id=0 created
0: : IO method (general relay thread): epoll (with changelist)
0: : turn server id=1 created
socket: Protocol not supported
0: : IPv4. TCP listener opened on : 127.0.0.1:3478
socket: Protocol not supported
0: : IPv4. TCP listener opened on : 127.0.0.1:3478
socket: Protocol not supported
0: : IPv4. TCP listener opened on : 127.0.0.1:3479
socket: Protocol not supported
0: : IPv4. TCP listener opened on : 172.18.119.188:3478
socket: Protocol not supported
0: : IPv4. TCP listener opened on : 172.18.119.188:3479
socket: Protocol not supported
0: : IPv4. TCP listener opened on : 127.0.0.1:3479
socket: Protocol not supported
0: : IPv4. TCP listener opened on : 172.18.119.188:3478
socket: Protocol not supported
0: : IPv4. TCP listener opened on : 172.18.119.188:3479
0: : IPv4. UDP listener opened on: 127.0.0.1:3478
0: : IPv4. UDP listener opened on: 127.0.0.1:3479
0: : IPv4. UDP listener opened on: 172.18.119.188:3478
0: : IPv4. UDP listener opened on: 172.18.119.188:3479
0: : Total General servers: 2
0: : IO method (auth thread): epoll (with changelist)
0: : IO method (admin thread): epoll (with changelist)
0: : SQLite DB connection success: /usr/local/var/db/turndb
0: : IO method (auth thread): epoll (with changelist)
```

```bash
Ctrl + C 结束
```

### 2.3 搭建`Webrtc`服务端

```bash
# 安装webrtc服务器和浏览器端
git clone https://github.com/androidtencent/WebrtcNodeJS
cd WebrtcNodeJS
npm install
# 启动服务
node server.js 
# Server running at http://0.0.0.0:3000/
```

### 2.4  搭建`nginx`服务器环境

#### 2.4.1  安装`nginx`服务器（推荐用编译来安装）

```bash
wget http://nginx.org/download/nginx-1.12.0.tar.gz
tar xvf nginx-1.12.0.tar.gz
cd nginx-1.12.0

./configure --prefix=/usr/local/nginx --with-http_stub_status_module --with-http_ssl_module

make 
sudo make install 
```

**遇到的坑：**

①`./configure`时报错如下：

```bash
src/core/ngx_murmurhash.c:37:11: error: this statement may fall through [-Werror=implicit-fallthrough=]
         h ^= data[2] << 16;
         ~~^~~~~~~~~~~~~~~~
src/core/ngx_murmurhash.c:38:5: note: here
     case 2:
     ^~~~
src/core/ngx_murmurhash.c:39:11: error: this statement may fall through [-Werror=implicit-fallthrough=]
         h ^= data[1] << 8;
         ~~^~~~~~~~~~~~~~~
src/core/ngx_murmurhash.c:40:5: note: here
     case 1:
```

解决方案：

```bash
vim objs/Makefile
# CFLAGS =  -pipe  -O -W -Wall -Wpointer-arith -Wno-unused-parameter -Werror -g
# 去掉CFLAGS中的-Werror 然后执行 make 命令,不要`./configure`了
```

参考：[安装nginx时出现In function ‘ngx_murmur_hash2’等错误](https://blog.csdn.net/zolewit/article/details/93595253)

②执行`make`命令时报错如下：

```bash
src/os/unix/ngx_user.c:36:7: error: ‘struct crypt_data’ has no member named ‘current_salt’
     cd.current_salt[0] = ~salt[0];
```

解决方案：

```bash
vim src/os/unix/ngx_user.c
# 找到以下这行代码并注释掉
#ifdef __GLIBC__
    /* work around the glibc bug */
    /*cd.current_salt[0] = ~salt[0];*/  # 注释掉这行代码并重新 make
#endif
```

参考：[linux 安装 nginx ‘struct crypt_data’ has no member named ‘current_salt’ 解决办法](https://blog.csdn.net/humanyr/article/details/107405383)


#### 2.4.2 生成`nginx`中的`https`证书

> **x509证书一般会用到三类文，key，csr，crt**
>
> **Key 是私用密钥openssl格，通常是rsa算法。**
>
> **Csr 是证书请求文件，用于申请证书。在制作csr文件的时，必须使用自己的私钥来签署申，还可以设定一个密钥。**
>
> **crt是CA认证后的证书文，（windows下面的，其实是crt），签署人用自己的key给你签署的凭证。** 
>
> **1.key的生成** 
>
> ```bash
> mkdir certificate
> cd certificate
> openssl genrsa -des3 -out cert.key 2048
> # 输入密码
> # 确认密码
> ls
> # cert.key
> ```
>
> 这样是生成`rsa`私钥，`des3`算法，`openssl`格式，`2048`位强度。`server.key`是密钥文件名。为了生成这样的密钥，需要一个至少四位的密码。可以通过以下方法生成没有密码的`key`:
>
> ```bash
> openssl rsa -in cert.key -out cert.key
> # 输入密码
> ls
> # cert.key
> ```
>
> 这样`server.key`就是没有密码的版本了。 
>
> **2. 生成CA的crt**
>
> ```bash
> openssl req -new -x509 -key cert.key -out cert.crt -days 3650
> Country Name (2 letter code) [XX]:cn
> State or Province Name (full name) []:tian
> Locality Name (eg, city) [Default City]:tian
> Organization Name (eg, company) [Default Company Ltd]:tian
> Organizational Unit Name (eg, section) []:tian
> Common Name (eg, your name or your server's hostname) []:tian
> Email Address []:tian
> ls
> # cert.crt cert.key
> ```
>
> 这里生成的`ca.crt`文件是用来签署下面的`server.csr`文件。 
>
> **3. csr的生成方法**
>
> ```bash
> openssl req -new -key cert.key -out cert.csr
> Country Name (2 letter code) [XX]:cn
> State or Province Name (full name) []:tian
> Locality Name (eg, city) [Default City]:tian
> Organization Name (eg, company) [Default Company Ltd]:tian
> Organizational Unit Name (eg, section) []:tian
> Common Name (eg, your name or your server's hostname) []:tian
> Email Address []:tian
> 
> Please enter the following 'extra' attributes
> to be sent with your certificate request
> A challenge password []:tian
> An optional company name []:tian
> ls
> # cert.crt  cert.csr  cert.key
> ```
>
> 需要依次输入国家，地区，组织，email。最重要的是有一个`common name`，可以写你的名字或者域名。如果为了`https`申请，这个必须和域名吻合，否则会引发浏览器警报。生成的`csr`文件交给`CA`签名后形成服务端自己的证书。 
>
> **4. crt生成方法**
>
> `CSR`文件必须有`CA`的签名才可形成证书，可将此文件发送到`verisign`等地方由它验证，要交一大笔钱，何不自己做`CA`呢。
>
> ```bash
> openssl x509 -req -days 3650 -in cert.csr -CA cert.crt -CAkey cert.key -CAcreateserial -out cert.crt
> ls
> # cert.crt  cert.csr  cert.key  cert.srl
> ```
>
> 输入`key`的密钥后，完成证书生成。`-CA`选项指明用于被签名的`csr`证书，`-CAkey`选项指明用于签名的密钥，`-CAserial`指明序列号文件，而`-CAcreateserial`指明文件不存在时自动生成。
>
> 最后生成了私用密钥：`server.key`和自己认证的`SSL`证书：`server.crt`
>
> 证书合并：
>
> ```bash
> cat cert.key cert.crt > cert.pem
> ls
> # cert.crt  cert.csr  cert.key  cert.pem  cert.srl
> # 其中cert.crt和cert.pem是我们所需要的
> ```

#### 2.4.3 更改`nginx` 配置文件并运行（额外强调  其中包含https证书，所以需要先完成2.4.2步骤）

```bash
vim /usr/local/nginx/conf/nginx.conf
```

* 删除配置文件内容,更改为以下内容

```bash
user root;
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
	worker_connections 768;
        multi_accept on;
	}

http {
	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	keepalive_timeout 300;
	types_hash_max_size 2048;
	default_type application/octet-stream;


	ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
	ssl_prefer_server_ciphers on;

	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;

	gzip on;

	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;

        upstream web {
		server localhost:3000;      
        }
	
	upstream websocket {
		server localhost:3000;   
        }

	server { 
		listen       443; 
		server_name  localhost;
		ssl          on;

		ssl_certificate     /root/webrtc/certificate/cert.crt;#配置证书
		ssl_certificate_key  /root/webrtc/certificate/cert.pem;#配置密钥
			ssl_session_cache    shared:SSL:1m;
		ssl_session_timeout  50m;
		ssl_protocols TLSv1 TLSv1.1 TLSv1.2 SSLv2 SSLv3;
		ssl_ciphers  HIGH:!aNULL:!MD5;
		ssl_prefer_server_ciphers  on;
		
		location /wss {
		proxy_pass http://websocket/; # 代理到上面的地址去
		proxy_read_timeout 300s;
		proxy_set_header Host $host;
		proxy_set_header X-Real_IP $remote_addr;
		proxy_set_header X-Forwarded-for $remote_addr;
		proxy_set_header Upgrade $http_upgrade;
		proxy_set_header Connection 'Upgrade';	
 		 }
		location / {
		proxy_pass         http://web/;
		proxy_set_header   Host             $host;
		proxy_set_header   X-Real-IP        $remote_addr;
		proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
 		 }
	}
}
```

* 运行：

```bash
cd /usr/local/nginx/sbin
./nginx
```

报错如下：

```bash
nginx: [emerg] open() "/var/log/nginx/access.log" failed (2: No such file or directory)
```

解决办法：

```bash
mkdir /var/log/nginx
```

### 2.5 分别启动服务

#### 2.5.1 启动`turnserver`服务

```bash
/usr/local/bin/turnserver --syslog -a -f --min-port=32355 --max-port=65535 --user=dds:123456 -r dds --cert=/cert/cert.pem --pkey==/cert/cert.pem --log-file=stdout -v

# --syslog 使用系统日志
# -a 长期验证机制
# -f 使用指纹
# --min-port   起始用的最小端口
# --max-port   最大端口号
# --user=dds:123456  turn用户名和密码
# -r realm组别
# --cert PEM格式的证书
# --pkey PEM格式的私钥文件
# -l, --log-file,<filename> 指定日志文件
```

#### 2.5.2 启动`nginx`服务

```bash
cd /usr/local/nginx/sbin
./nginx
```

#### 2.5.3 启动`webrtc` 服务

```bash
cd /root/webrtc/WebrtcNodeJS
node server
```

### 2.6 后台服务

因为分别启动服务会阻塞命令行窗口，所以我们可以开后台进程来运行，生成`shell`脚本：

```bash
cd ~/webrtc/WebrtcNodeJS
vim run.sh
```

脚本内容如下：

```bash
#!/bin/bash
NODE=`which node`
nohup $NODE server.js & > log.txt  # nohup代表后台运行 &代表重定向 >表示重定向位置
nohup /usr/local/bin/turnserver --syslog -a -f --min-port=32355 --max-port=65535 --user=dds:123456 -r dds --cert=/cert/cert.pem --pkey==/cert/cert.pem --log-file=stdout -v & > log.txt
```

给脚本加上权限并运行：

```bash
chmod 777 run.sh
./run.sh
```

