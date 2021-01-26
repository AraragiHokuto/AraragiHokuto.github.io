+++
title = "「译」Telegram，又曰“退下，让老子的数学PhD来！”"
date = "2020-10-28"
categories = ["cryptography"]
tags = ["cryptography", "translation"]
+++

> **译注** 原文见《[Telegram, Aka "Stand back, we have Math PhDs!"](https://web.archive.org/web/20180420061726/http://unhandledexpression.com/2013/12/17/telegram-stand-back-we-know-maths/)》。该文发表于2013年，不少内容或许已经过期，故仅供参考。

---

> **免责声明** 本文如今已十分古老，未必能反应 Telegram 协议于时下的情况。这期间已经有了不少其他研究进展，故而此文不应当被用作你选择加密聊天软件的依据。虽说如此，就我个人而言，我依然认为 Telegram 的加密系统很古怪，并且他们对此的洗地言论站不住脚。如果你想要我推荐一个加密聊天软件：找一套基于 Axolotl/Signal 协议的体系。这套协议设计良好，且已经过大量审查。 Signal 和 WhatsApp，以及一些其他的程序都使用了这套协议。

本文是本系列中第二篇介绍奇怪加密 App 的，对象是最近甚嚣尘上的 [Telegram](https://telegram.org)。

据他们网站所言，Telegram “基于云计算而且大量运用加密”。它有多安全？

> 非常安全。我们的技术基于一套新协议，名为MTProto，由我们自己的专家研发，并运用了经过时间考验的加密算法。目前而言，你在 Telegram 上的消息泄漏的最大风险是你妈从你背后看你屏幕。其他的风险我们都能搞定。

(引自他们的 [FAQ](https://web.archive.org/web/20131210133113/http://telegram.org/faq#security))

嗯，非常安全，他们自己这么说的。

行，那我们就来看看到底多安全。

## 目前可以公开的安全情报

他们的网站上发布了协议的细节。他们其实可以多画些示意图，而不是写一堆纯文字，不过现在这样也算能读。还有个[拿 Java 写的开源协议实现](https://web.archive.org/web/20180420061726/https://github.com/ex3ndr/telegram-mt/)。这算个优点。

关于他们的团队（嗯，我还记得我说过不搞诉诸人身，但毕竟他们一直在吹这点）：

> Telegram 背后的团队，由 Nikolai Durov 牵头，共计六个 ACM 冠军组成，其中半数都是数学PhD。他们花了两年时间来设计当前的 MTProto。虽说学历未必代表能力，但至少能说明这套协议是海量专家精确计算的结果（原文：result of thougtful and prolonged work of professionals）

（来自 [Hacker News](https://web.archive.org/web/20180420061726/https://news.ycombinator.com/item?id=6916860)）

他们不是密码学家，但他们有数学学术背景。好耶！

那么，整个体系架构长什么样？**基本上就是世界各地放几台服务器，在客户端之间转发信息。** 身份验证只做在客户端和服务端之间，而不是客户端之间点对点验证。客户端和服务端之间有加密，但用的不是TLS（而是一些自制的协议）。加密可以端对端进行，但因为没有身份验证，所以服务器可以进行中间人攻击。

本质而言，他们的威胁模型就只是“信任服务器”。在公网上传输的内容可能被安全加密了，但我们毕竟无从了解他们的服务器间通信细节，也无法得知他们的数据存储方案。以今天的眼光看来，这套东西既没意思，又昭示着不安全和粗心。类似的系统可以参考 Lavabit 或者 iMessage。**这套系统并不能阻止执法部门的监听或者服务器渗透。更糟糕的：你都没法检测到和你聊天对象之间的中间人攻击。**

我可以就此收手，但这样就没意思了。真正的乐子其实常在他们的加密设计里。这些设计粗看起来似乎合理，但对加密算法的选择既奇怪又不安全，而且每一个设计都无谓地选择了最复杂的那种。

## 网络协议

网络协议有两个阶段：密钥交换和通信。

密钥交换过程会向服务器注册一个设备。为此他们自己搞了一套协议，理由是 TLS 太慢而且太复杂。有一说一，确实：TLS 需要在客户端和服务器间跑**两**个来回 (round trip) 才能完成密钥交换。而且还需求一个 x509 证书，以及一套 RSA 或者 DSA 这种公钥加密算法，最终还需要走一趟 Diffle-Hellman 密钥交换算法。

[Telegram 大幅简化了这个过程（迫真）](https://web.archive.org/web/20180420061726/http://core.telegram.org/mtproto/auth_key)，只需要**三**个 RT，用上 RSA、 AES-IGE（某个没其他人用的奇怪加密模式），以及 DH 密钥交换，同时还有一套工作量证明（客户端得因数分解个数字，估计是个 DoS 防护措施）。另外，他们还用了一套土制算法来从服务器和客户端生成的 nonce 导出 AES 密钥和 IV（ `server_nonce` 在通信过程中是明文传输的）：

```
key = SHA1(new_nonce + server_nonce) + substr(SHA1(server_nonce + new_nonce), 0, 12);
iv = substr(SHA1(server_nonce + new_nonce), 12, 8)+ SHA1(new_nonce + new_nonce) + substr (new_nonce, 0, 4);
```

注意 AES-IGE 是不带认证的。所以他们自行实现了完整性验证，方法是简单地对原文取个 SHA1 （没错，甚至都没用个正经的 MAC），然后把 Hash 结果和原文一起加密（没错，这就是个迫真 MAC-then-Encrypt）。

最终的 DH 交换会创建一个储存在服务端和客户端（搞不好还是明文存储）的认证密钥。

我实在难以理解他们为什么要搞一套这么复杂的协议。他们本来可以这么搞：客户端生成一对密钥，用服务端的公钥加密客户端的公钥，和 nonce 一起发给服务端，然后服务端把用客户端公钥加密过的 nonce 发回来。简单有效。而且这样还可以向客户端提供公钥，实现端对端加密。

至于通信阶段：他们用了一套服务端加盐、消息 ID 和消息计数器的组合来阻止重放攻击。有趣的是，他们的消息密钥是用 SHA1 值的低128位组成的。这个消息密钥**用明文传输**，所以如果你监听到了消息头，那么此处应有巨大多信息泄漏。

用来加密消息的 AES 密钥（还是 IGE 模式下的 AES）是这么生成的：

如下是从 `auth_key` 和 `msg_key` 计算 `aes_key` 和 `aes_iv` 的算法：

```
sha1_a = SHA1(msg_key + substr(auth_key, x, 32));
sha1_b = SHA1(substr(auth_key, 32+x, 16) + msg_key + substr(auth_key, 48+x, 16));
sha1_с = SHA1(substr(auth_key, 64+x, 32) + msg_key);
sha1_d = SHA1(msg_key + substr(auth_key, 96+x, 32));
aes_key = substr(sha1_a, 0, 8) + substr(sha1_b, 8, 12) + substr(sha1_c, 4, 12);
aes_iv = substr(sha1_a, 8, 12) + substr(sha1_b, 0, 8) + substr(sha1_c, 16, 4) + substr (sha1_d, 0, 8);
```

其中，在客户端给服务端的消息中 \\(x = 0\\)，而服务端给客户端的消息中\\(x = 8\\)。

<s>因为 `auth_key` 是永久的，而消息密钥只依赖于服务端提供的盐（24小时一换）、会话 ID（估计是永久的，不过可能会被服务端忘掉）以及消息的开始部分，可能很多消息都会生成同样的消息密钥。<strong>没错，很多消息会共享同样的 AES 密钥和 IV。</strong></s>

**编辑：据 Telegram 的评论所言，每条消息的 AES 密钥和 IV 都是不同的。但还是那句话，根据消息内容来生成密钥和 IV 是个及其糟糕的设计。这二者必须从 CSPRNG 生成，和加密内容保持无关。**

**编辑2：[新的协议示意图](https://core.telegram.org/img/mtproto_encryption.png)澄清了密钥是由一个很弱的密钥导出算法推出来的，而且部分内容直接明文传输。这里大概有不少可以拿统计分析搞事情的地方。**

<s><strong>编辑3：行吧，如果你把同一条消息发两遍（在一天当中，因为服务端生成的盐能活24小时），密钥和 IV 会是一样的，而且密文也会是一样的。这是个正儿八经的安全漏洞。通常的解决方式是常换 IV （即使是 WEP 这样的破烂协议都知道这么干）而且常换密钥（也即 TLS 或者 OTR 里的前向安全性）。</strong></s>消息原文里包含一个（时间相关）的消息 ID 以及自增的消息序列号，所以客户端不会接受重放的消息，或者太旧的消息。

**编辑4：有人在[端对端聊天](https://web.archive.org/web/20180420061726/http://web.archive.org/web/20131220000537/https://core.telegram.org/api/end-to-end)中找到了一个[漏洞](https://web.archive.org/web/20180420061726/http://habrahabr.ru/post/206900/)。从 DH 算法生成出来的密钥会和服务端提供的 nonce 结合：`key = (pow(g_a, b) mod dh_prime) xor nonce`。这样，服务端就可以通过在两个客户端间生成同样的密钥，以此进行中间人攻击，让密钥验证失效。Telegram 已经更新了协议描述，打算修复这个漏洞。（这个 nonce 的引入是为了修复某些移动设备上的 RNG 问题）。**

> **译注** 我觉得这个洗地站不住脚。RNG 缺陷可以有多种方式解决：比如 EdDSA 演示实现中将128个随机字节 Hash 到 32个，以此避免暴露 RNG 原始输出防止逆推 RNG 状态。TG 这个解决方法在我看来非蠢即坏——在加密相关问题上我倾向于假定后者。

正经说，我从来没见过有人用 MAC 来生成加密用密钥的。就算你想埋后门，也不用埋得这么明显啊……

结论：**快跑**。里面没有什么全新发明，而且他们还通过自行组合 RSA、AES-IGE、纯 SHA1 完整性验证、MAC后加密、和自制 KDF 引入了一堆自制安全漏洞。比起 Telegram，你更应该用个有名的而且被审计过的协议，比如说 OTR （可用于 IRC 及 Jabber ），或者 TextSecure 的 Axolotl key ratcheting。