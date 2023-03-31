---
layout: post
title:  "Nginx Stream SSL反代和mTLS验证"
date:   2023-03-31 15:05:00 +0800
category: Linux
---
# 场景描述

手头上有两台机器A机和B机。A机上通过lxd启动了几台容器虚拟机，上面部署了一些应用。由于历史原因，lxd上的桥接网络用的是nat的方案，所以和A机在同一个子网的B机是无法直接访问LXD上部署的应用。这里我们尝试通过Nginx来实现B机对LXD应用的访问。

我们在A机上启动一个nginx监听在443端口。我们的目标就是通过这个443端口能访问到环境上的各种服务，比如两台linux主机的ssh服务，一个http服务等等。

# 资源清单

1. A机的局域网ip为10.0.92.10， 端口 443
2. lxd上的Linux 服务器A， ip为 192.168.121.21，端口为22
3. lxd上的Linux 服务器B， ip为 192.168.121.22，端口为22
4. lxd上某个服务器启动的http，url 为 http://192.168.121.23:8080s
5. B机的局域网ip为10.0.92.11

# 初步方案

如果A机上可以开放多个端口，我们可以简单的通过四层的反代来实现这个功能，也就是stream模块。对于单端口复用，其实我们也并不陌生，HTTP的Server_name就是用来实现单端口对应多服务的。整体还是可以遵循这个思路，在A机上通过server name来区分不同的服务，对应在B机上也启动一个nginx，用于将不同的server name映射到不同的端口。

我们引导Bing chat，让它提供类似如下的配置。
A机主要配置：

```ini
stream {
    map_hash_bucket_size 64;
    map $ssl_preread_server_name $proxy_pass_target {

    ssh1.example.com 127.0.0.1:30001;
    ssh2.example.com 127.0.0.1:30002;
    web.example.com 127.0.0.1:30003;
    default 127.0.0.1:65535;
    }

    server {
        listen 443;
        ssl_preread on;
        proxy_pass $proxy_pass_target;
    }

    server {
        listen 127.0.0.1:30001 ssl;
        ssl_certificate /etc/nginx/ssl/example.com/server.crt;
        ssl_certificate_key /etc/nginx/ssl/example.com/server.key;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        proxy_pass 192.168.121.21:22;
    }

    server {
        listen 127.0.0.1:30002 ssl;
        ssl_certificate /etc/nginx/ssl/example.com/server.crt;
        ssl_certificate_key /etc/nginx/ssl/example.com/server.key;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        proxy_pass 192.168.121.22:22;
    }

    server {
        listen 127.0.0.1:30003 ssl;
        ssl_certificate /etc/nginx/ssl/example.com/server.crt;
        ssl_certificate_key /etc/nginx/ssl/example.com/server.key;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        proxy_pass 192.168.121.23:8080;
    }
}
```

B机主要配置项

```ini
stream {
    server {
        listen 50001;
        proxy_ssl on;
        ssl_certificate /etc/nginx/ssl/example.com/server.crt;
        ssl_certificate_key /etc/nginx/ssl/example.com/server.key;
        proxy_ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        proxy_ssl_server_name on;
        proxy_ssl_name ssh1.example.com;
        proxy_pass 10.0.92.10:443;
    }

    server {
        listen 50002;
        proxy_ssl on;
        ssl_certificate /etc/nginx/ssl/example.com/server.crt;
        ssl_certificate_key /etc/nginx/ssl/example.com/server.key;
        proxy_ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        proxy_ssl_server_name on;
        proxy_ssl_name ssh2.example.com;
        proxy_pass 10.0.92.10:443;
    }
}
```

# 自签名证书

有了上面这个配置，我们只需要创建对应的证书就能部署到环境上。

如果是生产环境建议通过可信的CA来签发证书，Let's Encrypt/ZeroSSL是不错的选择,如果你有公网ip，并且80端口可以被访问，那么可以通过校验文件的方式来签发证书；否则可以通过DNS上设置TXT值的方式来签发证书。这种免费证书的有效期一般是3个月，所以证书更新也是一个比较麻烦的运维开销，当然我们也可以通过一些bot来自动更新证书。

阿里云账号给每个用户提供了20个免费证书额度，这种证书的有效期是一年。

此处我们采用自签名证书，签发的脚本如下：

