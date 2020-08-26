# 服务器列表Ping

**服务器列表Ping** (**Server List Ping, SLP**) 是Minecraft服务器提供的接口，支持通过常用端口查询MOTD、玩家数量、最大玩家人数和服务器版本。服务器列表Ping是 [协议](Protocol.md) 的一部分，所以不像 [Query](https://wiki.vg/Query) ，接口始终处于启用状态。Notchian客户端使用此接口显示多人游戏服务器列表，因此得名。在 [1.7](http://minecraft.gamepedia.com/1.7) 中以非向后兼容的方式更改了服务器列表Ping的过程，但 [当前版本](#当前版本) 和 [旧版本](#1.6) 仍然支持。

## 当前版本

它使用常规的 [协议](Protocol.md) ，关于一般的数据包格式，请参阅这篇文章。

### 握手

首先，客户端发送一个 [握手](Protocol.md#握手) 包，其 `Next State` 为 `1` 。

| 包ID |     字段名称     |    字段类型    |                             备注                             |
| :--: | :--------------: | :------------: | :----------------------------------------------------------: |
| 0x00 | Protocol Version |     VarInt     | 参见[协议版本号](https://wiki.vg/Protocol_version_numbers) 。客户端计划用于连接到服务器的版本（这对于ping来说并不重要）。如果客户端正在ping以确定要使用的版本，则应按约定设置 `-1` 。 |
|      |  Server Address  |     String     | 主机名或 IP ，例如 `localhost` 或 `127.0.0.1 `，这会被用于连接。官方服务端不使用该信息。 注意，SRV记录会被完全重定向，例如 `_minecraft._tcp.example.com` 代表 `mc.example.com` ，当用户试图连接 `mc.example.com` 时，服务器会自动重定向到`_minecraft._tcp.example.com` 所指向的服务器。 |
|      |   Server Port    | Unsigned Short |         默认值为 `25565` ，官方服务端不使用该信息。          |
|      |    Next state    |     VarInt     | `1` 表示 [状态](Protocol.md#状态) ， `2` 表示 [登录](Protocol.md#登录) |

### 请求

客户随后发出 [请求](Protocol.md#请求) 包，这个包里没有字段。

| 包ID | 字段名称 | 字段类型 | 备注 |
| :--: | :------: | :------: | :--: |
| 0x00 | *无字段* |          |      |

### 应答

服务器应使用 [应答](Protocol.md#应答) 包。注意，Notchian服务器由于未知原因将等待接收以下 [Ping](Protocol.md#Ping) 在超时和发送响应前30秒发送数据包。

| 包ID | 字段名称 | 字段类型 |                             备注                             |
| :--: | :------: | :------: | :----------------------------------------------------------: |
| 0x00 | JSON应答 |  String  | 如下所示，与所有字符串一样，这是以其长度作为前缀的VarInt类型变量 |

JSON响应字段是一个 [JSON](http://en.wikipedia.org/wiki/JSON) 对象，具有以下格式:

```json
{
    "version": {
        "name": "1.8.7",
        "protocol": 47
    },
    "players": {
        "max": 100,
        "online": 5,
        "sample": [
            {
                "name": "thinkofdeath",
                "id": "4566e69f-c907-48ee-8d71-d7ba5aa00d20"
            }
        ]
    },	
    "description": {
        "text": "Hello world"
    },
    "favicon": "data:image/png;base64,<data>"
}
```

其中 `description` 字段是一个 [Chat](https://wiki.vg/Chat) 对象。请注意，Notchian服务器无法提供实际的聊天组件数据。取而代之的是，将基于节符号的代码嵌入到对象文本的文本中。

其中 `favicon` 字段是可选的。而 `sample` 字段必须设置，不过可以为空。

其中 `favicon` 字段需要提供一个基于 [Base64](http://en.wikipedia.org/wiki/Base64) 编码的 [PNG](http://en.wikipedia.org/wiki/Portable_Network_Graphics) 图片 (不能包含换行符: `\n`, 自从 1.13 起就不能包含换行符了) ，并使用如下数据头: `data:image/png;base64,` 。

客户端收到响应数据包后，可以发送下一个数据包以帮助计算服务器的延迟，或者如果仅对以上信息感兴趣，则可以在此处断开连接。

如果客户端未收到格式正确的响应，则它将尝试 [旧版Ping](#1.6).

### Ping

如果进程继续，客户端现在将发送 [Ping](Protocol.md#Ping) 包，其中包含一些不重要的载荷。

| 包ID | 字段名称 | 字段类型 |                             备注                             |
| :--: | :------: | :------: | :----------------------------------------------------------: |
| 0x01 | Payload  |   Long   | 可以是任何数字。Notchian客户端使用依赖于系统的时间值，以毫秒计。 |

### Pong

服务器将响应 [Pong](Protocol.md#Pong) 包然后关闭连接。

| 包ID | 字段名称 | 字段类型 |          备注          |
| :--: | :------: | :------: | :--------------------: |
| 0x01 | Payload  |   Long   | 应该与客户端发送的相同 |

### 例子

- [C#](https://gist.github.com/csh/2480d14fbbb33b4bbae3)
- [Java](https://gist.github.com/zh32/7190955)
- [Python](https://gist.github.com/1209061)
- [Python3](https://gist.github.com/ewized/97814f57ac85af7128bf)
- [PHP](https://github.com/xPaw/PHP-Minecraft-Query)

## 1.6

这使用了一个与Netty重写之前的客户端-服务端协议兼容的协议。现代服务器通过开始字节 `0xFE` 而不是通常的 `0x00` 来识别此协议。

### 客户端到服务器

客户机在标准端口上启动到服务器的TCP连接。而不是进行身份验证和登录 (参见 [协议](Protocol.md) and [Protocol Encryption](https://wiki.vg/Protocol_Encryption))，它发送以下以十六进制表示的数据:

1. `FE` — 服务器列表Ping的数据包标识符。
2. `01` — 服务器列表Ping的有效载荷(总是 `1` )。
3. `FA` — 插件消息的包标识符。
4. `00 0B` — 以下字符串的长度，以字符为单位，作为短字符串(始终为 `11` )。
5. `00 4D 00 43 00 7C 00 50 00 69 00 6E 00 67 00 48 00 6F 00 73 00 74` — 字符串 `MC|PingHost` 经过 [UTF-16BE](http://en.wikipedia.org/wiki/UTF-16) 编码后的样子。
6. `XX XX` — 其余数据的长度，作为short类型。计算方式为 `7 + len(hostname)` ，其中 `len(hostname)` 是以UTF-16BE 编码主机名的字节数。
7. `XX` — [protocol version](https://wiki.vg/Protocol_version_numbers#Versions_before_the_Netty_rewrite) 。例如， `4a` 是最新版本 (74)。
8. `XX XX` — 以下字符串的长度，以字符为单位，作为short类型。
9. `...` — 客户端连接到的主机名，以 [UTF-16BE](http://en.wikipedia.org/wiki/UTF-16) 编码的字符串。
10. `XX XX XX XX` —客户端连接到的端口，作为int类型。

所有数据类型都是以大端模式(big-endian)存储的。

数据包存储示例:

```markdown
0000000: fe01 fa00 0b00 4d00 4300 7c00 5000 6900  ......M.C.|.P.i.
0000010: 6e00 6700 4800 6f00 7300 7400 1949 0009  n.g.H.o.s.t..I..
0000020: 006c 006f 0063 0061 006c 0068 006f 0073  .l.o.c.a.l.h.o.s
0000030: 0074 0000 63dd                           .t..c.
```

### 服务器到客户端

服务器以 `0xFF` kick包响应。包以单字节标识符 `0xFF` 开头，然后是一个双字节的big-endian short，以字符形式给出以下字符串的长度。实际上可以忽略长度，因为服务器在发送响应后关闭连接。

在前3个字节之后，数据包是一个UTF-16BE字符串。它以两个字符开头: `§1`，后跟一个空字符。在数据流上看起来像 `00 a7 00 31 00 00` 。

剩余部分为空字符（即 `00 00` ）分隔的字段:

1. 协议版本 (例如 `74`)
2. Minecraft 服务器版本 (例如 `1.8.7`)
3. MOTD消息 (例如 `A Minecraft Server`)
4. 在线玩家人数
5. 最大玩家人数

整个数据包看起来像这样:

```markdown
                <---> first character
0000000: ff00 2300 a700 3100 0000 3400 3700 0000  ....§.1...4.7...
0000010: 3100 2e00 3400 2e00 3200 0000 4100 2000  1...4...2...A. .
0000020: 4d00 6900 6e00 6500 6300 7200 6100 6600  M.i.n.e.c.r.a.f.
0000030: 7400 2000 5300 6500 7200 7600 6500 7200  t. .S.e.r.v.e.r.
0000040: 0000 3000 0000 3200 30                   ..0...2.0
```

**Note:** 在1.7.x及更高版本的服务器上使用此协议时，响应中的协议版本(第一个字段)将始终为 `127` ，这不是真正的协议号，因此较旧的客户端将始终认为此服务器不兼容。

### 例子

- [Ruby](https://gist.github.com/6281388)
- [PHP](https://github.com/winny-/mcstat)

## 1.4 到 1.5

在Minecraft 1.6之前，客户机到服务器的操作要简单得多，只发送 `FE 01` ，没有别的任何数据。

### 例子

- [PHP](https://gist.github.com/5795159)
- [Java](https://gist.github.com/4574114)
- [C#](https://gist.github.com/6223787)

## 1.3 到 1.8 Beta 版

在Minecraft 1.4之前，客户端只发送 `FE` 。

另外，来自服务器的响应只包含3个字段，用 `§` 分隔:

1. MOTD消息 (例如 `A Minecraft Server`)
2. 在线玩家人数
3. 最大玩家人数

整个数据包看起来像这样:

```markdown
                <---> first character
0000000: ff00 1700 4100 2000 4d00 6900 6e00 6500  ....A. .M.i.n.e.
0000010: 6300 7200 6100 6600 7400 2000 5300 6500  c.r.a.f.t. .S.e.
0000020: 7200 7600 6500 7200 a700 3000 a700 3100  r.v.e.r.§.0.§.1.
0000030: 30                                       0
```



From https://wiki.vg/Server_List_Ping