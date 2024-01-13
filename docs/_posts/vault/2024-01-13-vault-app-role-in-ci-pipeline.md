---
layout:         post
title:          "Vault 系列：再探 CI 流水线场景下的应用角色使用"
subtitle:       "Explore the AppRole usage in CI pipeline"
date:           2024-01-13 14:22:00 +0800
author:         "fuzhidai"
header-style:   text
tags:
    - Vault
    - AppRole
    - CI Pipeline
---
### 目录

* 内容概述
* 流程分析
  * CI Worker 初始化获取访问令牌 Token
  * CI Worker 请求 Wrapped Secret ID
  * CI Worker 将 Wrapped Secret ID 传递给 CI Job
  * CI Job unwrap 解包 Wrapped Secret ID
    * 是什么 Wrapped Secret ID
  * CI Job 获取访问令牌 Token
  * CI Job 基于访问令牌进行后续的请求操作
* 流程总结
  * CI Worker 与 CI Job 对 Vault 的四次请求
* 相关参考

### 内容概述

在 AppRole 使用模式的文章中，我们对应用角色在 CI 流水线中的使用有了一个全局的了解。但在翻译原文的过程中，发现原文存在许多一带而过的地方，也存在许多值得品味和学习的地方，因此在本篇文章中，将继续对 CI 流水线中的应用角色使用进行学习和分析。

### 流程分析

![approle-ci](/img/post-vault-approle-ci.jpg)

*图片来源: Hashicorp Vault*

#### CI Worker 初始化获取访问令牌 Token

在 CI 流水线的应用角色使用场景下，作为核心的受信代理系统（trusted broker），CI Worker 承载着将 Secret ID 下发至 CI Job 的职责，并且作为整套授权认证系统的核心节点，CI Worker 理论上会经手所有下发到 CI Job 的 Secret ID，因此也是整套系统的中心节点。

在整个使用流程中，CI Worker 的初始化是整个流程的起点，其核心步骤对应着上图中的第 1 步和第 2 步，即向 Vault 服务器进行身份认证，以及从 Vault 获取认证后颁发的令牌。这一环节是整个受信链的第一环，当 Vault 完成对 CI Worker 的身份认证后，标志着 Vault 已经信任了该 CI Worker，将信任链条向下传递。

在 Vault 的官方文档中，官方推荐基于平台级的认证方式完成对 CI Worker 的身份认证，这里的平台级认证方法，就是基于可信的三方平台完成认证。但另一方面，在文章的 “Vault returns a token” 部分，官方也表达了对 Vault 下发令牌到 CI Worker 这个流程的放心，甚至表示管理员可以直接将 CI Worker 需要使用的令牌预先下发到 Worker 中，或者直接干脆硬编码 Role ID 和 Secret ID。这主要是因为在最佳实践中，Vault 向 CI Worker 颁发的令牌，应当仅具有获取 Wrapped Secret ID 的权限（权限示例如下），这使得当仅具有 CI Worker 的令牌时，是无法完成 CI Job 中秘密的解密操作的。

```Shell
path "auth/approle/role/+/secret*" {
  capabilities = [ "create", "read", "update" ]
  min_wrapping_ttl = "100s"
  max_wrapping_ttl = "300s"
}
```

#### CI Worker 请求 Wrapped Secret ID

当 CI Worker 完成初始化后，它已经获取到了具有后续操作权限的令牌，有权向 Vault 服务器请求获取待 CI Job 所需使用的 Wrapped Secret ID 。这里对应着上图中的第 3 步和第 4 步，CI Worker 根据 CI Job 的角色，向 Vault 请求获取其对应的 Wrapped Secret ID。

这里需要注意两个概念，一个是 Wrapped Secret ID，通过参考 Vault 文档可知，Wrapped Secret ID 的含义是为 Secret ID 增加了一个有效期，使得只有在有效期内，才能通过 `vault unwrap` 命令完成 Secret ID 的解包。这样的操作流程符合 Vault 的两个基本原则中的第二个，即尽可能缩短认证的有效时间，保证权限的闭环。

