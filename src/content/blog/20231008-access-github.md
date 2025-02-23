---
title: 用caddy本地反代github
pubDate: 2023-10-08
#updatedDate: 2025-02-11
description: "使用 caddy 在本地对 github 进行反向代理"
tags:
  - 计算机技术
  - caddy
  - 反向代理
  - github
---

# 前言

github的网络问题已经是老生常谈的了。

## 准备工具

需要安装的软件有 [caddy](https://caddyserver.com/) [openssl](https://github.com/openssl/openssl/)

### 安装 openssl

windows用户需从这里下载 openssl：[https://slproweb.com/products/Win32OpenSSL.html](https://slproweb.com/products/Win32OpenSSL.html)

安装后记得配置下环境变量（Windows用户专属）：把openssl安装目录下的bin文件夹的路径加进path变量里

## 签发证书

参考资料： [OpenSSL 自签 CA 及 SSL 证书](https://2heng.xin/2018/12/16/your-own-ca-with-openssl/)

### 准备工作

找一个空文件夹，我们将在此文件夹内签发证书

新建一个文本文件，命名为 openssl.conf （注意文件的后缀名）

打开它，写入以下内容：

```
[ ca ]
default_ca = CA_default

[ CA_default ]
dir = ./CA
database = $dir/index.txt
new_certs_dir = $dir/newcerts
certificate = $dir/cacert.pem
serial = $dir/serial
private_key = $dir/cakey.pem
default_md = default
policy = policy_match

[ policy_match ]
countryName = match
stateOrProvinceName = optional
organizationName = match
organizationalUnitName = optional
commonName = supplied
emailAddress = optional

```

随后在此文件夹打开终端，执行以下命令：

```
mkdir ./CA/newcerts
touch ./CA/index.txt
echo 01 > ./CA/serial
```

### 制作CA证书

在此文件夹里，新建一个文件夹,命名为 CA

新建一个文本文件，命名为 root.conf （注意文件的后缀名）

打开它，写入以下内容：

```
[ req ]

default_bits = 2048
default_keyfile = r.pem
default_md = sha256
string_mask = nombstr
distinguished_name = req_distinguished_name
req_extensions = req_ext
x509_extensions = x509_ext

[ req_distinguished_name ]

countryName = CN
countryName_default = CN
organizationName = Local Fake CA LLC
organizationName_default = Local Fake CA LLC
commonName = Local Fake CA
commonName_default = Local Fake CA

[ x509_ext ]

subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
basicConstraints = CA:TRUE
keyUsage = digitalSignature, keyEncipherment, keyCertSign, cRLSign

[ req_ext ]

subjectKeyIdentifier = hash
basicConstraints = CA:TRUE
keyUsage = digitalSignature, keyEncipherment, keyCertSign, cRLSign

```

然后在之前的终端里输入命令

```
openssl genrsa -out ./CA/cakey.pem 2048
openssl req -new -x509 -key ./CA/cakey.pem -out ./CA/cacert.pem -days 7200 -config ./CA/root.conf
openssl x509 -inform PEM -in ./CA/cacert.pem -outform DER -out ./CA/CA.cer
```

之后，请将 CA.cer 导入为 ‘受信任的根证书颁发机构’

### 用这个CA证书签SSL证书

在制作CA证书的文件夹里，再新建一个文件夹,命名为 github

在 github 文件夹里新建一个文本文件，命名为 github.conf （注意文件的后缀名）

打开它，写入以下内容：

```
[ req ]
 
default_bits = 2048
default_keyfile = r.pem
default_md = sha256
string_mask = nombstr
distinguished_name = req_distinguished_name
req_extensions = req_ext
x509_extensions = x509_ext

[ req_distinguished_name ]
 
countryName = CN
countryName_default = CN
organizationName = Local Fake CA LLC
organizationName_default = Local Fake CA LLC
commonName = Github Local Fake CA
commonName_default = Github Local Fake CA

[ x509_ext ]
 
subjectKeyIdentifier = hash
authorityKeyIdentifier = keyid,issuer
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
subjectAltName = @alt_names

[ req_ext ]
 
subjectKeyIdentifier = hash
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
subjectAltName = @alt_names

[ alt_names ]
 
IP.1 = 127.0.0.1
IP.2 = ::1

DNS.01 = github.com
DNS.02 = *.github.com
DNS.03 = githubusercontent.com
DNS.04 = *.githubusercontent.com
DNS.05 = github.githubassets.com

```

然后在之前的终端里输入命令

```
openssl genrsa -out ./github/github.key 2048
openssl req -new -key ./github/github.key -out ./github/github.csr -config ./github/github.conf
openssl ca -config ./openssl.conf -in ./github/github.csr -out ./github/github.crt -days 3650 -extensions x509_ext -extfile ./github/github.conf
```

于是，你得到了后文 caddy 要用的文件 github.crt github.key

## 制作 Caddyfile

在你放caddy的文件夹里，新建一个txt文件 Caddyfile (没有后缀名)

如果是用包管理器安装的，则请打开 caddy 的配置文件目录

其中最重要的就是 auto_https off 。若是省去，caddy会自动帮你申请证书。但是我们是要在本地进行反向代理，申请不了证书的。

编辑 Caddyfile：

```
{
	# General Options
	# http_port 80
	# https_port 443
	default_bind 127.0.0.1

	# TLS Options
	# auto_https off|disable_redirects|ignore_loaded_certs|disable_certs
	auto_https off

	# log
}

```

然后是对 github 的配置

```
## github
github.com {
	tls path/to/github.crt path/to/github.key
	reverse_proxy {
		to 140.82.112.3:443
		to 140.82.112.4:443
		to 140.82.113.3:443
		to 140.82.113.4:443
		to 140.82.114.3:443
		to 140.82.114.4:443
		to 140.82.121.3:443
		to 140.82.121.4:443
		header_up Host {http.request.host}
		transport http {
			tls
			tls_insecure_skip_verify
		}
		lb_policy random
	}
}
github-releases.githubusercontent.com objects.githubusercontent.com {
	tls path/to/github.crt path/to/github.key
	reverse_proxy {
		# github
		to 185.199.108.133:443
		to 185.199.108.154:443
		to 185.199.109.133:443
		to 185.199.109.154:443
		to 185.199.110.133:443
		to 185.199.110.154:443
		to 185.199.111.133:443
		to 185.199.111.154:443
		header_up Host {http.request.host}
		transport http {
			tls
			tls_insecure_skip_verify
		}
		lb_policy random
	}
}
codeload.github.com {
	tls path/to/github.crt path/to/github.key
	reverse_proxy {
		to 140.82.112.9:443
		to 140.82.112.10:443
		to 140.82.121.9:443
		to 140.82.121.10:443
		header_up Host {http.request.host}
		transport http {
			tls
			tls_insecure_skip_verify
		}
		lb_policy random
	}
}
raw.githubusercontent.com raw.github.com camo.githubusercontent.com cloud.githubusercontent.com avatars.githubusercontent.com avatars0.githubusercontent.com avatars1.githubusercontent.com avatars2.githubusercontent.com avatars3.githubusercontent.com user-images.githubusercontent.com {
	tls path/to/github.crt path/to/github.key
	reverse_proxy {
		to 151.101.0.133:443
		to 151.101.64.133:443
		to 151.101.108.133:443
		to 151.101.128.133:443
		to 151.101.192.133:443
		to 199.232.36.133:443
		header_up Host {http.request.host}
		transport http {
			tls
			tls_insecure_skip_verify
		}
		lb_policy random
	}
}
```

请不要忘记将 github.crt 和 github.key 分别换成之前得到的 SSL 证书及密钥的绝对路径

你还可以用这方式上steam :)

## [](#启动 "启动")启动

直接在你放caddy的文件夹里开一个终端运行条命令就启动caddy了

```
./caddy run
```

然后修改hosts文件，加入这些：

```
## github

127.0.0.1 github.com
127.0.0.1 github-releases.githubusercontent.com
127.0.0.1 objects.githubusercontent.com
127.0.0.1 codeload.github.com
127.0.0.1 raw.githubusercontent.com
127.0.0.1 raw.github.com
127.0.0.1 camo.githubusercontent.com
127.0.0.1 cloud.githubusercontent.com
127.0.0.1 avatars.githubusercontent.com
127.0.0.1 avatars0.githubusercontent.com
127.0.0.1 avatars1.githubusercontent.com
127.0.0.1 avatars2.githubusercontent.com
127.0.0.1 avatars3.githubusercontent.com
127.0.0.1 user-images.githubusercontent.com
```

好了，github的使用体验应该已经得到了改善

## 一些问题

### 1.ssh用不了

个人技术有限，而且也不是必须用ssh： [https://docs.github.com/zh/get-started/getting-started-with-git/caching-your-github-credentials-in-git](https://docs.github.com/zh/get-started/getting-started-with-git/caching-your-github-credentials-in-git)

### 2.白屏或502

多试几次就行了
