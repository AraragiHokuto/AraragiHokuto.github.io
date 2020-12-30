+++
title = "思路整理：对于 illumos 上 SO_REUSEPORT 补丁的笔记"
date = "2020-12-26"
categories = ["operating-system", "networking"]
tags = ["operating-system", "networking", "illumos", "personal-note"]
+++

把目前世界上能找到的最正统的 SysV 的源代码扒过来，对着它的网络栈一通改动，还试图把改动作为补丁提交回去——这大概是我今年干出的事情里面最不知天高地厚的一件了。毕竟一头扎进有着数十年历史的代码堆里面本身就是需要勇气的一件事，更不用我改动的地方还有着炸掉世界各地的所有相关系统的潜力——在 IP 模块里面搞出个死锁干掉所有跑着的 illumos 系统这种事情想想就刺激。

不过身为软件工程师，总得有梦想，而上面提到的那些心理障碍在让自己的代码跑在世界各地的服务器上的浪漫面前显然无足挂齿。所以四月的时候我还是头铁了上去，而且还[真的搞出了一份 patch](https://code.illumos.org/c/illumos-gate/+/463)。截至本文成文之日，补丁仍在热烈 review 当中（显然，这种变动需要慎之又慎）；但我还是决定先花点时间，整理一下我自己的思路和所得，以方便后来者——包括以后的我自己。

## 背景资料

### illumos

[illumos](https://en.wikipedia.org/wiki/Illumos) 这个名字的认知度意外地不高——不少我见过的人都向我表示根本没听说过。不过如果我把它的原名拿出来，那恐怕是如雷贯耳：[OpenSolaris](https://en.wikipedia.org/wiki/OpenSolaris)。

虽说 Solaris 这个名字的所有权现在处于 Oracle 手里，官方意义上的 OpenSolaris 项目也早已停止，但鉴于 illumos 社区内有大量的 Sun 前员工，而且 [Oracle 已经事实上放弃了 Solaris](https://www.infoq.com/news/2017/09/solaris-sunsets/)，我觉得将 illumos 视作 Solaris 的正统后继而非 fork 没什么问题。

考虑到如今 illumos 的知名度，以及 Solaris 身为正统 SysV 的名号，或许很多人会先入为主地认为这不过是个沉浸在自己旧日荣光之中的腐朽操作系统。

远非如此。

先不说 illumos 作为 OpenSolaris 的二进制兼容物这点，如今 illumos 的应用也远没到已死的程度—— 身为三星子公司的 joyent 目前仍然运行着由 SmartOS 支撑的云服务，且正积极地为 illumos 社区回馈贡献；我也时常能见到在线上运行着 OmniOS 服务器的管理员来到社区讨论（我自己也可算作其中一员：我在 Linode 上跑着一台 OmniOS VPS）。而社区的热度则可以从我挤满了未读消息的 IRC 和塞了几千封未读邮件的邮件列表中体现。

相比于 Linux 这种主流 OS，illumos 确实小众；但这不代表它缺乏活力。而能为这样一个血统高贵（虽说 Unix 的血统不足惜）却又生机勃勃的 OS 贡献代码，不也是一种浪漫吗？

题外话：“菜是原罪”风气流行的开源社区，在对待新人的态度上一贯没什么好风评（就我个人而言，对此我可以理解）；所以身为一个名不见经传还操着残废英语的萌新，在初进社区的时候我其实是做好了被喷个狗血淋头的心里准备的。

完全没必要。illumos 社区里的友好和耐心简直令我感动。所以我在这里自愿地打个广告：如果有人希望为开源项目做点贡献的话，考虑一下 illumos 如何？从 bite-size bug 到各种高端变动，总有一款适合你。

### SO_REUSEPORT

[我们谈论的这个选项最早引入于 Linux 3.9](https://github.com/torvalds/linux/commit/c617f398edd4db2b8567a28e899a88f8f574798d)。该选项可以为 TCP 和 UDP socket 提供一套内核负载均衡功能：多个 socket 可以绑定到同一个 IP 地址上，而内核会将新入连接/数据在各个 socket 间分发。这个选项对于多进程服务器（例如 nginx）有着相当大的意义：进程之间的负载均衡可以被转交给内核处理，而无需上层应用再实现一个 accept 锁；并且惊群问题可以得到更为优雅及彻底的解决（因为新入连接在内核网络栈里就已经被分配到特定进程了，所以从一开始就不会唤醒所有进程）。

[DragonflyBSD 随后也跟进了这个变动](https://gitweb.dragonflybsd.org/dragonfly.git/commitdiff/740d1d9f7b7bf9c9c021abb8197718d7a2d441c9)，其实现语义与 Linux 基本一致，不过我印象中在监听 socket 数量变化的时候略有不同：Dragonfly 上增加或者减少监听的 socket 的时候不会产生 reset。

[FreeBSD 现在也已经引入了这个选项](https://reviews.freebsd.org/D11003)，但将选项名改成了 SO_REUSEPORT_LB。其理由主要有二：一是 FreeBSD 社区认为 Linux 的行为是在滥用 SO_REUSEPORT 的语义（毕竟 FreeBSD 一贯有点学院派作风），二是该平台上本来就有了 SO_REUSEPORT 选项，所以通过改名增加新选项来保持原选项的行为一致性。

就我个人而言，其实我觉得 FreeBSD 的做法从理论层面上更为优雅；但我们毕竟不是真空中的球形鸡。Linux 的现有语义的接受度显然广泛得多，而且 Solaris/illumos 上原本并没有 SO_REUSEPORT 选项，所以也不存在破坏向前兼容性的问题。因而在社区内讨论过后，我们决定跟从 Linux 的命名。

## 动手之前：明确需求

在动手做事情之前，我们首先需要明确到底该做什么——这听起来像是废话，但却经常被人忘掉（软件工程上尤其如此）。所以在真正动手写代码之前，我们先要明确我们所要遵从的语义。

## 动手之前：现有成果

在我着手自己的实现之前，我打算先拿出些时间看看前人给我们留下的现有成果。