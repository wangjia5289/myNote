---
title: 笔记：HTTPS协议
date: 2025-03-24
categories:
  - 网络服务
  - HTTP / HTTPS 协议
  - HTTPS
tags: 
author: 霸天
layout: post
---
### 1. HTTPS 概述

HTTPS（HyperText Transfer Protocol Secure，超文本传输安全协议）是 HTTP 的安全版本，它通过 **SSL/TLS** 加密传输数据，防止被窃听、篡改和劫持。

其实也不用把 HTTPS 看得太神，它的主要作用就是在你和目标服务器之间建立加密通信，防止中间人窃听数据，或者偷偷把你的请求导向一台恶意服务器。比如，你要访问的服务跑在 `192.168.136.8` 上，HTTPS 证书中也明确写了 `192.168.136.8` 是可信地址。这时通信是安全的。但如果有中间人把流量引导到一台伪装的服务器，比如 `10.10.2.3`，而这个地址并没有出现在证书中，浏览器就会识别出不匹配并给出安全警告。

一个常见的误区是以为 HTTPS 能帮你判断哪个网站是“好人”、哪个是“坏人”，其实它根本不负责做价值判断。它只负责把你和“证书里写明的那个服务器”之间的通信加密，至于那个服务器本身是不是邪恶的，它并不关心。

---


### 2. HTTPS 工作流程（从申请 HTTPS 证书到 HTTPS 通信完整流程）

==1.登录网站服务器==
登录你的网站服务器（Linux 系统），后续操作将在服务器上进行。


==2.安装 OpenSSL 服务==
在 `Ubuntu` 系统上，可执行以下命令安装 OpenSSL：
```
sudo apt install openssl
```


==3.生成服务器私钥（Private Key）==
使用 OpenSSL 生成 RSA 算法的服务器私钥，并用 AES-256 加密保护私钥文件，这个服务器私钥我们务必好好保管、留存一份
例如下面指令的流程是：
1. OpenSSL 使用 RSA 算法生产了一对秘钥（私钥和公钥）
2. 然后用 AES-256 加密算法，把生成的私钥加密保存成 `ca.key` （加密秘钥当我们执行下面的指令后，会提示输入密码），以后如果想要解密这个秘钥，OpenSSL 就会要求我们再输入这个密码
```
openssl genpkey -algorithm RSA -out private.key -pkeyopt rsa_keygen_bits:2048 -aes256
```
1. `-algorithm RSA`：
	1. 指定了**生成的私钥使用的加密算法**，这里使用的是 RSA（Rivest-Shamir-Adleman）算法
	2. RSA 是一种常见的秘钥加密算法，广泛用于 SSL/TLS 证书中
2. `-out private.key`：
	1. 指定生成的私钥文件保存的路径和文件名
3. `-pkeyopt rsa_keygen_bits:2048`：
	1. 指定 RSA 秘钥长度，常有 2048、4096 位
4. `-aes256`：
	1. 表示使用 AES 256 位加密 来**加密私钥文件**。


==4.创建 HTTPS 证书签名请求（CSR）==
CSR 是申请证书时提供给 CA 的请求文件，其中包含了服务器的公钥（自动从私钥推导出的公钥）和身份信息：
```
# 1. 创建并编辑配置文件 san.cnf
vim san.cnf


# 2. 添加下面的内容（下面是权威 CSR内容）
# 2.1. 注解版
[ req ]
distinguished_name = req_distinguished_name       # 指定那个段落作为证书的基本信息
req_extensions     = req_ext                      # 指定那个段落用来添加扩展字段（SAN）
prompt             = no                           # 不需要交互式输入，内容都从配置文件中读取

[ req_distinguished_name ]
C  = CN                                # 国家代码，比如 CN
ST = BeiJing                           # 省份
L  = BeiJing                           # 城市
O  = MyCompany                         # 公司名（Let’s Encrypt 可省略，但其他 CA 可能需要）
CN = es.example.com                    # 通常是你网站的域名，主流浏览器只看 SAN，但是 CN 在老客户端可能会检查

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = es.example.com                # 其实这个写主机名也无所谓，只要我们配置了主机名解析
DNS.2 = kibana.example.com
IP.1 = 192.168.136.8                  # 这个字段，大部分权威 CA 是不接受的，它是私有网段，无法公开验证归属权；只有你自己搭建的内部 CA（或某些企业 CA）会接受签发这种证书如果是公网访问服务，就用域名（DNS）字段，CA 会通过域名验证如果只是内网测试，就自签证书或自己搭建 CA；
一般，如果你确定只有多少台 IP，那么我们一般可以把所有ip 列举出来，然后公用一份 HTTPS 证书，但是在生产环境中，我们也不确定是不是需要扩展，所以一般都是一个服务器生成一份证书


# 2.2. 精简版
[ req ]
distinguished_name = req_distinguished_name
req_extensions     = req_ext
prompt             = no

[ req_distinguished_name ]
C  = CN
ST = BeiJing
L  = BeiJing
O  = MyCompany
CN = es.example.com

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = es.example.com
DNS.2 = kibana.example.com
IP.1 = 192.168.136.8


# 3. 创建 HTTPS 证书签名请求（输入命令后，会让你输入秘钥 aes 加密密码，因为要解密私钥）
openssl req -new -key private.key -out server.csr -config san.cnf
```
1. `-key private.key`：
	1. 指定用于生成 CSR 的服务器私钥文件。
