---
layout: post
title: https相关
tags: [https, CA, 证书]
author-id: zqmalyssa
---

https，证书等相关内容在工作中也是需要的，总结一下

#### 概况

https，CA，证书

随着互联网的快速发展，我们几乎离不开网络了，聊天、预订酒店、购物等等，我们的隐私无时无刻不暴露在这庞大的网络之中，HTTPS 能够让信息在网络中的传递更加安全，增加了 haker 的攻击成本。
HTTPS 区别于 HTTP，它多了加密(encryption)，认证(verification)，鉴定(identification)。它的安全源自非对称加密以及第三方的 CA 认证。

SSL Handshake(RSA) Without Keyless SSL

1、客户端生成一个随机数 random-client，传到服务器端（Say Hello)
2、服务器端生成一个随机数 random-server，和着公钥，一起回馈给客户端（I got it)
3、客户端收到的东西原封不动，加上 premaster secret（通过 random-client、random-server 经过一定算法生成的东西），再一次送给服务器端，这次传过去的东西会使用公钥加密
4、服务器端先使用私钥解密，拿到 premaster secret，此时客户端和服务器端都拥有了三个要素：random-client、random-server 和 premaster secret
5、此时安全通道已经建立，以后的交流都会校检上面的三个要素通过算法算出的 session key

random-client、random-server 和 premaster secret这三个要素在客户端和服务端都有

但是，如果网站只靠上面流程运作，可能会被中间人攻击，试想一下，在客户端和服务端中间有一个中间人，两者之间的传输对中间人来说是透明的，那么中间人完全可以获取两端之间的任何数据，
然后将数据原封不动的转发给两端，由于中间人也拿到了三要素和公钥，它照样可以解密传输内容，并且还可以篡改内容。

为了确保我们的数据安全，我们还需要一个 CA 数字证书。HTTPS的传输采用的是非对称加密，一组非对称加密密钥包含公钥和私钥，通过公钥加密的内容只有私钥能够解密。上面我们看到，整个传输过程，
服务器端是没有透露私钥的。而 CA 数字认证涉及到私钥，整个过程比较复杂，我也没有很深入的了解，后续有详细了解之后再补充下。但是也有一说是分阶段，看下面的表

