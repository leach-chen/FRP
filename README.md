# FRP内网穿透(域名绑定+Nginx代理+HTTPS支持+SSH登录)

## 一、介绍

内网穿透，网上有多种方案，有些是付费的，有些是开源的，其实各有优劣，付费的不需要自己拥有服务器，而开源的比如FRP则需要有一台公网的服务器，如果你拥有一台公网服务器（服务器可能不是你买的，但是你有使用权限或者说该服务器性能要差点），但是部署的软件如网站、gitlab等想放在自己的物理电脑上，那使用FRP进行内网穿透将是比较好的选择，有比较强的灵活性

主要能实现的功能：

1：外网通过域名访问自己电脑上任意端口程序；

2：外网ssh登录自己电脑；

3：总之云服务器能有的功能，大部分都可以部署到自己电脑上；

网上也有较多FRP内网穿透的文章，但是看的有点乱，我将结合实际穿透过程将每一步记录下来，大家按照步骤来是可以成功完成穿透的（可能看着步骤有点多，但是一步步完成实际还是简单的），总的来说只需要改三个文件（配置文件放在文末）：

**1：服务端的frps.ini文件**

**2：服务端的nginx.conf文件**

**3：客户端的frpc.ini文件**

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/adf08788a86c4add8abc13b31025d667~tplv-k3u1fbpfcp-zoom-1.image#?w=794\&h=141\&s=8818\&e=webp\&b=181818)

官网文档：<https://gofrp.org/docs/overview/>

## 需要的资源：

