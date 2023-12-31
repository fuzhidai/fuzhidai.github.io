# 引擎介绍
Transit Secrets Engine（传输秘密引擎），是 Vault 提供的一个用于对被传输数据进行加解密的秘密引擎，它不会对加解密的数据进行存储，而仅仅是作为一种 “加密即服务” 的服务提供者，并提供了对数据进行签名和验证的功能。

该引擎的主要目标，是将待传输数据的加解密工作，从应用的主体业务中剥离出来，降低业务主体的复杂度，同时提升加解密服务的可复用性和稳定性。

值得一提的是，Transit 秘密引擎还支持 “密钥派生” 功能，即能够通过用户提供的上下文，来 “派生” 出相应的新密钥，并将其分别应用到不同的场景中。默认设置下，密文的生成是不收敛的，即同一个明文多次加密，每次都会生成不同的密文，但你也可以选择进行 “收敛加密”，即允许对相同的明文产生相同的密文。

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

### 3. 查看新创建的加密密钥
通过 `vault list` 命令，我们可以查看当前 transit 引擎下现存的密钥列表。在执行命令后，我们可以看到刚刚创建的名称为 “my-key” 的加密密钥。

```shell
$ vault list transit/keys

Keys
----
my-key
```

而后再通过 `vault read` 命令，我们可以进一步获取密钥更多的详细信息。`vault read` 命令本质是发起对资源的读取操作，并通过参数来指定待读取的资源路径。在当前场景下，我们需要读取的资源为 “transit” 加密引擎下，名称为 “my-key” 的密钥资源，所以路径为 `transit/keys/:name`，`:name` 为 “my-key”。

运行完命令后，我们可以获取到密钥的详细信息，包括密钥的最新版本（latest_version），密钥下的所有版本（keys），最小解密版本（min_decryption_version），最小加密版本（min_encryption_version），以及当前的加密类型（type）等。这里现在需要重点关注的是 “type” 和 “version” 相关的信息，其中 “type” 指明了密钥的加密算法为 “aes256-gcm96”，这是 transit 加密引擎的默认加密算法。而密钥版本信息，则会在后续的密钥轮转中发挥作用。

```shell
$ vault read transit/keys/my-key

Key                       Value
---                       -----
allow_plaintext_backup    false
auto_rotate_period        0s
deletion_allowed          false
derived                   false
exportable                false
imported_key              false
keys                      map[1:1702100117]
latest_version            1
min_available_version     0
min_decryption_version    1
min_encryption_version    0
name                      my-key
supports_decryption       true
supports_derivation       true
supports_encryption       true
supports_signing          false
type                      aes256-gcm96
```

## 使用密钥进行加密（Encrypt）
在上一节中，我们已经完成了加密密钥的创建，接下来我们将使用上一节中创建的密钥，来对凭证进行加密。与上文相同，凭证的加密也是基于 REST 资源管理的方式来完成的，即我们需要向 `transit` 密钥引擎下写入（`write`）一个 `encrypt/:name` 资源，加密的参数通过 `plaintext` 的方式来进行指定。需要注意的是，在 Vault 加密中，加密的内容均需使用 base64 进行编码，这主要是为了增加加密场景的通用性，因此，通过 vault encrypt 我们不止可以对文本进行加密，也可以对二进制文件进行加密。

在下面的脚本中，我们通过执行 `vault write` 命名，对 base64 编码后的内容 `my secret data` 进行了加密，而后得到了一个加密后的 ciphertext 密文。在密文中，起始的 “vault” 表示当前密文是由 vault 进行管理的，而后的 “v1” 表示当前密钥的版本为 v1 版本，该版本主要用于后续的凭证轮转。

```shell
$ vault write transit/encrypt/my-key plaintext=$(echo "my secret data" | base64)

Key           Value
---           -----
ciphertext     vault:v1:LxLIHRN8sRdZ0t8dQPZ4CcgRKM9mdRZ89iAPV3+GKF6r2YiremX/JEq+BQ==
key_version    1
```

这里需要注意的是，当我们多次重复使用同一个密钥（版本也相同），来对同一个明文进行加密时，获取到的密文是不同的，即 “默认设置下” 加密操作是非幂等操作。

```shell
$ vault write transit/encrypt/my-key plaintext=$(echo "my secret data" | base64)

Key            Value
---            -----
ciphertext     vault:v1:ijM9KlWXXMyPtsEy0WPbwjU87sFUwMl7qqrud4vhgsbg1N0ANNj3PK+Tpg==
key_version    1
```

## 使用密钥进行解密（Decrypt）
完成密钥的加密后，接下来要完成的是密钥的解密流程。解密操作与加密操作相同，本质上都是发送一个资源请求，在解密场景下，是 `write` 写入一个 `decrypt/:name` 资源，并通过 `ciphertext` 参数来携带待解密的密文。同时，我们可以通过 `-filed` 参数来对响应结果进行解析，获取指定的字段内容。

在下面的脚本中，我们通过写入一个 `my-key` 加密密钥的解密资源请求，对密文进行解密，并通过 `-field=plaintext` 参数来提取解密后的 `plaintext` 字段内容，然后对齐进行 base64 解码，最终获取到明文。
 
```shell
$ vault write -field=plaintext transit/decrypt/my-key ciphertext=vault:v1:LxLIHRN8sRdZ0t8dQPZ4CcgRKM9mdRZ89iAPV3+GKF6r2YiremX/JEq+BQ== | base64 --decode

my secret data
```

同时，加密过程中产生的所有密文，均可以使用密钥进行解密，获取到其对应的明文内容.

```shell
$ vault write -field=plaintext transit/decrypt/my-key ciphertext=vault:v1:ijM9KlWXXMyPtsEy0WPbwjU87sFUwMl7qqrud4vhgsbg1N0ANNj3PK+Tpg== | base64 --decode

my secret data
```

