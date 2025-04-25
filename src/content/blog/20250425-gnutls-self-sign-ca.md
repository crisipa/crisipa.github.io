---
title: 使用 gnutls 替代 openssl 签发 CA 及 SSL 证书
pubDate: 2025-04-25
description: "gnutls 可比 openssl 方便简单多了！"
tags:
  - 计算机技术
  - gnutls
---

# 怎么想起用 gnutls 了？

你知道的，windows 10 是“微软推出的最后一代windows”，而微软又推出了 windows 11。虽然我大多数时间在使用 archlinux，但总有些事情必须得用 windows (对吧，matlab？)。由于忍受不了win11，我于今年年初将电脑上的 win11 重装为 win10。但是在下载 openssl 时我遇到了一点网络问题，遂下载了gnutls。

## 来签发证书吧！

此处以 [我之前用 openssl 签的证书](https://blog.crisipa.icu/posts/20231008-access-github/#%E7%AD%BE%E5%8F%91%E8%AF%81%E4%B9%A6) 举例。

参考资料： [红帽关于 gnutls 的文章](https://docs.redhat.com/zh-cn/documentation/red_hat_enterprise_linux/8/html/securing_networks/creating-a-private-ca-using-gnutls_creating-and-managing-tls-keys-and-certificates#creating-a-private-ca-using-gnutls_creating-and-managing-tls-keys-and-certificates)

### 准备工作

新建一个空文件夹并在此文件夹内打开终端，我们将在此文件夹内签发证书。
在此文件夹内新建两个文件夹，一个名为'CA'(放我们签的根证书的) 一个名为'github'(放我们签的 SSL 证书的)

### 制作CA证书

在 CA 文件夹内新建文件 CA.cfg ,写入以下内容：

```
country = "CN"
organization = "Local Fake CA LLC"
cn = "Local Fake CA"

expiration_days = 3650

ca
signing_key
cert_signing_key
crl_signing_key

```

在终端里执行命令

```
# 生成私钥
certtool --generate-privkey --outfile ./CA/ca.key
# 签发证书
certtool --generate-self-signed --load-privkey ./CA/ca.key --template ./CA/ca.cfg --outfile ./CA/ca.crt
```

### 用这个 CA 证书签发 SSL 证书

在 github 文件夹内新建文件 github.cfg ,写入以下内容：

```
country = "CN"
organization = "Github Local Fake LLC"
cn = "Github Local Fake CA"

expiration_days = 365

signing_key
encryption_key
key_agreement
tls_www_server

dns_name = "github.com"
dns_name = "*.github.com"
dns_name = "githubusercontent.com"
dns_name = "*.githubusercontent.com"
dns_name = "github.githubassets.com"

ip_address = "127.0.0.1"
ip_address = "::1"

```

在终端里执行命令

```
# 生成私钥
certtool --generate-privkey --outfile ./github/github.key
# 签发证书
certtool --generate-certificate --load-privkey ./github/github.key --template ./github/github.cfg --load-ca-certificate ./CA/ca.crt --load-ca-privkey ./CA/ca.key --outfile ./github/github.crt
```

## 后记

就上面的用途而言，gnutls 比 openssl 方便多了。
似乎 cloudflare 也有个这方面的工具 [cfssl](https://github.com/cloudflare/cfssl)，找个机会试一试。