```html

Https协议在内容传输上使用的加密是“对称加密”，而“非对称加密”只作用于证书验证阶段。
Https的整体实现过程分为“证书验证”和“数据传输”两个阶段：

https://www.jianshu.com/p/e80530f4708f


证书验证阶段
1.浏览器发起Https请求；
2.服务器端返回Https证书；
3.浏览器客户端验证证书是否合法，若不合法则提示警告

数据传输阶段
4.当证书验证合法后，在客户端本地生成随机数；
5.通过公钥加密随机数，并将加密后的随机数传输到服务端；
6.服务端通过私钥对接收到的加密随机数进行解密操作；
7.服务端通过客户端传入的随机数构造对称加密算法，对返回结果内容进行加密操作后再进行内容传输。

为什么数据传输是用对称加密？
首先，非对称加密的加密效率是非常低的，而http的应用场景通常存在着端与端之间的大量数据交互，从效率来说是无法接受的；
其次，在Https场景中只有服务端保存了私钥，而一对公私钥只能实现单向的加解密（即服务端无法使用私钥对传回浏览器客户端的数据进行加密，只能用于解密），所以Https中内容传输加密采取的是对称加密，而不是非对称加密（此处随机数则是对称加密的介体，即客户端和服务器端所拥有的随机数都是一致的，能够进行双向加解密）。

为什么需要CA认证机构颁发证书？

Http协议被认为不安全是因为传输过程容易被监听者勾线监听、伪造服务器，而Https协议主要就是解决网络传输的安全性问题。我们假设不存在认证机构，任何人都可以制作证书，这存在的风险便是经典的“中间人攻击”问题

1.客户端请求被劫持（如DNS劫持等），所有的客户端请求均被转发至中间人的服务器；
2.中间人服务器返回中间人伪造的“伪证书”（包含伪公钥）；
3.客户端创建随机数，通过中间人证书的伪公钥对随机数进行加密后传输给中间人，然后凭随机数构造对称加密算法对要进行传输的数据内容进行对称加密后传输；
4.中间人因为拥有客户端生成的随机数，从而能够通过对称加密算法进行数据内容解密；
5.中间人再以“伪客户端”的身份向正规的服务端发起请求；
6.因为中间人与服务器之间的通信过程是合法的，正规服务端通过建立的安全通道返回加密后的数据内容；
7.中间人凭借与正规服务器建立的对称加密算法进行数据内容解密；
8.中间人再通过与客户端建立的对称加密算法对正规服务器返回的数据内容进行加密传输；
9.客户端通过中间人建立的对称加密算法对返回的数据内容进行解密；

由于缺少对证书的真伪性验证，所有客户端即使发起了Https请求，但客户端完全不知道自己发送的请求已经被第三方拦截，导致其中传输的数据内容被中间人窃取

浏览器如何确保CA证书的合法性？

首先，权威机构是需要通过认证的。其次证书的可信性基于信任制，CA认证机构需要对其颁发的证书进行信用担保，只要是CA认证机构颁发的证书，我们就认为是合法的。CA认证机构会对证书申请人的信息进行审核的。

浏览器发起https请求时，服务器会返回网站的SSL证书，浏览器需要对证书做以下验证：

a.验证域名、有效期等信息是否正确；
b.判断证书来源是否合法。每份签发证书都可以根据验证链查找到对应的根证书，操作系统、浏览器会在本地存储权威机构的根证书，利用本地根证书可以对对应机构签发的证书进行来源验证；
c.判断证书是否被篡改。需要与CA服务器进行对比校验；
d.判断证书是否已被吊销。通过CRL(Certificate Revocation List 证书注销列表) 和 OCSP（Online Certificate Status Protocol 在线证书状态协议）实现，其中OCSP可用于第3步中以减少与CA服务器的交互，提高验证效率。

客户端的本地随机数被窃取了怎么办？

其实https并不包含对随机数的安全保证，https保证的只是数据传输过程安全，而随机数存储于本地，本地的安全属于另一安全范畴，应对的措施有安装杀毒软件、反木马、浏览器升级修复漏洞等。（这也反映了Https协议并不是绝对的安全的）

使用Https被抓包了会怎样？

由于Https的数据是加密，常规下抓包工具代理请求后抓到的包内容是加密状态的，无法直接查看。通常， HTTPS 抓包工具的使用方法是会生成一个证书，用户需要手动把证书安装到客户端中！！！！！然后终端发起的所有请求通过该证书完成与抓包工具的交互，然后抓包工具再转发请求到服务器，最后把服务器返回的结果在控制台输出后再返回给终端，从而完成整个请求的闭环。

证书安装到客户端中

linux tomcat导入根证书方式

keytool -import -alias xxx -keystore /usr/java/jdk1.7.0_51/jre/lib/security/cacerts -file /tmp/xxxroot.cer -trustcacerts
如果没设置过密码输入默认密码：changeit
参数解释：
-import：导入证书
-alias：证书别名，方便查找证书
-keystore：证书存储路径，一般为jre主目录下的lib/security/cacerts
-file：源证书路径
-trustcacerts：添加信任

/usr/java/jdk1.8.0_60/bin/keytool -import -alias xxx -keystore /usr/java/jdk1.8.0_60/jre/lib/security/cacerts -file /tmp/xxxroot.cer -trustcacerts
/usr/java/jdk1.8.0_60/bin/keytool -list -v -keystore /usr/java/jdk1.8.0_60/jre/lib/security/cacerts  |  grep xxx

// 11没有这个目录


```

公钥与私钥是一对，如果用公钥对数据进行加密，只有用对应的私钥才能解密。因为加密和解密使用的是两个不同的密钥，所以这种算法叫作非对称加密算法！！！

这边就说下对称加密和非对称加密