2. `-out server.csr`：
	1. 指定生成文件的保存的路径和文件名

> [!NOTE] 注意事项
> 1. 虽然多个节点可以共用一份 HTTPS 证书，只需在 `alt_names` 中添加所有节点的 IP（如：`ip.1 = 192.168.136.8`、`ip.2 = 192.168.136.9`、`ip.3 = 192.168.136.10`），这样无论通过 `https://192.168.136.8:5601` 还是 `https://192.168.136.9:5601` 访问 Kibana，证书中都能匹配对应 IP，避免中间人将流量引导至其他服务器。然而，这种方式的缺点是：如果集群后续新增节点（如 `192.168.136.11`），就必须重新生成包含新 IP 的证书。因此，严格来说，这不算真正的“多个节点共用一份证书”。此外，共用证书意味着共用私钥，一旦某个节点被攻破，攻击者便可窃取私钥，从而伪造或解密集群中的通信。因此 **不推荐**这种做法，建议每个节点使用只包含自身 IP 的独立证书。
> 2. 相比之下，使用域名方式更为灵活。即使新增多个节点，只需将新节点 IP 添加到域名解析中，原证书依然有效，无需重新生成。因此，在使用域名时，我们推荐多个节点共用一份证书。但仍需注意，一旦证书或私钥被伪造或破解，整个集群都可能受到影响。


==5.向 CA 递交 CSR 并申请证书==
将生成的 `server.csr` 文件提交给受信任的证书颁发机构（CA），如 DigiCert、Let’s Encrypt 等。


==6.CA 验证你的身份==
CA 会对域名所有权、组织身份等进行验证，验证通过后进入下一步；


==7.CA 签署 HTTPS 证书==
当你的身份验证通过后，证书颁发机构（CA）会使用它的 **私钥** 对你的证书签名请求（CSR）进行数字签名。

这个过程的核心是：CA 使用私钥对 CSR 中的内容计算出一个**数字签名**，并将这个签名附加到最终的证书中，从而确保该证书的**真实性**和**完整性**（防止篡改）。

签名完成后，CA 会生成一个标准的 **X.509 数字证书**（通常为 `.crt` 或 `.pem` 格式），该证书包含以下关键内容：
1. <font color="#00b0f0">服务器公钥</font>
    - 即你在 CSR 中提供的公钥，用于后续与客户端建立安全连接。
2. <font color="#00b0f0">数字签名</font>
    - CA 用其私钥生成的签名，任何人都可以使用该 CA 的 **公钥** 验证它的有效性，确保证书确实由该 CA 签发，且未被篡改。
3. <font color="#00b0f0">证书元信息</font>
    - 包括颁发者（CA）的信息、证书有效期、序列号、适用域名、使用范围等信息。


==8.用户向你的网站发起 HTTPS 请求==
用户访问你的网站（如 `https://es.example.com`）时，浏览器会尝试建立加密连接。


==9.网站服务器返回 HTTPS 证书==
服务器将刚刚由 CA 签发的 X.509 证书发送给浏览器。


