---
title: "Master Authentication"
date: 2022-12-07T15:15:48+08:00
draft: true
author: "whiledoing"
tags: ["cryptology"]
categories: ["tech"]
---

学习鉴权相关技术的记录.

<!--more-->

## JWT(json web token)

> JSON Web Token (JWT) is an open standard (RFC 7519) that defines a compact and self-contained way for securely transmitting information between parties as a JSON object. This information can be verified and trusted because it is digitally signed. JWTs can be signed using a secret (with the HMAC algorithm) or a public/private key pair using RSA or ECDSA.

JWT有3个部分复合而成, 形式上为`header.payload.signature`:

- header: 记录元信息, 比如签名的方法

```json
{
    "alg": "RS256",
    "typ": "JWT"
}
```

- payload: 实际透传内容, 需使用`Base64URLEncode`来编码

```bash
echo -n '{"sub": "1234567890", "name": "John Doe", "admin": true}' | base64 \
    | tr -d "\n=" \     # 去掉换行和=号
    | tr "+/" "-_"      # 替换base64编码为url方式
```

因为JWT可能被放在`URL`请求参数中, 所以内容必须是纯文本, 且不能有`HTTP`协议特殊字符. 标准base64编码使用`+/`编码, 这里需要改为`_-`:

```go
const encodeStd = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/"
const encodeURL = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789-_"
```

- signature

> A digital signature is a mathematical technique used to validate the authenticity and integrity of a message, software or digital document. It's the digital equivalent of a handwritten signature or stamped seal, but it offers far more inherent security. A digital signature is intended to solve the problem of tampering and impersonation in digital communications.

数字签名可保证payload数据完整性以及不可伪造篡改. 其计算机制可描述为: `secret(hash(base64(header) + "." + base64(payload)))`.

其中, `hash`方法提取摘要信息, `secret`用来加密或者混淆数据. `hash`保证完整性, 且提取的摘要信息不会太大; 而`secret`方法保证数据不可伪造篡改: 别人即使知道计算方法, 也没法知道`secret`, 也就没法正向伪造出签名.

签名的另一面是验证, 其逻辑和构造签名一样: **就是重新计算一遍**: 验证计算结果和原始签名一致, 就可说明数据本身是完整的, 且没有被恶意伪造或者篡改.

![digital-signature-verify](./digital-signature.svg "")

---

JWT常用的计算算法有`HMAC SHA256(HS256)`和`RSA SHA256(RS256)`, 前者是对称加密, 后者是非对称加密.

在分布式系统中, 私钥一般不会大范围共享的, 更多是共享公钥, 所以用`RS256`方法更安全. 其计算公式为:

```golang
// RSA中, 一般用公钥做加密, 而私钥用来解密, 所以这里用encrypt.
// 那么对应的, 签名流程中描述的secret, 就是用私钥进行decrypt.
// 在RSA中, 先用公钥加密再用私钥解密 == 先用私钥解密再用公钥加密
// @see https://github.com/golang-jwt/jwt/blob/main/rsa.go#L49
rsa.encrypt(
    SHA256(
        base64UrlEncode(header) + "." +
        base64UrlEncode(payload)
    ),
    rsa_public_key
)
```

![key-pair](./rsa-key-pair.jpeg "key-pair")

可以用`openssl dgst`进行签名和验证, [参考文章](https://techdocs.akamai.com/iot-token-access-control/docs/generate-jwt-rsa-keys):

```bash
# 加签
echo -n '{"sub": "1234567890", "name": "John Doe", "admin": true}' | base64 \
    | tr -d "\n=" \
    | tr "+/" "-_" \
    | openssl dgst -sha256 -binary -sign jwt-private.pem -out sign.temp

# 验签
echo -n '{"sub": "1234567890", "name": "John Doe", "admin": true}' | base64 \
    | tr -d "\n=" \
    | tr "+/" "-_" \
    | openssl dgst -sha256 -verify jwt-public.pem -signature sign.temp
```

## RSA

## X.509

## SSO
