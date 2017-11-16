# Hash MAC CBC-MAC HMAC AEAD GCM 笔记整理

[Letty51](https://github.com/letty51)

## Hash（Hash Function）

散列函数 Hash Function，又称散列算法、哈希函数，就是把任意长度的输入，通过 Hash 算法，变换成固定长度的输出，该输出就是散列值（Hash 值）。这种转换是一种压缩映射，Hash 值的空间通常远小于输入的空间。不同的输入可能会散列成相同的输出，这种情况称为碰撞。

![](https://github.com/Letty51/pictures/blob/master/Hash.png)

### Hash 函数分为两类：

- 改动检测码 MDC（Manipulation Detection Code），不带密钥的哈希函数，主要用于消息完整性验证；
- 消息验证码 MAC（Message Authentication Code），带密钥的哈希函数，主要用于消息源认证和消息完整性验证。

## MAC（Message Authentication Code）

消息验证码 MAC 是一种消息认证技术，用来检查一段信息、消息的可靠性和完整性。一个 MAC 算法接收一段任意长度的消息和一个固定长度的密钥 key，产生一个 MAC 值。
发送者传输的数据必然包含两个部分，即加密的数据部分以及消息验证码部分。
1. 发送者和接收者共享密钥 K，发送者通过MAC算法计算出消息的MAC值，并和原消息一起发给接收者；
2. 接收者对收到的消息用相同的密钥 K 进行相同的MAC计算得到 MAC 值，并与接收到的 MAC 值进行对比。若相等，则：
> - 接收者可以相信消息未被修改；
> - 接收者可以确信消息来自真正的发送者。

![](https://github.com/Letty51/pictures/blob/master/MAC.png)

### MAC 和 Encrypt 有三种结合使用的方式：

- Encrypt then MAC，先将明文加密，然后计算密文的 MAC，最后将密文和MAC值一起发送出去。
> $$ C = E(K_c,P) $$
> $$ t = MAC(K_M,C) $$

![](https://github.com/Letty51/pictures/blob/master/Encrypt%20then%20MAC.png)

- MAC then encrypt，先计算明文的 MAC，然后把明文和计算出来的 MAC 值一块加密，最后将密文发送出去。
> $$ t = MAC(K_M,P) $$
> $$ C = E(K_c,P||t) $$

![](https://github.com/Letty51/pictures/blob/master/MAC%20then%20Encrypt.png)

- Encrypt and MAC，分别对明文进行加密和 MAC 计算，最后分别将密文和 MAC 值发送出去。
> $$ C = E(K_c,P) $$
> $$ t = MAC(K_M,P) $$

![](https://github.com/Letty51/pictures/blob/master/Encrypt%20and%20MAC.png)

总体来说，更推荐使用 Encrypt then MAC，优点在于收到密文和 MAC 值之后，可以先检验 MAC 值，如果不正确，就不用对密文进行解密了。MAC then encrypt 也可以使用，但是收到密文之后，必须先全部解密出来之后，才能校验消息是否完整。Encrypt and MAC 的安全性最差，通过 MAC 值就可以暴露出明文的一些特征，同样的明文会得到同样的 MAC 值。

MAC 算法包含 CBC-MAC，HMAC 等。

## CBC-MAC

CBC-MAC 的基本思想就是对消息使用 CBC 模式进行加密，取密文的最后一块作为 MAC 值。

![](https://github.com/Letty51/pictures/blob/master/CBC-MAC.png)

## HMAC（Hash-based Message Authentication Code）

散列消息鉴别码 HMAC，主要利用 Hash 算法，以一个密钥和一个消息为输入，生成一个固定大小的消息摘要，并将其加入到消息中作为输出。HMAC 算法可以与任何迭代 Hash 函数捆绑使用，例如 MD5 和 SHA-1 函数。HMAC 是目前最常用，也是目前比较安全的一种 MAC 算法。其安全性依赖于 Hash 函数，故也称为带密钥的 Hash 函数。

$$ HMAC(K,M) = H(K \oplus opad || H(K \oplus ipad) || M) $$
> H ：采用 Hash 算法得到的散列值 \
> K ：密钥 \
> M ：一个消息输入 \
> opad ：外部填充（0x5c5c5c…5c，0x5c 重复 B 次） \
> ipad ：内部填充（0x363636…36，0x36 重复 B 次） \
> B ： 中所处理的块大小（MD5、SHA-1、SHA-256 处理的块大小为 64 Bytes；SHA-384、SHA-512 处理的块大小为 128 Bytes） \
> L ：最后输出的 MAC 值的大小（MD5 的输出大小为 128 bits；SHA-1 的输出大小为 160 bits；SHA-256 的输出大小为 256 bits; SHA-384 的输出大小为 384 bits …）

![](https://github.com/Letty51/pictures/blob/master/HMAC.png)

Step 1：对密钥 $K$ 进行处理：如果密钥长度小于 $B$，就在密钥 $K$ 后面填充 $0$ 来生成一个长为 $B$ 的子密钥；如果密钥长度大于 $B$，就先对密钥进行 $Hash$ 计算。（$K$ 可以有不超过 $B$ 字节的任意长度，但一般建议其长度不小于 $L$。） \
Step 2：将子密钥与 $ipad$ 进行异或运算生成 $S_1$； \
Step 3：将消息输入 $M$ 填充到 $S_1$ 之后； \
Step 4：对 Step 3 中生成的数据流进行 $Hash$ 计算； \
Step 5：将子密钥与 $opad$ 进行异或运算生成 $S_2$； \
Step 6：将 Step 4 中生成的 $Hash$ 值填充到 $S_2$ 之后； \
Step 7：对 Step 6 中生成的数据流进行 $Hash$ 计算，输出最终结果。

## AEAD（Authenticated-Encryption With Additional Data）

带附加数据的认证加密模式AEAD 的作用类似于 Encrypt then MAC，GCM 加密模式就是 AEAD 的一种。

## GCM（Galois/Counter Mode）

GCM 的设计思路是利用 CTR 模式和泛 Hash 设计 MAC 算法。其中泛 Hash 的运算是在二元域上的操作，因此软硬件实现代价小，速度快。

先决条件：
> - 允许使用分组大小为 128-bit 的分组加密算法 $CIPH$；
> - 密钥 $K$；
> - 定义支持的输入、输出长度；
> - 支持的 $tag$ 长度 $t$，与密钥相关。

输入：
> - 初始化向量 $IV$；
> - 明文 $P$；
> - 附加验证数据 $A$。

输出：
> - 密文 $C$；
> - 验证码 $T$。

![](https://github.com/Letty51/pictures/blob/master/GCM-AE.png)

Step 1：使用密钥 $K$ 通过分组加密算法 $CIPH$ 生成 $Hash$ 函数的子密钥 $H = CIPH_K(0^{128})$；\
Step 2：根据初始化向量 $IV$ 得到计数器的预设值 $J_0$；
> - 如果 $IV$ 的长度等于 $96 bits$，则 $J_0 = IV || 0^{31} || 1$；
> - 如果 $IV$ 的长度不等于 $96 bits$，则在 $IV$ 后面填充 $s = 128\lceil len(IV)/128\rceil - len(IV)$ 个 0 ，直至其长度为 $128 bits$ 的整数倍，并在其后增加 $64 bits$ 来表示原始 $IV$ 的长度，最后使用子密钥 $H$ 通过 $GHASH_H$ 函数生成 $J_0 = GHASH_H(IV || 0^{s+64} || [len(iv)]_{64})$。

Step 3：通过 $GCTR_K$ 函数生成密文 $C = GCTR_K(inc_{32}(J_0), P)$，其中 $inc_{32}$ 将 $J_0$ 的后 $32 bits$ 按 $1 mod 2^{32}$ 递增；\
Step 4：在附加验证数据 $A$ 后面填充 $v = 128\lceil \frac{lan(A)}{128} \rceil - len(A)$ 个 0 ，密文 $C$ 后面填充 $u = 128\lceil \frac{lan(C)}{128} \rceil - len(C)$ 个 0 ，直至它们的长度都为 $128 bits$ 的整数倍。将扩充后的附加验证数据 $A$ 添加到密文 $C$ 之前，再在扩充后的密文 $C$ 后增加 $64 bits$ 来表示原始 $A$ 的长度，增加 $64 bits$ 来表示原始 $C$ 的长度。\
Step 5：将 Step 4 得到的字符串使用子密钥 $K$ 通过 $GHASH_H$ 函数生成 $S = GHASH_H(A || 0^v || C || 0^u || [len(A)]_{64} || [len(C)]_{64})$；\
Step 6：使用 Step 2 得到计数器的预设值 $J_0$ 通过 $GCTR_K$ 函数对 Step 5 得到的结果 $S$ 进行加密，并用 $MSB_t$ 函数取其密文的前 $t bits$ 得到最终的验证码 $T = MSB_t(GCTR_K(J_0,S))$。\
Step 7：返回 Step 3 得到的密文 $C$ 和 Step 6 得到的验证码 $T$。

![](https://github.com/Letty51/pictures/blob/master/GHASH.png)

![](https://github.com/Letty51/pictures/blob/master/GCTR.png)