```html

对称加密是指加解密使用的是同样的密钥

非对称加密是指加解密使用的密钥不同。

对称加密的特点是简单快速，密钥越大，加密越强，但加解密过程越慢，密钥容易被黑客拦截

非对称加密使用了一对密钥，公钥和私钥，私钥由解密方安全保管，公钥可以发给任何请求它的人。数据使用公钥加密，私钥解密，因为私钥不通过网络发送出去，所以非对称加密的安全性很高，非对称加密很安全，但和对称加密比起来，非常慢

还有一种，对称密钥使用非对称方式发送

对称密钥使用非对称方式发送，解决了对称密钥易被获取，和非对称密钥加解密慢的问题。使用步骤如下：

1)A生成一个随机数作为对称密钥
2)A向B申请公钥
3)B将公钥发给A
4)A使用公钥加密对称密钥，将加密后的结果发给B
5)B使用私钥解密出对称密钥
6)A和B可以通过对称密钥对信息加解密了

https://www.huaweicloud.com/articles/739b7e08bbfed7eb06cfdcfb9a6df5de.html


```


CA 认证分为三类：DV ( domain validation)，OV ( organization validation)，EV ( extended validation)，证书申请难度从前往后递增，貌似 EV 这种不仅仅是有钱就可以申请的。

对于一般的小型网站尤其是博客，可以使用自签名证书来构建安全网络，所谓自签名证书，就是自己扮演 CA 机构，自己给自己的服务器颁发证书！！！

生成密钥、证书：

```html

第一步，为服务器端和客户端准备公钥、私钥

# 生成服务器端私钥
openssl genrsa -out server.key 1024
# 生成服务器端公钥
openssl rsa -in server.key -pubout -out server.pem


# 生成客户端私钥
openssl genrsa -out client.key 1024
# 生成客户端公钥
openssl rsa -in client.key -pubout -out client.pem

第二步，生成 CA 证书

# 生成 CA 私钥
openssl genrsa -out ca.key 1024
# X.509 Certificate Signing Request (CSR) Management.
openssl req -new -key ca.key -out ca.csr
# X.509 Certificate Data Management.
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt

在执行第二步时会出现：

➜  keys  openssl req -new -key ca.key -out ca.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:CN
State or Province Name (full name) [Some-State]:Zhejiang
Locality Name (eg, city) []:Hangzhou
Organization Name (eg, company) [Internet Widgits Pty Ltd]:My CA
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:localhost
Email Address []:

注意，这里的 Organization Name (eg, company) [Internet Widgits Pty Ltd]: 后面生成客户端和服务器端证书的时候也需要填写，不要写成一样的！！！可以随意写如：My CA, My Server, My Client。
然后 Common Name (e.g. server FQDN or YOUR name) []: 这一项，是最后可以访问的域名，我这里为了方便测试，写成 localhost，如果是为了给我的网站生成证书，需要写成 barretlee.com。

第三步，生成服务器端证书和客户端证书

# 服务器端需要向 CA 机构申请签名证书，在申请签名证书之前依然是创建自己的 CSR 文件
openssl req -new -key server.key -out server.csr
# 向自己的 CA 机构申请证书，签名过程需要 CA 的证书和私钥参与，最终颁发一个带有 CA 签名的证书
openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial -in server.csr -out server.crt

# client 端
openssl req -new -key client.key -out client.csr
# client 端到 CA 签名
openssl x509 -req -CA ca.crt -CAkey ca.key -CAcreateserial -in client.csr -out client.crt

此时，我们的 keys 文件夹下已经有如下内容了：

.
├── https-client.js
├── https-server.js
└── keys
    ├── ca.crt
    ├── ca.csr
    ├── ca.key
    ├── ca.pem
    ├── ca.srl
    ├── client.crt
    ├── client.csr
    ├── client.key
    ├── client.pem
    ├── server.crt
    ├── server.csr
    ├── server.key
    └── server.pem

看到上面两个 js 文件了么，我们来跑几个demo。

HTTPS本地测试，服务器代码：

// file http-server.js
var https = require('https');
var fs = require('fs');

var options = {
  key: fs.readFileSync('./keys/server.key'),
  cert: fs.readFileSync('./keys/server.crt')
};

https.createServer(options, function(req, res) {
  res.writeHead(200);
  res.end('hello world');
}).listen(8000);

短短几行代码就构建了一个简单的 https 服务器，options 将私钥和证书带上。然后利用 curl 测试：

➜  https  curl //localhost:8000
curl: (60) SSL certificate problem: Invalid certificate chain
More details here: http://curl.haxx.se/docs/sslcerts.html


curl performs SSL certificate verification by default, using a "bundle"
 of Certificate Authority (CA) public keys (CA certs). If the default
 bundle file isn't adequate, you can specify an alternate file
 using the --cacert option.
If this HTTPS server uses a certificate signed by a CA represented in
 the bundle, the certificate verification probably failed due to a
 problem with the certificate (it might be expired, or the name might
 not match the domain name in the URL).
If you'd like to turn off curl's verification of the certificate, use
 the -k (or --insecure) option.

当我们直接访问时，curl //localhost:8000 一堆提示，原因是没有经过 CA 认证，添加 -k 参数能够解决这个问题：

➜  https  curl -k //localhost:8000  （注意这种访问是不安全的）
hello world%

这样的方式是不安全的，存在我们上面提到的中间人攻击问题。可以搞一个客户端带上 CA 证书试试，看下面的客户端的代码：

// file http-client.js
var https = require('https');
var fs = require('fs');

var options = {
  hostname: "localhost",
  port: 8000,
  path: '/',
  methed: 'GET',
  key: fs.readFileSync('./keys/client.key'),
  cert: fs.readFileSync('./keys/client.crt'),
  ca: [fs.readFileSync('./keys/ca.crt')]
};

options.agent = new https.Agent(options);

var req = https.request(options, function(res) {
  res.setEncoding('utf-8');
  res.on('data', function(d) {
    console.log(d);
  });
});
req.end();

req.on('error', function(e) {
  console.log(e);
});

先打开服务器 node http-server.js，然后执行

➜  https  node https-client.js
hello world

如果你的代码没有输出 hello world，说明证书生成的时候存在问题。也可以通过浏览器访问：

```

