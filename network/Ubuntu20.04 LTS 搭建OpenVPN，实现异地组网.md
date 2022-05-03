# Ubuntu20.04 LTS 搭建OpenVPN，实现异地组建局域网

## 安装OpenVPN、easy-rsa

OpenVPN用于搭建VPN服务，easy-ras用于生成证书，搭建VPN需要使用证书，因此需要我们自己搭建一个简单的CA服务。

安装

```bash
sudo apt install -y openvpn libssl-dev openssl easy-rsa 
```

 ```bash
 mkdir /etc/openvpn/easy-rsa
 cd /etc/openvpn/easy-rsa
 cp -r /usr/share/easy-rsa/* ./
 cp vars.example vars
 vim vars
 ```

添加以下内容：

```bash
set_var EASYRSA_REQ_COUNTRY "CN"
set_var EASYRSA_REQ_PROVINCE    "SY"
set_var EASYRSA_REQ_CITY    "ShenYang"
set_var EASYRSA_REQ_ORG "NEU"
set_var EASYRSA_REQ_EMAIL   "haiqiang225@gamil.com"
set_var EASYRSA_REQ_OU      "haiqiang"

export KEY_NAME="openvpn"
```

执行

```bash
./easyrsa init-pki
./easyrsa build-ca nopass
./easyrsa build-server-full openvpn nopass
./easyrsa build-client-full client nopass
./easyrsa gen-dh
cd ..
cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz  /etc/openvpn
gzip -d server.conf.gz
cp easy-rsa/pki/ca.crt ./
cp easy-rsa/pki/issued/openvpn.crt ./
cp easy-rsa/pki/private/openvpn.key ./
cp easy-rsa/pki/dh.pem ./dh2048.pem
vim server.conf
```

修改配置，有几项是注释掉的，将注释去掉：

```bash
# TCP or UDP server?
proto tcp 
;proto udp


ca ca.crt
cert openvpn.crt # 修改成自己的
key openvpn.key  # This file should be kept secret

; tls-auth ta.key 0 # This file is secret

comp-lzo

log-append  /var/log/openvpn/openvpn.log
;explicit-exit-notify 1
```

启动

```bash
nohup /usr/sbin/openvpn --config /etc/openvpn/server.conf &
```

查看

```bash
lsof -iTCP -nP | grep 1194
```

服务已经正常启动

![image-20220503192940524](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/openvpn01.png)

将下面前三个文件传输的客户端，可以使用SFTP等，新建client.ovpn

![image-20220503195204012](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/openvpn02.png)

内容如下：

```bash
client

dev tun0

proto tcp

remote 202.199.6.118 1194

resolv-retry infinite

nobind

persist-key

persist-tun

ca ca.crt

cert client.crt

key client.key

ns-cert-type server

compress lz4-v2

comp-lzo

verb 3
```

测试连接：

![image-20220503195427974](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/openvpn03.png)

## Ubuntu OpenVPN客户端

将所有文件上传到客户端服务器。

![image-20220503195916065](https://haiqiang-picture.oss-cn-beijing.aliyuncs.com/blog/image-20220503195916065.png)

执行

```bash
nohup openvpn client.ovpn &
```

