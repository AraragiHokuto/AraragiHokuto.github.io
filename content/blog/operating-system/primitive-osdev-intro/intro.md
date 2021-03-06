+++
title = "初等屠龙技：阿库娅也能看懂的 OS 编写入门 序"
date = "2020-12-24"
categories = ["operating-system"]
tags = ["operating-system"]
+++

# 其之一 引

如是我闻：程序员有三大浪漫，曰OS，曰CG，曰编译器。虽说引用*乎上业界爱抖露的言论是个高风险行为，不过我还是打算拿这句话作为开篇之语——反正这是我自己的博客，管他那么多。

我还记得小学三年级时的自己。那时候我还是个对计算机充满好奇的熊孩子，脑袋里充斥着一行代码没读过的外行人对编程的幻想。当时我听说 Linux 和 Windows 都是用 C 写的，于是误以为花个一两年的时间把 C 入了门，就足以写出一个操作系统来自娱自乐——我到现在还能记得当时我幻想中的那个自制 OS 的 BIOS 式蓝色背景文本 GUI（现在回首，在 UI 的设想上我还真是意外地有自知之明，没幻想一个华丽的3D 桌面出来）。

当然不必说，买回来一本 VB 教程和一本国内大学 C++ 教材的我自然是连门都没碰到，甚至连编程本身也没学到几分。真正要入门 OS 开发，那还得等到初二的时候有人给我指了路，我才跟着一个相对实用的教程写了半个内核出来。

关于我自己的闲话就到此为止。至于提笔编写这个教程的动机——如前文所述——是浪漫。虽说既然是如此普遍的浪漫，教程自然不会少，但我总觉得编写玩具 OS 可以采取一个不那么实用向的角度（不会真的有人觉得自己花一个月就能打败 Windows&Linux&macOS 吧？）。然而不幸，我所能找到的 OS Tutorial 几乎都始于从 0x7c00 启动开始读扇区，或者从 multiboot 启动之后开始往 0xb8000 写字符显示 VGA 文本。

这无可非议。毕竟写 OS 总得让人有点成就感，而在自己的 x86 机器上跑一个能往屏幕上吐出东西的裸机程序大概是获得成就感的最快方式了。但我还是觉得不高兴：既然都已经放弃实用性追求浪漫了，身为活在新世纪的第二个十年的年轻人，我们何不抛开这些历史的苟且，搞些更优雅更浪漫的东西出来呢？

正因如此，我打算在本系列中采取一个不那么寻常的路子。我们将首先实现一个 RISC-V 上的类 Unix 操作系统，一直到能跑起一个基础的 shell 为止，但我们并不止于此；之后，我们还要把我们这个玩具 OS 移植到 x86-64 上。

这种路径会带来几个好处：

- 因为我们已经预定了多体系支持，这便要求我们在编写内核的时候留个心眼，而不会满足于平台相关的七拼八凑。
- 不同体系之间的对比能更加明显地凸显出哪些部分是操作系统的本质，哪些部分是与体系交互的抽象层。我个人希望这种概念上的明晰能让入门读者受益更多。
- 众所周知，x86 的臃肿和混乱可谓臭名昭彰。从 RISC-V 上开始能让我们更快地真正开始编写我们的操作系统，而不用花大力气对付实模式/保护模式/长模式，以及其他那堆历史遗留的东西。

当然，考虑到大部分读者还在用着 x86 的机器，所以如果有读者觉得这种路径太繁琐/太慢热的话，本系列恐怕不适合你。无妨——优秀的 OS Tutorial 到处都是，相信读者稍微检索一下即可找到。

而如果你是那部分还没被劝退的读者的话——欢迎登机，祝旅途愉快。

## 背景知识

鉴于本系列是 OS 编写入门教程，而不是编程入门教程，我会假定读者具有以下背景知识：

- 以 C 语言编写程序的能力。本教程中绝大部分内容会用 C 语言实现，故而假定读者具有理解示例，以及自行补足或修改示例的能力。（另，别跟我推销 Rust，也别问我为什么不接受推销。）
- 对计算机系统组成的基本概念。OS 不止是 HAL，但 OS 通常需要实现一个 HAL。在使用硬件之前显然需要对硬件有概念。
- OS 相关的基本理论。当然本系列并不要求读者成为 OS 专家，但没有理论的引导我们只会晕头转向，所以基础的理论知识也是必要的。

以上列表我会随时补充。感到自己知识有所欠缺的读者，可以预先补足，或者随着我们的进度推进一道补足。

## 参考资料

教程本身是有极限的，额外的参考资料必不可少。以下将会列出我本人认为有价值的材料，方便读者随时查阅：

- [OSDev Wiki](https://wiki.osdev.org/) OS 开发相关的 wiki，上面零散地分布着大量实用而有价值的资料。十星推荐。
- [RISC-V 标准](https://riscv.org/technical/specifications/) RISC-V 体系的标准。在我们编写 RISC-V 上的程序的时候将会大量参考这些内容。
- 《AMD64 Architecture Programmer's Manual》AMD 官方提供的 AMD64 参考手册。在我们进行 x86-64 移植的时候将会大量参考该手册。

以上列表我会随时补充。

---

闲话少叙，是时候开始干点实事了。下一话，你好，THE WORLD!