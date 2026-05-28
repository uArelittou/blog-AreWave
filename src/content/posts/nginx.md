---
title: Nginx学习笔记
published: 2026-05-27
description: 整理好nginx部署配置笔记
image: " "
tags: [Nginx]
category: 学习笔记
draft: false
---

## 下载

sudo apt install nginx -y 

*动态部署相关的前置下载：*

sudo apt install php php-fpm -y

sudo apt install php-mysql -y

### mysql 下载：

先去mysql官网登陆下载对应版本的安装包

解压:
tar -xf mysql-server_9.6.0-1debian13_amd64.deb-bundle.tar

然后安装:	

sudo apt install ./*.deb

*(可能会出现依赖缺失，补全：

sudo apt-get -f install)*

接下来要填写个root密码

完成后用查看运行状态:	

sudo systemctl status mysql

显示active（ruuning）就说明安装好了

mysql -uroot -p

enter password : <填写你设置的密码>

就登陆上去了




## 静态部署：

/etc/nginx/sites-available/default （端口和root文件）

/var/www/html   (网站配置文件夹)




## 动态部署：

### 创建数据库：
**进入mysql里**

create database word;

create user 'are'@'%' identified  by 'ql';

grant all privileges on word.* to 'are'@'%';

flush privileges;

exit

### 修改nginx配置文件：

在/etc/nginx/conf.d/下创建一个   .conf 的文件：

```
server{
        listen  2001;
        server_name www.arewave.net;
        root /home/are/web/word;
        location / {
                index index.php;
        } 

        location ~ \.php$ {
                fastcgi_pass 127.0.0.1:9000;
                fastcgi_index index.php;
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        }
}
```

### 检查语法：

sudo nginx -t


### 修改php配置文件：
sudo vi /etc/php/8.4/fpm/pool.d/www.conf

### 注释和加入： 
listen = /run/php/php8.4-fpm.sock

listen = 127.0.0.1:9000

sudo systemctl restart php8.4-fpm

可以检查一下端口（有没有2001和9000）：

sudo ss -tunlp



## workpress：

### 进入网站目录里安装
```
cd  < >
wget https://cn.wordpress.org/latest-zh_CN.tar.gz
tar -zxvf latest-zh_CN.tar.gz
rm -rf wordpress latest-zh_CN.tar.gz
```

### 放权：
```
sudo chown -R www-data:www-data <>\
sudo chown -R 755 <>
```

然后直接进入web控制台就好了(进不去可能是防火墙没放行端口)：

< ip >:2001  (自定义端口)

之后就把刚刚创建的专门的数据库和用户和数据库主机（127.0.0.1）填进去



## 反向代理：
cond下的独立配置文件：

```
server {
    # 1. 监听端口：Nginx 对外提供服务的端口
    listen 80;
    
    # 2. 域名：用户通过什么域名访问。如果没有域名，填服务器的公网 IP 也可以，或者填 localhost 仅供本地测
试
    server_name www.proxy1.com; 

    # 3. 核心路由规则：所有以 / 开头的请求，都走这里的规则
    location / {
        # 【关键】把请求转发到后端的 <> 端口
        proxy_pass http://<ip>:<prot>

        # --- 以下是保留真实用户信息的配置 ---
        # 默认情况下，后端程序会以为所有请求都是 Nginx (127.0.0.1) 发来的。
        # 加上这几行，Nginx 就会把用户的真实 IP 和域名一起打包发给后端                                                          
        # 默认情况下，后端程序会以为所有请求都是 Nginx (127.0.0.1) 发来的。
        # 加上这几行，Nginx 就会把用户的真实 IP 和域名一起打包发给后端。

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```

## 负载均衡：
主要加上这个模块和修改location下的proxy_pass

```
# 分摊负载的服务器ip和端口
upstream awa {
        server 127.0.0.1:7000;
        server 127.0.0.1:7022;
}


server {
    # 1. 监听端口：Nginx 对外提供服务的端口
    listen 80;
    
    # 2. 域名：用户通过什么域名访问。如果没有域名，填服务器的公网 IP 也可以，或者填 localhost 仅供本地测
试
    server_name www.proxy1.com; 

    # 3. 核心路由规则：所有以 / 开头的请求，都走这里的规则
    location / {
        # 【关键】把请求转发到后端的 8080 端口
        proxy_pass http://awa;

        # --- 以下是保留真实用户信息的配置 ---
        # 默认情况下，后端程序会以为所有请求都是 Nginx (127.0.0.1) 发来的。
        # 加上这几行，Nginx 就会把用户的真实 IP 和域名一起打包发给后端。
                                                                                      14,0-1       顶端
        # 默认情况下，后端程序会以为所有请求都是 Nginx (127.0.0.1) 发来的。
        # 加上这几行，Nginx 就会把用户的真实 IP 和域名一起打包发给后端。

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```



 ***如果要做实验的话可以用py脚本来占用端口模拟服务器负载：***
编辑好后在后台运行，在nginx conf配置文件下对接好端口就可以实验了

```
import sys
from http.server import BaseHTTPRequestHandler, HTTPServer

# 获取命令行传入的端口，改需要用的端口
port = int(sys.argv[1]) if len(sys.argv) > 1 else < prot >

class MyHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header("Content-type", "text/plain; charset=utf-8")
        self.end_headers()
        # 关键点：返回响应时，带上自己的端口号
        response = f"==== Hello! I am the backend running on PORT {port} ====\n"
        self.wfile.write(response.encode('utf-8'))

print(f"Backend is starting on http://127.0.0.1:{port}")
HTTPServer(('127.0.0.1', port), MyHandler).serve_forever()
```


## SSL证书：
先用openssl工具生成练习测试用的证书和密钥：
```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/nginx-selfsigned.key -out /etc/ssl/certs/nginx-selfsigned.crt
```
把之前做到配置文件备份更改：

```
# 负载均衡池（假设你后台还在跑着 7001 和 7002）
upstream my_backend_pool {
    server 127.0.0.1:7000;
    server 127.0.0.1:7022;
}

# 第一个 Server 块：看守 80 端口（HTTP）
server {
    listen 80;
    server_name www.proxy1.com;

    # 如果有人用不安全的 http 访问，强行把他一脚踢到 https 去
    return 301 https://$host$request_uri;
}

# 第二个 Server 块：看守 443 端口（HTTPS）
server {
    listen 443 ssl;
    server_name www.proxy1.com;

    # 告诉 Nginx 证书和私钥藏在哪里
    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

    location / {
        proxy_pass http://my_backend_pool;

        # 灵魂四件套（注意最后一行，此时传给后端的 scheme 就会变成 https）
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**写完记得用  nginx -t 检查语法问题**

然后把后端测试的py脚本启动后

curl  http://www.proxy1.com

返回301报错页面说明成功启用了强制https了

curl -k https://www.proxy1.com

发现成功正常返回，多返回几次，能发现运用了负载均衡返回了多个端口

没错，这一版本就是结合了反向代理，负载均衡，ssl的配置文件...



## 关于502问题：
返回502 Bad Gateway

先去看看日志：   /var/log/nginx/error.log

sudo tail -n 10 /var/log/nginx/error.log

大概率后端掉了

sudo ss -tunlp | grep -E "7000|7021"

检查一下，发现什么都没有返回，那就是了

启动对应脚本文件让后运行发现解决了