==10.浏览器验证 HTTPS 证书==
浏览器内置了多个 **受信任的 CA 根证书（Root CA 证书）**，这些证书是由各大权威 CA 自己给自己签发的 X.509 证书，包含了对应 CA 的 **公钥**。
浏览器会执行以下操作，用于验证你的 HTTPS 证书：
1. <font color="#00b0f0">验证证书签名</font>：
	- 使用内置的 CA 公钥验证其数字签，检查该证书是否由受信任的 CA 签发
2. <font color="#00b0f0">验证有效期</font>：
	- 确认证书当前是否处于有效期内；
3. <font color="#00b0f0">验证域名</font>：
	- 它会检查 HTTPS 证书中的 SAN，这个字段里列出了允许用来访问服务器的域名、IP 地址，然后会检查你访问的地址是否在证书的 SAN 列表中出现
		- 例如你访问`https://es.example.com`，他会去 SAN 列表中查找 ex.example.com（注意不是去找域名解析出的 IP 地址，除非你 IP 地址直连，如下面）
		- 例如你访问`https://192.168.136.8` ，他会去 SAN 列表中查找 `192.168.136.8`
	- 这个主要是<font color="#ff0000">防止中间人把流量导到另一台危险服务器上</font>，保证，我访问的这台服务器，证书里面也有，那说明这台服务器是安全的，如果你访问的这台服务器，证书里面没有，那你就要小心了
	- 这个字段就是告诉客户端或浏览器，这个证书对那些域名或ip 有效
	- 检查证书中绑定的域名是否与你当前访问的网站域名一致。
    
如果任意一项验证失败，浏览器将提示用户该网站的连接不安全，并发出安全警告，用户可以选择终止访问或继续前往


==11.浏览器生成会话秘钥（对称秘钥）==
1. 浏览器会随机生成一个 **会话密钥**（对称加密所用的密钥），用于后续数据传输中的加密和解密。
2. 然后，浏览器使用你网站证书中的 **服务器公钥** 对这个会话密钥进行加密，并将加密后的内容发送给服务器。
3. 由于只有服务器持有对应的 **私钥**，能够解密加密后的内容，即使数据在传输过程中被截获，攻击者也无法解密这个加密后的会话密钥。
4. 注意：非对称加密（如 RSA）计算复杂、速度较慢，通常只用于密钥交换，而不是用于大数据量的加密通信。


==12.服务器解密会话秘钥==
1. 服务器接收到浏览器发来的加密会话密钥后，使用自己的 **私钥** 进行解密，成功还原出浏览器生成的会话密钥。


==13.后续再通信==
1. 一旦浏览器和服务器都持有相同的会话密钥，后续所有通信内容将通过**该密钥（会话秘钥）** 进行 **对称加密** 和 **解密**。
2. 这样可以在保证通信安全的同时大幅提升传输效率，即使中间数据被截获，也因加密而无法被解读
3. 注意：对称加密（如 AES）加解密速度快，适合处理大量传输数据，是实际 HTTPS 通信中的核心加密方式。


==14.补充：如何从私钥推导出公钥==
在非对称加密中（例如 RSA），私钥（`private.key`）和公钥（`public.key`）是一对通过特定数学算法生成的密钥。

私钥本身包含了推导出对应公钥所需的全部信息，但前提是必须使用与私钥生成时相同的加密算法。此外，只要私钥保持不变，每次推导出的公钥也始终是一致的。
```
# 1. 使用 RSA 算法生成受 AES-256 加密保护的 2048 位私钥
openssl genpkey -algorithm RSA -out private.key -pkeyopt rsa_keygen_bits:2048 -aes256


# 2. 从私钥中导出对应的公钥
openssl rsa -in private.key -pubout -out public.key
```


==15.补充：客户端证书==
HTTPS 可以要求客户端上传证书，常用于：
1. 企业内部应用：
	1. 当只有特定的设备或员工才能访问某些系统时，可以使用客户端证书进行身份验证。
2. API 调用验证：
	1. 一些需要保护的 API 接口可能会要求客户端通过证书来证明自己的身份。
3. 金融服务：
	1. 例如，在线银行或支付系统，客户端证书可以增加安全性，防止非法访问。


---


### 3. HTTPS 证书

#### 3.1. HTTPS 证书 概述

