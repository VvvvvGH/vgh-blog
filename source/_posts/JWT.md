---
title: JWT
date: 2024-05-06 11:23:42
tags:
---
  
## 什么是 JSON Web Token？

JSON Web Token（JWT）是一种通用的基于文本的消息传输格式，用于以紧凑和安全的方式传输信息。与普遍认知相反，JWT不仅仅适用于在网络上发送和接收身份令牌——尽管这是最常见的用例。JWT还可以用于传输任何类型的数据。

一个最基本的 JWT 包含两部分：

- 称为负载（payload）的主要数据
- 一个代表关于负载和消息本身的元数据的名称/值对的 JSON 对象，称为头部（header）

JWT 的负载可以是任何东西——任何可以表示为字节数组的东西，如字符串、图片、文档等。

由于 JWT 头部是一个 JSON 对象，因此理论上 JWT 负载也可以是一个 JSON 对象。在许多情况下，开发人员希望负载是表示用户、计算机或类似身份概念的数据的 JSON。当以这种方式使用时，负载被称为 JSON Claims 对象，该对象中的每一个名称/值对都被称为一个声明——每一条信息都在声明有关某个身份的信息。

虽然关于身份提出“声明”很有用，但实际上任何人都可以这么做。重要的是，你需要通过验证这些声明是否来自你信任的人或计算机来信任这些声明。

JWT 的一个特性是它可以通过各种方式进行安全保护。JWT 可以进行加密签名（JWS）或加密（JWE）。这为 JWT 增加了一个强大的可验证性层——JWS 或 JWE 的接收者可以通过验证签名或解密来高度信任它来自于他们信任的人。正是这种可验证性特性使得 JWT 成为发送和接收敏感信息（如身份声明）的良好选择。

最后，虽然具有人类可读性的带空格的 JSON 很好，但它并不是一种非常高效的消息格式。因此，JWT 可以被压缩（甚至压缩）成最小的表示形式——基本上是 Base64URL 编码的字符串——这样它们就可以在网络上更有效地传输，如在 HTTP 头或 URL 中。

### JWT 示例

一旦你拥有了负载和头部，它们是如何为网络传输而压缩的，最终的 JWT 实际上是什么样的？让我们通过一些伪代码简化这个过程：

假设我们有一个带有 JSON 头部和简单文本消息负载的 JWT：
alg 是 JWT 的算法，none 代表无算法

```
String header = '{"alg":"none"}'; 
String payload = '智慧的真正标志不是知识而是想象力。'
```

获取 UTF-8 字节并对每一个进行 Base64URL 编码：
```
String encodedHeader = base64URLEncode(header.getBytes("UTF-8")); 
String encodedPayload = base64URLEncode(payload.getBytes("UTF-8"));
```

用点（'.'）字符连接编码后的头部和声明：

```
String compact = encodedHeader + '.' + encodedPayload + '.';
```

最终连接后的紧凑 JWT 字符串如下所示：
`eyJhbGciOiJub25lIn0.VGhlIHRydWUgc2lnbiBvZiBpbnRlbGxpZ2VuY2UgaXMgbm90IGtub3dsZWRnZSBidXQgaW1hZ2luYXRpb24u.`

这个就是最简单的 JWT，没有数字签名也没有加密吗。也是相当不安全的。


如果使用 JWS ，添加 Hmac-sha256 签名算法，则数据就变成
```
String header = '{"alg":"HS256"}'; 
String payload = '智慧的真正标志不是知识而是想象力。'
byte[] signature = hmacSha256(concatenated, key);
```
则最终拼接的 JWT 数据为
```
String concatenated = encodedHeader + '.' + encodedClaims + '.' + base64URLEncode(signature);
```