提示错误：

此服务器无法证明它是localhost；您计算机的操作系统不信任其安全证书。出现此问题的原因可能是配置有误或您的连接被拦截了。

原因是浏览器没有 CA 证书，只有 CA 证书（看清楚，是CA证书，为什么没有呢，看下面），服务器才能够确定，这个用户就是真实的来自 localhost 的访问请求（比如不是代理过来的）。

你可以点击 继续前往localhost（不安全） 这个链接，相当于执行 curl -k //localhost:8000。如果我们的证书不是自己颁发，而是去靠谱的机构去申请的，那就不会出现这样的问题，因为靠谱机构的证书会放到浏览器中！！！，
浏览器会帮我们做很多事情。初次尝试的同学可以去 https://store.wotrus.com/ 申请一个免费的证书。

这边的话客户端会带上ca.crt了

Nginx 部署
ssh 到你的服务器，对 Nginx 做如下配置：

server_names barretlee.com *.barretlee.com
ssl on;
ssl_certificate /etc/nginx/ssl/barretlee.com.crt;  // 服务器证书
ssl_certificate_key /etc/nginx/ssl/barretlee.com.key;  // 服务器私钥
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_ciphers "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4EECDH EDH+aRSA RC4 !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS !MEDIUM";
# Add perfect forward secrecy
ssl_prefer_server_ciphers on;
add_header Strict-Transport-Security "max-age=31536000; includeSubdomains";

会发现，网页 URL 地址框左边已经多出了一个小绿锁。当然，部署好了之后可以去  https://www.ssllabs.com/ssltest/analyze.html 看看测评分数，如果分数是 A+，说明你的 HTTPS 的各项配置都还不错，速度也很快。

