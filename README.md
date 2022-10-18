# LLVM 学习笔记

我在阅读 LLVM 源代码、学习 LLVM 相关内容时记录的笔记。如果发现错误，有意见或者建议，欢迎提 Issue 指出。

Read the Docs 在线文档地址：https://llvm-study-notes.readthedocs.io/

## Contents:

- Important and useful LLVM APIs
  - [RTTI in LLVM](https://llvm-study-notes.readthedocs.io/en/latest/important-and-useful-llvm-apis/RTTI-in-LLVM.html)
  - [StringRef & Twine](https://llvm-study-notes.readthedocs.io/en/latest/important-and-useful-llvm-apis/StringRef-and-Twine.html)
- LLVM IR
  - [ConstantExpr](https://llvm-study-notes.readthedocs.io/en/latest/llvm-ir/ConstantExpr.html)
- SSA
  - [SSA construction](https://llvm-study-notes.readthedocs.io/en/latest/ssa/SSA-Construction.html)
  - [Mem2Reg](https://llvm-study-notes.readthedocs.io/en/latest/ssa/Mem2Reg.html)
- Analysis
  - [Alias Analysis](https://llvm-study-notes.readthedocs.io/en/latest/analysis/alias-analysis/index.html)
- Transform
  - [Aggressive Dead Code Elimination](https://llvm-study-notes.readthedocs.io/en/latest/transform/aggressive-dead-code-elimination/index.html)
  - [Called Value Propagation](https://llvm-study-notes.readthedocs.io/en/latest/transform/called-value-propagation/index.html)
  - [Correlated Value Propagation](https://llvm-study-notes.readthedocs.io/en/latest/transform/correlated-value-propagation/index.html)
  - [SLP Vectorizer](https://llvm-study-notes.readthedocs.io/en/latest/transform/slp-vectorizer/index.html)
- Link Time Optimization
  - [LTO Remove Dead Symbol](https://llvm-study-notes.readthedocs.io/en/latest/lto/RemoveDeadSymbol.html)
- Sanitizer
  - [How To Write a Sanitizer](https://llvm-study-notes.readthedocs.io/en/latest/sanitizer/writing-a-sanitizer/index.html)
  - [How Sanitizer Runtime Initialized](https://llvm-study-notes.readthedocs.io/en/latest/sanitizer/sanitizer-runtime-init/index.html)
  - [How Sanitizer Interceptor Works](https://llvm-study-notes.readthedocs.io/en/latest/sanitizer/sanitizer-interceptor/index.html)
  - [How Sanitizer Get Stack Trace](https://llvm-study-notes.readthedocs.io/en/latest/sanitizer/sanitizer-stacktrace/index.html)
  - [ThreadSanitizer](https://llvm-study-notes.readthedocs.io/en/latest/sanitizer/tsan/index.html)
  - [GWP-ASan](https://llvm-study-notes.readthedocs.io/en/latest/sanitizer/gwp-asan/index.html)
- Misc
  - [Exploring C++ Undefined Behavior Using Constexpr](https://llvm-study-notes.readthedocs.io/en/latest/misc/UB_Constexpr.html)