HTTPS 证书就是 SSL/TLS 证书，最初，HTTPS 使用的是 SSL（Secure Sockets Layer，安全套接字层） 进行加密，但 SSL 3.0 已被淘汰，现在主要使用 TLS（Transport Layer Security，传输层安全协议）。

| 名称      | 状态    | 说明               |
| ------- | ----- | ---------------- |
| SSL 1.0 | 未发布   |                  |
| SSL 2.0 | 废弃 ❌  | 存在严重漏洞           |
| SSL 3.0 | 废弃 ❌  | 容易受 POODLE 攻击    |
| TLS 1.0 | 废弃 ❌  | 2020 年起主流浏览器不再支持 |
| TLS 1.1 | 废弃 ❌  | 安全性不足            |
| TLS 1.2 | 推荐 ✅  | 目前最广泛使用          |
| TLS 1.3 | 最优 🏆 | 更快更安全            |

---


#### 3.2. HTTPS 证书 分类

1. ==DV 证书（Domain Validation）==：
    1. 仅验证域名所有权，适合个人博客、小型网站
    2. 例如：Let’s Encrypt（免费）、ZeroSSL
2. ==OV 证书（Organization Validation）==：
    1. 需要验证企业身份，适用于企业网站、电商平台
3. ==EV 证书（Extended Validation）==：
    1. 最高级别的验证，浏览器地址栏显示企业名称，适合金融、银行、政府网站


---


#### 3.3. 获取 HTTPS 证书

##### 3.3.1. 通过证书颁发机构（CA）购买证书

选择一个受信任的证书颁发机构（CA），如 DigiCert、Comodo、GlobalSign、Symantec 等，通常是获取 HTTPS 证书的最常见方式，适用于大多数企业和网站。

CA 机构颁发 HTTPS 证书的过程是：首先，你需要创建一个证书签名请求（CSR），然后，CA 机构会使用它们的 CA 证书（根证书）签署这个 CSR，从而生成一个有效的 HTTPS 证书并提供给你。

---


##### 3.3.2. 使用免费证书服务（Let’s Encrypt）

---


##### 3.3.3. 通过云服务商获取证书

如果你使用了云服务（如 GoDaddy、Bluehost、阿里云、腾讯云等），这些提供商通常会提供一键获取 HTTPS 证书的服务，方便快捷。

---


##### 3.3.4. 自签名证书（开发、测试环境）

如果用于测试或开发环境，可以生成一个自签名的 HTTPS 证书，但不推荐在生产环境中使用

==1. 生成自签名证书==
```
openssl x509 -req -in server.csr -signkey private.key -out server.crt -days 365
```
1. `-in server.csr`：
	1. 指定用于生成自签名证书文件的证书签名请求文件
2. `-signkey private.key`：
	1. 指定用于生成自签名证书文件的秘钥文件（CA 私钥）
3. `-out server.crt`：
	1. 指定生成的签名证书文件保存的路径和文件名
4. `-days 365`：
	1. 自签名证书的有效期，以天为单位

==2.检查自签名证书==
```
openssl x509 -in server.crt -text -noout
```
1. `-in server.crt`：
	1. 用于检验的自签名证书文件
2. `-text`：
	1. 以文本格式显示证书的详细信息。
	2. 包括证书的各种字段，如有效期、发行者、主题、序列号、公钥信息等。
3. `-noout`：
	1. 表示只显示证书的详细信息，而不输出证书的内容

---


#### 3.4. 配置 HTTPS 证书

##### 3.4.1. 基于 Nginx
假设证书文件位于 `/path/to/server.crt`，私钥文件位于 `/path/to/private.key`
```
# 在 http 块中的 server 块配置 HTTPS 证书
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /path/to/server.crt;                 # 指定证书文件
    ssl_certificate_key /path/to/private.key;            # 指定私钥文件

    # 其他配置项
}
```

---


##### 3.4.2. 基于 Apache
假设证书文件位于 `/path/to/server.crt`，私钥文件位于 `/path/to/private.key`
```
<VirtualHost *:443>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html
    ServerName example.com

    SSLEngine on
    SSLCertificateFile /path/to/server.crt            # 指定证书文件
    SSLCertificateKeyFile /path/to/private.key        # 指定私钥文件

    # 其他配置项
</VirtualHost>
```

---
