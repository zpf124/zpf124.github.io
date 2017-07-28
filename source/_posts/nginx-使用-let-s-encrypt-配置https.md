---
title: nginx 使用 let's encrypt 配置https
date: 2016-04-02 23:44:34
categories: 
 - Operations
tags: 
 - linux
 - debian8
 - nginx
 - let's encrypt
---
# 前言

本文中使用的linux为debian8，nginx为最新版（1.9.9）
注：nginx自1.9.5后才默认包含http2模块，所以如果要开启http2需要使用1.9.5以上版本。

## 2016年6月30日更新
	注意现在let's Encrypt客户端已经改名，命令名称稍有变化，参数不变。

<!-- more -->


# 安装nginx
我采用的是通过`apt`进行安装，我们按照 [nginx官方文档][1] 中的教程进行操作。
我将其中对于debian系统的操作步骤单独截图出来，并附带图中的"[this key][2]"的链接。

![nginx官方安装步骤截图][3]

##  1. 为第三方源添加添加受信任的key

当添加第三方源时，需要为系统添加对应的密钥。

``` bash
    curl -sL http://nginx.org/keys/nginx_signing.key | sudo -E apt-key add -
```
##  2.  添加apt源列表
编辑`/etc/apt/source.list`文件，在其中添加以下内容。
也可以选择在`/etc/apt/source.list.d/`目前下新建`*.list`文件，并添加以下内容。

	deb http://nginx.org/packages/mainline/debian/ jessie nginx
	deb-src http://nginx.org/packages/mainline/debian/ jessie nginx

## 3. 更新apt软件列表，然后安装nginx

``` bash
sudo aptitude update && sudo aptitude install nginx
```

# 使用let's Encrypt程序生成证书

## 下载let's Encrypt程序。

[let'sEncrypt项目][4]发布在github上，所以需要安装git工具
``` bash
    sudo aptitude install git
    git clone https://github.com/letsencrypt/letsencrypt.git
```
## 运行程序
当运行let'sEncrypt程序时，程序会自动编译项目处理依赖关系，所以只需要进入程序目录，执行auto脚本即可。
```bash
    cd letsencrypt
    sudo ./letsencrypt-auto --help all
```
甚至最后那句可以直接这样指定程序参数，然后当项目编译完成后就会弹窗要求你填写相关信息，然后变自动生成了网站的证书。

```bash
    # 目前（2016年4月2日）let'sEncrypt还不支持nginx自动生成程序。
    # 所以需要先停止nginx服务器，然后使用standalone参数生成证书。
    sudo service nginx stop
    sudo ./letsencrypt-auto certonly --standalone 
```
生成的证书会存放在`/etc/letsencrypt/live/{domain}/`下
nginx 中用到的是`fullchain.pem` 和 `privkey.pem` 其他为apache使用的证书。

# nginx配置https

Mozilla 有一个[协助生成SSL配置的页面][5]各位可以尝试使用

 这一段直接贴我的配置吧
 
```conf
    server {
        #nginx 监听端口，443为默认https端口，ssl指使用https，http2为开启http2
        listen                  443 ssl http2;
        # 服务器名称
        server_name             example.org;

        # 网站根目录 和 无后缀时默认查找文件
        root                    /var/www/;
        index                   index.html;

        # 开启 ssl （其实实际是tls）
        ssl                     on;
        ssl_prefer_server_ciphers on;
        # 支持的加密协议
        ssl_protocols           TLSv1 TLSv1.1 TLSv1.2;
        # 支持的加密套件
        ssl_ciphers             HIGH:!RC4:!3DES:!aDSS:!aNULL:!kPSK:!kSRP:!MD5:@STRENGTH:+SHA1:+kRSA;
        # 定义session缓存大小
        ssl_session_cache       shared:TLSSL:16m;
        # 定义session过期时间
        ssl_session_timeout     10m;
        # https证书公钥
        ssl_certificate         /etc/letsencrypt/live/example.org/fullchain.pem;
        # https证书私钥 要注意保存！
        ssl_certificate_key     /etc/letsencrypt/live/example.org/privkey.pem;
        # nginx默认会使用Diffiel-Hellman交换密钥是1024位的，相对不安全，所以需要替换使用更安全的。
        #使用 openssl dhparam -out dh4096.pem 4096 可以生成，然后我将其与网站证书的密钥放到了一起
        ssl_dhparam             /etc/letsencrypt/live/example.org/dh4096.pem;

        # 禁止被外站frame嵌入引用
        add_header X-Frame-Options SAMEORIGIN;
        # 为响应头添加要求浏览器使用https重定向的 header
        add_header Strict-Transport-Security max-age=16000000;
    }
```
 

 
[1]: http://nginx.org/en/linux_packages.html "nginx安装文档"
[2]: http://nginx.org/keys/nginx_signing.key "nginx key"
[3]: /assets/images/20160201135825676.png "nginx官方安装步骤截图"
[4]: https://github.com/letsencrypt/letsencrypt "let's Encrypt"
[5]: https://mozilla.github.io/server-side-tls/ssl-config-generator/ "Mozilla SSL Configuration Generator"