CA，Catificate Authority，它的作用就是提供证书（即服务器证书，由域名、公司信息、序列号和签名信息组成）加强服务端和客户端之间信息交互的安全性，以及证书运维相关服务。
任何个体/组织都可以扮演 CA 的角色（连接会有红色提示那种，是使用的证书没有签证，或者未在浏览器受信的 CA 签证），只不过难以得到客户端的信任，能够受浏览器默认信任的 CA 大厂商有很多，其中 TOP5 是 Symantec、Comodo、Godaddy、GolbalSign 和 Digicert（公司有用）

类型：
DV（Domain Validation），面向个体用户，安全体系相对较弱，验证方式就是向 whois 信息中的邮箱发送邮件，按照邮件内容进行验证即可通过，DV单域名、多域名支持，泛域名、多泛域名不支持
OV（Organization Validation）， 面向企业用户，证书在 DV 证书验证的基础上，还需要公司的授权，CA 通过拨打信息库中公司的电话来确认，OV单域名、多域名、泛域名、多泛域名都支持
EV（Extended Validation），打开 Github 的网页，你会看到 URL 地址栏展示了注册公司的信息，这会让用户产生更大的信任，这类证书的申请除了以上两个确认外，还需要公司提供金融机构的开户许可证，要求十分严格
EV单域名、多域名支持，泛域名、多泛域名不支持

OV 和 EV 证书相当昂贵，使用方可以为这些颁发出来的证书买保险，一旦 CA 提供的证书出现问题，一张证书的赔偿金可以达到 100w 刀以上。

证书由域名、公司信息、序列号和签名信息组成，当我们通过 HTTPS 访问页面时，浏览器会主动验证证书信息是否匹配，也会验证证书是否有效。

CA 供应商很多，提供服务的侧重点可能也存在一些差异，比如很多 CA 都没有提供证书吊销的服务，这一点对于安全性要求很高的企业来说是完全不能接受的，那么对 CA 供应商的评估需要注意写什么呢？

1. 内置根

所谓内置根，就是 CA 的根证书内置到各种通用的系统/浏览器中，只有根证书的兼容性够强，它所能覆盖的浏览器才会越多。

2. 安全体系

两个指标可以判断 CA 供应商是否靠谱，一是看价格，价格高自然有它的理由，必然提供了全套的安全保障体系；二是看黑历史，该 CA 供应商有没有爆出过什么漏洞，比如之前的 DigiNotar，被伊朗入侵，签发了 500 多张未授权的证书，结果直接被各系统/浏览器将其根拉入黑名单，毫无疑问公司直接倒闭。


3. 核心功能和扩展功能

这就需要从业务上考虑了，不同的规模的企业、不同的业务对证书的要求不一样，比如证书是否会考虑无 SNI 支持的浏览器问题，是否支持在 reissue 的时候添加域名，是否支持 CAA，是否支持短周期证书等等。

4. 价格

企业完全没必要购买 Github 那样的 EV 证书，太昂贵，而且一般的企业也未必能够申请到这样的证书。供应商很大，价格可以好好评估下，不一定要最贵，最适合的就行。

自建 Root CA

然而在一些情况下，我们没必要去 CA 机构购买证书，比如在内网的测试环境中，为了验证 HTTPS 下的一些问题，我们不需要部署昂贵的证书，这个时候自建 Root CA，给自己颁发证书就显得很有价值了。

创建 root pair

扮演 CA 角色，就意味着要管理大量的 pair 对，而原始的一对 pair 对叫做 root pair，它包含了 root key（ca.key.pem）和 root certificate（ca.cert.pem）。通常情况下，root CA 不会直接为服务器或者客户端签证，它们会先为自己生成几个中间 CA（intermediate CAs），这几个中间 CA 作为 root CA 的代表为服务器和客户端签证。

注意：一定要在绝对安全的环境下创建 root pair，可以断开网络、拔掉网线和网卡，当然，如果是测试玩一玩就不用这么认真了。

