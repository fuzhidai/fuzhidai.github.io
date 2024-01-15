---
layout:         post
title:          "Vault 系列：聊聊神秘的 Cubbyhole"
subtitle:       "Talk about the mysterious Cubbyhole in the Vault"
date:           2024-01-15 20:55:00 +0800
author:         "fuzhidai"
header-style:   text
tags:
    - Vault
    - Cubbyhole
---
> 本文部分摘录自  Hashicorp Vault 的官方文档：[Vault cubbyhole response wrapping](https://developer.hashicorp.com/vault/tutorials/secrets-management/cubbyhole-response-wrapping)，仅供学习使用。

### 什么是 Cubbyhole

Cubbyhole 的中译为小隔间，按照 Vault 自身的定义来说，术语 Cubbyhole 源自一种美式定义，可以将其作为一个储物柜或一个安全屋来存放贵重的物品。在 Vault 中，Cubbyhole 是用户的带锁储物柜，用户可以将全部的秘密都存储在该储物柜中，而其解锁方式就是使用一个关联的令牌。但当该令牌过期或被撤销后，也意味着储物柜中的秘密也一并被撤销了。

与键值秘密引擎不同，在 Vault 的规则中，即使作为根用户，也无权访问另一个 Cubbyhole，而对于键值秘密引擎来说，只要策略允许，任何令牌都可以访问它有权限的键值秘密引擎中存储的秘密。

### Cubbyhole 要解决的问题是什么

在 Vault 中，为了对秘密进行严格的管理，用户可以使用策略来控制谁有权限访问什么样的资源，并通过将策略附加到令牌及角色上的方式，实现对资源访问权限的控制。

但存在这样的一些情况，当用户存在一个受信实体，该实体需要从 Vault 中读取秘密，此时该受信实体必须要获取一个有权限的令牌，而如果该受信实体或其主机发生了重启，那该主体必须使用有效的令牌与 Vault 重新进行身份认证，此时如何才能安全地将初始令牌分发给该受信实体，这就是 Cubbyhole 要解决的一个问题。

### Cubbyhole 的解决方案是什么

针对上述问题，Vault 提供的解决方案就是基于 Cubbyhole 来实现初始令牌的安全下发。Vault 通过包装响应值的方式，将初始令牌存储在 Cubbyhole 秘密引擎中，使用方可以通过使用一次性令牌，实现对包装秘密的解包。而即使是创建初始令牌的用户或系统，也无权查看令牌的原始值。一方面，包装令牌的生命周期是有限的，另一方面，它也可以像其它令牌一样被撤销，因此其被未授权非法访问的安全风险会被降至最低。

#### 什么是 Cubbyhole 响应包装（Cubbyhole Response Wrapping）

当请求响应被包装时，Vault 会创建一个临时且仅供单次使用的令牌（Wapping Token），并将该令牌和其有效期一并存储到响应中。只有持有该临时令牌的客户端，才有权解包（unwrap）获取原始的响应数据。理论上，任何 Vault 请求响应都可以使用这种方式来进行包装。

#### Cubbyhole 响应包装的好处

响应包装通过引用秘密原始值，而非传输秘密原始值，来实现对原始秘密的保护。通过临时令牌的仅一次解包有效机制，保证只有一方可以解包令牌，且仅可解包一次，从而实现对非预期非法行为的提前检测。响应包装令牌的有效时间，与秘密自身的有效时间是隔离的，因此可以通过尽可能缩短包装令牌的有效时间，来降低秘密的曝光时间。

### 方案介绍

#### 场景介绍

如下图所示，应用程序需要从 Vault 的键值秘密引擎中获取秘密数据，但该应用程序当前没有一个有效的令牌可以用来读取该秘密数据，因此它需要系统管理员将有权限的令牌安全地下发给它。系统管理员使用 Cubbyhole 响应包装机制来包装秘密值，并将包装后的令牌下发给应用程序，此时应用程序就可以使用该封装令牌，在令牌到期之前完成秘密的获取了。

![cubbyhole-example](/img/post-vault-cubbyhole-example.jpg)

*图片来源: Hashicorp Vault*


### 引用参考

**【1】**[Vault cubbyhole response wrapping](https://developer.hashicorp.com/vault/tutorials/secrets-management/cubbyhole-response-wrapping)
