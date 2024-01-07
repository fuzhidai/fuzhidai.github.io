---
layout:         post
title:          "Vault 系列：推荐的 AppRole 使用模式"
subtitle:       " \"Recommended pattern for Vault AppRole use\""
date:           2024-01-07 10:45:00 +0800
author:         "fuzhidai"
header-style:   text
tags:
    - Vault
    - 翻译
---
> 本文翻译自  Hashicorp Vault 的官方文档：[Recommended pattern for Vault AppRole use](https://developer.hashicorp.com/vault/tutorials/recommended-patterns/pattern-approle#anti-patterns)，仅供学习使用。

## Catelog

* Objective（目标）
* Vault best practice（Vault 最佳实践）
  * Blast-radius of an identity（身份的爆炸半径）
  * Duration of authentication（认证的持续时间）
* Reference material（参考资料）
* Prerequisites（预先准备）
* Glossary（术语表）
* Vault AppRole overview（Vault 应用角色概述）
  * AppRole in a CI pipeline with wrapped secretIDs（在 CI 流水线中使用应用角色）
* Anti-patterns（反模式）
  * CI worker retrieves secrets
  * CI worker passes RoleID and SecretID to the runner
  * CI worker passes a Vault token to the runner
* Security considerations（安全考量）
* Vault AppRole references（Vault 应用角色的相关参考）

## Objective（目标）

> [HashiCorp Vault](https://www.hashicorp.com/products/vault/) is an identity-based secrets and encryption management system. It provides encryption services gated by authentication and authorization methods to ensure secure, auditable, and restricted access to secrets. It is used to secure, store, and tightly control access to tokens, passwords, certificates, encryption keys for protecting secrets, and other sensitive data using a UI, CLI, or HTTP API.

HashiCorp Vault 是一个基于身份构建的秘密（Secrets）及加密管理系统，它通过身份的授权（authorization）和认证（authentication）来提供加密服务，并对机密提供访问权限控制，以此保证机密的安全性和操作的可审计性。Vault 通常用于保护、存储和严格控制敏感数据的访问权限，这里所说的敏感数据即包括令牌、密码、证书和加密密钥，也包括其它通过 UI、CLI 或 HTTP API 进行使用和传输的敏感数据。

> At the core of Vault's usage is authentication and authorization. Understanding the methods that Vault surfaces these to the client is the key to understanding how to configure and manage Vault.
>
> Vault provides authentication to a client by the use of [auth methods](https://developer.hashicorp.com/vault/docs/concepts/auth).
>
> Vault provides authorization to a client by the use of [policies](https://developer.hashicorp.com/vault/docs/concepts/policies).
>
> Vault provides several internal and external authentication methods. External methods are called *trusted third-party authenticators* such as AWS, LDAP, GitHub, etc. A trusted third-party authenticator is not available in some situations, so Vault has an alternate approach -  **AppRole** . (If another platform method of authentication is available via a trusted third-party authenticator, it is best practice to use that instead of AppRole.)

Vault 的核心用途是身份的授权（authorization）和认证（authentication），因此理解 Vault 向客户端提供这些身份授权和身份验证的方式，是理解如何配置和管理 Vault 的关键。

Vault 通过使用 [auth 方法](https://developer.hashicorp.com/vault/docs/concepts/auth) 向客户端提供身份验证。

Vault 通过使用 [策略](https://developer.hashicorp.com/vault/docs/concepts/policies) 向客户端提供授权。

Vault 提供了几种内部和外部的身份验证方法。外部的身份验证方法被称为 “可信三方认证器（trusted third-party authenticators）”，比如 AWS、LDAP、GitHub 等，它们都是有效的可信三方认证器。但在某些情况下，可能无法使用它们来完成身份的验证操作，因此 Vault 提供了另外一种身份验证方法 **AppRole**（需要注意的是，如果可以通过某个可信三方认证器来完成身份的验证，那最佳的实践是使用该认证器来进行身份验证，而不是使用 AppRole）。

> This guide will detail the high-level concepts of AppRole and outline two detailed uses following the recommended patterns explored in the high-level concepts. This guide will also detail anti-patterns to help readers avoid insecure use of this feature.
>
> If you are unfamiliar with AppRole, refer to the [AppRole Pull Authentication](https://developer.hashicorp.com/vault/tutorials/auth-methods/approle) tutorial for step-by-step instructions. Also, refer to [AppRole With Terraform &amp; Chef](https://developer.hashicorp.com/vault/tutorials/auth-methods/approle-trusted-entities) for more example.

本指南将详细介绍 AppRole 的高级概念，并遵循高级概念的推荐模式来概述两个具体的使用方式。还将详细介绍相应的反模式，避免读者通过不安全的方式来使用该特性。

如果你不熟悉 AppRole，请参阅 [AppRole 拉取认证](https://developer.hashicorp.com/vault/tutorials/auth-methods/approle) 教程来获取分步的操作说明。另外，可以参阅 [AppRole 与 Terraform &amp; Chef](https://developer.hashicorp.com/vault/tutorials/auth-methods/approle-trusted-entities) 来获取更多的用例。

## Vault best practice（Vault 最佳实践）

> This guide relies heavily on two fundamental principles for Vault: limiting both the blast-radius of an identity and the duration of authentication.

本指南在很大程度上依赖于 Vault 的两个基本原则：限制身份的爆炸半径和认证的持续时间。

### Blast-radius of an identity（身份的爆炸半径）

> Vault is an identity-based secrets management solution, where access to a secret is based on the known and verified identity of a client. It is crucial that authenticating identities to Vault are identifiable and only have access to the secrets they are the users of. Secrets should never be proxied between Vault and the secret end-user and a client should never have access to secrets they are not the end-user of.

Vault 是一个基于身份的秘密管理解决方案，在该方案中，对秘密的访问需要基于客户端已验证的身份才能够完成。因此，通过 Vault 进行身份验证的身份必须是可识别的，并且验证完成后，该身份只能访问其关联用户的秘密。同时，永远不应该在 Vault 和秘密终端用户（end-user）之间代理秘密，客户端永远不应该有权限访问其它终端用户的秘密。

### Duration of authentication（认证的持续时间）

> When Vault verifies an entity's identity, Vault then provides that entity with a [token](https://developer.hashicorp.com/vault/docs/concepts/tokens). The client uses this token for all subsequent interactions with Vault to prove authentication, so this token should be both handled securely and have a limited lifetime. A token should only live for as long as access to the secrets it authorizes access to are needed.

当 Vault 验证实体的身份时，如果身份验证通过，Vault 会为该实体颁发一个[令牌](https://developer.hashicorp.com/vault/docs/concepts/tokens)。因为客户端与 Vault 的所有后续交互，都需要使用该令牌来进行身份的验证，因此该令牌应该得到安全的处理，保证其仅具有有限的生命周期，只要能够满足其进行秘密访问操作的操作周期即可（或理解为令牌应该只在需要访问它授权访问的秘密时有效）。

## Reference material（参考资料）

> * [Reference Architecture](https://developer.hashicorp.com/vault/tutorials/day-one-consul/reference-architecture) covers the recommended production Vault cluster architecture
> * [Deployment Guide](https://developer.hashicorp.com/vault/tutorials/day-one-consul/deployment-guide) covers how to install and configure Vault for production use
> * [Auth Methods](https://developer.hashicorp.com/vault/docs/auth) are used to authenticate users and machines with Vault
> * [AppRole](https://developer.hashicorp.com/vault/docs/auth/approle) is the auth method we will be discussing
> * [Response-wrapped tokens](https://developer.hashicorp.com/vault/docs/concepts/response-wrapping)

* [参考架构](https://developer.hashicorp.com/vault/tutorials/day-one-consul/reference-architecture) 包含推荐的 Vault 生产集群架构；
* [部署指南](https://developer.hashicorp.com/vault/tutorials/day-one-consul/deployment-guide) 介绍了如何安装和配置生产环境使用的 Vault；
* [认证方法](https://developer.hashicorp.com/vault/docs/auth) 用于通过 Vault 对用户和机器进行身份验证；
* [AppRole](https://developer.hashicorp.com/vault/docs/auth/approle) 是我们将要讨论的认证方法；
* [响应包装的令牌](https://developer.hashicorp.com/vault/docs/concepts/response-wrapping)；

## Prerequisites（预先准备）

> * You have followed the [Reference Architecture](https://developer.hashicorp.com/vault/tutorials/day-one-consul/reference-architecture) for Vault to provision the necessary resources for a highly-available Vault cluster.
> * You have followed the [Deployment Guide](https://developer.hashicorp.com/vault/tutorials/day-one-consul/deployment-guide) for Vault to install and configure Vault on each Vault server.
> * You have followed the [Production Hardening Guide](https://developer.hashicorp.com/vault/tutorials/operations/production-hardening) for Vault to improve the Vault cluster's security.
> * Vault is unsealed. See the documentation on [unsealing](https://developer.hashicorp.com/vault/docs/commands/operator/unseal).
>
> Here we refer to response-wrapped tokens introduced in version 0.8 of Vault; therefore, it is assumed that you're applying the following recommendations to Vault v0.8 or later.

- 你已经按照 Vault 的 [参考体系结构](https://developer.hashicorp.com/vault/tutorials/day-one-consul/reference-architecture)，为高可用的 Vault 集群配置了必要的资源；
- 请按照 Vault 的 [部署指南](https://developer.hashicorp.com/vault/tutorials/day-one-consul/deployment-guide) 在每个 Vault 服务器上安装和配置 Vault；
- 已按照 Vault 的 [生产加固指南](https://developer.hashicorp.com/vault/tutorials/operations/production-hardening) 提高了 Vault 集群的安全性；
- Vault 已启封，可参考 [unsealed](https://developer.hashicorp.com/vault/docs/commands/operator/unseal) 文档进行操作；

这里我们使用 Vault 0.8 版本中引入的响应包装令牌，因此，你应该将以下建议应用于 Vault v0.8 或更高版本。

## Glossary（术语表）

> * **Authentication** - The process of confirming identity. Often abbreviated to *AuthN*
> * **Authorization** - The process of verifying what an entity has access to and at what level. Often abbreviated to *AuthZ*
> * **RoleID** - The semi-secret identifier for the role that will authenticate to Vault. Think of this as the 'username' portion of an authentication pair.
> * **SecretID** - The secret identifier for the role that will authenticate to Vault. Think of this as the 'password' portion of an authentication pair.
> * **AppRole role** - The role configured in Vault that contains the authorization and usage parameters for the authentication.

* **身份验证** - 确认身份的过程，通常缩写为 *AuthN；*
* **身份授权** - 验证（verifying）实体有权访问什么级别内容的过程，通常缩写为 *AuthZ；*
* **角色标识** - 向 Vault 进行身份验证时使用的角色标识（半秘密标识符），可以将其视为身份验证流程中的 “用户名”；
* **秘密标识** - 向 Vault 进行身份验证时使用的角色密钥（秘密标识符），可以将其视为身份验证流程中的 “密码”；
* **AppRole 角色** - 在 Vault 中配置的角色，包含认证的授权参数和使用参数；

## Vault AppRole overview（Vault 应用角色概述）

> The AppRole authentication method is for machine authentication to Vault. Because AppRole is designed to be flexible, it has many ways to be configured. The burden of security is on the configurator rather than a trusted third party, as is the case in other Vault auth methods.
>
> AppRole is not a trusted third-party authenticator, but a *trusted broker* method. The difference is that in AppRole authentication, the onus of trust rests in a securely-managed broker system that brokers authentication between clients and Vault.
>
> The central tenet of this security is that during the brokering of the authentication to Vault, the RoleID and SecretID are only ever together on the end-user system that needs to consume the secret.

AppRole 身份验证方法是 Vault 用于对机器进行身份验证的方法，它的设计十分灵活，存在很多种配置的方法。与 Vault 的其他验证方法相同，身份验证的安全性责任由配置器承担，而不是由受信任的三方平台承担。

AppRole 不是一个受信任的第三方身份验证器，而是一个受信任的代理系统（trusted broker）。这两者之间的不同之处在于，在 AppRole 的身份验证中，信任（trust）的责任在于一个安全管理的代理系统，该系统在客户端和 Vault 之间代理身份验证。

这种安全机制的核心原则是，从代理到 Vault 的身份验证期间，RoleID 和 SecretID 只在需要使用密钥的最终用户系统（end-user system）上，才会一起出现。

> In an AppRole authentication, there are three players:
>
> * **Vault** - The Vault service
> * **The broker** - This is the trusted and secured system that brokers the authentication.
> * **The secret consumer** - This is the final consumer of the secret from Vault.

在 AppRole 的身份验证方法中，存在着三种参与者：

* **Vault** - Vault 服务；
* **代理** - 代理身份验证的可信且安全的系统；
* **秘密消费者** - Vault 秘密（secret）的最终消费者；

### AppRole in a CI pipeline with wrapped secretIDs

> In this scenario, the CI needs to run a job requiring some data classified as secret and stored in Vault. The CI has a master and a worker node (such as Jenkins). The worker node runs jobs on spawned container runners that are short-lived. The process here should be:
>
> 1. CI Worker authenticates to Vault
> 2. Vault returns a token
> 3. Worker uses token to retrieve a wrapped secretID for the **role** of the job it will spawn
> 4. Wrapped secretID returned by Vault
> 5. Worker spawns job runner and passes wrapped secretID as a variable to the job
> 6. Runner container requests unwrap of secretID
> 7. Vault returns SecretID
> 8. Runner uses RoleID and SecretID to authenticate to Vault
> 9. Vault returns a token with policies that allow read of the required secrets
> 10. Runner uses the token to get secrets from Vault
>
> Here are more details on the more complicated steps of that process.

在当前场景中，CI 需要运行一个作业，该作业需要将一些数据分类为机密并存储在 Vault 中。CI 有一个主节点和一个工作节点（比如 Jenkins），工作节点会在其生成的短期容器运行器上运行作业，这里的具体流程应该是：

1. CI Worker 向 Vault 请求进行身份认证；
2. Vault 返回一个令牌；
3. CI Worker 使用令牌为它将生成的作业，根据其角色，获取一个包装好的 SecretID；
4. 由 Vault 返回包装好的 SecretID；
5. CI Worker 生成任务运行器（Runner），并将封装的 SecretID 作为变量传递给任务；
6. 运行器容器（Runner）请求解封 SecretID；
7. Vault 返回解封后的 SecretID；
8. 运行器（Runner）使用角色标识（RoleID）和秘密标识（SecretID）向 Vault 进行认证；
9. Vault 返回一个令牌，该令牌具有读取所需秘密的权限策略；
10. 运行器（Runner）用这个令牌从 Vault 获取秘密（Secret）；

下面是该过程中更复杂步骤的更多细节。

![approle-ci](/img/post-vault-approle-ci.jpg)
*图片来源: Hashicorp Vault*

#### CI worker authenticates to Vault

> The CI worker will need to authenticate to Vault to retrieve wrapped SecretIDs for the AppRoles of the jobs it will spawn.
>
> If the worker can use a platform method of authentication, then the worker should use that. Otherwise, the only option is to pre-authenticate the worker to Vault in some other way.

CI Worker 需要向 Vault 进行身份验证，以便为它将生成的作业，根据其 AppRole 获取包装好的 SecretID。

如果 Worker 可以使用平台提供的身份验证方法，那 Worker 应该使用它进行身份认证，否则，唯一的选择就是以其他的方式让 Worker 向 Vault 完成身份的预认证。

#### Vault returns a token

> The worker's Vault token should be of limited scope and should only retrieve wrapped SecretIDs. Because of this the worker could be pre-seeded with a long-lived Vault token or use a hard-coded RoleID and SecretID as this would present only a minor risk.
>
> The policy the worker should have would be:

CI Worker 的 Vault 令牌的作用域应该是有限的，并且应该只能获取包装好的 SecretID。因此，可以使用长期存在的 Vault 令牌预先下发到 Worker，或者使用硬编码的 RoleID 和 SecretID，在这种情况下，风险是可控的。

Worker 的策略应该如下所示：

```shell
path "auth/approle/role/+/secret*" {
  capabilities = [ "create", "read", "update" ]
  min_wrapping_ttl = "100s"
  max_wrapping_ttl = "300s"
}
```

#### Worker uses token to retrieve a wrapped SecretID

> The CI worker now needs to be able to retrieve a wrapped SecretID. This command would be something like:
>
> Notice that the worker only needs to know the **role** for the job it is spawning. In the example above, that is `my-role` but not the RoleID.

CI Worker 现在需要能够获取包装好的 SecretID，这个命令类似于:

```shell
vault write -wrap-ttl=120s -f auth/approle/role/my-role/secret-id
```

注意，Worker 只需要知道它正在生成的作业的角色即可，在上面的例子中，需要的角色是 “my-role”，而不是 RoleID。

#### Worker spawns job runner and passes wrapped SecretID

> This could be achieved by passing the wrapped token as an environment variable. Below is an example of how to do this in Jenkins:

Worker 可以通过将包装好的令牌，作为环境变量来传递给待运行的任务。下面是一个如何在 Jenkins 中做到这一点的例子：

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

#### Runner uses RoleID and SecretID to authenticate to Vault

> The runner would authenticate to Vault and it would only receive the policy to read the exact secrets it needed. It could not get anything else. An example policy would be:

Worker 将向 Vault 请求进行身份验证，并且只获得权限策略来读取所需的指定秘密，不能得到除此之外的其他权限，一个策略示例是:

```shell
path "kv/my-role_secrets/*" {
  capabilities = [ "read" ]
}
```

#### Implementation specifics

> As additional security measures, create the required role for the App bearing in mind the following:
>
> * [`secret_id_bound_cidrs` (array: [])](https://developer.hashicorp.com/vault/api-docs/auth/approle#secret_id_bound_cidrs) - Comma-separated string or list of CIDR blocks; if set, specifies blocks of IP addresses which can perform the login operation.
> * [`secret_id_num_uses` (integer: 0)](https://developer.hashicorp.com/vault/api-docs/auth/approle#secret_id_num_uses) - Number of times any particular SecretID can be used to fetch a token from this [AppRole](https://developer.hashicorp.com/vault/tutorials/recommended-patterns/pattern-approle#vault-approle-overview), after which the SecretID will expire. A value of zero will allow unlimited uses.
>
> For best security, set `secret_id_num_uses` to `1` use. Also, consider changing `secret_id_bound_cidrs` to restrict the source IP range of the connecting devices.

作为额外的安全措施，请为应用程序创建所需的角色，并牢记以下内容:

* [`secret_id_bound_cidrs(array: [])`](https://developer.hashicorp.com/vault/api-docs/auth/approle#secret_id_bound_cidrs) - 逗号分隔的字符串或 CIDR 块列表，如果设置，则指定可执行登录操作的 IP 地址块；
* [`secret_id_num_uses(integer: 0)`](https://developer.hashicorp.com/vault/api-docs/auth/approle#secret_id_num_uses) - 使用任何特定的 SecretID 可以从这个 [AppRole](https://developer.hashicorp.com/vault/tutorials/recommended-patterns/pattern-approle#vault-approle-overview) 获取令牌的次数，超过这个次数后，SecretID 将过期，值为 0 时将允许无限使用；

为了获得最佳的安全性，请将 `secret_id_num_uses` 设置为 `1`  次。另外，考虑通过修改 `secret_id_bound_cidrs` 来限制可连接设备的源 IP 范围。

## Anti-patterns（反模式）

> Consider avoiding these anti-patterns when using Vault's AppRole auth method.

在使用 Vault 的 AppRole 认证方案时，请避免使用这些反模式。

### CI worker retrieves secrets

> The CI worker could just authenticate to Vault and retrieve the secrets for the job and pass these to the runner, but this would break the first of the two best practices listed above.
>
> The CI worker may likely have to run many different types of jobs, many of which require secrets. If you use this method, the worker would have to have the authorization (policy) to retrieve many secrets, none of which is the consumer. Additionally, if a single secret were to become compromised, then there would be no way to tie an identity to it and initiate break-glass procedures on that identity. So all secrets would have to be considered compromised.

CI Worker 可以向 Vault 进行身份验证，然后直接获取作业的秘密（secrets），并将其传递给 Runner，但这会破坏上面列出的两个最佳实践中的第一个，即控制身份的爆炸范围。

CI Worker 可能必须同时运行多个不同类型的作业，并且其中许多作业都需要获取秘密（secrets）。此时如果使用上面的方案，CI Worker 必须同时具有获取多个秘密的权限（策略），而这些秘密的使用者都不是 CI Worker。此外，如果一个秘密被泄露，那么就没有办法将它与某个身份进行关联，也无法启动该身份的破窗（break-glass）程序，所以只能认为，此时所有的秘密都可能已经被泄露了。

### CI worker passes RoleID and SecretID to the runner

> The worker could be authorized to Vault to retrieve the RoleId and SecretID and pass both to the runner to use. While this prevents the worker from having Vault's authorization to retrieve all secrets, it has that capability as it has both RoleID and SecretID. This is against best practice.

CI Worker 可以被授权向 Vault 来获取 RoleId 和 SecretID，并将它们传递给 Runner 进行使用。虽然这阻止了 Worker 获得 Vault 的授权来检索所有的秘密，但它仍存在具有这种能力的可能性，因为它同时具有了 RoleID 和 SecretID，这违背了最佳实践。

### CI worker passes a Vault token to the runner

> The worker could be authorized to Vault to generate child tokens that have the authorization to retrieve secrets for the pipeline.
>
> Again, this avoids authorization to Vault to retrieve secrets for the worker, but the worker will have access to the child tokens that would have authorization and so it is against best practices.

CI Worker 可以被授权使用 Vault 来生成子令牌，这些子令牌具有检索流水线秘密的权限。

同样，这虽然避免了对 Worker 进行授权，使其能从 Vault 检索秘密，但 Worker 仍有权限访问具有检索秘密权限的子令牌，因此这违反了最佳实践。

## Security considerations（安全考量）

> In any trusted broker situation, the broker (in this case, the Jenkins worker) must be secured and treated as a critical system. This means that users should have minimal access to it and the access should be closely monitored and audited.
>
> Also, as the Vault audit logs provide time-stamped events, monitor the whole process with alerts on two events:
>
> * When a wrapped SecretID is requested for an AppRole, and no Jenkins job is running
> * When the Jenkins slave attempts to unwrap the token and Vault refuses as the token has already been used
>
> In both cases, this shows that the trusted-broker workflow has likely been compromised and the event should investigated.

对于任何存在 “可信代理的情况下，可信代理（在本例中为 Jenkins worker）都必须得到保护，并被视为关键系统。这意味着用户应该尽量减少对它的访问，并且应该密切监视和审计这些访问。

此外，由于 Vault 审计日志提供带时间戳的事件，因此可以通过以下两个事件发出的警报来监视整个过程：

* 当为一个 AppRole 请求一个包装好的 SecretID，但并没有 Jenkins 作业正在运行时；
* 当 Jenkins slave 试图打开（unwrap）已经被使用过的令牌而被 Vault 拒绝时；

在这两种情况下，都表明可信代理的工作流可能已经被破坏，应该调查这些事件。

## Vault AppRole references（Vault 应用角色的相关参考）

* [How (and Why) to Use AppRole Correctly in HashiCorp Vault](https://www.hashicorp.com/blog/how-and-why-to-use-approle-correctly-in-hashicorp-vault)
* Vault auth methods

  * [CLI Enable/Disable](https://developer.hashicorp.com/vault/docs/auth)
  * [API](https://developer.hashicorp.com/vault/api-docs/auth/approle)
* Vault AppRole authentication

  * [Pull authentication](https://developer.hashicorp.com/vault/tutorials/auth-methods/approle)
  * [Auth method](https://developer.hashicorp.com/vault/docs/auth/approle)
* Vault cubbyhole response wrapping

  * [Response wrapping concept](https://developer.hashicorp.com/vault/docs/concepts/response-wrapping)
  * [Learn tutorials](https://developer.hashicorp.com/vault/tutorials/secrets-management/cubbyhole-response-wrapping)
* Vault policies

  * [Policy overview](https://developer.hashicorp.com/vault/docs/concepts/policies)
  * [ACL policies](https://developer.hashicorp.com/vault/tutorials/policies/policies)
  * [Sentinel policies](https://developer.hashicorp.com/vault/tutorials/policies/sentinel)
* Vault tokens

  * [Create Vault tokens](https://developer.hashicorp.com/vault/docs/commands/token/create)
  * [Token types](https://developer.hashicorp.com/vault/docs/concepts/tokens)
* [Vault policy capabilities](https://developer.hashicorp.com/vault/docs/concepts/policies#capabilities)
* [Vault token periods and TTLs](https://developer.hashicorp.com/vault/docs/concepts/tokens#token-time-to-live-periodic-tokens-and-explicit-max-ttls)
