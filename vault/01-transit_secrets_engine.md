# 引擎介绍
// TODO

# 相关定义
## 加密与解密（Encrypt & Decrypt）
加密与解密的主要目的是为了保护数据的安全，防止出现未授权的数据访问。其中加密是将明文转换为密文的过程，这一过程通常会使用加密密钥来完成，而因为加密密钥通常是保密的，所以只有拥有正确密钥的主体才能完成解密操作，并恢复获取到原始的数据。而解密则是加密的逆过程，即将密文转换为明文，需要借助加密密钥来实现。常见的加密算法有 AES、DES、RSA 等。

## 编码与解码（Encode & Decode）
编码与解码的目的主要是为了保证数据格式的正确性，使得数据更加适合传输和存储。在这个过程中，编码是将当前数据转换为另一种数据形式的过程，比如将文本字符转换为二进制数字，以此实现数据的网络间传输。而解码则是其逆流程，即将编码后的数据还原为原始形式的过程，同时，解码方式通常是公开的，任何主体都可以使用相应的规则或算法，来对编码后的数据进行解码。常见的编码格式有 UTF-8、BASE64、ASCII 等。

## 差异点
- **安全性**：加密是专门为了安全性所设计，只有拥有正确密钥的主体，才能对数据进行访问，而编码则不涉及安全性，只要掌握了编码规则或算法，就可以对编码数据进行解码；
- **密钥**：加密通常具有一个或多个密钥，而编码则通常不存在密钥；
- **公开性**：编码规则通常是公开的，任何人都可以进行编码和解码操作，而加密虽然可能使用公开的算法，但密钥却是保密的；
- **目的**：加密是为了保证数据的安全性，保护数据不会被未经授权的主体所访问，而编码则主要是为了数据能够被正常的传输和存储；


# 基本操作

## 创建加密使用的密钥

### 1. 启用 Transit 密钥引擎
由于 Vault 的密钥引擎采用的是显示管理的方案，因此，当我们需要使用 Transit 密钥引擎时，首先需要通过 `secrets enable` 命令，来启用对应的引擎。在当前的场景下，即使用如下所示的命令：

```shell
$ vault secrets enable transit
Success! Enabled the transit secrets engine at: transit/
```

### 2. 创建具名的加密密钥
Vault 采用了一种 REST 的设计风格，即将所有操作均抽象为一种具有层级的资源管理行为。所以当我们需要创建一个加密密钥时，需要采用增加 “加密密钥” 资源的方式来进行，即在 `transit` 的密钥管理引擎下，通过 `write -f` 强制写入的方式，创建一个名称为 `my-key` 的加密密钥资源（`key`）。通过资源路径可以表示为 `transit/keys/:name`，当命令完成后，我们就完成了加密密钥的创建。

```shell
$ vault write -f transit/keys/my-key
Success! Data written to: transit/keys/my-key
```

### 3. 查看加密密钥列表
通过 `vault list` 命令，我们可以查看当前 transit 引擎下现存的密钥列表。

```shell
$ vault list transit/keys

Keys
----
my-key
```


## 使用密钥进行加密（Encrypt）
在上一节中，我们已经完成了加密密钥的创建，接下来我们将使用上一节中创建的密钥，来对凭证进行加密。与上文相同，凭证的加密也是基于 REST 资源管理的方式来完成的，即我们需要向 `transit` 密钥引擎下写入（`write`）一个 `encrypt/:name` 资源，加密的参数通过 `plaintext` 的方式来进行指定。需要注意的是，在 Vault 加密中，加密的内容均需使用 base64 进行编码，这主要是为了增加加密场景的通用性，因此，通过 vault encrypt 我们不止可以对文本进行加密，也可以对二进制文件进行加密。

在下面的脚本中，我们通过执行 `vault write` 命名，对 base64 编码后的内容 `my secret data` 进行了加密，而后得到了一个加密后的 ciphertext 密文。在密文中，起始的 **vault** 表示当前密文是由 vault 进行管理的，而后的 **v1** 表示当前密钥的版本为 v1 版本，该版本主要用于后续的凭证轮转。

```shell
$ vault write transit/encrypt/my-key plaintext=$(echo "my secret data" | base64)

Key           Value
---           -----
ciphertext    vault:v1:8SDd3WHDOjf7mq69CyCqYjBXAiQQAVZRkFM13ok481zoCmHnSeDX9vyf7w==

```

## 使用密钥进行解密（Decrypt）
完成密钥的加密后，接下来要完成的是密钥的解密流程。解密操作与加密操作相同，本质上都是发送一个资源请求，在解密场景下，是发送（`write`）一个 `decrypt/:name` 请求，并通过 `ciphertext` 参数来携带待解密的密文。同时，我们可以通过 `-filed` 参数来对响应结果进行解析，获取指定的字段内容。

在下面的脚本中，我们通过写入一个 `my-key` 加密密钥的解密资源请求，对密文进行解密，并通过 `-field=plaintext` 参数来提取解密后的 `plaintext` 字段内容，然后对齐进行 base64 解码，最终获取到明文。
 
```shell
$ vault write -field=plaintext transit/decrypt/my-key ciphertext=... | base64 --decode
my secret data
```

# 原理介绍
// TODO


# 参考文档

【1】[Transit secrets engine](https://developer.hashicorp.com/vault/docs/secrets/transit)