1：域名（没有域名只能通过ip进行访问），这里我申请的域名是：[www.***.com](https://www.leachchen.com)

2：一台云服务器（FRP内网穿透需要依赖公网服务器，若没有服务器的同学，可以在网上找其他内网穿透方案也有比较多，不过功能全点的要付费）

3：一台自己的物理电脑（我的是win10+ubuntu双系统，ubuntu系统作为物理服务器）

4：FRP客户端，github地址 <https://github.com/fatedier/frp/releases>

## 二、目标

我在自己物理电脑上安装了nginx(默认端口80)和gitlab(gitlab端口我设置成8000)

本地可以通过\*\*<http://127.0.0.1**访问网站>

本地可以通过\*\*<http://127.0.0.1:8000**访问代码服务器>

希望能做到：

1：通过<https://www.***.com> 访问物理电脑上的网站

2：通过<https://xxx.***.com> 访问物理电脑上的代码仓库(xxx，自己可以在域名平台配置你想要的域名前缀)

## 三、配置过程

### 我申请的域名是[www.***.com](https://www.***.com),假设服务器IP是100.10.10.10

### 第一步（云服务器FRP客户端配置）：

1：下载FRP客户端，下载地址上面有提供，我当时下载的最新版

2：解压

    1：tar xvf frp_0.44.0_linux_amd64.tar.gz
    2：cd frp_0.44.0_linux_amd64.tar.gz
    3：vim frps.ini  这里注意是frps.ini，客户端是frpc.ini

3：配置frps.ini（具体含义注意看配置里的注释）

```
[common]
# frp监听的端口，默认是7000，可以改成其他的
bind_port = 7000
# 授权码自己可以随便取，客户端配置需要与服务端配置保持一致
token = 123123

# 对user开放的web访问端口
vhost_http_port = 81

#通过ssh登录时的端口
[ssh]
listen_port = 6000

# frp管理后台端口，请按自己需求更改，可通过http://服务器IP:7500访问后台管理系统，如：http://100.10.10.10:7500
dashboard_port = 7500
# frp管理后台用户名和密码，请改成自己的
dashboard_user = admin
dashboard_pwd = admin
enable_prometheus = true

# frp日志配置
log_file = /var/log/frps.log
log_level = info
log_max_days = 3

```

4：运行：nohup ./frps -c ./frps.ini &

5：此时服务端会运行起以下端口：
**7000**，frp软件

**81**，服务端的web端口，比如通过<http://100.10.10.10:81就能访问到物理机上的nginx网站，不能用80端口会跟云服务端的nginx> 80端口冲突

**6000**，用于ssh登录的

**7500**，frp看板系统，可通过<http://服务器IP:7500进行访问>

### 第二步（客户端FRP客户端配置，在自己物理机上）：

1：下载FRP客户端，下载地址上面有提供，我当时下载的最新版

2：解压

    1：tar xvf frp_0.44.0_linux_amd64.tar.gz
    2：cd frp_0.44.0_linux_amd64.tar.gz
    3：vim frpc.ini

3：配置frpc.ini（具体含义注意看配置里的注释）

```
# 客户端配置
[common]
 #这里填你自己实际的域名，要是没有域名这里填云服务器公网ip
server_addr = www.***.com
#与frps.ini的bind_port一致
server_port = 7000
#与frps.ini的token一致
token = 123123

#配置ssh服务
[ssh]
type = tcp
local_port = 22
#与服务器ssh部分listen_port保持一致
remote_port = 6000

[web]
type = http
local_port = 80
#这里填你自己实际的域名，要是没有域名这里填云服务器公网ip
custom_domains = www.***.com 

[gitlab]
type = http
local_port = 8000
#这里填你自己实际的域名，要是没有域名这里填云服务器公网ip
custom_domains = xxx.***.com

```

4：运行：nohup ./frpc -c ./frpc.ini &

### 第三步（域名与云服务器IP绑定）

我的是阿里云的域名，不同服务商绑定过程都差不多，这里主要将www、gitlab这两个与云服务器IP进行绑定
![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c305d182551341929c1e94c92bf864f4~tplv-k3u1fbpfcp-zoom-1.image#?w=1500\&h=185\&s=5938\&e=webp\&b=fefdfd)

### 第四步（Nginx配置）

1：访问[www.***.com](https://www.***.com)时默认是80端口，此时将请求转发到81端口上，frp服务端和客户端保持了连接，此时会访问到客户端配置了相同域名的端口上

2：访问xxx.***.com时默认是80端口，此时将请求转发到81端口上，frp服务端和客户端保持了连接，此时会访问到客户端配置了相同域名的端口上

若不需要支持https，则将这几行注释掉：
listen 443 ssl;
ssl\_certificate /root/temp/ssl/xxx.pem;
ssl\_certificate\_key /root/temp/ssl/xxx.key;

nginx配置文件路径：/etc/nginx/nginx.conf，有些是在/etc/nginx/sites-enabled/default

    user root;
    worker_processes 4;
    pid /run/nginx.pid;

    events {
    	worker_connections 768;
    	# multi_accept on;
    }

    http {

    	include /etc/nginx/conf.d/*.conf;
    	include /etc/nginx/sites-enabled/*;

    	server {
                    listen 80;
    		listen 443 ssl;
                    ssl_certificate /root/temp/ssl/xxx.pem; #在域名服务商申请的证书，不同的域名证书不一样
                    ssl_certificate_key /root/temp/ssl/xxx.key; #在域名服务商申请的证书，不同的域名证书不一样

                    server_name  www.***.com;  #这里填你自己实际的域名
                    client_max_body_size 10M;
                    location / {
    					proxy_pass http://127.0.0.1:81;    
                                        #访问www.***.com时默认是80端口，此时将请求转发到81端口上，frp服务端和客户端保持了连接，此时会访问到客户端配置了相同域名的端口上
    					proxy_redirect http://$host/ http://$http_host/;
    					proxy_set_header X-Real-IP $remote_addr;
    					proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    					proxy_set_header Host $host;

                    }

            }

    	server {
                    listen 80;
                    listen 443 ssl;
                    ssl_certificate /root/temp/ssl/yyy.pem;     #在域名服务商申请的证书，不同的域名证书不一样 
                    ssl_certificate_key /root/temp/ssl/yyy.key; #在域名服务商申请的证书，不同的域名证书不一样

                    server_name xxx.***.com; #这里填你自己实际的域名
                    client_max_body_size 10M;
                    location / {
                        proxy_pass http://127.0.0.1:81;     #访问xxx.***.com时默认是80端口，此时将请求转发到81端口上，frp服务端和客户端保持了连接，此时会访问到客户端配置了相同域名的端口上
                        proxy_redirect http://$host/ http://$http_host/;
                        proxy_set_header X-Real-IP $remote_addr;
                        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                        proxy_set_header Host $host;

                    }

            }

    }

到此就完成了物理机上的网站，通过https域名进行外网访问

### SSH登录（通过putty登录）：

输入物理机的账号密码就可以进行登录了

![image.png](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/21726aff43a0407f8b4af807670eea30~tplv-k3u1fbpfcp-zoom-1.image#?w=697\&h=686\&s=45052\&e=webp\&b=f6f4f4)

### 配置文件下载：<https://github.com/leach-chen/FRP>

个人网站：<https://www.***.com/>