## 轮转加密密钥（Rotate）

### 1. 轮转密钥
在前面的几节中，我们已经完成了使用密钥对明文进行加密的操作，获取到了加密后的密文，而后又通过使用该密钥对密文进行了解密，并最终获取到了被加密的明文，这已经完成了常见的凭证加解密管理操作。但当密钥出现泄漏时，或出于安全性考虑需要对密钥进行周期性更换时，我们就需要使用到密钥的轮转功能，通过密钥轮转来生成新版本的密钥，再对需要更换的密文进行重新加密的操作。

与前面创建具名密钥的操作类似，密钥的轮转操作也是通过 `vault write` 命令来完成的，只不过写入的资源变成了具体密钥下的 “轮转操作”，即 `transit/keys/:name/rotate`，其中 `:name` 是密钥的名称，完整的命令如下所示。

```shell
$ vault write -f transit/keys/my-key/rotate
Success! Data written to: transit/keys/my-key/rotate
```

当完成密钥的轮转操作后，我们可以再次通过 `vault read` 命令，来查询最新的密钥信息。此时可以看到，密钥的最新版本（latest_version）变成了 “2”，同时密钥版本集（keys）中多了一个新的密钥，但此时最小解密版本（min_decryption_version）仍为 “1”，也就意味着，我们仍然可以使用该密钥对 v1 版本的密文进行解密操作。

```shell
Key                       Value
---                       -----
allow_plaintext_backup    false
auto_rotate_period        0s
deletion_allowed          false
derived                   false
exportable                false
imported_key              false
keys                      map[1:1702100117 2:1702101111]
latest_version            2
min_available_version     0
min_decryption_version    1
min_encryption_version    0
name                      my-key
supports_decryption       true
supports_derivation       true
supports_encryption       true
supports_signing          false
type                      aes256-gcm96
```

### 2. 探索向前兼容性
因此，当我们再次使用该密钥，来对 v1 版本的密文进行解密时，会发现解密的流程依然是可以正常进行的，密钥的轮转带来的版本升级，并没有对存量输出产生影响。这也意味着，在 “transit” 加密引擎中，密钥的轮转操作是可以做到数据的前向兼容的。

```shell
$ vault write -field=plaintext transit/decrypt/my-key ciphertext=vault:v1:LxLIHRN8sRdZ0t8dQPZ4CcgRKM9mdRZ89iAPV3+GKF6r2YiremX/JEq+BQ== | base64 --decode

my secret data
```

甚至，我们还可以通过指定密钥版本 `key_version` 的方式，继续使用 v1 版本的密钥，来对数据进行加密操作。

```shell
$ vault write transit/encrypt/my-key key_version=1 plaintext=$(echo "my secret data" | base64)
Key            Value
---            -----
ciphertext     vault:v1:oRR/lbMeiPet6Gfkaq7KUOCk85zEGWo0O0kdqV3WO9tlEwF+2Ins+EEuvw==
key_version    1

```

### 3. 使用新版密钥

但此时，当我们再次使用同一个密钥 “my-key”，来对同一个明文进行加密时，会发现获取到的密文版本已经变成了 v2 版本，同时密文的内容也发生了变化。

```shell
$ vault write transit/encrypt/my-key plaintext=$(echo "my secret data" | base64)

Key            Value
---            -----
ciphertext     vault:v2:fX66cCjdelJqlR91YPuwBmLNid7S5/E1P9LunuBoFJINP0cF6QK4RSjbrA==
key_version    2
```

而后对该密文进行解密，可以确认新版本密钥的解密操作，也是符合预期的。

```shell
$ vault write -field=plaintext transit/decrypt/my-key ciphertext=vault:v2:fX66cCjdelJqlR91YPuwBmLNid7S5/E1P9LunuBoFJINP0cF6QK4RSjbrA== | base64 --decode

my secret data
```

## 升级加密数据的版本

Vault 中存量密文数据的密钥升级（密文重封装）操作，是通过 `rewrap` 来完成的。通过 `vault write` 的写入命令，向指定的资源路径下 `transit/rewrap/:name` 写入一个密文重封装请求，并携带要重封装的密文数据，Vault 即可完成基于旧版密钥的密文数据 “解密” 流程，以及基于新版密钥的 “加密” 流程，并最终返回重封装后的密文数据。在当前场景下，具体的命令如下所示。

```shell
$ vault write transit/rewrap/my-key ciphertext=vault:v1:LxLIHRN8sRdZ0t8dQPZ4CcgRKM9mdRZ89iAPV3+GKF6r2YiremX/JEq+BQ==

Key            Value
---            -----
ciphertext     vault:v2:vcT+1bA3Of4FU6KTdk6Y6eMifF+hlytxckgLQQXexfydgkmYe9xQK4UETw==
key_version    2
```

需要注意的是，密文重封装 `rewrap` 操作，本质上是对原有密文进行解密操作，然后再使用最新版本的密钥对其进行加密操作，以此获取最新版本密钥对应的密文，仅此而已，并不会涉及对原有密钥的禁用操作。这也就意味着，在进行密文的重封装操作后，原有的密文依然是可以被正常使用（解密）的。

```shell
$ vault write -field=plaintext transit/decrypt/my-key ciphertext=vault:v1:LxLIHRN8sRdZ0t8dQPZ4CcgRKM9mdRZ89iAPV3+GKF6r2YiremX/JEq+BQ== | base64 --decode

my secret data
```

# 原理介绍

## 轮转流程

### 1. 如何实现轮转的向前兼容性？
// TODO


# 参考文档

【1】[Transit secrets engine](https://developer.hashicorp.com/vault/docs/secrets/transit)