# 协议加密

从12w17a开始，Minecraft为在线模式服务器使用加密连接。

## 概述

```markdown
  C->S : Handshake State=2
  C->S : Login Start
  S->C : Encryption Key Request
  (Client Auth)
  C->S : Encryption Key Response
  (Server Auth, Both enable encryption)
  S->C : Login Success
```

1. 参见 [协议常见问题解答](Protocol_FAQ.md) 以获取下一步发生什么的信息。

## 服务器ID字符串

**现代版 (1.7.x):** 服务器ID现在以空字符串的形式发送。哈希也使用公钥，因此它们仍然是正确的。

**旧版-1.7.x:** 服务器ID字符串是随机生成的字符串，最大长度为20个代码点(如果长度超过20个，则客户端会断开连接，例外情况下会断开连接)。

如果服务器ID字符串包含某些无法打印的字符，则客户端似乎得到了不正确的哈希值。因此，为了获得一致的结果，只应发送代码点在 `U+0021` ~ `U+007E` (包含)范围内的字符。除了空格字符( `U+0020` )和所有控制字符( `U+0000 ` ~ `U+001F` , `U+007F` )外，此范围对应于所有ASCII。

如果服务器ID字符串太短，则客户端似乎得到了不正确的哈希值。已从Notchian服务器上观察到15到20个(包括15个)长度的字符串，并确认从1.5.2开始工作。

## 密钥交换

服务器在启动时生成1024位的RSA密钥对。公钥以DER编码格式在加密请求包中发送。更严格地说，它是x.509定义的ASN.1格式，其结构如下所示:

```Java
SubjectPublicKeyInfo ::= SEQUENCE {
  algorithm SEQUENCE {
    algorithm         OBJECT IDENTIFIER
    parameters        ANY OPTIONAL
  }
  subjectPublicKey  BITSTRING
}

SubjectPublicKey ::= SEQUENCE {
  modulus           INTEGER
  publicExponent    INTEGER
}
```

如果您正努力使用加密库导入它，请尝试找到一个加载DER编码公钥的函数。如果找不到，可以将其转换为更常见的PEM编码，方法是对原始字节进行base64编码，并将base64文本包装在 `-----BEGIN PUBLIC KEY-----` 和 `-----END PUBLIC KEY-----` 。请参见下面的PEM编码密钥示例: https://git.io/v7Ol9

## 对称加密

接收到来自服务器的加密请求后，客户端将生成一个随机的16字节共享机密，与AES / CFB8流密码一起使用。然后，它使用服务器的公共密钥（已填充PKCS＃1 v1.5）对其进行加密，并以相同的方式对在加密请求数据包中接收到的验证令牌进行加密，然后将两者都通过加密应答数据包发送到服务器。由于有填充，加密应答数据包中的两个字节数组都将为128个字节长。这是客户端唯一使用服务器的公共密钥的时间。

服务器使用其私钥解密共享密钥和令牌，并检查令牌是否相同。然后，它发送登录成功，并启用AES / CFB8加密。对于初始向量(IV)和AES设置，双方都将共享机密用作IV和密钥。同样，客户端也将在发送加密应答后启用加密。从现在开始，所有内容都被加密。注意: 整个数据包都是加密的，包括长度字段和数据包的数据。

登录成功数据包被加密发送。

Note: AES密码会不断更新，并不是每一个包都完成并重新启动的。

## 身份验证

如果服务器处于联机模式，则服务器和客户端都需要向 `sessionserver.mojang.com` 发出请求。

### 客户端

生成公钥后，客户端生成以下哈希:

```markdown
sha1 := Sha1()
sha1.update(ASCII encoding of the server id string from Encryption Request) 
sha1.update(shared secret) 
sha1.update(server's encoded public key from Encryption Request) 
hash := sha1.hexdigest()  # String of hex characters
```