另一个概念是 Role，当 CI Worker 向 Vault 请求获取 Wrapped Secret ID 时，它所仅知的两个信息只有 CI 任务标识（CI Job ID）和应用角色的角色名称（Approle RoleName），此时 CI Worker 不知道任何关于 CI Job 的 Role ID 信息。因此，CI Worker 仅能通过应用角色的角色名称，来向 Vault 请求 Wrapped Secret ID。

这也是为什么 Vault 对 CI Worker 的令牌如此之放心，因为 CI Worker 永远不会获取到 Role ID，仅能获取到 Secret ID 。按照 Vault 自己的叙述来说，这种安全机制的核心原则是，从代理到 Vault 的身份验证期间，Role ID 和 Secret ID 只在需要使用密钥的最终用户系统（end-user system）上，才会一起出现。

具体的请求命令如下所示，`wrap-ttl` 是为本次申请的 secret ID 设置了一个有效期，从而生成一个 Wrapped Secret ID，在当前示例中该有效期为 120 秒，而具体请求路径中的 `my-role` 则是本次请求的角色名称（Role Name），而非 Role ID。

```Shell
vault write -wrap-ttl=120s -f auth/approle/role/my-role/secret-id
```

#### CI Worker 将 Wrapped Secret ID 传递给 CI Job

在第 5 步中，CI Worker 从 Vault 获取到 Wrapped Secret ID 后，会在完成 CI Job 的创建后，将 Wrapped Secret ID 以环境变量的形式下发到 CI Job 中，这样 CI Job 就可以从环境变量中获取该值了。请求 Wrapped Secret ID 及执行下发命令的示例如下所示，以 CI Job 的名称作为角色名称，向 Vault 请求其对应的 Wrapped Secret ID（有效期为 300 秒），然后再将其设置为任务的环境变量 `WRAPPED_SID`。

```shell
environment {
   WRAPPED_SID = """$s{sh(
                    returnStdout: true,
                    Script: ‘curl --header "X-Vault-Token: $VAULT_TOKEN"
       --header "X-Vault-Namespace: ${PROJ_NAME}_namespace"
       --header "X-Vault-Wrap-Ttl: 300s"
         $VAULT_ADDR/v1/auth/approle/role/$JOB_NAME/secret-id’
         | jq -r '.wrap_info.token'
                 )}"""
  }
```

#### CI Job unwrap 解包 Wrapped Secret ID

CI Job 启动后，将会执行 Wrapped Secret ID 的 unwrap 解包操作，即将具有生命周期且被封装的 Wrapped Secret ID 还原为真实有效的 Secret ID，该步骤可以通过 `vault unwrap` 命令或其对应的 API 请求来完成，对应下面流程图中的第 6 步和第 7 步。需要注意的是，每个 Wrapped SID 仅能被执行一次 `unwrap` 操作，以此保证整个流程的安全性。

##### 是什么 Wrapped Secret ID

这里想要着重分析一下 Wrapped Secret ID，不知道你有没有跟我一样的疑惑，在 Vault 的这篇最佳实践中，Wrapped Secret ID 仿佛是一个突然出现的天外来物，全文多次提到了 Wrapped SID（Wrapped Secret ID），但却没有任何一处详细解释了到底 Wrapped SID 是什么。所以从这个角度出发，我想跟你一起探究下到底 Wrapped SID 是什么，以及它到底有什么作用。

首先是 CI Worker 向 Vault 请求生成 Wrapped SID 的流程，此时请求的路径为 `/v1/auth/approle/role/<role_name>/secret-id`，在源码中检索该路径的处理器为 `*backend.pathRoleSecretIDUpdate`，可以看到该方法的整体流程为：基于 UUID 随机生成 Secret ID，然后将其与相关的配置进行持久化存储。而 `wrap` 包装的操作则是由外部的 `request_handling `请求处理器来处理，当请求处理器发现请求参数中包含与 wrapping 相关的参数时，就会自动对响应进行包装操作，并将操作生成的相关数据存储在 `response.wrap_info `中，包括下面将要提到的 `wrappingToken `，其路径为 `response.wrap_info.token`。

