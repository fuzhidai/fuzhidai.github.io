---
layout:         post
title:          "Vault 系列：Cubbyhole 响应包装秘密引擎"
subtitle:       "Vault Cubbyhole response wrapping secret engine"
date:           2024-01-20 22:28:00 +0800
author:         "fuzhidai"
header-style:   text
tags:
    - Vault
    - Cubbyhole
    - 翻译
---
> 本文摘录自  Hashicorp Vault 的官方文档：[Vault cubbyhole response wrapping](https://developer.hashicorp.com/vault/tutorials/secrets-management/cubbyhole-response-wrapping)，大部分为意译，包含个人理解，仅供学习使用。

### 什么是 Cubbyhole

Cubbyhole 单词的中译为「小隔间」，按照 Vault 自身的定义，术语 Cubbyhole 源自一种美式定义，可以将其看作为是一个储物柜或一个安全屋来存放贵重的物品。在 Vault 中，Cubbyhole 是用户的带锁储物柜，用户可以将全部的秘密都存储到该储物柜中，而后使用一个关联的令牌来进行解锁获取。同时，但当该令牌过期或被撤销时，也意味着储物柜中的秘密也一并被销毁了。

在访问权限管控方面，Cubbyhole 与 Kv Secret 存储非常不同。对于键值秘密引擎（kv secret engine）来说，只要当前用户的令牌策略允许，任何用户都可以访问它有权限的键值秘密引擎中存储的秘密。但对于 Cubbyhole 来说，即使作为根用户，也无权访问另一个他不持有特定令牌的 Cubbyhole。

### Cubbyhole 要解决的问题是什么

在 Vault 中，为了对秘密进行严格的管理，用户可以使用策略来控制谁有权限访问什么样的资源，并通过将策略附加到令牌及角色上的方式，实现对资源访问权限的控制。

但存在这样的一些情况，当用户存在一个受信实体，该实体需要从 Vault 中读取秘密，此时该受信实体必须要获取一个有权限的令牌，而如果该受信实体或其主机发生了重启，那该主体将必须使用有效的令牌与 Vault 重新进行身份认证，此时如何才能安全地将初始令牌分发给该受信实体，正是 Cubbyhole 要解决的一个问题。

### Cubbyhole 的解决方案是什么

针对上述问题，Vault 提供的解决方案就是基于 Cubbyhole 来实现初始令牌的安全下发。Vault 通过包装响应值的方式，将初始令牌存储在 Cubbyhole 秘密引擎中，使用方可以通过使用一次性令牌，实现对包装秘密的解包。即使是创建初始令牌的用户或系统，也无权查看令牌的原始值。一方面，包装令牌的生命周期是有限的，另一方面，它也可以像其它令牌一样被撤销，因此其被未授权非法访问的安全风险会被降至最低。

#### 什么是 Cubbyhole 响应包装（Cubbyhole Response Wrapping）

当请求响应被包装时，Vault 会创建一个「临时」且「仅供单次使用」的令牌（Wapping Token），并将该令牌和其有效期一并存储到响应中。只有持有该临时令牌的客户端，才有权解包（unwrap）获取原始的响应数据。理论上，任何 Vault 请求响应都可以使用这种方式来进行包装，以此实现请求响应的安全下发。

#### Cubbyhole 响应包装的好处

Cubbyhole 响应包装（Cubbyhole Response Wrapping）通过引用秘密原始值，而非传输秘密原始值，来实现对原始秘密的保护。通过临时令牌的仅一次解包有效机制，保证只有一方可以解包令牌，且仅可解包一次，从而实现对非预期非法行为的提前检测。同时，响应包装令牌的有效时间，与秘密自身的有效时间是隔离的，因此可以通过尽可能缩短包装令牌的有效时间，来降低秘密的曝光时间。

### 方案介绍

#### 场景介绍

如下图所示，应用程序需要从 Vault 的键值秘密引擎中获取秘密数据，但该应用程序当前没有一个有效的令牌可以用来读取该秘密数据，因此它需要系统管理员将有权限的令牌安全地下发给它。系统管理员使用 Cubbyhole 响应包装机制来包装秘密值，并将包装后的令牌下发给应用程序，此时应用程序就可以使用该封装令牌，在令牌到期之前完成秘密的获取了。

![cubbyhole-example](/img/post-vault-cubbyhole-example.jpg)

*图片来源: Hashicorp Vault*

#### 操作方式

##### 步骤一：创建秘密并获取包装令牌

当对一个秘密（Secret）进行包装操作时，Vault 会将该秘密放置到一个具有一次性包装令牌的 Cubbyhole 中，然后返回该 Cubbyhole 的一次性令牌。当后续获取该秘密时，仅需对这个包装的一次性令牌进行解包即可。

**1）写入测试数据**

首先，向 `secret/dev` 中写入一些测试数据。

```shell
vault kv put secret/dev username="webapp" password="my-long-password"
```

写入后的输出如下：

```shell
= Secret Path =
secret/data/dev

======= Metadata =======
Key                Value
---                -----
created_time       2022-03-30T18:50:59.153742Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            1
```

**2）获取包装响应**

读取 `secret/dev` 中的秘密，使用 `-wrap-ttl` 标志来包装响应输出，并指定包装令牌的有效期为120 秒。

```shell
vault kv get -wrap-ttl=120 secret/dev
```

读取后的输出如下，此时响应中包含的是包装令牌（Wrapping Token），而非在 `secret/dev` 中存储原始秘密。

```shell
Key                              Value
---                              -----
wrapping_token:                  hvs.CAESIGu9Ulc0Uhmqa8hmXx7DyvnrFvOGmlecFcl8kkGz-tLoGh4KHGh2cy5uYzZsZEMxa3VKdE5UakpiR25LZXpjczA
wrapping_accessor:               3dyFk8GHlmLNvKEjxcL9TDz2
wrapping_token_ttl:              2m
wrapping_token_creation_time:    2022-04-05 21:09:08.2289 -0700 PDT
wrapping_token_creation_path:    secret/data/dev
```

**3）设置一次性包装令牌**

将上一步骤请求中返回的包装令牌值，存储到应用所在容器的 `WRAPPING_TOKEN` 环境变量中。

```shell
export WRAPPING_TOKEN="hvs.CAESIGu9Ulc0Uhmqa8hmXx7DyvnrFvOGmlecFcl8kkGz-tLoGh4KHGh2cy5uYzZsZEMxa3VKdE5UakpiR25LZXpjczA"
```

##### 步骤二：解包秘密

在完成了上一步的操作后，应用就可以通过环境变量从管理员那里接受到一个一次性的包装令牌，此时为了让应用能从 `secret/dev` 中读取其所需的秘密，它必须对这个令牌执行 `unwrap` 操作。

需要注意的是，如果应用一直没有收到其期望获取的响应包装令牌，可能是其它攻击者拦截了该令牌，并对该令牌提前执行了解包操作，然后阻断了该令牌的继续下发传播，因此，此时应该立即进行排查。

**1）解包获取秘密**

参考下面的命令，通过传递包装令牌并执行 `vault unwrap` 来解包获取秘密。此时，应用不必持有客户端令牌，而仅需使用有效的包装令牌即可。

```shell
VAULT_TOKEN=$WRAPPING_TOKEN vault unwrap
```

执行解包操作后的输出如下，可以看到，这里已经获取到了原始的秘密数据。但需要注意的是，此时的秘密数据仍然是存储在 `secret/dev` 中的。

```shell
Key         Value
---         -----
data        map[password:my-long-password username:webapp]
metadata    map[created_time:2022-04-06T04:24:24.391513Z custom_metadata:<nil> deletion_time: destroyed:false version:1]
```

而如果我们再次尝试对该令牌执行解包操作，将会返回一个错误，提示 “包装令牌无效或不存在”，这是因为包装令牌仅限于一次性使用。

```shell
VAULT_TOKEN=$WRAPPING_TOKEN vault unwrap
```

### 引用参考

**【1】**[Vault cubbyhole response wrapping](https://developer.hashicorp.com/vault/tutorials/secrets-management/cubbyhole-response-wrapping)
`<br>`**【2】**[Cubbyhole secrets engine](https://developer.hashicorp.com/vault/docs/secrets/cubbyhole)
`<br>`**【3】**[Response wrapping](https://developer.hashicorp.com/vault/docs/concepts/response-wrapping)