```html

设定文件夹结构，并且配置好 openssl 设置：

$ cd /root/ca
$ mkdir certs crl newcerts private
$ chmod 700 private
$ touch index.txt
$ echo 1000 > serial
$ wget -O /root/ca/openssl.cnf \
//raw.githubusercontent.com/barretlee/autocreate-ca/master/cnf/root-ca

创建 root key，密码可为空，设定权限为只可读：

$ cd /root/ca
$ openssl genrsa -aes256 -out private/ca.key.pem 4096

Enter pass phrase for ca.key.pem: secretpassword
Verifying - Enter pass phrase for ca.key.pem: secretpassword

$ chmod 400 private/ca.key.pem

创建 root cert，权限设置为可读：

$ cd /root/ca
$ openssl req -config openssl.cnf \
      -key private/ca.key.pem \
      -new -x509 -days 7300 -sha256 -extensions v3_ca \
      -out certs/ca.cert.pem

$ chmod 444 certs/ca.cert.pem

验证证书：

$ openssl x509 -noout -text -in certs/ca.cert.pem

正确的输出应该是这样的：

Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number:
            87:e8:c0:a0:4b:e2:12:5d
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: C=CN, ST=Zhejiang, O=Barret Lee, OU=Barret Lee Certificate Authority, CN=Barret Lee Root CA
        Validity
            Not Before: Apr 23 05:46:36 2016 GMT
            Not After : Apr 18 05:46:36 2036 GMT
        Subject: C=CN, ST=Zhejiang, O=Barret Lee, OU=Barret Lee Certificate Authority, CN=Barret Lee Root CA
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
            RSA Public Key: (4096 bit)
                Modulus (4096 bit):
                    // ...
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Subject Key Identifier:
                E5:2D:B8:2B:DC:88:FE:CE:DA:93:D8:6F:2E:74:04:D2:39:E7:C8:03
            X509v3 Authority Key Identifier:
                keyid:E5:2D:B8:2B:DC:88:FE:CE:DA:93:D8:6F:2E:74:04:D2:39:E7:C8:03

            X509v3 Basic Constraints: critical
                CA:TRUE
            X509v3 Key Usage: critical
                Digital Signature, Certificate Sign, CRL Sign
    Signature Algorithm: sha256WithRSAEncryption
        // ...

包含：

数字签名（Signature Algorithm）
有效时间（Validity）
主体（Issuer）
公钥（Public Key）
X509v3 扩展，openssl config 中配置了 v3_ca，所以会生成此项

cdb中的都有对应项（注意对应查看）

创建 intermediate pair

目前我们已经拥有了 Root Pair，事实上已经可以用于证书的发放了，但是由于根证书很干净，特别容易被污染，所以我们需要创建中间 pair 作为 root pair 的代理，生成过程同上，只是细节略微不一样。

$ mkdir /root/ca/intermediate
$ cd /root/ca/intermediate
$ mkdir certs crl csr newcerts private
$ chmod 700 private
$ touch index.txt
$ echo 1000 > serial
$ echo 1000 > /root/ca/intermediate/crlnumber
$ wget -O /root/ca/openssl.cnf \
    //raw.githubusercontent.com/barretlee/autocreate-ca/master/cnf/intermediate-ca

创建 intermediate key，密码可为空，设定权限为只可读：

$ cd /root/ca
$ openssl genrsa -aes256 \
      -out intermediate/private/intermediate.key.pem 4096

Enter pass phrase for intermediate.key.pem: secretpassword
Verifying - Enter pass phrase for intermediate.key.pem: secretpassword

$ chmod 400 intermediate/private/intermediate.key.pem

创建 intermediate cert，设定权限为只可读，这里需要特别注意的一点是 Common Name 不要与 root pair 的一样！！！ ：

$ cd /root/ca
$ openssl req -config intermediate/openssl.cnf -new -sha256 \
      -key intermediate/private/intermediate.key.pem \
      -out intermediate/csr/intermediate.csr.pem

Enter pass phrase for intermediate.key.pem: secretpassword
You are about to be asked to enter information that will be incorporated
into your certificate request.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name []:Zhejiang
Locality Name []:
Organization Name []:Barret Lee
Organizational Unit Name []:Barret Lee Certificate Authority
Common Name []:Barret Lee Intermediate CA
Email Address []:

使用 v3_intermediate_ca 扩展签名，密码可为空，中间 pair 的有效时间一定要为 root pair 的子集：

$ cd /root/ca
$ openssl ca -config openssl.cnf -extensions v3_intermediate_ca \
      -days 3650 -notext -md sha256 \
      -in intermediate/csr/intermediate.csr.pem \
      -out intermediate/certs/intermediate.cert.pem

Enter pass phrase for ca.key.pem: secretpassword
Sign the certificate? [y/n]: y

$ chmod 444 intermediate/certs/intermediate.cert.pem

此时 root 的 index.txt 中将会多出这么一条记录：

V 260421055318Z   1000  unknown .../CN=Barret Lee Intermediate CA

验证中间 pair 的正确性：

$ openssl x509 -noout -text \
      -in intermediate/certs/intermediate.cert.pem
$ openssl verify -CAfile certs/ca.cert.pem \
      intermediate/certs/intermediate.cert.pem

intermediate.cert.pem: OK

浏览器在验证中间证书的时候，同时也会去验证它的上一级证书是否靠谱，创建证书链，将 root cert 和 intermediate cert 合并到一起，可以让浏览器一并验证：

$ cat intermediate/certs/intermediate.cert.pem \
      certs/ca.cert.pem > intermediate/certs/ca-chain.cert.pem
$ chmod 444 intermediate/certs/ca-chain.cert.pem

创建服务器/客户端证书

终于到了这一步，生成我们服务器上需要部署的内容，上面已经解释了为啥需要创建中间证书。root pair 和 intermediate pair 使用的都是 4096 位的加密方式，一般情况下服务器/客户端证书的过期时间为一年，所以可以安全地使用 2048 位的加密方式。

$ cd /root/ca
$ openssl genrsa -aes256 \
      -out intermediate/private/www.barretlee.com.key.pem 2048
$ chmod 400 intermediate/private/www.barretlee.com.key.pem

创建 www.barretlee.com 的证书：

$ cd /root/ca
$ openssl req -config intermediate/openssl.cnf \
      -key intermediate/private/www.barretlee.com.key.pem \
      -new -sha256 -out intermediate/csr/www.barretlee.com.csr.pem

Enter pass phrase for www.barretlee.com.key.pem: secretpassword
You are about to be asked to enter information that will be incorporated
into your certificate request.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name []:Zhejiang
Locality Name []:Hangzhou
Organization Name []:Barret Lee
Organizational Unit Name []:Barret Lee's Personal Website
Common Name []:www.barretlee.com
Email Address []:barret.china@gmail.com

使用 intermediate pair 签证上面证书：

$ cd /root/ca
$ openssl ca -config intermediate/openssl.cnf \
      -extensions server_cert -days 375 -notext -md sha256 \
      -in intermediate/csr/www.barretlee.com.csr.pem \
      -out intermediate/certs/www.barretlee.com.cert.pem
$ chmod 444 intermediate/certs/www.barretlee.com.cert.pem

可以看到 /root/ca/intermediate/index.txt 中多了一条记录：

V 170503055941Z   1000  unknown .../emailAddress=barret.china@gmail.com

验证证书：

$ openssl x509 -noout -text \
      -in intermediate/certs/www.barretlee.com.cert.pem
$ openssl verify -CAfile intermediate/certs/ca-chain.cert.pem \
      intermediate/certs/www.barretlee.com.cert.pem

www.barretlee.com.cert.pem: OK

此时我们已经拿到了几个用于部署的文件：

ca-chain.cert.pem
www.barretlee.com.key.pem
www.barretlee.com.cert.pem

添加信任 CA 和证书的调试

双击 /root/ca/intermediate/certs/ca-chain.cert.pem 将证书安装到系统中，目的是让本机信任这个 CA，将其当作一个权威 CA，安装 root pem 或者 intermediate chain pem 都是可以的，它们都具备验证能力。如果不执行这一步，浏览器依然会提示 net:ERR_CERT_AUTHORITY_INVALID。

上面申请测试证书时，我设置的 Common Name 为 www.barretlee.com，由于不在线上机器测试，可以将其添加到 hosts：

127.0.0.1 www.barretlee.com

执行下方测试代码，server端代码

// https-server.js
var https = require('https');
var fs = require('fs');

var options = {
  key: fs.readFileSync('/root/ca/intermediate/private/www.barretlee.com.key.pem'),
  cert: fs.readFileSync('/root/ca/intermediate/certs/www.barretlee.com.cert.pem'),
  passphrase: 'passoword' // 如果生成证书的时候设置了密码，请添加改参数和密码
};

https.createServer(options, function(req, res) {
  res.writeHead(200);
  res.end('hello world');
}).listen(8000, function(){
  console.log('Open URL: //www.barretlee.com:8000');
});

可以看到这样的效果：

hello world

查看证书的详细信息：

www.barretlee.com，可以查看基本的信息

一般公司内网的电脑都会强制安装一些安全证书，此时就可以把我们自建自签名的证书导入/引导安装到用户的电脑中啦~

```

