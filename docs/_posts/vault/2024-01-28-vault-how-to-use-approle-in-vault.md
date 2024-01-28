---
layout:         post
title:          "Vault 系列：如何在 HashiCorp Vault 中正确使用 AppRole"
subtitle:       "How (and Why) to Use AppRole Correctly in HashiCorp Vault"
date:           2024-01-28 18:31:00 +0800
author:         "fuzhidai"
header-style:   text
tags:
    - Vault
    - AppRole
    - 翻译
---
> 本文翻译自 HashiCorp Vault 的官方 Blog，仅供学习使用，原文地址为：[How (and Why) to Use AppRole Correctly in HashiCorp Vault](https://www.hashicorp.com/blog/how-and-why-to-use-approle-correctly-in-hashicorp-vault?product_intent=vault)

与许多综合的解决方案一样，[HashiCorp Vault](https://www.vaultproject.io/?ajs_aid=5caed021-6539-4314-a163-e4ce4230bef5&product_intent=vault) 也有一个学习的曲线。它固执己见地认为安全地做事才是正确，但有时这种固执己见带来的复杂性会导致人们选择去走 “捷径”，而这些所谓的 “捷径” 可能会在秘密或用于验证 Vault 的凭据周围产生安全的漏洞。因此，让我们浏览一下使用 [AppRole 身份验证方法](https://developer.hashicorp.com/vault/docs/auth/approle?ajs_aid=5caed021-6539-4314-a163-e4ce4230bef5&product_intent=vault) 的一些最佳实践，并了解我们为什么不应该走那些 “捷径”。

注意：这不是 Vault 集群加固指南，请参阅我们的 [Vault 产品加固指南](https://developer.hashicorp.com/vault/tutorials/operations/production-hardening)，并从攻击者的角度了解如何试图破坏Vault，我推荐我长期同事 John Boero 的博客文章：[我如何攻击您的 HashiCorp Vault（以及您如何阻止我）系统加固。](https://medium.com/hashicorp-engineering/how-id-attack-your-hashicorp-vault-and-how-you-can-prevent-me-system-hardening-ce151454e26b)

### 当你需要 AppRole：Secret Zero

在应用程序可以从 Vault 获取秘密之前，需要为它们提供一个可以进行身份验证的秘密 —— 这有点像先有鸡还是先有蛋的问题，我们称之为 “安全引入” 或 “秘密零” 难题。应用程序可以通过以下三种基本方法来解决这个难题，向 Vault 进行身份验证，并获取令牌：

* 通过使用底层平台身份（云提供商 IAM 角色、Kubernetes 服务帐户等）；
* 通过使用运营商提供的非平台身份验证（用户名/密码）；
* 通过完全绕过身份验证，使用直接提供给应用程序的令牌 —— 我称之为 “从天而降的令牌”；

从天而降给应用程序一个令牌是最糟糕的 —— 你必须保证自己安全交付这个令牌，而且你也无法获取这个令牌与应用程序身份之间的关联关系，除非你自己通过实体别名为你创建的每个应用令牌建立这种关联。相比之下，为应用程序提供非平台证书会更好一些（它将身份与应用程序关联起来），但你仍然面临安全方面的挑战。

通过平台进行应用程序身份验证的方式是最好，但并不是每个平台都集成了 Vault 的身份验证。你可能在云上没有任何 Vault 的身份验证插件，或者你可能使用的是裸机。幸运的是，Vault 提供了一种身份验证方法，使得即使没有本地平台集成，也可以为你提供基于平台进行身份验证的诸多优点：AppRole 能够让你轻松有效地为应用程序构建可信的代理。

### 什么让 AppRole 更好？

使 AppRole 优于直接分发令牌的最重要的特性是，凭据被分成 [角色 ID（Role ID）](https://www.vaultproject.io/docs/auth/approle?ajs_aid=5caed021-6539-4314-a163-e4ce4230bef5&product_intent=vault#roleid)和 [秘密 ID（Secret ID）](https://www.vaultproject.io/docs/auth/approle?ajs_aid=5caed021-6539-4314-a163-e4ce4230bef5&product_intent=vault#secretid)，并通过不同的渠道进行传递下发。此外，Secret ID 仅在预期使用的时间（通常是在应用程序启动时）才会交付给应用程序。

这种通过独立交付链路，即时、部分地交付信息的授权模式，与标准的多因素身份验证方法非常相似：首先你需要登陆到服务中，此时你已经有一个已知的身份，但你还是需要在登陆时生成并交付一个一次性的令牌。

> 译者注：这里可以类比于登陆一些知识论坛，当你登陆时需要输入你的账号和密码，但账号和密码验证通过后，可能还是会出于某些安全因素，让你从手机上获取一个动态临时的验证码，然后你需要将该验证码提供给平台进行最终的身份核验。

Role ID 并非敏感信息，因此可以将其用于指定应用程序的任意数量的实例，你可以将它硬编码到 VM 或容器映像中（尽管作为最佳实践，你不应该将其提供给不需要它的进程，比如，你不应该将它提供给那些管理角色的进程，而是应该将它提供给那些使用它来进行身份验证的进程）。

而对于 Secret ID，则与此相反：

* 一方面，其旨在限制访问，因此只能由被授权的应用程序进行使用。它可能仅供单个应用程序使用，甚至可能仅供单个应用程序的单个实例使用；
* 另一方面，其旨在缩短暴露的时间窗口，所以它的有效期可能仅为几秒钟；

我们后续在 AppRole 完整的使用过程中介绍的许多步骤，都旨在保证这些属性中的某一个或另一个。

### 如何正确的使用 AppRole

对于我们将逐步完成的过程，请参阅下面的图，取自 [AppRole 拉取认证](https://developer.hashicorp.com/vault/tutorials/auth-methods/approle?in=vault%2Fauth-methods) 教程的 [Secret ID 的响应包装](https://developer.hashicorp.com/vault/tutorials/auth-methods/approle?in=vault%2Fauth-methods#response-wrap-the-secretid) 部分（稍后我们将介绍什么是响应包装）。

![how-to-use-approle](/img/post-vault-how-to-use-approle.png)

*图片来源: Hashicorp Vault Blog*

图 1 - AppRole 使用流程示意图

这里显示了 11 个步骤，从简单的操作到具有多个选项或备选方案的操作。

重要提示：“你的应用程序”

在整个过程中，我将讨论 “你的应用程序” 如何使用 AppRole 进行身份验证。但是，如果你有一个应用程序期望能够直接获取到 Vault 令牌，而你无法更改它以执行必要的步骤，此时该怎么办？Vault 二进制文件具有代理模式，在这种情况下，该模式可以使用文件中提供的 AppRole 认证组件向 Vault 进行身份验证，并将生成的令牌保存到接收的文件中，而后应用程序可以从中读取令牌。默认情况下，它甚至会更新可再生令牌。在这两种情况下，这里的工作流程都同样适用，它只是由代理进行处理，而不是由你的应用程序直接进行处理 —— 有效地，代理成为你应用程序的插件。

### AppRole：一步一步

本教程的 “第一步” 是使用 TLS 保护你与 Vault 的通信。如果没有这一步，Vault 的其他安全措施从一开始就已经被破坏了。说真的，如果你还没有使用 TLS 保护你的 Vault 部署，请在阅读本文的其余部分之前进行保护。

### 步骤 1

[启用 AppRole 认证方法](https://www.vaultproject.io/docs/auth?ajs_aid=5caed021-6539-4314-a163-e4ce4230bef5&product_intent=vault#enabling-disabling-auth-methods)，是时候考虑管理角色之类的事情了。如果你将角色的管理委托给某人，他是否需要管理每个角色？如果不是，你可能需要在多个路径上启用 AppRole，以便你可以轻松地授予权限来管理角色集。

如果你正在使用 Vault 企业版，你可能还希望能够设置命名空间，并在其下启用此身份验证方法。这将使你能够更轻松地授予权限给那些你创建的角色，使得你可以与它们一起协同来管理机密。

### 步骤 2

为应用程序 [创建角色](https://www.vaultproject.io/docs/auth/approle?ajs_aid=5caed021-6539-4314-a163-e4ce4230bef5&product_intent=vault#configuration) 和策略。角色是一个可重用的单元，包含一组 Vault 策略，这些策略决定了应用程序可以访问哪些秘密，如果两个应用程序具有相同的角色，那它们将可以访问相同的秘密。因此，请确保创建正确数量的角色，以此提供应用程序之间所需的区分级别，同时，应考虑确保每个应用程序都遵循 [最小权限原则](https://en.wikipedia.org/wiki/Principle_of_least_privilege)。

记住，一个角色可以附加多个策略，所以如果你的应用程序需要共享一部分秘密，但不是全部秘密，你可以创建一个策略，授予一组应用程序对公共秘密的访问权限，同时再创建一些特定于应用程序的策略，以此满足每个应用程序的独特需求。

在创建角色时，应尽可能更多地使用相关的特性：

* [TTL（生存时间）、使用次数和 Secret ID 的源 CIDR 限制](https://developer.hashicorp.com/vault/api-docs/auth/approle?ajs_aid=5caed021-6539-4314-a163-e4ce4230bef5&product_intent=vault#secret_id_bound_cidrs)，它们能够控制通过使用 Secret ID 向 Vault 进行身份验证的方式，比如可以从哪里发起对 Secret ID 的使用，以及该 Secret ID 可以被使用多少次；
* [同样的限制](https://developer.hashicorp.com/vault/api-docs/auth/approle?ajs_aid=5caed021-6539-4314-a163-e4ce4230bef5&product_intent=vault#token_ttl) 也适用于通过身份验证创建的令牌；

### 步骤 3 和步骤 4

请求并接收角色的 Role ID。

通常情况下，你应该将 Role ID 插入到与应用程序相关的组件中 —— 如它运行的虚拟机映像、配置文件模板等。

### 步骤 5

在将来的某个时候，你将运行你的应用程序。一旦完成上述的配置后，应用程序就已经具备访问 Vault 认证所需的半数信息了。从下一步开始，我们将向它提供认证所需的另一半信息。

重要提示：确保流程的执行顺序

步骤 6-8（生成并交付 Secret ID）有时在步骤 5 之前完成，但最安全的方式是按照这里所列出的顺序来执行。在交付 Secret ID 之前就完成应用程序的启动，以此保证程序能够在获取到 Secret ID 后就立即使用它。这样 Secret ID 就不会出现被闲置的情况（此时闲置的 Secret ID 可以很容易被拦截，无论拦截的程度有多低）。

理想情况下，应用程序应等待 Secret ID 出现，然后立即使用它，或者干脆终止并重新启动（例如由监督编排器），直到 Secret ID 可用为止。但在某些情况下，如果应用程序无法在 Secret ID 出现之前就完成启动，则可能必须预先交付 Secret ID 给应用才可以。

### 步骤 6 和步骤 7

你的运行时平台现在需要请求并接收一个 Secret ID，并将其提供给应用程序。为此需要使用 Vault 标识，该标识只与允许执行此操作的策略相关联。

你可以让你的平台检索一个明文的 Secret ID，并将其直接提供给你的应用，但这再次引入了一个你试图避免的问题：你必须确保这个敏感的凭据能够被安全的处理。

为了解决这种情况，Vault 支持响应封装 Secret ID（Wrapped Secret ID） —— 不是字面意义上的 Secret ID，响应封装 Secret ID 会返回一个一次性令牌，可用于 Vault API 中的 “解封” 操作。在解封时，Vault 将返回原始的秘密 —— 在本例中是一个 AppRole 的 Secret ID。

Secret ID 响应封装提供了三个基本好处：

* 隐匿性：当封装令牌通过你的可信平台传递给最终运行的应用程序时，任何处理并传递它的服务都不需要知道原始的 Secret ID；
* 有限的曝光时间：封装令牌具有自己的 TTL（有效期），以此保证未使用的封装令牌在未更新时会自动过期，防止其因长期处于活动状态，而被攻击者所利用；
* 防篡改：如果一个受感染的应用程序或恶意进程拦截并解封了令牌，那么当授权的应用程序再次试图打开该令牌时，它将得到一个错误的响应。通过在新路径上存储未封装的 Secret ID，并传递来自该新秘密的新的封装令牌（Wrapping Token），可以提供进一步的保护层，防止攻击者逃避解包检测 —— 封装令牌总是包含一个属性，给出它们所封装的秘密的路径。你的应用程序应该知道并验证此路径，以检测恶意的 Secret ID 重封装操作。如果你正在使用 Vault Agent 并向其提供源路径，则它将自动执行此验证。

如果封装令牌解封失败，或者封装令牌的创建路径不符合预期，则系统应当发出警报，此时应立即进行调查。你还应该在发现任何围绕角色 Secret ID 的意外活动时，发出警报并进行调查。例如，如果你正在为 [CI/CD 流水线使用该模式](https://developer.hashicorp.com/vault/tutorials/recommended-patterns/pattern-approle?in=vault%2Frecommended-patterns)，并为流水线角色生成了一个 Secret ID，但此时却没有流水线正在运行，那可能是发生了异常的情况。

你可以通过在创建 Secret ID 时，应用一个强制使用 TTLs 的策略，来确保 Secret ID 会被执行响应封装 —— 在 Vault 策略文档中有一个 [示例策略](https://developer.hashicorp.com/vault/docs/concepts/policies?ajs_aid=5caed021-6539-4314-a163-e4ce4230bef5&product_intent=vault#required-response-wrapping-ttls) 就是这样做的（注意，此处不需要提供 Role ID，只需提供角色路径即可）。

### 步骤 8

将检索到的 Secret ID 封装令牌提供给授权的应用程序。

如何做到这一点取决于你 —— 你可以通过调用应用程序中的 API 来推送它，也可以让应用程序通过读取文件来获取它，或者你也可以通过其他的方式来完成此操作。但无论使用哪种方法，请确保它不是通过传递 Role ID 的相同通道来下发的。

另一个好的做法是，在使用封装令牌进行身份验证后，从获取它的地方删除原始的封装令牌，例如，可以通过删除能够读取到封装令牌的文件来完成此操作（如果使用代理，默认情况下它将尝试删除该文件）。

下面是一个示例工作流动画，用于向容器内的应用程序传递一个封装的 Secret ID：
![post-vault-approle-in-container](/img/post-vault-approle-in-container.gif)

*图片来源: Hashicorp Vault Blog*

### 步骤 9-11

应用程序解封封装的 Secret ID，使用 Role ID 和 Secret ID 向 Vault 进行身份验证，并接收 Vault 返回的令牌。

方便的是，如果你正在使用代理，它将自动识别封装令牌，并在尝试进行身份验证之前，使用它来获取 Secret ID。

一个关键的原则是，应用程序应尽快解封并使用 Secret ID —— 记住，如果 Secret ID 长时间未被使用，那么以下的一个或多个推论将可能会出现：

* 你将暴露一个可被利用的凭证；
* 封装令牌或原始的 Secret ID 最终会过期，这意味着你的应用程序将无法再使用它；
* 如果你的凭证已被拦截并被利用，那么在你尝试进行身份验证之前，你将不会发现该泄露情况；

### 步骤 12

前面的图中没有第 12 步，但执行前面所有步骤的基本要点是：使用返回的令牌来访问 Vault 中的秘密（具体的方法会因秘密类型的不同而有所不同）。

代理在这里也可以提供帮助：如果你的应用程序根本不支持 Vault，[代理不仅可以代表它向 Vault 进行身份验证，还可以检索你应用程序需要的秘密](https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent/template?ajs_aid=5caed021-6539-4314-a163-e4ce4230bef5&product_intent=vault)，并以应用程序期望的格式将它们写入文件。

此时，你的应用程序将有一个 Vault 令牌，它已经检索了它需要访问的秘密，相关凭证的组件也已经被清理，并且应该运行正常。下一步是复习 [Vault 策略的最佳实践](https://developer.hashicorp.com/vault/tutorials/policies/policies)、[Vault 中的秘密租约](https://developer.hashicorp.com/vault/docs/concepts/lease?ajs_aid=5caed021-6539-4314-a163-e4ce4230bef5&product_intent=vault) 以及 [令牌的过期和续订](https://developer.hashicorp.com/vault/tutorials/tokens/tokens#renew-service-tokens)。

### AppRole 的最佳实践

在实际情况中，如何实现 AppRole 的细节可能会有所不同，但请确保牢记以下这些基本原则：

* 使用与最小权限策略绑定的身份来管理角色。监视角色的管理操作，并对可能执行恶意活动的意外操作发出警告；
* 通过单独的路径，仅在需要时才提供凭据组件（Role ID 和 Secret ID），并仅在需要使用它们进行身份验证的应用程序需要获取它们的地方才提供它们；
* 确保凭据能够被立即使用，并设置可选配置来避免将其暴露给未经授权的访问对象（响应包装、ttl、使用限制和源 CIDR 限制）。监视和告警封装解封或身份验证中的任何错误，这些错误可能表明凭证在传输中被恶意拦截；
* 最后，遵循常规的 Vault 最佳实践，通过 [使用策略](https://developer.hashicorp.com/vault/tutorials/policies/policies) 来控制对机密的访问权限，就像你对任何其他客户端标识所做的那样；

如果你希望在操作 Vault 以扩展用例时，消除大部分的复杂性，请考虑使用 HCP Vault 的按需部署进行管理。
