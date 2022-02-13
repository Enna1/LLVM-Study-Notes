# Prologue

据我所知，目前中文互联网上介绍 ThreadSanitizr(AKA TSan) 原理的文章很少，只有 [ThreadSanitizer——跟 data race 说再见](https://zhuanlan.zhihu.com/p/38687826) 写的很清晰。

但是实际上这篇知乎文章是基于 **ThreadSanitizer** **V1** 的论文 [ThreadSanitizer: data race detection in practice (WBIA ’09)](https://static.googleusercontent.com/media/research.google.com/en//pubs/archive/35604.pdf) 来讲解的。ThreadSanitzer V1 是基于 Valgrind 实现的，基于 happen-before 和 lockset 算法。

而目前集成在 GCC/Clang 中的 ThreadSanitizer 实际上已经是 V2 版本了，底层算法也与 V1 不同，根据 [AddressSanitizer, ThreadSanitizer, and MemorySanitizer: Dynamic Testing Tools for C++ (GTAC'2013)](http://goo.gl/FPVd8)，TSan V2 使用的 fast happens-before 算法，**类似**于 FastTrack(PLDI'09) 中提出的算法。

所以本节关于 TSan 的学习笔记包含两篇文章：

1. 第一篇是对 [FastTrack: efficient and precise dynamic race detection (PLDI'09)](https://users.soe.ucsc.edu/~cormac/papers/pldi09.pdf) 这篇论文的学习笔记

2. 第二篇是从 TSan 代码实现的角度，理解 TSan 背后的检测算法

P.S.1 TSan runtime 最新是 [V3](https://reviews.llvm.org/D112603) 版本，截止本文撰写时 LLVM14 还没有发布，应该从 LLVM14 开始 TSan 默认使用 V3 runtime 。

P.S.2 我就 TSan 的底层算法[请教](https://reviews.llvm.org/D119417)了 TSan 作者 Dmitry Vyukov，得到回复是：TSan V2/V3 使用的算法都是类似于作者的另外一个 data race 检测工具 **[Relacy'08](https://github.com/dvyukov/relacy)** ，但是该工具没有论文，只有源码。

P.S.3 Konstantin Serebryany 诚不欺我，TSan V2 使用算法确实**类似** FastTrack(PLDI'09) 算法，学习 FastTrack(PLDI'09) 对于理解 TSan V2 的底层算法非常有帮助。