无 SNI 支持问题

很多公司由于业务众多，域名也是相当多的，为了方便运维，会让很多域名指向同样的 ip，然后统一将流量/请求分发到后端，此时就会面临一个问题：由于 TLS/SSL 在 HTTP 层之下
客户端和服务器握手的时候还拿不到 origin 字段，所以服务器不知道这个请求是从哪个域名过来的，而服务器这边每个域名都对应着一个证书，服务器就不知道该返回哪个证书啦。

SNI 就是用来解决这个问题的，官方解释是

SNI（Server Name Indication）是为了解决一个服务器使用多个域名和证书的SSL/TLS扩展。一句话简述它的工作原理就是，在连接到服务器建立SSL链接之前先发送要访问站点的域名（Hostname），这样服务器根据这个域名返回一个合适的证书。

然后有将近 25% 的浏览器不支持该字段的扩展，这个问题有两个通用解决方案：

1 使用 VIP 服务器，每个域名对应一个 VIP，然后 VIP 与统一接入服务对接，通过 ip 来分发证书，不过运维成本很高，可能也需要大量的 VIP 服务器

2 采用多泛域名，将多个泛域名证书打包进一个证书，可以看看 淘宝 页面的证书

它的缺点是每次添加域名都需要更新证书。

几个细节知识点

