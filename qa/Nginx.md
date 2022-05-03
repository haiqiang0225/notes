[toc]

# Nginx

## 安装部署

### Nginx发行版

[nginx版本对比](https://www.cnblogs.com/lizexiong/p/15003543.html)

- [Nginx 开源版](https://nginx.org/)，功能简单，很多功能需要二次开发。
- [Nginx plus 商业版](https://www.nginx.com/)，商业版本，需要付费，功能全面。
- [Openresty](http://openresty.org/cn/)，以lua脚本的形式扩展功能。
- [Tengine](http://tengine.taobao.org/)，用C语言的形式扩展功能。

### 安装开源版

Linux版本：`Ubuntu 20.04 LTS`

[nginx: download](http://nginx.org/en/download.html)

下载后解压。

![image-20220501232840692](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/image-20220501232840692.png)

![image-20220501233011895](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/nginx_download_02.png)

执行命令解压：

```bash
tar -xzvf nginx-1.20.2.tar.gz
```

解压完成后，在Nginx根目录执行

```bash
./configure 
```

发现报错，安装依赖：

```bash
apt-get install libpcre3 libpcre3-dev libssl-dev zlib1g zlib1g-dev openssl libssl-dev -y
```

编译安装：

```bash
make 
make install
```

进入Nginx目录，启动Nginx：

```bash
cd /usr/local/nginx/sbin
./nginx
```

测试：

![image-20220501234202931](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/image-20220501234202931.png)

## 启动Nginx

```bash
./nginx						启动
./nginx -s stop 	快速停止
./nginx -s quit		在退出前完成已经接受的连接请求
./nginx -s reload	重新加载配置
```

 

## 将Nginx添加到系统服务

```bash
vim /usr/lib/systemd/system/nginx.service
```

添加：

```bash
[Unit]
Description=nginx - high performance web server
After=network.target remote-fs.target nss-lookup.target

[Service]
Type=forking
PIDFile=/usr/local/nginx/logs/nginx.pid
ExecStartPre=/usr/local/nginx/sbin/nginx -t -c /usr/local/nginx/conf/nginx.conf
ExecStart=/usr/local/nginx/sbin/nginx -c /usr/local/nginx/conf/nginx.conf
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop
ExecQuit=/usr/local/nginx/sbin/nginx -s quit
PrivateTmp=true
[Install]
WantedBy=multi-user.target
```

重新加载系统服务：

`systemctl daemon-reload`

启动服务：

`systemctl start nginx.service`

如果提示`Failed to start nginx.service: Unit nginx.service is masked.`

执行：`systemctl unmask nginx.service`然后再次启动。

设置开机启动：

`systemctl enable nginx.service`




## 配置虚拟主机与域名解析

网站有多个子域名，主机只有一台，只有一个公网IP，但是想根据域名的不同来解析到不同的页面，这个时候就可以使用Nginx的虚拟主机配置来实现。

比如`blog.seckill.cc`和`seckill.cc`，解析的地址是同一台主机的公网ip地址，但是我想解析到不同的网页。

```bash
vim /usr/local/nginx/conf/nginx.conf
```

配置如下：

```bash
worker_processes  8;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  65;
		
		# 虚拟主机1 解析seckill.cc
    server {
        listen       80;
        server_name  seckill.cc;
        
        # 文件查找位置，root代表查找html文件的根目录，这里使用的相对目录，即Nginx根目录下的html目录。
        location / {
            root   html;
            index  index.html index.htm;
        }
        
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

    }
		# 虚拟主机2 解析blog.seckill.cc
    server {
        listen       80;
        server_name  blog.seckill.cc;

        location / {
            root    html/blog;
            index   index.html index.htm;
        }

        error_page    500 502 503 504   /50x.html;
        location = /50x.html {
            root    html/blog;
        }
    }

}
```

创建blog域名的html文件

```bash
cd /usr/local/nginx/html
mkdir -p /usr/local/nginx/html/blog
cp *.html ./blog/
cd blog
vim index.html

```

```html
<!DOCTYPE html>
<html>
<head>
<title>Hello, World!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
blog!
</body>
</html>
```

重新加载Nginx

```bash
systemctl reload nginx
```

![image-20220502170814837](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/nginx_download_04.png)

![image-20220502170838321](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/nginx_download_05.png)

不同的域名解析到了不同的html。

### ServerName匹配

servername匹配分先后顺序，写在前面的匹配上就不会继续往下匹配了。

- 完整匹配

    ```bash
    server_name  seckill.cc pay.seckill.cc
    ```

- 通配符匹配

    ```bahs
    server_name  *.seckill.cc
    或者
    server_name  seckill.*
    ```

- 正则匹配

    ```bash
    server_name  ~^[0-9]+\.seckill\.cc$;
    ```

## 反向代理

- 正向代理

    典型的正向代理就是VPN，我们作为客户端，是知道服务端是谁的，比如Google。我们将对Google的请求发给VPN代理商，由代理商请求Google，得到结果后再将结果传给我们。因此我们既知道代理是谁，也知道服务端是谁，而服务端却不知道我们是谁，只知道VPN代理的存在。代理是客户端主动配置的。

- 反向代理

    典型的反向代理就是Nginx，我们作为客户端，只知道Nginx，而不知道Nginx之后真正提供服务的服务端是谁。我们将请求发给Nginx，然后由Nginx转发给真正的服务端，服务端知道客户端是谁，而客户端却不知道具体的服务端，这样的方式称为反向代理。代理是由后端配置的。

- 隧道式代理

    请求和响应都要经过代理服务器

- LVS-DR模式
    请求由代理服务器转发，响应由后端返回给客户端，不经过代理服务器。

### 反向代理配置

```bash
 server {
        listen       80;
        server_name  proxy.seckill.cc;

        location / {
            proxy_pass http://www.baidu.com;
        }

        error_page    500 502 503 504   /50x.html;
        location = /50x.html {
            root    html ;
        }
    }
```

访问proxy.seckill.cc即可代理到百度。（百度的服务器自己会重定向，所以地址栏会变，不需要纠结，别的网站基本也会重定向到自己真实的域名，可以改成代理到自己的某个域名，这样地址栏就不会变了）。

### 基于反向代理的负载均衡

默认配置：

```bash
vim /usr/local/nginx/conf/nginx.con
```

添加`upstream NAME {}`项，并添加对应的代理`server`，我这里是`test_group`。命中了这个`test_group`的请求就会被转发。

```bash
    # ----------------------
    #        负载均衡        
    # ----------------------

    upstream test_group{
        server tx_cloud:80;
        server vm_host:80;
    }

    server {
        listen       80;
        server_name  seckill.cc;
        
            location / {
            proxy_pass http://test_group;
        }
        
            error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

    }
```

> `tx_cloud`和`vm_host`是我在本机的hosts文件中配置的，这里需要换成你自己对应的ip地址或者域名。

![image-20220502205507106](../../../Documents/tmp/nginx_download_06.png)

![image-20220502205450826](../../../Documents/tmp/nginx_download_07.png)

- 轮询算法

    默认的负载均衡策略，每台服务器访问一次。

- 权重weight

    给每个服务器添加一个权重，按照权重访问，权重越高的服务器访问次数越大。对应配置：

    ```bash
    upstream test{
        server tx_cloud:80 weight=8;
        server vm_host:80 weight=2; 
    }
    ```

- IP哈希

    根据客户端的ip地址转发，同一个ip会转发到同一个后端服务器。现在不一定能保持会话了，比如用手机访问网站，在家用的wifi网是一个ip，出门后换成了5G信号，对应的ip可能就变了，因此可能不适合用来保持会话。

      ```bash
      upstream test{
          ip_hash;
          server tx_cloud:80 weight=8;
          server vm_host:80 weight=2; 
      }
      ```

- 随机算法

    通过系统的随机算法，随机选择一台服务器进行转发。所有服务器被访问的期望是相等的。

    ```bash
    upstream test{
        random;
        server tx_cloud:80;
        server vm_host:80;
    }
    ```

- 加权随机

    与随机不同的是，每台服务器被访问的期望值是不相等的，按照。

    ```bash
    upstream test{
        random;
        server tx_cloud:80 weight=8;
        server vm_host:80 weight=2; 
    }
    ```

- 最小连接数

    根据当前服务器的链接情况，动态地选取其中当前积压连接数最少的一台服务器来处理当前的请求。

      ```bash
      upstream test{
          least_conn;
          server tx_cloud:80 weight=8;
          server vm_host:80 weight=2; 
      }
      ```

- fair

    根据后端服务器响应时间转发请求。需要第三方组件。

    ```bash
    upstream test{
        fair;
        server tx_cloud:80 weight=8;
        server vm_host:80 weight=2; 
    }
    ```

- 一致性hash

    需要使用一致性Hash模块。

    ```bash
    upstream test{
        hash $request_uri consistent;
        server tx_cloud:80 weight=8;
        server vm_host:80 weight=2; 
    }
    ```

 ## 动静分离

将静态资源放到nginx服务器，非静态资源才交由后端服务器。

- 使用多个location配置

    ```bash
        location /css {
          root   /usr/local/nginx/static;
          index  index.html index.htm;
        }
        location /images {
          root   /usr/local/nginx/static;
          index  index.html index.htm;
        }
        location /js {
          root   /usr/local/nginx/static;
          index  index.html index.htm;
        }
    ```

- 利用正则使用一个location

    location地址匹配`location [=|~|~*|~^] /uri/ { ... }` ：

    - `=` 精确匹配，不是以指定模式开头。优先级1。
    - `^~ `非正则匹配，匹配以指定模式开头的location，表示uri以某个常规字符串开头。优先级2。
    - `~  `正则匹配，区分大小写。优先级3。
    - `~* `正则匹配，不区分大小写。优先级3。
    - `/ `通用匹配，任何请求都会匹配到,优先级最低，只有别的都匹配不了时，才匹配它。优先级4。

- 配置`proxy_pass URL`时，路径拼接规则

    - 如果`URL`有/结尾，则不会拼接`location`中匹配部分

        ```bash
        location ^~ /abc {
            proxy_pass http://localhost/
        }
        ```

        请求地址`http://seckill.cc/abc/index`，会转发到`http://localhost/index`

    - 如果没有/结尾，则会拼接`location`中匹配部分

        ```bash
        location ^~ /abc {
            proxy_pass http://localhost
        }
        ```

        请求地址`http://seckill.cc/abc/index`，会转发到`http://localhost/abc/index`

- `root`和`alias`

    `root`和`alias`都可以定义在`location`模块中，都是用来指定请求资源的真实路径。

    区别是指定`root`时，真是的路径是`root指定的路径`+`location指定的路径`，而`alias`指定的是真实的路径；`alias`只能在`location`中使用，`root`可以在`server`、`http`、`location`中使用。

    ```bash
        location /js {
          root   /usr/local/nginx/static;
          index  index.html index.htm;
        }
    ```

    真实的路径是`/usr/local/nginx/static/js`

    ```bash
        location /js {
          alias  /usr/local/nginx/static;
          index  index.html index.htm;
        }
    ```

    真实的路径是`/usr/local/nginx/static`

## URLRewrite

可以隐藏真实的后端服务器的地址。比如真实的请求是`http://seckill.cc/get?param=123`，通过URLRewrite可以变成`http://seckill.cc/123.html`

用法：`rewrite    <regex>   <replacement>    [flag] `，`replacement`可以使用正则捕获组，`$`+组号。

`[flag]`:

- `last` :本条规则匹配完成后，继续向下匹配新的location URI规则
- `break`：本条规则匹配完成即终止，不再匹配后面的任何规则
- `redirect`：返回302临时重定向，浏览器地址会显示跳转后的URL地址
- `permanent`：返回301永久重定向，浏览器地址栏会显示跳转后的URL地址

```bash
    location /get {
        rewrite     ^/([0-9]+).html$    /get?param=$1    break;
        index  index.html index.htm;
    }
```

## 防盗链

Referer防盗链，是基于HTTP请求头中Referer字段（例如，Referer黑白名单）来设置访问控制规则，实现对访客的身份识别和过滤，防止网站资源被非法盗用。

Referer是HTTP请求头的一部分，携带了HTTP请求的来源地址信息（协议+域名+查询参数），可用于识别请求来源。但是这个仅限于使用的是合法的浏览器访问才会遵守HTTP协议关于Referer的规范，我们完全可以使用程序伪造HTTP请求而不带Referer。

**用法**：

```bash
 valid_referers none | blocked | server_names | strings ....;
```

- `none`：允许`Referer`不存在的请求访问
- `blocked`：检测 Referer 头域的值被防火墙或者代理服务器删除或伪装的情况。这种情况该头域的值不以 “http://” 或 “https://” 开头。
- `server_names`：设置一个或多个 URL ，检测` Referer `头域的值是否是这些 `URL `中的某一个。
- `strings`：任意字符串，表示域名及URL的字符串，域名可以在前缀或者后缀中含有`*`通配符，若`Referer`头部的值匹配字符串后，则允许访问。

**示例：**

```bash
    location /js {
        valid_references seckill.cc localhost # 只有来源是localhost才允许访问。
        if ($invalid_referer) {
            return 403; 
        }
        root   /usr/local/nginx/static;
        index  index.html index.htm;
    }
```

# Nginx高可用配置

## Keepalived

### 安装配置

[下载地址](https://www.keepalived.org/download.html)

![image-20220503150756963](../../../Documents/tmp/nginx_download_08.png)

解压：

```bash
tar -xzvf keepalived-2.2.7.tar.gz
```

执行

```bash
apt install libnl-3-dev libnl-genl-3-dev -y
./configure --prefix=/usr/local/keepalived
make
make install
```

或者

```bash
apt install keepalived -y
```

**配置：**

```bash
vim /etc/keepalived/keepalived.conf
```

主机配置：

```bash
global_defs {
    router_id lb01        # 你的ip地址
}

vrrp_instance VI_1 {
    state MASTER          # 备份服务器上将 MASTER 改为 BACKUP
    interface ens33     # 网卡，可以通过ifconfig查看
    virtual_router_id 51  # 主、备机的 virtual_router_id 必须相同
    priority 100          # 主、备机取不同的优先级，主机值较大，备份机值较小
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
         192.168.107.128    # VRRP H 虚拟地址
    }
}
```

从机配置：

```bash
global_defs {
    router_id lb02        # 你的ip地址
}

vrrp_instance VI_1 {
    state BACKUP          # 备份服务器上将 MASTER 改为 BACKUP
    interface ens33       # 网卡，可以通过ifconfig查看
    virtual_router_id 51  # 主、备机的 virtual_router_id 必须相同
    priority 50          # 主、备机取不同的优先级，主机值较大，备份机值较小
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
         192.168.107.222    # VRRP H 虚拟地址
    }
}
```

分别启动服务

```bash
systemctl start keepalived.service
```

查看网卡信息`ip addr`，可以看到`vip`已经生效了，且在其它机器可以ping通该ip:

![image-20220503211831225](../../../Documents/tmp/nginx_download_09.png)

关掉第一个机器，发现ip漂移成功。

![image-20220503212042724](../../../Documents/tmp/nginx_download_10.png)