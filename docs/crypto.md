# Crypto

## 概览

加密算法主要分为以下几类：

1. **对称加密 (Symmetric Encryption)**: 加密和解密使用同一个密钥。速度快，适合大数据量。
2. **非对称加密 (Asymmetric Encryption)**: 使用一对密钥（公钥和私钥）。
3. **摘要算法 (Hash Algorithms)**: 将任意长度数据映射为固定长度的哈希值，不可逆。
4. **消息认证码 (MAC)**: 带密钥的哈希，用于验证完整性和认证。
5. **数字签名 (Digital Signature)**: 保证消息的不可否认性、完整性和认证。

---

## 1. 对称加密 (Symmetric Encryption)

### AES (Advanced Encryption Standard)

目前最广泛使用的对称加密算法，安全性高。

* **密钥长度**: 128, 192, 256 位。
* **常见分组模式**:
    * **ECB (Electronic Codebook)**: 最简单，不安全（相同明文块产生相同密文块），**不推荐使用**。
    * **CBC (Cipher Block Chaining)**: 引入初始化向量 (IV)，安全，最常用的模式之一。串行处理，无法并行。
    * **CTR (Counter)**: 流式加密模式，支持并行计算。
    * **GCM (Galois/Counter Mode)**: 推荐使用。提供机密性的同时提供完整性认证（AEAD），性能优异。

### DES / 3DES

* **DES**: Data Encryption Standard。56位密钥，已**极不安全**，易被暴力破解。
* **3DES**: Triple DES。对数据应用三次 DES。比 DES 安全，但处理速度慢，逐渐被 AES 取代。

### ChaCha20

* Google 推广，专为移动设备设计。在没有硬件 AES 指令集的设备上，性能优于 AES。
* 常与 Poly1305 结合使用 (ChaCha20-Poly1305) 提供 AEAD。

---

## 2. 非对称加密 (Asymmetric Encryption)

加密和解密使用的是两个不同的密钥。

### RSA

* **原理**: 基于大整数分解难题（两个大质数相乘容易，反之分解困难）。
* **用途**:
    * **加密**: **公钥加密，私钥解密**。适合加密少量数据（如对称密钥、Session Key）。
        * *如果公钥加密，任何人都能获得公钥，所以只能保证机密性，不能保证发送者身份。*
    * **签名**: **私钥签名，公钥验签**。
        * *私钥只有持有者有，保证了不可否认性。*
* **缺点**: 运算速度慢，密钥长度长（通常 2048 位或 4096 位）。

### ECC (Elliptic Curve Cryptography)

* **原理**: 基于椭圆曲线离散对数难题。
* **优势**: 相同安全强度下，密钥长度远小于 RSA（256位 ECC ≈ 3072位 RSA），计算更快，带宽占用更小。
* **常见曲线**: P-256, secp256k1 (Bitcoin/Ethereum 使用)。

---

## 3. 摘要算法 (Hash Algorithms)

### MD5 (Message-Digest Algorithm 5)

* **长度**: 128 bit。
* **现状**: **已不安全**（容易构造碰撞）。
* **用途**: 仅用于非安全场景的文件完整性校验（校验和），不可用于密码存储或数字签名。

### SHA (Secure Hash Algorithm) 家族

* **SHA-1**: 160 bit。**已不安全**（Google 和 CWI Amsterdam 已演示碰撞攻击）。
* **SHA-2**: 目前主流标准。包括 SHA-256, SHA-384, SHA-512。
    * **SHA-256**: 广泛用于区块链、TLS 证书、密码存储。
* **SHA-3**: 新一代标准 (Keccak 算法)，内部结构不同于 SHA-2，抗攻击性更强。

### SM3

* 中国国家密码管理局发布的密码杂凑算法，安全性等同于 SHA-256。

---

## 4. 消息认证码 (MAC)

### HMAC (Hash-based MAC)

* 结合 Hash 算法和密钥。
* **原理**: `HMAC(K, m) = H((K' xor opad) || H((K' xor ipad) || m))`
* **用途**: API 接口签名（防止篡改）、身份验证。常见如 `HmacSHA256`。

---

## 5. 数字签名 (Digital Signature)

### RSA 签名

* 流程：`Hash(Message) -> 用私钥加密 Hash 值 -> 签名`。
* 验证：`用公钥解密签名 -> 得到 Hash1；Hash(Message) -> 得到 Hash2；对比 Hash1 == Hash2`。

### DSA (Digital Signature Algorithm)

* 专用于签名的标准（NIST DSS），不能用于数据加密。速度通常比 RSA 签名快，但验签慢。

### ECDSA (Elliptic Curve DSA)

* 基于 ECC 的签名算法。
* **特点**: 签名生成和验证速度快，签名数据极短。
* **风险**: 对随机数 (Nonce) 质量要求极高。如果随机数重复或可预测，私钥直接暴露（索尼 PS3 破解就是因为 Nonce 固定）。

### EdDSA (Edwards-curve DSA)

* 基于 Twisted Edwards 曲线（如 Ed25519）。
* **优势**: 确定性签名（不需要随机数发生器），速度极快，避免了 ECDSA 的随机数风险。现代应用（如 SSH, TLS 1.3）首选。

---

## 总结对比表

| 算法类型      | 常见算法                | 密钥长度(位)     | 典型应用场景                  |
|:----------|:--------------------|:------------|:------------------------|
| **对称加密**  | AES (GCM/CBC)       | 128/256     | HTTPS 通信内容、数据库字段加密、文件加密 |
| **非对称加密** | RSA, ECC            | 2048+, 256+ | HTTPS 握手(交换密钥)、SSH 登录   |
| **摘要算法**  | SHA-256, SHA-3      | 256+        | 密码加盐存储、文件指纹、签名预处理       |
| **数字签名**  | RSA, ECDSA, Ed25519 | -           | 软件防篡改、区块链交易、JWT、CA 证书   |

### SHA-256 vs HmacSHA256

| 特性       | SHA-256                       | HmacSHA256                                           |
|:---------|:------------------------------|:-----------------------------------------------------|
| **全称**   | Secure Hash Algorithm 256-bit | Hash-based Message Authentication Code using SHA-256 |
| **类型**   | **摘要算法 (Hash)**               | **消息认证码 (MAC)**                                      |
| **密钥**   | **无密钥**                       | **需要密钥 (Secret Key)**                                |
| **功能**   | 保证**数据完整性** (Integrity)       | 保证**数据完整性** + **身份认证** (Authentication)              |
| **谁能生成** | 任何人 (只要有数据)                   | 仅持有密钥的人                                              |
| **典型用途** | 文件校验、密码存储(加盐)、数字签名预处理         | API 接口签名、JWT 签名、支付回调验证                               |

**核心差异举例**：

* **SHA-256**: 你下载了一个文件，网站给出了 SHA-256 值。你计算下载文件的 SHA-256，如果一致，说明文件**未损坏**。但你不知道是谁放上去的，黑客可以同时替换文件和 SHA-256 值。
* **HmacSHA256**: 客户端和服务器共享一个密钥 `secret`。客户端发送消息 `msg` 并附带 `HmacSHA256(msg, secret)`。服务器收到后用同样的 `secret` 计算 HMAC。如果一致，服务器确信：
    1. 消息没被篡改（完整性）。
    2. 消息一定是由持有 `secret` 的客户端发出的（身份认证）。