Note: Minecraft使用的Sha1.hexdigest()方法是非标准的。它与大多数编程语言和库中的摘要方法不匹配。它的工作原理是将sha1输出字节作为2的补码中的一个大整数，然后打印以16为基数的整数，如果解释的数字为负数，则放置一个负号。Minecraft中sha1的一些样例如下: 

```markdown
sha1(Notch) :  4ed1f46bbe04bc756bcb17c0c7ce3e4632f06a48
sha1(jeb_)  : -7c9d5b0044c130109a5d7b5fb5c317c02b4e28c1
sha1(simon) :  88e16a1019277b15d58faf0541e11910eb756f6
```

结果散列然后通过HTTP POST请求发送到

```http
https://sessionserver.mojang.com/session/minecraft/join
```

包含以下作为POST数据发送的 `Content-Type` 为 `application/json`

```json
  {
    "accessToken": "<accessToken>",
    "selectedProfile": "<player's uuid without dashes>",
    "serverId": "<serverHash>"
  }
```

在 [authentication](https://wiki.vg/Authentication#Authenticate) 期间会收到字段 `<accessToken>` 和玩家的 UUID。

如果一切顺利，客户端将收到 `HTTP/1.1 204 No Content` 应答。

### 服务端

在第二个加密应答中解密公钥后，服务器生成如上所述的登录哈希，并发送一个HTTP GET到

```http
https://sessionserver.mojang.com/session/minecraft/hasJoined?username=username&serverId=hash&ip=ip
```

用户名不区分大小写，必须与客户端的用户名(在登录开始包中收到)匹配。请注意，这是所选配置文件的游戏内昵称，而不是Mojang帐户名(从不发送到服务器)。服务器应使用 `name` 字段中发送的名称。

`ip` 字段是可选的，当出现时应该是玩家的IP地址；它是最初发起会话请求的那个。Notchian服务器只有在 `server.properties` 中将 `prevent-proxy-connections` 设置为 `True` 时才包含此项。

应答是一个JSON对象，包含用户的UUID和皮肤blob:

```json
{
    "id": "<profile identifier>",
    "name": "<player name>",
    "properties": [ 
        {
            "name": "textures",
            "value": "<base64 string>",
            "signature": "<base64 string; signed data using Yggdrasil's private key>"
        }
    ]
}
```

然后使用登录成功包将 `id` 和 `name` 字段发送回客户端。JSON应答中的配置文件id的格式为 `111111111222333445555555555` ，在将其发送回客户端之前，需要将其更改为格式 `11111111-2222-3333-4444-55555555` 。

### 例程

生成Minecraft风格的十六进制摘要的示例:

- C++: https://git.io/JfTkx
- C#: https://git.io/fhjp6
- Go: http://git.io/-5ORag
- Java: http://git.io/vzbmS
- node.js: http://git.io/v2ue_A
- PHP: https://git.io/fxcFY
- Python: https://git.io/vQFUL
- Rust: https://git.io/fj6P0

## 其他链接

[DER Encoding of ASN.1 Types](https://msdn.microsoft.com/en-us/library/windows/desktop/bb648640.aspx)

[A Layman's Guide to a Subset of ASN.1, BER, and DER](http://luca.ntop.org/Teaching/Appunti/asn1.html)

[Serializing an RSA Key Manually](https://gist.github.com/Lazersmoke/9947ada8acdd74a8b2e37d77cf1e0fdc)

[Encrypt shared secret using OpenSSL](https://gist.github.com/3900517)

[Generate RSA-Keys and building the ASN.1v8 structure of the x.509 certificate using Crypto++](http://pastebin.com/8eYyKZn6)

[Decrypt shared secret using Crypto++](http://pastebin.com/7Jvaama1)

[De/Encrypt data via AES using Crypto++](http://pastebin.com/MjvR0T98)

[C# AES/CFB support with bouncy castle on Mono](https://github.com/SirCmpwn/Craft.Net/blob/master/source/Craft.Net.Networking/AesStream.cs)