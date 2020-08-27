# Mojang API

## 前言

- 所有公共API均受速率限制，因此您应该缓存结果。当前设置为每10分钟限制600个请求，但可能会更改。
- 对于API的某些部分，有时会包括演示帐户，有时则不包括。 Mojang不断改变这一点。

**目录**
<!-- TOC -->

- [前言](#前言)
- [请求状态](#请求状态)
  - [应答](#应答)
- [name -> UUID （在时间戳上）](#name---uuid-在时间戳上)
  - [应答](#应答-1)
- [UUID -> 历史 name](#uuid---历史-name)
  - [应答](#应答-2)
- [多个 name -> 多个 UUID](#多个-name---多个-uuid)
  - [载荷](#载荷)
  - [应答](#应答-3)
- [UUID -> 个人资料 + 皮肤/斗篷](#uuid---个人资料--皮肤斗篷)
  - [应答](#应答-4)
- [更改皮肤](#更改皮肤)
  - [应答](#应答-5)
  - [请求头](#请求头)
  - [载荷](#载荷-1)
  - [例子](#例子)
- [上传皮肤](#上传皮肤)
  - [应答](#应答-6)
  - [请求头](#请求头-1)
  - [载荷](#载荷-2)
  - [例子](#例子-1)
- [重置皮肤](#重置皮肤)
  - [应答](#应答-7)
  - [请求头](#请求头-2)
  - [例子](#例子-2)
- [安全问题解答流程](#安全问题解答流程)
  - [检查是否需要安全问题](#检查是否需要安全问题)
  - [获取安全问题列表](#获取安全问题列表)
  - [发回答案](#发回答案)
- [被封禁的服务器](#被封禁的服务器)
  - [应答](#应答-8)
- [统计](#统计)
  - [载荷](#载荷-3)
  - [应答](#应答-9)
- [例程](#例程)

<!-- /TOC -->

## 请求状态

```http
 GET https://status.mojang.com/check
```

返回各种Mojang服务的状态。 可能的值为 `green` (绿色，无问题)， `yellow` (黄色，有些问题) 和 `red` (红色，服务不可用)。

### 应答

```http
[
  {
    "minecraft.net": "yellow"
  },
  {
    "session.minecraft.net": "green"
  },
  {
    "account.mojang.com": "green"
  },
  {
    "auth.mojang.com": "green"
  },
  {
    "skins.minecraft.net": "green"
  },
  {
    "authserver.mojang.com": "green"
  },
  {
    "sessionserver.mojang.com": "yellow"
  },
  {
    "api.mojang.com": "green"
  },
  {
    "textures.minecraft.net": "red"
  },
  {
    "mojang.com": "green"
  }
]
```

## name -> UUID （在时间戳上）

```http
 GET https://api.mojang.com/users/profiles/minecraft/<username>?at=<timestamp>
```

这将在提供的时间戳上返回 `Username` 的 `UUID` 。

`?at=0` 可以用来获取该用户名的原始用户的 `UUID` ，但是，只有在名称至少更改一次或帐户是旧帐户的情况下，它才有效。

- 时间戳是 [UNIX 时间戳](http://en.wikipedia.org/wiki/Unix_time) (无毫秒)
- 如果不发送 `at` 参数，则使用当前时间

### 应答

```http
{
  "id": "7125ba8b1c864508b92bb5c042ccfe2b",
  "name": "KrisJelbring"
}
```

- `name` 是 **该 `UUID` 的当前名称**，**不是请求名称**
- `legacy` 仅在为true时显示（不迁移到mojang帐户）
- `demo` 仅在为true时显示（帐户未付款）

如果没有使用给定用户名的玩家，则返回状态码204且不包含任何正文。
如果时间戳不是数字、太大或太小都会发送HTTP状态代码400（错误请求），并显示如下错误消息:

```http
{
  "error": "IllegalArgumentException",
  "errorMessage": "Invalid timestamp."
}
```

## UUID -> 历史 name

```http
  GET https://api.mojang.com/user/profiles/<uuid>/names
```

返回该用户过去使用的所有用户名和当前使用的用户名。 UUID必须不带连字符。

### 应答

```http
[
  {
    "name": "Gold"
  },
  {
    "name": "Diamond",
    "changedToAt": 1414059749000
  }
]
```

`changedToAt` 字段是Java时间戳(以毫秒为单位)

## 多个 name -> 多个 UUID

```http
 POST https://api.mojang.com/profiles/minecraft
```

这将返回多个玩家的UUID和一些其他内容。

### 载荷

```http
[
    "maksimkurb",
    "nonExistingPlayer" //Test for non-existing player
]
```

### 应答

```http
[
    {
        "id": "0d252b7218b648bfb86c2ae476954d32",
        "name": "maksimkurb",
        "legacy": true,
        "demo": true
    }
]
```

- `name` 区分大小写
- 仅当为true时才会显示旧版（配置文件未迁移到mojang.com）
- 演示仅在为true时显示（帐户未付款）
- 当任何用户名为null或""时，返回 `IllegalArgumentException` 异常
- `Content-Type` HTTP头必须为 `application/json`
- 每个请求最多只能请求10个名字

## UUID -> 个人资料 + 皮肤/斗篷

```http
 GET https://sessionserver.mojang.com/session/minecraft/profile/<uuid>
```

这将返回玩家的用户名以及有关它们的所有其他信息（例如皮肤）。比如: https://sessionserver.mojang.com/session/minecraft/profile/4566e69fc90748ee8d71d7ba5aa00d20

这有一个更严格的速率限制：您可以每分钟请求一次相同的配置文件，但是可以发送任意多的唯一请求。

### 应答

```http
{
    "id": "<profile identifier>",
    "name": "<player name>",
    "properties": [ 
        {
            "name": "textures",
            "value": "<base64 string>",
            "signature": "<base64 string; signed data using Yggdrasil's private key>" // Only provided if ?unsigned=false is appended to url
        }
    ]
}
```



- 如果用户尚未将minecraft.net帐户迁移到mojang，则 `"legacy": true` 将出现在响应中。

The "value" base64 string for the "textures" object decoded:

应答对象中，将 `properties` 中的 `value` 键作为base64解码后得到的 `textures` 对象:

```http
{
    "timestamp": <java time in ms>,
    "profileId": "<profile uuid>",
    "profileName": "<player name>",
    "signatureRequired": true, // Only present if ?unsigned=false is appended to url
    "textures": {
        "SKIN": {
            "url": "<player skin URL>"
        },
        "CAPE": {
            "url": "<player cape URL>"
        }
    }
}
```

- 时间戳有时是过去的（可能是由于缓存的结果？）
- 如果玩家的模型手臂很纤细，则 `"SKIN"` 对象中会出现 `"metadata": {"model": "slim"}` (Alex风格). 对于方形手臂，则不会出现 `"metadata"` (Steve风格)。
- 如果未设置自定义皮肤， `"SKIN"` 就不会在应答中出现。
- 玩家是否拥有Alex风格或Steve风格的外观取决于其UUID的 [Java hashCode](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/classes/java/util/UUID.java#l394) 。该值如果是偶数则为Steve风格。示例实现：

  - [PHP](https://github.com/mapcrafter/mapcrafter-playermarkers/blob/c583dd9157a041a3c9ec5c68244f73b8d01ac37a/playermarkers/player.php#L8-L19)
  - [Go](https://github.com/LapisBlue/Lapitar/blob/55ede80ce4ebb5ecc2b968164afb40f61b4cc509/mc/uuid.go#L34-L36)
  - [JavaScript](https://github.com/crafatar/crafatar/blob/9d2fe0c45424de3ebc8e0b10f9446e7d5c3738b2/lib/skins.js#L90-L108) (包含说明)
  - [Java](https://web.archive.org/web/20151022205119/https://gist.github.com/jomo/9968b8d572c38e1b1f4c) (包含样例 UUIDs)
- 同样，如果该账户没有披肩，那么 `"CAPE"` 也不会在应答中出现。

## 更改皮肤

```http
POST https://api.mojang.com/user/profile/<uuid>/skin
```

这将为所选配置文件设置外观，但是Mojang的服务器将从URL获取外观。 这也适用于旧帐户。

### 应答

发生错误时，服务器将发回带有错误的JSON（成功是空有效载荷）。

### 请求头

```http
Authorization: Bearer <access token>
```

### 载荷

此API的有效载荷包含两个url编码的表单字段（以'＆'组合）。

```http
model=<""/"slim">&url=<skin url>
```

model为默认模型则将字符串留空，使用slim模型填写`slim` 。

### 例子

```http
curl -H "Authorization: Bearer <access token>" --data-urlencode "model=" --data-urlencode "url=http://assets.mojang.com/SkinTemplates/steve.png" https://api.mojang.com/user/profile/<uuid>/skin
POST /user/profile/<uuid>/skin HTTP/1.1
Host: api.mojang.com
User-Agent: curl/7.49.0
Accept: */*
Authorization: Bearer <access token>
Content-Length: 69
Content-Type: application/x-www-form-urlencoded

model=&url=http%3A%2F%2Fassets.mojang.com%2FSkinTemplates%2Fsteve.png
```

## 上传皮肤

```http
PUT https://api.mojang.com/user/profile/<uuid>/skin
```

这会将皮肤上传到Mojang的服务器。 它还会设置用户的皮肤。 这也适用于传统计数。

### 应答

除非有错误，否则无回应

### 请求头

```http
Authorization: Bearer <access token>
```

### 载荷

该API的有效负载由多部分表单数据组成。具体有两部分（顺序与边界的b/c无关）：

| **model** | **默认模型为空字符串，slim模型为 `slim`** |
| --------- | ----------------------------------------- |
| **file**  | **原始图像文件数据**                      |

### 例子

```http
curl -X PUT -H "Authorization: Bearer <access token>" -F model=alex -F file="@alex.png;type=image/png" https://api.mojang.com/user/profile/<uuid>/skin
PUT /user/profile/<uuid>/skin HTTP/1.1
Host: api.mojang.com
User-Agent: curl/7.49.0
Accept: */*
Authorization: Bearer <access token>
Content-Length: <length>
Content-Type: multipart/form-data; boundary=<boundary>

--<boundary>
Content-Disposition: form-data; name="model"

slim
--<boundary>
Content-Disposition: form-data; name="file"; filename="alex.png"
Content-Type: image/png

<image data>
--<boundary>--
```

## 重置皮肤

```http
DELETE https://api.mojang.com/user/profile/<uuid>/skin
```

将用户的皮肤重置为默认皮肤。

### 应答

除非有错误，否则无回应

### 请求头

```http
Authorization: Bearer <access token>
```

### 例子

```http
curl -X DELETE -H "Authorization: Bearer <access token>" https://api.mojang.com/user/profile/<uuid>/skin
DELETE /user/profile/<uuid>/skin HTTP/1.1
Host: api.mojang.com
User-Agent: curl/7.46.0
Accept: */*
Authorization: Bearer <access token>
```

## 安全问题解答流程

如果服务尚不信任您的IP，则需要此选项才能使皮肤更改端点正常工作。

### 检查是否需要安全问题

```http
GET https://api.mojang.com/user/security/location
Authorization: Bearer <access token>
```

IP被信任:

```http
204 NO CONTENT
```

IP不被信任:

```http
{
    "error": "ForbiddenOperationException",
    "errorMessage": "Current IP is not secured"
}
```

### 获取安全问题列表

```http
GET https://api.mojang.com/user/security/challenges
Authorization: Bearer <access token>
```

应答:

```http
[
    {
        "answer": {
            "id": 123
        },
        "question": {
            "id": 1,
            "question": "What is your favorite pet's name?"
        }
    },
    {
        "answer": {
            "id": 456
        },
        "question": {
            "id": 2,
            "question": "What is your favorite movie?"
        }
    },
    {
        "answer": {
            "id": 789
        },
        "question": {
            "id": 3,
            "question": "What is your favorite author's last name?"
        }
    }
],
```

可能的问题范围:

```markdown
1  What is your favorite pet's name?
2  What is your favorite movie?
3  What is your favorite author's last name?
4  What is your favorite artist's last name?
5  What is your favorite actor's last name?
6  What is your favorite activity?
7  What is your favorite restaurant?
8  What is the name of your favorite cartoon?
9  What is the name of the first school you attended?
10 What is the last name of your favorite teacher?
11 What is your best friend's first name?
12 What is your favorite cousin's name?
13 What was the first name of your first girl/boyfriend?
14 What was the name of your first stuffed animal?
15 What is your mother's middle name?
16 What is your father's middle name?
17 What is your oldest sibling's middle name?
18 In what city did your parents meet?
19 In what hospital were you born?
20 What is your favorite team?
21 How old were you when you got your first computer?
22 How old were you when you got your first gaming console?
23 What was your first video game?
24 What is your favorite card game?
25 What is your favorite board game?
26 What was your first gaming console?
27 What was the first book you ever read?
28 Where did you go on your first holiday?
29 In what city does your grandmother live?
30 In what city does your grandfather live?
31 What is your grandmother's first name?
32 What is your grandfather's first name?
33 What is your least favorite food?
34 What is your favorite ice cream flavor?
35 What is your favorite ice cream flavor?
36 What is your favorite place to visit?
37 What is your dream job?
38 What color was your first pet?
39 What is your lucky number?
```

### 发回答案

```http
POST https://api.mojang.com/user/security/location
Authorization: Bearer <access token>
[
    {
        "id": 123,
        "answer" : "foo"
    },
    {
        "id": 456,
        "answer" : "bar"
    },
    {
        "id": 589,
        "answer" : "baz"
    }
]
```

失败时，您将得到某种错误。 除非是语法或json结构错误，否则将是这样:

```http
{
  "error": "ForbiddenOperationException",
  "errorMessage": "At least one answer was incorrect"
}
```

成功时:

```http
204 NO CONTENT
```

## 被封禁的服务器

```http
  GET https://sessionserver.mojang.com/blockedservers
```

返回SHA1哈希列表，用于在客户端尝试连接时对照服务器地址进行检查。

客户端使用ISO-8859-1字符集对照此列表检查小写名称。 他们还将尝试检查子域，将每个级别替换为 `*`。 具体来说，它根据域中的 `.` 进行拆分，遍历每个节，一次删除一个节。 例如，对于 `mc.example.com` ，它将尝试 `mc.example.com` 、 `*.example.com` 和 `* .com` 。 使用IP地址（通过划分为4个部分进行验证，每个部分都是包括0和255之间的有效整数），替换将从头开始，因此对于 `192.168.0.1` ，它将尝试 `192.168.0.1` 、`192.168.0.*` 、`192.168.*` 和 `192.*` 。

此检查由netty中的bootstrap类完成。 在启动程序加载的com.mojang:netty依赖项中，默认的netty类被1覆盖。 这使其可以影响使用netty (1.7+)的任何版本

### 应答

所有SHA1哈希的以行分隔。

当前约2200个哈希值中的一些已被破解。

```http
6f2520f8bd70a718c568ab5274c56bdbbfc14ef4:*.minetime.com
7ea72de5f8e70a2ac45f1aa17d43f0ca3cddeedd:*.trollingbrandon.club
c005ad34245a8f2105658da2d6d6e8545ef0f0de:*.skygod.us
c645d6c6430db3069abd291ec13afebdb320714b:*.mineaqua.es
8bf58811e6ebca16a01b842ff0c012db1171d7d6:*.eulablows.host
8789800277882d1989d384e7941b6ad3dadab430:*.moredotsmoredots.xyz
e40c3456fb05687b8eeb17213a47b263d566f179:*.brandonlovescock.bid
278b24ffff7f9f46cf71212a4c0948d07fb3bc35:*.brandonlovescock.club
c78697e385bfa58d6bd2a013f543cdfbdc297c4f:*.mineaqua.net
b13009db1e2fbe05465716f67c8d58b9c0503520:*.endercraft.com
3e560742576af9413fca72e70f75d7ddc9416020:*.insanefactions.org
986204c70d368d50ffead9031e86f2b9e70bb6d0:*.playmc.mx
65ca8860fa8141da805106c0389de9d7c17e39bf:*.howdoiblacklistsrv.host
7dca807cc9484b1eed109c003831faf189b6c8bf:*.brandonlovescock.online
c6a2203285fb0a475c1cd6ff72527209cc0ccc6e:*.brandonlovescock.press
e3985eb936d66c9b07aa72c15358f92965b1194e:*.insanenetwork.org
b140bec2347bfbe6dcae44aa876b9ba5fe66505b:*.phoenixnexus.net
27ae74becc8cd701b19f25d347faa71084f69acd:*.arkhamnetwork.org
48f04e89d20b15de115503f22fedfe2cb2d1ab12:brandonisan.unusualperson.com
9f0f30820cebb01f6c81f0fdafefa0142660d688:*.kidslovemy500dollarranks.club
cc90e7b39112a48064f430d3a08bbd78a226d670:*.eccgamers.com
88f155cf583c930ffed0e3e69ebc3a186ea8cbb7:*.fucktheeula.com
605e6296b8dba9f0e4b8e43269fe5d053b5f4f1b:*.mojangendorsesbrazzers.webcam
5d2e23d164a43fbfc4e6093074567f39b504ab51:touchmybody.redirectme.net
f3df314d1f816a8c2185cd7d4bcd73bbcffc4ed8:*.mojangsentamonkeyinto.space
073ca448ef3d311218d7bd32d6307243ce22e7d0:*.diacraft.org
33839f4006d6044a3a6675c593fada6a690bb64d:*.diacraft.de
e2e12f3b7b85eab81c0ee5d2e9e188df583fe281:*.eulablacklist.club
11a2c115510bfa6cb56bbd18a7259a4420498fd5:*.slaughterhousepvp.com
75df09492c6c979e2db41116100093bb791b8433:*.timelesspvp.net
d42339c120bc10a393a0b1d2c6a2e0ed4dbdd61b:*.herowars.org
4a1b3b860ba0b441fa722bbcba97a614f6af9bb8:justgiveinandblockddnsbitches.ddns.net
b8c876f599dcf5162911bba2d543ccbd23d18ae5:brandonisagainst.health-carereform.com
9a9ae8e9d0b6f3bf54c266dcd1e4ec034e13f714:brandonwatchesporn.onthewifi.com
336e718ffbc705e76b4a72884172c6b95216b57c:canyouwildcardipsplease.gotdns.ch
27cf97ecf24c92f1fe5c84c5ff654728c3ee37dd:letsplaysome.servecounterstrike.com
32066aa0c7dc9b097eed5b00c5629ad03f250a2d:mojangbrokeintomy.homesecuritymac.com
39f4bbfd123a5a5ddbf97489877831c15a70d7f2:*.primemc.org
f32f824d41aaed334aef248fbe3a0f8ecf4ac1a0:*.meep.in
c22efe4cf7fb319ca2387bbc930c1fdf77ab72fc:*.itsjerryandharry.com
cc8e1ae93571d144bf4b37369cb8466093d6db5a:*.thearchon.net
9c0806e5ffaccb45121e57e4ce88c7bc76e057f1:*.goatpvp.com
5ca81746337088b7617c851a1376e4f00d921d9e:*.gotpvp.com
a5944b9707fdb2cc95ed4ef188cf5f3151ac0525:*.guildcraft.org
```

## 统计

```http
  POST https://api.mojang.com/orders/statistics
```

获取有关《我的世界》销量的统计信息。

### 载荷

有效载荷是 `metricKeys` 键下的选项的json列表。 您将收到与所请求类型的销售总额相对应的单个对象。 您必须要求至少一种销售类型。 以下是 https://minecraft.net/en/stats/ 使用的默认列表:

```http
{
    "metricKeys": [
        "item_sold_minecraft",
        "prepaid_card_redeemed_minecraft"
    ]
}
```

有效选项包括:

```http
   item_sold_minecraft
   prepaid_card_redeemed_minecraft
   item_sold_cobalt
   item_sold_scrolls
   prepaid_card_redeemed_cobalt
   item_sold_dungeons
```

### 应答

返回一个json对象，其中包含已售出的副本总数，最近24小时内售出的副本数量以及每秒的销售量。

```http
{
    "total": integer total amount sold,
    "last24h": integer total sold in last 24 hours,
    "saleVelocityPerSeconds": decimal average sales per second
}
```

## 例程

[C#](https://github.com/hawezo/MojangSharp) | 完整的API包装器

[Go](https://github.com/Lukaesebrot/mojango) | 完整的API包装器

[Go](https://github.com/PhilipBorgesen/minecraft/tree/master/profile) | UUID或name到具有皮肤，斗篷和名称历史的配置文件

[Python](https://github.com/SynchronousX/mojang-api) | 完整的API包装器

[Python](https://github.com/techkid6/AccountsClientPython) | UUID或name到个人资料

[Python](https://gist.github.com/jomo/74944770e7647855ac9d) | name文件到uuid+name

[PHP](https://github.com/elyby/mojang-api) | 完整的Mojang API包装器

[PHP](https://github.com/MineTheCube/MojangAPI) | UUID或name到具有皮肤，斗篷和名称历史的配置文件

[PHP](https://gist.github.com/ezfe/a71feccd3a837a2592f1) | UUID 到 name

[PHP](https://github.com/ozzyfant/AccountsClientPHP) | UUID 到 name 双向

[Java](https://github.com/SparklingComet/java-mojang-api) | 近乎完整的API包装器

[JavaScript](https://github.com/thechunknetwork/mojang-api) | UUID或name到具有皮肤，斗篷和名称历史的配置文件