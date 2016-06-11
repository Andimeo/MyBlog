title: Certificate
date: 2016-06-11 18:17:53
categories: 密码学
tags:
- Certificate
- DER
- PEM
- OpenSSL
---
在Luminate里接手的第一个项目，是做一个IPsec的客户端，来测试已有的IPsec gateway。在这个项目中，已经快把[RFC7296](https://tools.ietf.org/html/rfc7296)翻烂了，在这中间学习了C++，学习了emacs。接下来要做的支持在IKE negotiation中的Certificate Payload，之前对证书等加密的概念都是一知半解，今天发现一篇[文章](<https://support.ssl.com/Knowledgebase/Article/View/19/0/der-vs-crt-vs-cer-vs-pem-certificates-and-how-to-convert-them>
)阐述了各种证书格式的异同，翻译于此。

# 证书与编码 #
X.509本质上是按照RFC 5280编码或签名的一个数字文件。

实际上，X.509这个词是指X.509 v3标准的IETF’s PKIX Certificate and CRL Profile，在RFC 5280中说明，通常被称作PKIX for Public Key Infrastructure (X.509)

<!-- more -->

# X509 文件扩展名 #
首先我们要弄明白每一种扩展名对应的是哪种类型。DER、PEM、CRT和CER这些扩展名给我们带来了很多困惑，很多人都错误的认为这些可以随意使用，彼此等价。然而只有在某些情况下，其中的一些才可以交换，而最好的办法就是知道你的证书是什么编码，然后用合适的扩展名标记它。正确命名的证书更容易使用。

## 编码(扩展名) ##
  * `.DER` = 扩展名DER用于按照DER格式编码的二进制证书。这些文件也可以被命名为CER或CRT。正确的说法是“我有一个DER编码的证书”，而不是“我有一个DER证书”。
  * `.PEM` = 扩展名PEM用于不同类型的X.509 v3文件。这些证书包含ASCII(Base64)的数据，以“--BEGIN ...” 开头。
  
## 通用扩展 ##
  * `.CRT` = 扩展名CRT用于证书。证书的内容可能是DER格式的二进制数据，也可能是ASCII的PEM数据。CER和CRT几乎是一样的意思，而且在*nix系统中最常见。
  * `.CER` = 扩展名CER是CRT的另一个形式，是微软的习惯。你可以用MS把CRT转换为CER，不论是DER格式还是PEM格式。.CER扩展名的文件可以被IE识别为一个执行微软cryptoAPI的命令，IE会弹出一个对话框供你导入或查看证书。
  * `.KEY` = 扩展名KEY同时用于PKCS#8的公钥和私钥。这些公钥私钥既可能是DER格式的，也可能是PEM格式的。
  
# OpenSSL的证书操作 #
一般而言有四种类型的证书操作：查看，转换，组合和提取。

## 查看 ##
即使PEM文件是ASCII的，他们仍然不是供人阅读的。下面是一些让你以人类可读的方式输出证书内容的一些命令。

### 查看PEM格式证书 ###
将下面命令中的证书替换为你自己的对应扩展名的证书：

``` shell
openssl x509 -in cert.pem -text -noout
openssl x509 -in cert.cer -text -noout
openssl x509 -in cert.crt -text -noout
```

如果你碰到下面的错误，那以为着你在试图查看一个DER格式的证书，DER格式的证书在下面一节说明。

``` shell
unable to load certificate
12626:error:0906D06C:PEM routines:PEM_read_bio:no start line:pem_lib.c:647:Expecting: TRUSTED CERTIFICATE
```

### 查看DER格式证书 ###

``` shell
openssl x509 -in certificate.der -inform der -text -noout
```
如果你碰到下面的错误，那以为着你在试图查看一个PEM格式的证书，DER格式的证书在下面一节说明。

``` shell
unable to load certificate
13978:error:0D0680A8:asn1 encoding routines:ASN1_CHECK_TLEN:wrong tag:tasn_dec.c:1306:
13978:error:0D07803A:asn1 encoding routines:ASN1_ITEM_EX_D2I:nested asn1 error:tasn_dec.c:380:Type=X509
```

## 转换 ##
转换可以把一种编码格式的证书变为另一种格式的编码，如把PEM格式变为DER格式。

### PEM to DER ###

``` shell
openssl x509 -in cert.crt -outform der -out cert.der
```

### DER to PEM ###

``` shell
openssl x509 -in cert.crt -inform der -outform pem -out cert.pem
```

## 组合 ##
在某些情况下，我们需要把多个X509的片段放到一个证书文件中。一个例子就是我们需要把公钥和私钥放在同一个证书中。

最简单的办法是把多个证书转换为PEM格式，然后复制到一个新的文件中。Apache就会需要这种格式的证书。

## 提取 ##
一些证书是上节提到的组合后的格式。一个文件可能包含证书、私钥、公钥、签名、CA或Authority Chain。
