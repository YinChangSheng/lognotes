# 使用 Let’s Encrypt 申请免费的 https 证书

标签（空格分隔）： https

---

## 必备条件

> 1.一个自主域名，拥有域名解析管理权限
> 2.一台linux服务器(windows类似)，并保证该服务器能访问到公网。

## Let’s Encrypt 工作的大致原理
`Let’s Encrypt` 工作的原理很简单，其实就是验证当前申请人对申请证书的是否拥有对相应的域名的所有权。

如果是就会使用`Let’s Encrypt`的根CA正式颁发给该域名相应的全套证书文件

一句话就是通过一些方式检查域名是不是你的，如果是就颁发，不是就报错

## 安装 Let’s Encrypt 软件
官方用python开发了一个证书生成软件`certbot`。

* ubuntu-16 上安装
```
$ sudo apt-get update
$ sudo apt-get install software-properties-common
$ sudo add-apt-repository ppa:certbot/certbot
$ sudo apt-get update
$ sudo apt-get install python-certbot-nginx
```
至此certbot就装好了

* 其他操作系统自行下载[https://certbot.eff.org/](https://certbot.eff.org/)
目前还没有提供windows的安装版本(截止2018年2月26号)

* certbot是如何检测你是否对该域名有权限呢？
> 1. certbot会向`Let’s Encrypt`服务器发起一个证书生成的申请。
> 2. `Let’s Encrypt`会返回一个很长的随机字符串，类似这样`R1iZB5e54rA5jNbShC--OvXunrkz1BhcLVSrSeBFJj0.M89SNuoalbWqJuvaWnSm7oUKuPgT8v6eUNG9TryJJQg`
> 3. 然后`Let’s Encrypt`会在公网上检查对应域名能否检查到上面返回的字符串。如果一致验证通过，不一致则会失败。
certbot将这种验证行为称为`challenge`，certbot支持多种`challenge`方式。其实不管任何方式certbot最终都制作这样的一件事罢了。
甚至你可以定制自己的`challenge`。多个`challenge`的集合叫做`plugins`.是的一个`plugins`支持至少一个`challenge`。
以下我们介绍简单也是常用的几个`plugins`。

目前certbot官方的`plugins`有以下几种

* `apache`          Use the Apache plugin for authentication & installation
* `standalone`      Run a standalone webserver for authentication
* `nginx`           Use the Nginx plugin for authentication & installation
* `webroot`         Place files in a server's webroot folder for authentication
* `manual`          Obtain certificates interactively, or using shell script hooks

## 常见使用方式
一下我们都使用 `open.yinchangsheng.cc` 和 `blog.yinchangsheng.cc` 来作为生成证书的域名

### 方式1 - 使用DNS解析的模式 - 该方式是由`manual`插件提供的`dns challenge`实现的
这种方式最简单，属于入门级。

```bash
$ certbot certonly --manual --preferred-challenges dns --domains open.yinchangsheng.cc,blog.yinchangsheng.cc
```

来解释一下：
> certonly 只获取证书不发生安装行为，这里的安装就是服务器配置。说白了就是只获取生成，其他的不做。
> --manual 使用`manual plugins`
> --preferred-challenges 指定该 `plugins` 中提供的插件 `dns challenge`
> --domains 验证的域名列表
以上语句的意思就是使用manual`plugins`中的dns`challenge`来获取域名`open.yinchangsheng.cc`,`blog.yinchangsheng.cc`的证书

输入以上语句后，会进入交互模式。
大概意思是，该服务器的公网ip是否是域名解析的ip。一般情况下输入`Y`确认。

```
-------------------------------------------------------------------------------
NOTE: The IP of this machine will be publicly logged as having requested this
certificate. If you're running certbot in manual mode on a machine that is not
your server, please ensure you're okay with that.

Are you OK with your IP being logged?
-------------------------------------------------------------------------------
(Y)es/(N)o: y

```
确认后会提示以下消息，大概意思是用你的DNS解析工具，
主机记录: _acme-challenge.open.yinchangsheng.cc 
记录类型: TXT
记录值: QC9Ohixx32HBbQVE-vjxdQeoV-Gin70x-CcMy1snjZs
完成解析后`Enter`键继续
```

-------------------------------------------------------------------------------
Please deploy a DNS TXT record under the name
_acme-challenge.open.yinchangsheng.cc with the following value:

QC9Ohixx32HBbQVE-vjxdQeoV-Gin70x-CcMy1snjZs

Before continuing, verify the record is deployed.
-------------------------------------------------------------------------------
Press Enter to Continue
```
如果出现以下信息，说明证书获取成功了。是不是很简单。
证书存放位置`/etc/letsencrypt/live/open.yinchangsheng.cc`
```
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/open.yinchangsheng.cc/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/open.yinchangsheng.cc/privkey.pem
   Your cert will expire on 2018-05-27. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

### 方式2 - 与 nginx 配合的模式
因为nginx的简单稳定高效，现在使用nginx作为web服务器的人越来越多。基本已经成为主流了。
所以配合nginx自动生成证书的方式自然是少不了的。这叫做开箱即用。
这种方式是使用`nginx plugins`和`webroot plugins`来完成的。不必多说主机上需要有，一个nginx服务器，并且监听在80端口上。

与上一个方法相比，这个方法不需要多余解析一个域名。只需要在`nginx`配置上做一点改动。
验证过程: `certbot`会向`Let’s Encrypt`发起申请。`Let’s Encrypt`会告诉`certbot`，将会请求验证域名下的`/.well-known/acme-challenge/`访问一个随机字符串。并能够得到期望的字符串，这个期望的字符串也是`Let’s Encrypt`返回的。

nginx 类似一下方法配置

/etc/nginx/snippets/ssl.conf

```nginx
ssl_session_timeout 1d;
ssl_session_cache shared:SSL:50m;
ssl_session_tickets off;

ssl_protocols TLSv1.2;
ssl_ciphers EECDH+AESGCM:EECDH+AES;
ssl_ecdh_curve secp384r1;
ssl_prefer_server_ciphers on;

ssl_stapling on;
ssl_stapling_verify on;

add_header Strict-Transport-Security "max-age=15768000; includeSubdomains; preload";
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
```

/etc/nginx/vhost.d/yinchangsheng.cc.conf

```nginx
server {
    server_name open.yinchangsheng.cc blog.yinchangsheng.cc;
    listen 80;
    listen 443 ssl default_server;
    include /etc/nginx/snippets/ssl.conf; # 该文件见付文
    access_log /var/log/nginx/access.log vhost;
    index index.html;

    location ^~ /.well-known/acme-challenge/ {  # 该路径为必须配置，属于验证协议的的一部分
      default_type "text/plain";
      root /var/www/letsencrypt; # 该目录需要创建，certbot 会在适当的时机将验证文本放在该目录下面
    }

    location / {
      try_files $uri $uri/ =404;
    }
}
```

```bash
$ sudo certbot --webroot -w /var/www/letsencrypt --domains open.yinchangsheng.cc,blog.yinchangsheng.cc
```
来解释一下：
> --webroot 指定一个验证器插件, 验证器其实就是我们之前说的`challenges`
> -w 这是webroot插件的选项，指代域名所在的web根目录。需要与nginx配置一致.
> --domains 需要申请证书的域名

如果返回如下信息，说明已经成功了
```
Waiting for verification...
Cleaning up challenges
Generating key (2048 bits): /etc/letsencrypt/keys/0005_key-certbot.pem
Creating CSR: /etc/letsencrypt/csr/0005_csr-certbot.pem
Cannot find a VirtualHost matching domain open.yinchangsheng.cc.

IMPORTANT NOTES:
 - Unable to install the certificate
 - Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/openrpc1.qurenjia.com/fullchain.pem. Your
   cert will expire on 2018-05-27. To obtain a new or tweaked version
   of this certificate in the future, simply run certbot again with
   the "certonly" option. To non-interactively renew *all* of your
   certificates, run "certbot renew"
```

也有这样的方式

```bash
$ sudo certbot --nginx # 这种方式是官方首页推荐的方式
```

但是该方式存在问题。

`Client with the currently selected authenticator does not support any combination of challenges that will satisfy the CA`

这句话的全写`certbot --authenticator nginx --installer nginx`是这样的

来解释一下：
> --authenticator 指定一个验证器插件, 验证器其实就是我们之前说的`challenges`
> --installer 指定一个安装器插件
plugins 可以提供两种功能，一个是`authenticator`一个就是`installer`。所以这里的意思就是用`webroot`插件的方式获取证书，用nginx的方式安装证书.

这里面有一个小问题，之前`nginx authenticator`是可以使用的。后来官方修改了`TLS-SNI`的验证方式。导致`nginx authenticator`失效了。

此后生成证书需要使用`webroot`提供的`http-01 challenge`的方式了。

[TLS-SNI Validation Will Remain Disabled For New Accounts](https://community.letsencrypt.org/t/2018-01-11-update-regarding-acme-tls-sni-and-shared-hosting-infrastructure/50188)

类似的方式还有

```bash
$ sudo certbot --authenticator webroot --installer nginx --domains open.yinchangsheng.cc,blog.yinchangsheng.cc
```

nginx installer 安装器原本的作用是自动修改配置文件。
但是无一例外都会出现 Cannot find a VirtualHost matching domain open.yinchangsheng.cc. 这样的一个消息。
所以 nginx installer 我基本没有用过。具体原因是应为太复杂的nginx配置文件`certbot-nginx`无法正常解析。我推荐手动配置`https`。自动安装的话毕竟不知道它对nginx都做了什么。

### 如果没有 80 端口
有的时候我们没有 80 端口的权限。虽然这种情况出现的概率低。但是贴心的`certbot`还是为我们考虑到这么一点。
`--http-01-port` 该选项可以帮助我们指定一个验证端口。`webroot`默认使用的是`http-01 challenge`

### 自动证书续期
续期是`certbot`提供的一个贴心功能。

```bash
$ certbot renew --dry-run
```

就会对之前所有申请过的证书进行续期。他是怎么做到的。
每当你申请证书成功后，都会在`/etc/letencrypt/renewal`下生成对应的配置文件。该配置文件记录了当时生成该域名证书的所有信息。`renew`就是让这个过程在发生一变。

我们只需要一个`cron`任务就可以证书自动续期。
```
* 3 * * 1 certbot renew --post-hook "systemctl reload nginx"
```
这样每周早上3点都会尝试更新证书。就不用担心证书过期问题了。

### 使用编程 api 的方式自动化申请证书
这部分内容待续...

### 注意事项
由于`letsencrypt`是一个非营利组织。
所以提供的证书类型比较单一。而且需要三个月续期一次。
并且不能对一个域名反复申请证书，否则会被拒绝请求。

## 参考文档
[Certbot documentation](http://letsencrypt.readthedocs.io/en/latest/)
[Certbot Website](https://certbot.eff.org)
[Set up Let’s Encrypt with Nginx web server with webroot plugin](https://agock.com/2017/01/set-lets-encrypt-nginx-web-server-webroot-plugin/)
[letsencrypt_2017.md](https://gist.github.com/cecilemuller/a26737699a7e70a7093d4dc115915de8)