1. 证书选择

证书有多种加密方式，不同的加密方式对 CPU 计算的损耗不同，安全级别也不同。TLS 在进行第一次握手的时候，客户端会向服务器端 say hello，这个时候会告诉服务器，它支持哪些算法，此时服务器可以将最适合的证书发给客户端。

2. 证书的吊销

CA 证书的吊销存在两种机制，一种是在线检查，client 端向 CA 机构发送请求检查 server 公钥的靠谱性；
第二种是 client 端储存一份 CA 提供的证书吊销列表，定期更新。前者要求查询服务器具备良好性能，后者要求每次更新提供下次更新的时间，一般时差在几天。安全性要求高的网站建议采用第一种方案。

大部分 CA 并不会提供吊销机制（CRL/OCSP），靠谱的方案是为根证书提供中间证书，一旦中间证书的私钥泄漏或者证书过期，可以直接吊销中间证书并给用户颁发新的证书。
中间证书的签证原理于上上条提到的原理一样，中间证书还可以产生下一级中间证书，多级证书可以减少根证书的管理负担。

很多 CA 的 OCSP Server 在国外，在线验证时间比较长，如果可以联系 CA 供应商将 Server 转移到国内，效率可以提升 10 倍左右。

3. PKI 体系

比较主流的两种方案是 HPKP 和 Certificate Transparency：

HPKP 就是用户第一次访问的时候记下 sign 信息，以后不匹配则拒绝访问，这存在很大的隐患，比如 Server 更新了证书，或者用户第一次访问的时候就被人给黑了

Certificate Transparency 意思就是让 CA 供应商透明化 CA 服务日志，防止 CA 供应商偷偷签证