```ini
function create(){
HOST=$1
DAY=$2

# 创建ca私钥，服务器私钥，ca证书
openssl genrsa -out ca.key 2048
openssl genrsa -out server.key 2048
openssl req -new -x509 -days $DAY -key ca.key -subj "/CN=${HOST}" -out ca.crt

# 创建服务器证书签名请求
openssl req -new \
-key server.key \
-subj "/C=CN/OU=a/O=b/CN=c" \
-reqexts SAN \
-config <(cat /etc/pki/tls/openssl.cnf \
<(printf "\n[SAN]\nsubjectAltName=DNS:${HOST}")) \
-out server.csr

# 通过ca颁发服务器证书
openssl x509 -req -days $DAY \
-in server.csr -out server.crt \
-CA ca.crt -CAkey ca.key -CAcreateserial \
-extensions SAN \
-extfile <(cat /etc/pki/tls/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:${HOST}"))
}


# 接收两个参数 地址1, 有效期天数
create "*.example.com" 3650
```

这个脚本签发的证书有两个特点，一个是通配,所有example.com后缀的域名都能使用这个证书；另一个是有效期是3650天，这样就可以避免经常更新证书。

我们可以分别给A机和B机生成两套证书，也就是说我们在文中看到A机和B机上的证书文件虽然文件名是一致，其实我们给的是不同的证书，不要混淆了。

将证书拷贝到对应目录之后，我们试着启动下两边的Nginx进程，然后在B机下执行：

```shell
ssh -p 50001 testuser@127.0.0.1
ssh -p 50002 testuser@127.0.0.1
curl -so /dev/null -w "%{http_code}" https://web.example.com/ -k
```

