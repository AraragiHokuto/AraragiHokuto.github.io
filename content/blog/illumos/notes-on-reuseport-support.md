+++
title = "思路整理：对于 illumos 上 SO_REUSEPORT 补丁的笔记"
date = "2020-12-26"
categories = ["operating-system", "networking"]
tags = ["operating-system", "networking", "illumos", "personal-note"]
+++

把目前世界上能找到的最正统的 SysV 的源代码扒过来，对着它的网络栈一通搅合，还试图把改动作为补丁提交回去——这大概是我今年干出的事情里面最不知天高地厚的一件了。毕竟一头扎进有着数十年历史的代码堆里面本身就是需要勇气的一件事，更不用我改动的地方还有着炸掉世界各地的所有相关系统的潜力——在 IP 模块里面搞出个死锁干掉所有跑着的 illumos 系统这种事情想想就刺激。

不过身为软件工程师，总得有梦想，而上面提到的那些心理障碍在让自己的代码跑在世界各地的服务器上的浪漫面前显然无足挂齿；所以四月的时候我还是头铁了上去，而且还[真的搞出了一份 patch](https://code.illumos.org/c/illumos-gate/+/463)。截至本文成文之日，补丁仍在热烈 review 当中（显然，这种变动需要慎之又慎）；但我还是决定先花点时间，整理一下我自己的思路和所得，以方便后来者——包括以后的我自己。

## 背景资料

### illumos

[illumos](https://en.wikipedia.org/wiki/Illumos) 这个名字的认知度意外地不高——不少我见过的人都向我表示根本没听说过。不过如果我把它的前身拿出来，那恐怕是如雷贯耳：[Solaris](https://en.wikipedia.org/wiki/Solaris_(operating_system))

虽说 Solaris 这个名字的所有权现在处于 Oracle 手里，官方意义上的 OpenSolaris 项目也早已停止，但鉴于 illumos 社区内有大量的 Sun 前员工，而且 [Oracle 已经事实上放弃了 Solaris](https://www.infoq.com/news/2017/09/solaris-sunsets/)，我觉得将 illumos 视作 Solaris 的正统后继而非 fork 没什么问题。

考虑到如今 illumos 的知名度，以及 Solaris 身为正统 SysV 的名号，或许很多人会先入为主地认为这不过是个沉浸在自己旧日荣光之中的腐朽操作系统。

远非如此。

先不说基于 illumos 的 OpenIndiana 发行版作为 OpenSolaris 的二进制兼容物有其不可替代的应用场景这点，如今 illumos 的应用也远没到将死的程度—— 身为三星子公司的 joyent 目前仍然运行着由 SmartOS 支撑的云服务，且正积极地为 illumos 社区回馈贡献；我也时常能见到在线上运行着 OmniOS 服务器的管理员来到社区讨论（我自己也可算作其中一员：我在 Linode 上跑着一台 OmniOS VPS）。而社区的热度则可以从我挤满了未读消息的 IRC 和塞了几千封未读邮件的邮件列表中体现。

相比于 Linux 这种主流 OS，illumos 确实小众；但这不代表它缺乏活力。而能为这样一个血统高贵（虽说 Unix 的血统不足惜）却又生机勃勃的 OS 贡献代码，不也是一种浪漫吗？

题外话：“菜是原罪”风气流行的开源社区，在对待新人的态度上一贯没什么好风评（就我个人而言，对此我可以理解）；所以身为一个名不见经传还操着残废英语的萌新，在初进社区的时候我其实是做好了被喷个狗血淋头的心里准备的。

完全没必要。illumos 社区里的友好和耐心简直令我感动。所以我在这里自愿地打个广告：如果有人希望为开源项目做点贡献的话，考虑一下 illumos 如何？从 bite-size bug 到各种高端变动，总有一款适合你。

### SO_REUSEPORT

[SO_REUSEPORT 这个选项最早引入于 Linux 2.9](https://github.com/torvalds/linux/commit/c617f398edd4db2b8567a28e899a88f8f574798d)。该选项可以为 TCP 和 UDP socket 提供一套内核负载均衡功能：多个 socket 可以绑定到同一个 IP 地址上，而内核会将新入连接/数据在各个 socket 间分发。这个选项对于多进程服务器（例如 nginx）有着相当大的意义：进程之间的负载均衡可以被转交给内核处理，而无需上层应用再实现一个 accept 锁；并且惊群问题可以得到更为优雅及彻底的解决（因为新入连接在内核网络栈里就已经被分配到特定进程了，所以从一开始就不会唤醒所有进程）。

[DragonflyBSD 随后也跟进了这个变动](https://gitweb.dragonflybsd.org/dragonfly.git/commitdiff/740d1d9f7b7bf9c9c021abb8197718d7a2d441c9)，其实现语义与 Linux 基本一致，不过我听说在监听 socket 数量变化的时候略有不同：Dragonfly 上增加或者减少监听的 socket 的时候不会产生 reset（这点我没亲自测试）。

[FreeBSD 则在版本11引入了这个选项](https://reviews.freebsd.org/D11002)，但将选项名改成了 SO_REUSEPORT_LB。其理由主要有二：一是 FreeBSD 社区认为 Linux 的行为是在滥用 SO_REUSEPORT 的语义（毕竟 FreeBSD 一贯有点学院派作风），二是该平台上本来就有了 SO_REUSEPORT 选项，所以通过改名增加新选项来保持原选项的行为一致性。

就我个人而言，其实我觉得 FreeBSD 的做法从理论层面上更为优雅；但我们毕竟不是真空中的球形鸡。Linux 的现有语义的接受度显然广泛得多，而且 Solaris/illumos 上原本并没有 SO_REUSEPORT 选项，所以也不存在破坏向前兼容性的问题。因而在社区内讨论过后，我们决定跟从 Linux 的命名。

## 动手之前：明确需求

在动手做事情之前，我们首先需要明确到底该做什么——这听起来像是废话，但却经常被人忘掉（软件工程上尤其如此）。所以在真正动手写代码之前，我们先要明确选项的语义。

不幸地，“明确语义”从来都不是 Linux 所擅长的东西：原 commit 中不仅提交信息语焉不详，代码中也没几行注释；虽说 "Good Code Documents Itself"，但 Linux 的网络栈显然还没达到这里的 Good Code 的程度，而且有些背景信息毕竟不是能从代码上一眼看出来的。但无论如何，we are where we are，所以不足的部分只好我们自己写程序测出来。

具体的测试代码我就不展示了（过于 trivial，读者应当有能力根据我的描述自行实现），总之我整理出来的语义如下：

- 共通
    - 若要允许复用，绑定到同一地址的每个 socket 都要**在 bind() 之前**开启该选项；在 bind() 之后再开启该选项不会生效。
    - 关闭选项同理：在 bind() 之后再关闭该选项并不阻止其他 socket 绑定到同一地址，也不会让该 socket 不再接受负载均衡派发来的连接。
    - SO_REUSEPORT **并不隐含** SO_REUSEADDR。二者虽然语义类似但是行为不会互相影响。
- TCP
    - 在某一地址上绑定的 socket 数量发生变化（新 socket 绑定，或旧 socket 停止监听）时，各个平台对于半开连接的处理行为不同（我没仔细测试）：
        - Linux 在两种情况下都有可能对半开连接产生 RST。原因我没深究，不过 commit message 里提了一句，似乎是共享数据结构产生的问题。
        - DragonflyBSD 上据称两种情况都不会产生 RST。我没测试。
        - FreeBSD 上新 socket 绑定的情况我没测试，不过关闭旧 socket 的时候会有部分半开连接被 RST。
- UDP Unicast
    - 之前 SO_REUSEADDR 同样允许多个 socket 绑定到同一个地址上，但没有负载均衡行为：所有的新入数据都由新绑定的 socket 接收。SO_REUSEPORT 与该选项语义有重叠，但允许负载均衡。二者都打开的时候呢？Linux 没提，我测试的结论是依然会有负载均衡行为。
    - 因为负载均衡通过对 IP 四元组取 hash 决定接收者，所以在绑定的 socket 恒定的情况下，特定地址发来的信息总会被转发到同一客户端上。我不确定该行为是否有保证（也不觉得应当依赖于这个行为），但至少在目前所有的 Linux 版本上都适用。
- UDP Multicast/Broadcast
    - 原 commit 完全没提多播/广播情况下的处理。测试结论是 SO_REUSEPORT 此时行为和 SO_REUSEADDR 一致：所有 socket 都会收到一份数据的拷贝。
- 其他协议
    - 截至本文发布时，其他协议全部不支持该选项。

当然以上的结论里面其实漏掉了一个 corner case：若 SO_REUSEPORT 和 SO_REUSEADDR 同时在 UDP socket 上开启，但 bind() 时 SO_REUSEPORT 理当失败而 SO_REUSEADDR 理当成功（例如，新 socket 与现有 socket 所属进程具有的 effective UID 不同，或者复用连接数达到了 SO_REUSEPORT 允许的上限），那么绑定应当成功还是失败？但这个情况实在过于牛角尖，而且定义这个行为的收益实在太少（什么情况下会有人需要依赖这个行为？），再加上或许这里保持 flexible 反而更有利于未来的进一步实现，所以我决定不测试也不明确这个行为。

明白了我们要做什么之后，来看看我们手上现有的东西：

## 动手之前：底层架构

### Solaris 的网络接口，以及 TLI

illumos 脱胎于 OpenSolaris，而后者脱胎于 Solaris 10。Solaris 10 的 FireEngine 网络栈实现也就一直保留到了现有的 illumos 代码。

具体的网络栈实现在《Solaris Internals: Solaris 10 and OpenSolaris Kernel Architecture》一书中有详细的剖析，而且仔细分析网络栈实现远超本文的涵盖范围，所以这里我不会讲得太详细，只稍微提几句，好让读者诸君有个概念。（不过其实那本书还挺难买：英文原版我只能搞到一本二手的，好在品相不错。）

与 BSD 系的 Unix 不同，身为 SysV 系操作系统（而且大概最正统的 SysV）的 Solaris，其网络栈的原生 API 并非 BSD socket，而是基于 STREAMS 的 TLI (Transport Layer Interface) API；而在 VFS 层面上则通过一个 sockfs 模块将用户程序的 BSD socket 调用转发到 TLI 层面上的调用。

从理论层面上来说 STREAMS 其实算是个比较先进的模型：每个连接都至少有一个 STREAM 头以及一个底层设备，STREAM 头对接用户程序，而底层设备对接网卡驱动；在二者之间可以动态插入多个 STREAM 模块以构成一个 STREAM 链，该链中每个模块都有单独的读队列及写队列，以此与其相邻的模块“首尾相接”。故一个典型的 TCP 连接应当具有这样的结构：

{{<mermaid>}}
graph TD;
A["STREAM 头"] --> |写| B["TCP 协议实现"] --> |写| C["IP 协议实现"] --> |写| D["STREAM 底层设备"];
D --> |读| C --> |读| B --> |读| A;
{{</mermaid>}}

这结构理论上更符合 OSI 的七层模型，在刚出来之时用着也还不错；动态构建 STREAM 链确实开销较大，但在 Web 时代之前大部分连接都是长连接，所以平均下来这点开销基本可以忽略。

**在 Web 时代之前。**

Web 时代的到来使得情况发生了巨大转变：之前的连接主要为长连接的假设不再成立，大部分 HTTP 连接都是高频短连接；这点使得 STREAMS 创建连接成本较高的缺点逐渐凸显。

我们需要换个做法。

### Solaris 10 的 FireEngine

Solaris 10 引入了一套重新实现的网络栈：FireEngine。

即使以今天的眼光来看，FireEngine 的架构也相当先进。和不少操作系统上的网络实现不同，FireEngine 是完全并行化的：所有连接都可以并行处理，无关连接之间几乎不会产生干扰。

另外，为了解决创建连接成本较高的问题，整个 TCP&UDP/IP 网络栈都被整合成了一个 STREAM 模块，上可直接接 STREAM 头，下可直接接底层设备。因为不需要再动态组装 STREAM 链，创建连接的效率被大幅提高了。

至于并行化，则通过如下的机制实现：

引入 squeue 作为同步机制；每个 squeue 与 CPU 绑定，而每个连接则与 squeue 绑定。此机制用来保证线程对连接的独占访问，以及提升数据的本地关联性。

squeue 实现于 TCP/IP 模块内，但并非只保护 TCP/IP 模块本身，而是保护从 ipclassifier 到 sockfs 的整个链路——换言之，数据包从 ipclassifier 出来之后，到达用户程序的所有路径都通过同一个机制同步。

对于 TCP 连接，调整了 TCP 握手的处理方式：FireEngine 的实现在接收到 SYN 时会立即创建一个新的 TCP 连接（称为 eager），之后剩余的握手过程全部都在 eager 的回路中处理，而非在监听的 socket 上处理。

在连接与网卡驱动之间的部分是 ipclassifier，其作用是将数据包分发到对应的 squeue 中供连接处理。几乎只有这部分涉及跨 CPU 数据传递（将数据包从接收中断的 CPU 发送到对应连接所绑定的 CPU）。

这套设计使得 FireEngine 在架构上就实现了并行化，而无需在实现时再使用大量同步原语搞奇技淫巧。更棒的是，这种在早期便将数据包 fanout 上对应连接的设计正适合实现 SO_REUSEPORT。

## 动手之前：现有成果

在正式动手之前，我们还有最后一步要做：看看前人都为我们留下了些什么。

其实 illumos 上的 SO_REUSEPORT 实现并非是我从零开始写就的。joyent 在 几年前就已经引入了基本的 SO_REUSEPORT 支持[^1][^2]，目的是让 LX-Branded Zones （也即 Solaris 的 Linux 二进制兼容层）中的程序在使用 SO_REUSEPORT 的时候不至于直接炸掉。当然也因为如此，所以他们只实现了最最基本的行为；负载均衡语义则完全没有实现。

[^1]: https://github.com/joyent/illumos-joyent/commit/310084386497b66d8f5b66f4887dfe1a57e1cb94
[^2]: https://github.com/joyent/illumos-joyent/commit/4a663ad4eabd38f9c6397c5205cc12f083b474ca

不过无论如何，这两个补丁还是给我们打下了一个可以着手的基础，以及指明了修改的方向。所以作为正式分析我们自己的补丁之前的最后一步，我们还要仔细看看这两个提交。提交的源码和变动我不会直接贴在下面，所以请读者准备好浏览器窗口。

- tcp.h, tcp_impl.h

一点经验：读 C 代码的时候先看头文件总没错，因为这里是定义数据结构和接口的地方。

这点对于我们正讨论的几个提交也适用。这两个头文件的主要变动无非两点：

- 定义`struct tcp_rg_s`，用于表示由绑定在同一个地址上的所有`tcp_t`（表示一个 TCP 连接的数据结构）组成的 group，以此建立在这些`tcp_t`之间的关联；以及该结构相关的函数接口。
- 往`tcp_t`里增加一个`tcp_rg_bind`引用，指向其所属的`strut tcp_rg_s`组。

意义一目了然，我不做过多解释。

- tcp_opt_data.c

该文件负责实现 setsockopt 的底层操作，所以也是新选项在执行过程中最先碰到的变动所在。注意两个 commit 中都有对该文件的变动。

此处我们增加的操作全都是关于 SO_REUSEPORT 的行为。具体如下：

- 检查给定 socket 是否已经 fallback 到了 STREAMS。如果是，拒绝开启选项（fallback 到 STREAMS 的 socket 不支持该行为）。
- 如果是开启选项，那么在此时分配初始`struct tcp_rg_s`。我们之后在`bind`中里将会使用`tcp_rg_bind`的值决定是否允许重用，而非使用现有 socket 的选项值，这是因为在绑定 socket 之后再关闭该选项不应当阻止重用的可能。
- 如果是关闭选项，那么进行一系列检查。这里的各种状态比较复杂，但总之都遵循一个原则：如果当前 group 会在解除绑定的时候被释放，那么这里保留 group；否则释放 group。

这里因为我们提前分配了`tcp_rg_bind`组，导致状态检查有些复杂。不过没关系，总之我们先接着往下看。

- tcp_bind.c

这里的改动看上去很多，不过其实不可怕。大部分的变动都是`tcp_rg_*`族函数的实现，用于操作`struct tcp_rg_s`；真正对网络栈的改动其实无非如此：

在`bind`时，若找到冲突的 socket，检查该 socket 是否属于某个`tcp_rg_bind`组。若是，则说明该冲突 socket 在绑定时开启了 SO_REUSEPORT，因此将我们的 socket 也加入这个组，并继续。否则正常返回 EADDRINUSE。

- ipclassifier.c

这里面的改动主要是将`IPCL_HASH`族宏替换成了等价的函数实现，并且在将 socket 从 hash 中移除的时候处理`struct tcp_rg_s`的释放。实质 trivial，相信读者能够自行理解。

剩余的其他文件的改动一目了然，不必多言。

## 正式下手

完成了以上的准备之后，我们需要做的工作便自然浮现了出来。整理一下便是：

- 将原本只适用于 TCP 的选项扩展到 UDP 上
- 实现负载均衡

这样看来其实剩下的工作并不多。至于是否真是如此，我们写写试试便可得知。

### 选项适用范围扩展

原补丁中通过添加`struct tcp_rg_s`，并在`tcp_t`中添加相关引用实现了 TCP 上的 SO_REUSEPORT 选项支持；但显然，“绑定在同一地址上的一组 socket” 这个概念不仅仅适用于 TCP。要实现 UDP 上的支持，将这个概念扩展到 UDP 协议栈中似乎是个不错的着手点。

实现这点最简单的办法，当然是直接复制一份代码到 UDP 的实现中，然而通往屎山的道路往往就由 C&P 式复用铺就，所以这显然不是好主意。更好的办法是把 TCP 和 UDP 连接放在同一个更高层的抽象之下，在这个抽象的层面上实现如今`struct tcp_rg_s`的概念。那么现在的网络栈中有没有合适的抽象呢？

不难找。ipclassifier.h 里面定义了一个`conn_t`，从名称上就能猜出这个结构体表示的正是无关具体协议的“连接”这一概念。各处实现中对其的用法也能证实这点。

此时我们的做法就很明确了：我们将`struct tcp_rg_s`的定义移到 ipclassifier.h 中（同时顺手将其改名为`conn_rg_t`以保持命名风格统一），然后将原本在`tcp_t`中的引用挪到`conn_t`中。原`tcp_rg_*`系列接口也做同样处理。

下一步我们需要去 udp_opt_data.c 中为 UDP 增加相关选项的定义。一行改动即可解决。

不过在大功告成之前，我们还有一件事情要处理：tcp_opt_data.c 中在开关选项是似乎有比较复杂的逻辑，这些逻辑是否在 UDP 上也有必要？

要回答这个问题，我们首先需要分析这里的具体逻辑是什么。这里我觉得逐行分析大概会更清楚，所以我会把`tcp_set_reuseport`的所有代码全都贴过来。

```c
/*
 * Set a TCP connection's participation in SO_REUSEPORT.  This operation is
 * performed under the protection of the squeue via tcp_setsockopt.
 * The manipulation of tcp_rg_bind, as part of this operation, is subject to
 * these constraints:
 * 1. Prior to bind(), tcp_rg_bind can be set/cleared in tcp_set_reuseport
 *    under the protection of the squeue.
 * 2. Once the connection has been bound, the tcp_rg_bind pointer must not be
 *    altered until such time as tcp_free() cleans up the connection.
 * 3. A connection undergoing bind, which matches to a connection participating
 *    in port-reuse, will switch its tcp_rg_bind pointer when it joins the
 *    group of an existing connection in tcp_bindi().
 */
 ```

一上来我们就被砸了大段的逻辑解释。这里的注释主要说明的是对`conn_rg_t`进行操作时需要遵循的原则。具体而言：

- 在 socket 绑定之前，`conn_rg_t`可以随选项开关正常分配/去配。
- 一旦 socket 已被绑定，那么直到`tcp_free()`开始对连接数据进行清理为止，都不能再改动`conn_rg_bind`指针。这是因为 Linux 语义下 socket 绑定后再修改该选项并不阻止新的端口重用地址，而我们在绑定时判断是否重用的依据便是 `conn_rg_bind` 的值。
- 如果 socket 在绑定时找到了之前已经绑定的 socket，那么将其`conn_rg_bind`指针切换到现有的`conn_rg_t`。也即，这种情况下我们放弃已经分配的新组，而加入现有的组。

具体实现如下。

 ```c
static int
tcp_set_reuseport(conn_t *connp, boolean_t do_enable)
{
	tcp_t *tcp = connp->conn_tcp;
	struct tcp_rg_s *rg;

    /* ... */

	if (tcp->tcp_state <= TCPS_CLOSED) {
		return (EINVAL);
	}
```

显然在已经关闭了的 TCP 连接上开启该选项毫无意义。

```c
	if (connp->conn_reuseport == 0 && do_enable) {
		/* disabled -> enabled */
		if (tcp->tcp_rg_bind != NULL) {
			tcp_rg_setactive(tcp->tcp_rg_bind, do_enable);
		} else {
			/*
			 * Connection state is not a concern when initially
			 * populating tcp_rg_bind.  Setting it to non-NULL on a
			 * bound or listening connection would only mean that
			 * new reused-port binds become a possibility.
			 */
			if ((rg = tcp_rg_init(tcp)) == NULL) {
				return (ENOMEM);
			}
			tcp->tcp_rg_bind = rg;
		}
		connp->conn_reuseport = 1;
```

开启选项时检查是否已经加入现有组。是的话我们只更新一下组内簿记信息，否则我们在此时分配一个新组。

```c
	} else if (connp->conn_reuseport != 0 && !do_enable) {
		/* enabled -> disabled */
		ASSERT(tcp->tcp_rg_bind != NULL);
		if (tcp->tcp_state == TCPS_IDLE) {
			/*
			 * If the connection has not been bound yet, discard
			 * the reuse group state.  Since disabling SO_REUSEPORT
			 * on a bound socket will _not_ prevent others from
			 * reusing the port, the presence of tcp_rg_bind is
			 * used to determine reuse availability, not
			 * conn_reuseport.
			 *
			 * This allows proper behavior for examples such as:
			 *
			 * setsockopt(fd1, ... SO_REUSEPORT, &on_val...);
			 * bind(fd1, &myaddr, ...);
			 * setsockopt(fd1, ... SO_REUSEPORT, &off_val...);
			 *
			 * setsockopt(fd2, ... SO_REUSEPORT, &on_val...);
			 * bind(fd2, &myaddr, ...); // <- SHOULD SUCCEED
			 *
			 */
			rg = tcp->tcp_rg_bind;
			tcp->tcp_rg_bind = NULL;
			VERIFY(tcp_rg_remove(rg, tcp));
			tcp_rg_destroy(rg);
```

在未绑定的 socket 上关闭选项时，我们需要去配之前开启选项时分配的组结构。

```c
		} else {
			/*
			 * If a connection has been bound, it's no longer safe
			 * to manipulate tcp_rg_bind until connection clean-up
			 * during tcp_free.  Just mark the member status of the
			 * connection as inactive.
			 */
			tcp_rg_setactive(tcp->tcp_rg_bind, do_enable);
		}
		connp->conn_reuseport = 0;
	}
	return (0);
}
```

否则如前所述，我们不回收该组，而只更新簿记信息。

这套逻辑可以在 UDP 上大致复刻。但我们还可以做的更深入一点：

注意，之所以我们需要这么一大堆逻辑，是因为我们在开启选项时便分配了组的结构，而去配组结构的时候需要分析 socket 是否已经绑定。另外还有一点：在 socket 被绑定之前，这个分配的结构实际上根本没有被使用。

既然如此，我们何不把组的分配延迟到 bind 之时呢？假设我们这么做，那么这里需要的逻辑会变成这样：

- 开启选项只标记一个开关，组分配延迟到 bind 时再处理
- 关闭选项时，如果 socket 尚未绑定，我们只需要关闭开关，不需要去配组，因为组并没有被分配
- 关闭选项时，如果 socket 已绑定，我们只需要关闭开关，不需要去配组，因为此时组不当被去配

综合一下，我们所要做的事情就只剩下标记开关了；逻辑瞬间被大幅简化。

看上去不错，所以我们就这么干吧：

我们把所有的复杂逻辑全部去掉，只留标记开关一项，在 UDP 中也同样。因为两边的操作完全一致，所以我们不必在两边协议栈中重复实现，而可以把共通实现直接提取到 conn_opt.c 中处理。开启选项相关的修改便就此大功告成。

### 负载均衡：上

负载均衡这个行为，虽然说起来只有四个字，但仔细一想倒还挺吓人。我们要如何把数据包及连接负载均衡到同一地址的不同连接上，还要保证不重复，且不发到不同地址的连接上呢？

要解决这个问题，首先我们需要回顾一下 illumos 的网络栈的架构：一个数据包从网卡驱动出来，一路送到用户程序，中间具体经历了哪些步骤？

- 数据包离开网卡驱动进入协议栈以后，第一个关卡是 ipclassifier。它负责将数据包分发到对应的连接上。具体而言，它会按照 IP 四元组（对于已经建立的连接而言）或者二元组（对于 SYN 一类的数据包而言）查询一个 Hash 表，找到对应的连接，然后将数据包压入该连接的 squeue 中等待处理。

- 这之后的一切协议处理都发生在连接自己的回路上；它会在经过协议栈层层处理之后一路上达用户程序。这整个过程（甚至包括 sockfs 这个抽象层）的处理全都由 squeue 保证独占，所以不会和其他连接产生任何干扰。

画出图形来表示的话，就是这样的一套结构：

{{<mermaid>}}
graph TD;
A["TCP 连接 1"] --- B["IP Classifier"] --- C["网卡驱动"];
D["TCP 连接 2"] --- B;
E["TCP 连接 3"] --- B;
F["UDP 连接 1"] --- B;
G["UDP 连接 2"] --- B;
H["UDP 连接 3"] --- B;
I["其他协议连接"] --- B;
{{</mermaid>}}

这样捋一下以后，着手点就很清楚了：我们需要对 ipclassifier 动手脚。具体而言：我们需要修改 ipclassifier，让它知晓`conn_rg_t`的存在，并在分发数据包的时候不仅仅是找到（第一个）对应连接后直接发过去即可，而是需要在整个`conn_rg_t`里的所有连接之间负载均衡。

- ipclassifier.c

ipclassifier.c 是我的提交里改动最多的一个文件。目前的版本 (Patchset 10) 的改动多达375行。虽说其中大部分都是注释，不过也说明了本提交的关键所在正是这个组件。

我们先不考虑如何构造`conn_rg_t`的问题；这不是 ipclassifier 该操心的事情。所以在我们写 ipclassifier 中的变动的时候，可以假设`conn_rg_t`已经被正确地填充了，我们只需要利用其中的信息即可。

先理一下 ipclassifier.c 中的现有逻辑。整个文件中，有两个函数的名称引起了我们的注意：`ipcl_classify_v4`及`ipcl_classify_v6`。从名称上看，这两个函数分别是 IPv4 及 IPv6 数据包的分发函数；进一步阅读代码也能看出这点（我觉得读者应该有大致看懂这些代码的能力，所以不做具体分析了）：二者逻辑基本相似，而且大致操作都是按照协议栈和数据包地址查询对应 Hash 表，从而找到具体连接。显然这里便是我们的入手点。

这里我们先对 UDP 进行改动，因为 UDP 是无状态协议，需要操心的事情最少。增加的逻辑很简单：

```c
if (connp->conn_rg_bind != NULL) {
	/*
	 * Have multiple bindings by SO_REUSEPORT,
	 * do load balancing
	 */
	connp = conn_rg_lb_pick(
	    connp->conn_rg_bind,
	    ipha->ipha_src,
	    ipha->ipha_dst,
	    ports);
}
```

在直接把数据包丢给我们找到的第一个连接之前，我们先检查一下找到的`conn_t`是否加入了一个`conn_rg_t`组。如果是的话，我们不是直接返回找到的连接，而是按照数据包的地址信息在组内挑选一个连接返回。`conn_rg_lb_pick`实现了挑选连接的策略：求出数据包的 IP 地址信息的 Hash，然后按照 Hash 值选择一个连接。具体实现并不复杂（且就在文件下方），所以在此我们一笔带过。

`ipcl_classify_v6`中的改动类似，只是将 Hash 用的地址改成了 IPv6 地址。

UDP 的改动就到此为止；接下来我们考虑一下 TCP 的问题。对于 TCP 而言，除了上述的改动之外，我们有没有什么其他需要做的呢？

首先，TCP 是有连接的概念的。这也就是说，一个`conn_t`对应的 TCP 连接不一定是我们`bind`上来接受`accept()`的连接，还有可能是`accept`以后创建的单独连接；后者显然不应当参与负载均衡。

其次，`accept()`在创建 TCP 连接之时需要先完成 TCP 握手，该握手是有状态的，因此所有的数据包都应该发到同一个连接上。我们的负载均衡并不考虑数据包的内容。这会搞乱握手造成问题吗？

再回顾一下 TCP 握手的处理：

illumos 对 TCP 握手的处理方式是这样的：我们在监听 socket 的回路上接受 SYN 数据包，在 SYN 之后立即创建出新的连接（称为 eager）。之后的握手全部在 eager 的回路中完成，监听 socket 不再参与。

换句话说，实际上监听 socket 需要接受的只有一个 SYN 包而已。如果我们能保证创建出来的连接不参与负载均衡的话，我们自然也就不会搞乱握手的问题。那么保证这点又需要什么呢？

什么也不用做。因为我们预定要将`conn_rg_t`的创建和填充放到`bind()`相关的代码中去做，而创建出来的新连接不走`bind()`，自然也不会加入`conn_rg_t`；而不加入`conn_rg_t`的 socket 相关的代码我们完全没动，所以他们并不会参与负载均衡。

结论：TCP 没有额外的工作要做，采取和 UDP 完全一致的变动即可。似乎还是比我们预想中的轻松不少？

### 负载均衡：下

完成了利用`conn_rg_t`的代码，接下来我们需要的便是正确维护`conn_rg_t`，好让我们刚刚写出来的代码可以利用它。如前所述，我们会将维护`conn_rg_t`的操作放到`bind()`相关的代码中处理，所以首先我们还是来看看`bind()`的时候到底做了些什么。

- tcp_bind.c

这里面的函数挺多，调用链也有些复杂（涉及到 TLI 和 sockfs 的交互问题），不过最终都规约到一个函数上：`tcp_bindi()`。该函数里面的操作大致是检查现有 Hash 表，如果找到地址冲突的话，视选项情况决定是直接返回错误，还是插入新项覆盖旧监听 socket。很方便的一点是，因为`SO_REUSEADDR`选项的存在，我们需要的大部分逻辑都已经被实现了；剩下的只是对`conn_rg_t`的维护了。重点改动如下：

```c
if (attempt_reuse) {
	/* Attempt to join the existing group */
	int err;
	conn_rg_t *rg;

	ASSERT(ltcp != NULL);
	ASSERT(ltcp->tcp_connp != NULL);
	ASSERT(ltcp->tcp_connp->conn_rg_bind != NULL);
	ASSERT(connp != NULL);
	ASSERT(connp->conn_rg_bind == NULL);

	err = conn_rg_insert(
	    lconnp->conn_rg_bind, connp);
	if (err != 0) {
		mutex_exit(&tbf->tf_lock);
		*errcode = err;
		return (0);
	}
	connp->conn_rg_bind = lconnp->conn_rg_bind;
}
```
这段代码大致继承自 joyent 的原提交。如果目前有冲突的 socket，且该 socket 加入了一个`conn_rg_t`组，则说明它在绑定时开启了`SO_REUSEPORT`；因此我们试图加入现有组。`conn_rg_insert`中还有一些其他的权限检查，不过逻辑并不复杂，在这里同样一笔带过。
```c
a/*
 * If we are the first here and have SO_REUSEPORT set,
 * set up connp->conn_rg_bind
 */
if (connp->conn_reuseport &&
    (connp->conn_rg_bind == NULL)) {
	conn_rg_t *rg = conn_rg_init(connp);
	if (rg == NULL) {
		mutex_exit(&tbf->tf_lock);
		*errcode = ENOMEM;
		return (0);
	}
	connp->conn_rg_bind = rg;
}
```
在没有冲突 socket 的情况下，我们需要额外检查调用`bind()`的 socket 是否开启了`SO_REUSEPORT`。若已开启，则创建新的`conn_rg_t`以供后来者加入。

UDP 的逻辑也大致如此，在此不再赘述。

### 杂项

上述内核中的改动实现了功能的核心。但作为一个提交给整个项目的 patch，仅仅写出这些代码还是不够的；我们还需要加点佐料，例如测试。

illumos-gate 中已有一套成熟的测试框架；这套框架和利用它的测试位于 usr/src/test 下。仔细检查相关内容以后，我决定把我的测试加载 os-tests 目录下。

另外我还试图做了点 man page 的维护。虽说最终结果似乎不尽人意（显然我的英语写作水平还远没到 native speaker 的程度），不过社区里某位维护者亲切地提出他可以帮我完成文档问题，所以我也便恭敬不如从命了。

## 结语

于补丁完成以后再分析，似乎这个补丁的内容三言两语便能写完，但实际完成这个补丁却花了我好几个月时间；捋清思路以及反复验证实现所需要的功夫远比纸面看上去的要多得多。不过正是这样的过程才让我获益匪浅：如何参与现有项目，如何与其他人合作，如何保证成品的质量，这些东西绝非写几个玩具项目就能学到的。

另外在这个过程中 illumos 内核代码的高质量也给我带来了极大的帮助。这点让我更加深刻地体会到了代码结构清晰，以及文档详尽的重要性：这些品质在单次开发的时候未必有很明显的优点，但却能造福后人。不信的话，对比一下我的 patch 和原 Linux 的 patch，你立刻就能体会到这点（没错，我又在黑 Linux 了）。

最后我再打一次广告：身为 Solaris 的后继者，一个已经在高端生产环境历经数十年检验的项目，illumos 的质量毋庸置疑；而精密规划过的结构以及详尽的注释让它的代码上手难度比 Linux 低了不知道多少个数量级。对于任何有志成为内核 hacker 的人来说，我觉得 illumos 都是个不错的起点，所以强烈推荐诸位在选择项目的时候认真考虑一下 illumos。相信它不会让你失望。