对于应用角色认证场景下的 unwrap 解包流程，在 Vault 提供的源码中检索  `approle `这个关键字，会发现 `AppRoleAuth `这个关键模型，在这个模型提供的 `Login `方法中，Vault 会在从环境变量中解析出 `SecretIDVaule `后，将其作为 `wrappingToken `参数去调用 `unwrap `服务，并最终获取到 `unwrappedToken `，而后再从其 `data `字段中解析获取 `secret_id `用于后续的 `Login `登陆操作。

从上述的流程来看，Wrapped SID 本质上就是 Vault 中的一个 `Token` 令牌。外部调用方可以要求 Vault 对本次请求的响应进行 `wrap` 包装操作，并将包装后的相关参数存储在响应的 `wrap_info` 中，其中就包括 Wrapping Token（路径为 `wrap_info.token`）。而后调用方可以基于该 Wrapping Token 调用 unwrap 解包服务，从而获取到上次包装请求中的内容，在应用角色使用场景下，请求中的内容即为 Secret ID。

#### CI Job 获取访问令牌 Token

在完成 Wrapped SID 的解包操作后，CI Job 就获取到了真实有效的 Secret ID，同时因为 Role ID 天然是由 CI Job 进行保管的，所以当前运行已经同时具备了 Secret ID 和 Role ID，可以向 Vault 换取后续请求使用的 Token 令牌，这一流程对应上图中的第 8 步和第 9 步。

执行完这一步骤后，整个授权的可信链路已经建立完成。可以发现，在整个流程中，正如 Vault 所说，Wrapped Secret ID 由 CI Worker 动态下发到 CI Job 中，并在解包后才能获取到真实有效的 Secret ID，而 Role ID 则只有 CI Job 自己可知，这保证了 Secret ID 和 Role ID 仅在需要使用密钥的最终用户系统（当前为 CI Job）上才会一起出现，极大的保证了密钥的安全性。

#### CI Job 基于访问令牌进行后续的请求操作

最后，正如图中第 10 步所示，当 CI Job 获通过 Secret ID 和 Role ID 换取到访问令牌 Token 后，后续该任务的请求操作均会基于该令牌进行操作，其具有的访问权限也由该令牌进行控制。

### 流程总结

#### CI Worker 与 CI Job 对 Vault 的四次请求

在整个受信链路建立的过程中，CI Worker 与 CI Job 总共和 Vault 发生了 4 次交互操作，只有在全部的认证及授权交互完成后，CI Job 才能获取到用于后续请求使用的访问令牌，这 4 次操作可以概括为：

1. **兑换 Worker 令牌**：CI Worker 向 Vault 请求原始访问令牌，用于颁发后续 CI Job 使用的 Wrapped SID；
2. **生成 Wrapped SID**：CI Worker 根据 CI Job 的 Role Name 信息，向 Vault 请求对应的 Wrapped SID；
3. **解包获取 Secret ID**： 使用 CI Worker 下发的 Wrapped SID，向 Vault 发起解包操作，兑换 Secret ID；
4. **兑换 Job 令牌**：CI Job 使用自带的 Role ID 和刚刚获取到的 Secret ID，向 Vault 兑换后续访问使用的令牌 Token；

从这里可以看出，在 Vault 的整个授权认证体系中，外部调用方与 Vault 的交互认证均需基于访问令牌 Token 来进行，哪怕是基于应用角色的认证方式，最终的目的也是为了获取用于执行其他操作的访问令牌，而后再基于该令牌执行后续的操作。

### 相关参考

**【1】**[Vault token create](https://developer.hashicorp.com/vault/docs/commands/token/create)
<br>**【2】**[Vault API unwrap](https://developer.hashicorp.com/vault/api-docs/system/wrapping-unwrap)
<br>**【3】**[Vault AppRole Recommend Pattern](https://developer.hashicorp.com/vault/tutorials/recommended-patterns/pattern-approle#anti-patterns)