![](https://f.003721.xyz/2023/03/a62cbce45d8b9d1ab8d512d84f3619c9.png)

发现功能都正常.这里我们并没有在B机的Nginx中将web.example.com 转成http，而是直接通过curl去访问A机的https服务，并通过curl的-k指令去绕过证书安全性校验。q如果不想用-k参数，可以带上A机的ca证书来访问：
```
curl -so /dev/null --cacert /pathtoserverca/ca.crt  -w "%{http_code}" https://web.example.com/
```
上面这个指令可用的前提是web.example.com这个域名能解析到A机ip，也就是10.0.92.10，在测试环境上我们可以通过在B机的/etc/hosts增加类似如下的记录来实现：
```
10.0.92.10 web.example.com
```

生产环境上当然是通过正规DNS设置来做到这一点了。


当然我们也可以在B机增加如下配置，将https offload成http，然后直接访问http。
```
    server {
        listen 50003;
        proxy_ssl on;
        ssl_certificate /etc/nginx/ssl/example.com/server.crt;
        ssl_certificate_key /etc/nginx/ssl/example.com/server.key;
        proxy_ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        proxy_ssl_server_name on;
        proxy_ssl_name web.example.com;
        proxy_pass 10.0.92.10:443;
    }
```

`curl -so /dev/null -w "%{http_code}" http://127.0.0.1:50003`

我们关注下A机上的配置文件，首先是如下配置块：
```
server {
listen 443;
ssl_preread on;
proxy_pass $proxy_pass_target;
}
```
这个配置文件在最外层用到了 ngx_stream_core_module 模块，这个配置并不是默认启用的，需要在nginx编译时带上--with-stream参数。这个模块的listen参数用于指定这个server监听在哪个端口。

proxy_pass 这个配置是ngx_stream_proxy_module模块的配置。

ngx_stream_proxy_module可以对TCP、UDP或者linux 的sockets的数据流做反向代理。proxy_pass 参数指定了后端的地址，这里跟着的是个变量$proxy_pass_target。

我们再来看看下面这段配置:
```
map_hash_bucket_size 64;
map $ssl_preread_server_name $proxy_pass_target {

ssh1.example.com 127.0.0.1:30001;
ssh2.example.com 127.0.0.1:30002;
web.example.com 127.0.0.1:30003;
default 127.0.0.1:65535;
}
```
这段配置用到的模块是ngx_stream_map_module。map模块用于设置两个变量之间的映射关系。这个例子中，我们是根据$ssl_preread_server_name这个变量的取值来确认$proxy_pass_target的值。

map_hash_bucket_size设置的是hash的bucket的大小，这个值一般设置成和CPU的cache size一致，也就是64.

虽然前面listen的端口是443，但是由于listen后面没有接ssl参数，所以在这一步nginx并没有做ssl 的offload，只是流量透传，按正常的流程是没法获取server_name相关参数。这里映射用到的参数是ssl_preread_server_name，这个参数是通过ngx_stream_ssl_preread_module模块来获取的，我们在server里指定了ssl_preread on。ngx_stream_ssl_preread_module可以在不做ssl offload前提下，从ClientHello消息中提取出一些信息，比如server_name.

Nginx在对TCP/UDP会话处理时，有如下一些阶段：Post-accept，Pre-access，Access，SSL，Preread，Content和Log。ngx_stream_ssl_preread_module就工作在Preread阶段。

在这个配置中，ssh1.example.com是映射到127.0.0.1:30001，也就是说当我们ssl_preread_server_name取到ssh1.example.com值时，ngx_stream_proxy_module的proxy_pass会将我们的连接转发到127.0.0.1:30001。这个127.0.0.1:30001在下面这个server配置块中定义：

```

server {
listen 127.0.0.1:30001 ssl;
ssl_certificate /etc/nginx/ssl/example.com/server.crt;
ssl_certificate_key /etc/nginx/ssl/example.com/server.key;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
proxy_pass 192.168.121.21:22;
}

```
这个配置块是一个典型的ngx_stream_ssl_module配置，ssl_certificate指定了server的证书，ssl_certificate_key是对应的secret key文件。ssl_protocols则指定了支持的SSL协议版本。

ngx_stream_ssl_module实现了SSL的offload，或者说终结了SSL，后端是透明的TCP/UDP端口.当然我们这里的22端口其实也是加密的通道，不过这个加密就和nginx无关， 对于nginx来说它只是一个普通的tcp流。 下面这个30003转发的后端是8080端口，这个可以更清晰的看出后端是普通TCP端口：

```
server {
listen 127.0.0.1:30003 ssl;
ssl_certificate /etc/nginx/ssl/example.com/server.crt;
ssl_certificate_key /etc/nginx/ssl/example.com/server.key;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
proxy_pass 192.168.121.23:8080;
}
```

我们上面提到的模块，很多都不是默认的编译参数，需要在编译中开启对应选项，我们可以通过nginx -V检查我们的版本是不是已经支持了这些模块:

```
/# nginx -V
nginx version: nginx/1.23.1
built by gcc 10.2.1 20210110 (Debian 10.2.1-6)
built with OpenSSL 1.1.1n  15 Mar 2022
TLS SNI support enabled
configure arguments: --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-g -O2 -ffile-prefix-map=/data/builder/debuild/nginx-1.23.1/debian/debuild-base/nginx-1.23.1=. -fstack-protector-strong -Wformat -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -Wl,--as-needed -pie'
```

我们再来分析下B机上的Nginx配置。由于服务端的端口是被套了一层SSL，因此我们访问这些服务需要通过ssl协议。对于https来说没什么问题，192.168.121.23:8080被a机nginx反向代理之后由于套上了SSL，等效于https。这就是为什么我们一开始可以通过curl 指令的https直接访问到192.168.121.23:8080的服务。而对于ssh协议，我们需要一个中间人来做终结这个SSL，这就是为什么我们引入如下配置:

```
server {
listen 50001;
proxy_ssl on;
ssl_certificate /etc/nginx/ssl/example.com/server.crt;
ssl_certificate_key /etc/nginx/ssl/example.com/server.key;
proxy_ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
proxy_ssl_server_name on;
proxy_ssl_name ssh1.example.com;
proxy_pass 10.0.92.10:443;
}
```

这个配置块中proxy_ssl是ngx_stream_ssl_module模块的配置，它指示nginx和后端之间走SSL协议。proxy_ssl_protocols指定了支持的SSL协议类型。proxy_ssl_server_name on这个配置则开启了SNI功能，通过下面的proxy_ssl_name设置server_name.

这段配置中其实有两个冗余的参数，就是ssl_certificate和ssl_certificate_key。由于listen中并没有加ssl后缀，因此这段server配置启动的监听端口是普通的TCP端口，无需加SSL，这两个选项是无效的，我们将它们去除，整个功能是不受影响的。

所以整改后的B机配置如下所示：
```
stream {
server {
listen 50001;
proxy_ssl on;
proxy_ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
proxy_ssl_server_name on;
proxy_ssl_name ssh1.example.com;
proxy_pass 10.0.92.10:443;
}

server {
listen 50002;
proxy_ssl on;
proxy_ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
proxy_ssl_server_name on;
proxy_ssl_name ssh2.example.com;
proxy_pass 10.0.92.10:443;
}
}
```

涉及的模块：ngx_stream_core_module，ngx_stream_proxy_module，和ngx_stream_ssl_module。

这里我们就有个疑问，为啥原先的配置里，B机会有冗余的ssl_certificate和ssl_certificate_key。我查看了作者的描述，他的初始意图其实是想做双向认证，让服务端也校验客户端的SSL，或者说他想实现一个mTLS流程。

什么是mTLS，mTLS是双向传输层安全（Mutual Transport Layer Security）的缩写，是一种双向认证的方法。mTLS通过验证双方都拥有正确的私钥和TLS证书来确保网络连接两端的身份。

我们先看看单向的TLS认证流程：

客户端向服务器发送一个ClientHello消息，包含客户端支持的加密算法和协议版本，以及一个随机数（ClientRandom）。

服务器从客户端支持的列表中选择一个加密算法和协议版本，并向客户端发送一个ServerHello消息，包含服务器选择的加密算法和协议版本，以及一个随机数（ServerRandom）和服务器的TLS证书。

客户端验证服务器的证书是否有效，是否由可信任的CA签发，是否与服务器域名匹配。如果验证通过，客户端用Diffie-Hellman或Elliptic Curve Diffie-Hellman算法生成一个预主密钥（Pre-Master Secret），并用服务器的公钥加密它。

服务器用自己的私钥解密客户端发送的预主密钥，并用它和ClientRandom和ServerRandom生成主密钥（Master Secret）。

客户端和服务器使用主密钥来生成对称加密密钥、MAC密钥和初始化向量，并分别发送ChangeCipherSpec消息来通知对方使用新生成的参数来加密后续通信。双方还会发送Finished消息来确认握手过程完成。

客户端和服务器使用对称加密算法、MAC算法和初始化向量来加密和解密数据传输。

而mTLS流程则是这样的：

客户端向服务器发送一个ClientHello消息，包含客户端支持的加密算法和协议版本，以及一个随机数（ClientRandom）。

服务器从客户端支持的列表中选择一个加密算法和协议版本，并向客户端发送一个ServerHello消息，包含服务器选择的加密算法和协议版本，以及一个随机数（ServerRandom）和服务器的TLS证书。服务器还会发送一个CertificateRequest消息，要求客户端也提供证书。

客户端验证服务器的证书是否有效，是否由可信任的证书颁发机构（CA）签发，是否与服务器域名匹配。如果验证通过，客户端向服务器发送自己的TLS证书，并用Diffie-Hellman或Elliptic Curve Diffie-Hellman算法生成一个预主密钥（Pre-Master Secret），并用服务器的公钥加密它。客户端还会发送一个CertificateVerify消息，用自己的私钥对前面所有交换过的消息进行签名。

服务器验证客户端的证书是否有效，是否由可信任的CA签发。如果验证通过，服务器用自己的私钥解密客户端发送的预主密钥，并用它和ClientRandom和ServerRandom生成主密钥（Master Secret）。服务器还会验证客户端发送的CertificateVerify消息中的签名是否正确。

客户端和服务器使用主密钥来生成对称加密密钥、MAC密钥和初始化向量，并分别发送ChangeCipherSpec消息来通知对方使用新生成的参数来加密后续通信。双方还会发送Finished消息来确认握手过程完成。

客户端和服务器使用对称加密算法、MAC算法和初始化向量来加密和解密数据传输。

mTLS并不神秘，我们日常使用的kubectl其实就用到了mTLS。

对于我们这个案例来说，我们不仅没实现mTLS认证，甚至连单向认证都是不全的。假设我们身处一个复杂的网络环境，随时可能出现有人仿冒A机提供钓鱼网站，骗取我们的信任，也就是说我们原则上认为A机不可信，除非有CA给它背书。

如果A机的证书是可信的CA签发的，B机自然是能信任它，但是我们前面配置时，采用的是自签名的方式，因此B机如果没有提前导入A机的CA证书，是不能信任A机的证书，这就是为什么我们通过curl 访问A机https时，不指定CA证书，并且不加-k选项绕过校验，访问是会失败的。如果我们希望B机的Nginx同样需要通过指定CA来校验A机是可信的，我们可以使用ngx_stream_proxy_module的proxy_ssl_verify相关参数来实现这个功能，我们在B机的nginx配置中加入如下配置：

```
proxy_ssl_verify on;
proxy_ssl_trusted_certificate /etc/nginx/ssl/ca_for_a.crt;
proxy_ssl_verify_depth 6;
```

proxy_ssl_verify 为on时，代表我们需要校验A机是否可信；proxy_ssl_trusted_certificate 设置的是用于校验对端证书的CA，这里的ca_for_a.crt就是我们在A机上创建的自签名证书中的CA； proxy_ssl_verify_depth 定义的是对端证书链的深度，如果对端证书不是由根证书直接签发，经过了若干intermediate CA，这里的值就要适度设置下。

修改后B机的配置如下：
```
stream {
server {
listen 50001;
proxy_ssl on;
proxy_ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
proxy_ssl_verify on;
proxy_ssl_trusted_certificate /etc/nginx/ssl/ca_for_a.crt;
proxy_ssl_verify_depth 6;
proxy_ssl_server_name on;
proxy_ssl_name ssh1.example.com;
proxy_pass 10.0.92.10:443;
}

server {
listen 50002;
proxy_ssl on;
proxy_ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
proxy_ssl_verify on;
proxy_ssl_trusted_certificate /etc/nginx/ssl/ca_for_a.crt;
proxy_ssl_verify_depth 6;
proxy_ssl_server_name on;
proxy_ssl_name ssh2.example.com;
proxy_pass 10.0.92.10:443;
}
}
```

配置生效后，我们尝试连接ssh，发现可以正常连接。

如果这时候有个机器冒出来，声称自己就是A机，可以提供这个nginx服务，我们如何从真假大圣中辨别出谁是六耳猕猴呢？这里我们可以尝试把B机的/etc/nginx/ssl/ca_for_a.crt修改成其它CA文件，也就是让B机的proxy_ssl_trusted_certificate和A机的证书不匹配。这样我们再次测试时，会得到如下的失败结果：

```
$ ssh -p 50002 xxx@127.0.0.1
kex_exchange_identification: read: Connection reset by peer
```

当然在真实的场景下，B机的ca_for_a.crt是没有改变的，上面的流程只是做个等效的验证，我们就是需要通过这个CA来校验A机的可信。变动的是仿冒A机上的证书，由于它的证书并不是我们受信CA签发的，因此不能通过B机Nginx的验证。

至此我们完成了单向TLS认证，下面我们可以继续实现mTLS。

为了完成mTLS流程，我们需要在两边的nginx都做配置，首先是B机上的Nginx，我们需要在ngx_stream_ssl_module模块中增加proxy_ssl_certificate和proxy_ssl_certificate_key两个参数，用于指定同A机进行认证的证书文件。具体配置如下：

```
stream {
server {
listen 50001;
proxy_ssl on;
proxy_ssl_certificate /etc/nginx/ssl/example.com/server.crt;
proxy_ssl_certificate_key /etc/nginx/ssl/example.com/server.key;
proxy_ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
proxy_ssl_verify on;
proxy_ssl_trusted_certificate /etc/nginx/ssl/ca_for_a.crt;
proxy_ssl_verify_depth 6;
proxy_ssl_server_name on;
proxy_ssl_name ssh1.example.com;
proxy_pass 10.0.92.10:443;
}

server {
listen 50002;
proxy_ssl on;
proxy_ssl_certificate /etc/nginx/ssl/example.com/server.crt;
proxy_ssl_certificate_key /etc/nginx/ssl/example.com/server.key;
proxy_ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
proxy_ssl_verify on;
proxy_ssl_trusted_certificate /etc/nginx/ssl/ca_for_a.crt;
proxy_ssl_verify_depth 6;
proxy_ssl_server_name on;
proxy_ssl_name ssh2.example.com;
proxy_pass 10.0.92.10:443;
}
}
```

重启nginx生效后，我们发现ssh和https功能还是正常的，由于服务端没有开启验证，这一步不会引入失败。

然后我们回到A机，nginx需要增加ssl_verify_client相关配置，这是ngx_stream_ssl_module模块的参数。

ssl_verify_client 有 on,off,optional和optional_no_ca几个选项。

off选项是默认值，也就是默认不开启对客户端的证书校验。

如果配置成on，则 ssl_client_certificate 是必选的，这个参数指定的文件需要提供所有服务端信任的CA证书列表，这些证书会在TLS握手阶段传给客户端，客户端据此判断自己要发给服务端的证书是不是被服务端所认可。

如果配置成optional，ssl_client_certificate 也是必选的，但是客户端可以选择不发送自己的证书，在这种情况下，服务端不会强制去校验客户端的证书。

如果配置成optional_no_ca，情形就比较复杂，这时候ssl_client_certificate就不是必须的。在这个选项下，只要客户端不主动发证书，服务端都不会去校验客户端证书。

1）如果ssl_client_certificate没有配置，服务端不会把自己信任的CA证书发给客户端。 这时候，客户端不发证书或者发一个不受信任证书都能通过。 

2）在ssl_client_certificate没有配置的前提下， ssl_trusted_certificate 如果设置了，并且客户端发了证书的情况下，服务端会根据ssl_trusted_certificate校验客户端证书的可信情况。

3）如果ssl_client_certificate有值，则用它来校验客户端证书的可信，忽略ssl_trusted_certificate的值。


根据上面的分析，对于A机的Nginx，我们有了如下的mTLS配置：

```
stream {
map_hash_bucket_size 64;
map $ssl_preread_server_name $proxy_pass_target {

ssh1.example.com 127.0.0.1:30001;
ssh2.example.com 127.0.0.1:30002;
web.example.com 127.0.0.1:30003;
default 127.0.0.1:65535;
}

server {
listen 443;
ssl_preread on;
proxy_pass $proxy_pass_target;
}

server {
listen 127.0.0.1:30001 ssl;
ssl_certificate /etc/nginx/ssl/example.com/server.crt;
ssl_certificate_key /etc/nginx/ssl/example.com/server.key;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;

ssl_verify_client on;
ssl_verify_depth 6;
ssl_client_certificate  /etc/nginx/ssl/ca_for_b.crt;
# ssl_trusted_certificate    /etc/nginx/ssl/ca_for_b.crt;

proxy_pass 192.168.121.21:22;
}

server {
listen 127.0.0.1:30002 ssl;
ssl_certificate /etc/nginx/ssl/example.com/server.crt;
ssl_certificate_key /etc/nginx/ssl/example.com/server.key;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;

ssl_verify_client on;
ssl_verify_depth 6;
ssl_client_certificate  /etc/nginx/ssl/ca_for_b.crt;
# ssl_trusted_certificate    /etc/nginx/ssl/ca_for_b.crt;

proxy_pass 192.168.121.22:22;
}

server {
listen 127.0.0.1:30003 ssl;
ssl_certificate /etc/nginx/ssl/example.com/server.crt;
ssl_certificate_key /etc/nginx/ssl/example.com/server.key;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;

ssl_verify_client on;
ssl_verify_depth 6;
ssl_client_certificate  /etc/nginx/ssl/ca_for_b.crt;
# ssl_trusted_certificate    /etc/nginx/ssl/ca_for_b.crt;

proxy_pass 192.168.121.23:8080;
}
}
```

根据这个配置，我们可以通过修改客户端的证书，验证下服务端是不是真正校验了客户端的证书。这里就不一一展开。


我们再重新看下A机的Nginx配置，这个配置还是比较复杂的，套了两层的反向代理：第一层通过SSL preread取到server name之后，映射到下面一层反代。第二次反代上实现了SSL的termination。既然我们每个请求都要在第二层做SSL/TLS的卸载，我们是不是也可以在第一层直接把它offload了呢？于是对于A机的Nginx，我们有如下这个配置：

```
stream {
map_hash_bucket_size 64;

map $ssl_server_name $proxy_pass_target {
ssh1.example.com 192.168.121.21:22;
ssh2.example.com 192.168.121.22:22;
web.example.com 192.168.121.23:8080;
default 127.0.0.1:65535;
}

server {
listen 443 ssl;
ssl_certificate /etc/nginx/ssl/example.com/server.crt;
ssl_certificate_key /etc/nginx/ssl/example.com/server.key;
ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
ssl_verify_client on;
ssl_client_certificate /etc/nginx/ssl/ca_for_b.crt;
ssl_verify_depth 6;
#ssl_trusted_certificate /etc/nginx/ssl/ca_for_b.crt;
proxy_pass $proxy_pass_target;
}
}
```
主要区别在于 map中 $ssl_preread_server_name 被 $ssl_server_name  取代，因为新配置已经没有配置ssl_preread，直接将SSL卸载了，因而可以直接取到$ssl_server_name 。map里的后端也直接指到真实的后端，不再绕一圈。证书和校验相关的配置也挪到第一层了，listen后面现在也不是单纯的四层反代，而是ssl。这个配置功能和之前的版本也是一致的。


# 一点引申
在生产环境上，我们有时会碰到这样的需求，应用的集群部署在一个隔离的内网环境上，只有DMZ上的实例具备有限的公网访问权限。而这时候，应用需要访问多个公网上的https API，比如https://bing.domaina.com 和https://oapi.dingtalk.com。这时候我们需要通过DMZ机器做请求转发。这种转发实现方式有多种，比如在DMZ机器上部署一个正向代理，应用请求这些API时需要指定通过代理访问；或者在DMZ机器上做一个透明代理，应用机器把API的ip路由到DMZ机器上。 

受上面案例启发，我们也可以部署一个nginx，通过反向代理实现这个功能。https的反向代理其实我们用得很多，一般都是7层反代，这就需要在nginx上配置域名的证书，这些公网API并不是我们运营的，他们也不可能把私钥发给我们，7层代理方式也就走不通。这时候我们可以用上面提到的ssl_preread，这种模式并没有对SSL做卸载，也就不需要配置私钥和key，我们只要获取到$ssl_preread_server_name就可以做分域名的转发：

```
map_hash_bucket_size 64;
map $ssl_preread_server_name $proxy_pass_target {

bing.domaina.com bing.domaina.com:443;
oapi.dingtalk.com oapi.dingtalk.com:443;
default 127.0.0.1:65535;
}

server {
listen 443;
ssl_preread on;
proxy_pass $proxy_pass_target;
}

```

在DMZ机器上部署这个nginx之后，我们在客户机上手工将bing.domaina.com和oapi.dingtalk.com通过host记录指向DMZ机器，然后调用相关api，发现没有获得正确的结果:

```
curl https://oapi.dingtalk.com
curl: (35) schannel: failed to receive handshake, SSL/TLS connection failed
```

检查nginx日志，发现如下日志：

```
2023/03/17 08:19:45 [error] 23#23: *1 no resolver defined to resolve oapi.dingtalk.com, client: 10.xx.xx.xx, server: 0.0.0.0:443, bytes from/to client:0/0, bytes from/to upstream:0/0
```

于是我们修正下配置,增加dns解析配置：

```
resolver 223.5.5.5 valid=300s;
resolver_timeout 10s;

map_hash_bucket_size 64;
map $ssl_preread_server_name $proxy_pass_target {

bing.domaina.com bing.domaina.com:443;
oapi.dingtalk.com oapi.dingtalk.com:443;
default 127.0.0.1:65535;
}

server {
listen 443;
ssl_preread on;
proxy_pass $proxy_pass_target;
}
```

这次可以正常获得结果:

```
curl https://oapi.dingtalk.com
{"errcode":404,"errmsg":"请求的URI地址不存在"}
```

写到这里不得不提一下，HTTPS虽说是安全的，但是你访问哪个域名还是可以被轻易的探知，这也是防火墙实施SNI阻断，或者审计系统审计你摸鱼行为的原理。对于审计系统来说，如果网络控制严格些，也可以强制设备安装根证书，这样所有的流量都没有私密性可言。

参考文档：

<http://nginx.org/en/docs/stream/ngx_stream_core_module.html>
<http://nginx.org/en/docs/stream/ngx_stream_proxy_module.html>
<http://nginx.org/en/docs/stream/ngx_stream_ssl_module.html>
<http://nginx.org/en/docs/stream/ngx_stream_map_module.html>
<http://nginx.org/en/docs/stream/ngx_stream_ssl_preread_module.html>