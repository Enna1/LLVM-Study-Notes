How To Write A Dumb Sanitizer
=============================

本文所实现的 DumbSanitizer 的完整代码见 `DumbSanitizer.patch ·
GitHub <https://gist.github.com/Enna1/f8696072bd9dc36ac236ba63839b7c16>`__\ ，基于
llvm 14.0.4。

Introduction — What is a sanitizer?
-----------------------------------

Sanitizers 是由 Google 开源的动态代码分析工具，包括：

-  AddressSanitizer (ASan)

-  LeakSanitizer (LSan)

-  ThreadSanitizer (TSan)

-  UndefinedBehaviorSanitizer (UBSsan)

-  MemorySanitizer (MSan)

所有的 Sanitizer 都由编译时插桩和运行时库两部分组成，Sanitizer 自 Clang
3.1 和 GCC 4.8 开始被集成在 Clang 和 GCC 中。

例如 ASan 是用于检测 Use-after-free, heap-buffer-overflow,
stack-buffer-overflow 等内存错误的。对于如下代码：

.. code:: cpp

   // clang -O0 -g -fsanitize=address test.cpp && ./a.out
   int main(int argc, char **argv) {
     int *array = new int[100];
     delete [] array;
     return array[argc];  // BOOM
   }

使用命令 ``clang -O0 -g -fsanitize=address test.cpp`` 就可以得到开启
ASan 编译后的产物，然后运行编译产物 a.out
就会得到如下类似输入，说明在运行 a.out 时发现了一个 UAF：

::

   =================================================================
   ==6254== ERROR: AddressSanitizer: heap-use-after-free on address 0x603e0001fc64 at pc 0x417f6a bp 0x7fff626b3250 sp 0x7fff626b3248
   READ of size 4 at 0x603e0001fc64 thread T0
       #0 0x417f69 in main test.cpp:5
       #1 0x7fae62b5076c (/lib/x86_64-linux-gnu/libc.so.6+0x2176c)
       #2 0x417e54 (a.out+0x417e54)
   0x603e0001fc64 is located 4 bytes inside of 400-byte region [0x603e0001fc60,0x603e0001fdf0)
   freed by thread T0 here:
       #0 0x40d4d2 in operator delete[](void*) llvm/projects/compiler-rt/lib/asan/asan_new_delete.cc:61
       #1 0x417f2e in main test.cpp:4
   previously allocated by thread T0 here:
       #0 0x40d312 in operator new[](unsigned long) llvm/projects/compiler-rt/lib/asan/asan_new_delete.cc:46
       #1 0x417f1e in main test.cpp:3
   Shadow bytes around the buggy address:
     0x1c07c0003f30: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
     0x1c07c0003f40: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
     0x1c07c0003f50: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
     0x1c07c0003f60: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
     0x1c07c0003f70: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
   =>0x1c07c0003f80: fa fa fa fa fa fa fa fa fa fa fa fa[fd]fd fd fd
     0x1c07c0003f90: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
     0x1c07c0003fa0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd fd
     0x1c07c0003fb0: fd fd fd fd fd fd fd fd fd fd fd fd fd fd fa fa
     0x1c07c0003fc0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
     0x1c07c0003fd0: fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa fa
   Shadow byte legend (one shadow byte represents 8 application bytes):
     Addressable:           00
     Partially addressable: 01 02 03 04 05 06 07 
     Heap left redzone:     fa
     Heap righ redzone:     fb
     Freed Heap region:     fd
     Stack left redzone:    f1
     Stack mid redzone:     f2
     Stack right redzone:   f3
     Stack partial redzone: f4
     Stack after return:    f5
     Stack use after scope: f8
     Global redzone:        f9
     Global init order:     f6
     Poisoned by user:      f7
     ASan internal:         fe
   ==6254== ABORTING

Quick Start — Writing dumb sanitizer
------------------------------------

接下来这一节我们就来讲解下怎么实现一个简单的 Sanitizer（本文称之为
DumbSanitizer 或 DbSan）。我们的 DumbSanitizer
实现下述功能：对于程序中的每一个变量，我们都统计该变量在程序运行中被访问了多少次，并且在程序退出时打印出访问次数最多的变量。

Compile llvm project with compiler-rt
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如何编译 llvm 可以参考 `Building LLVM with
CMake <https://llvm.org/docs/CMake.html#quick-start>`__\ ，需要注意的是为了使用
Sanitizer 我们需要将 ``compiler-rt`` 加入到 LLVM_ENABLE_PROJECTS 这个
CMake varibale 里。

.. code:: shell

   $ git clone -b llvmorg-14.0.4 https://github.com/llvm/llvm-project.git --depth 1
   $ cd llvm-project
   $ cmake -DCMAKE_INSTALL_PREFIX=${HOME}/llvm-bin -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_PROJECTS="clang;compiler-rt" -DLLVM_TARGETS_TO_BUILD="X86" -DLLVM_ENABLE_DUMP=ON ../llvm-project/llvm
   $ make -j12
   $ make install

Implementing the instrumentation pass
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

我们在本文最开始提到：所有的 Sanitizer
都由编译时插桩和运行时库两部分组成，并且几乎所有的 Sanitizer
的插桩部分都是通过 LLVM pass 的方式实现的。我们的 DumbSanitizer
也不例外。（关于 LLVM pass 的编写，见 `Writing an LLVM
Pass <https://llvm.org/docs/WritingAnLLVMPass.html>`__\ ）

本节就来说明 DumbSanitizer 的插桩部分是如何实现的。

这里只对一些关键点进行说明，完整实现见 `DumbSanitizer.patch ·
GitHub <https://gist.github.com/Enna1/f8696072bd9dc36ac236ba63839b7c16>`__
中：

-  llvm-project/llvm/include/llvm/Transforms/Instrumentation/DumbSanitizer.h

-  llvm-project/llvm/lib/Transforms/Instrumentation/DumbSanitizer.cpp。

首先说一下，我们实现
“对于程序中的每一个变量，统计该变量在程序运行中被访问了多少次，并且在程序退出时打印出访问次数最多的变量”
该功能的思路：

-  编译时插桩：对于每一次 memory access (load, store)，我们都会在此次
   acccess 之前插入一个函数调用 (__dbsan_read,
   \__dbsan_write)，该函数调用是在运行时库中实现的。

-  运行时库：维护一个全局 map，该 map 记录了每一个 address
   被访问的次数。函数 \__dbsan_read, \__db_write 的实现就是去更新该 map
   中 key 为本次访问变量的 address 所对应的 value 的值。

即，程序使用 DumbSanitizer 编译后，每一次对变量 x 的读/写之前都会先调用
\__dbsan_read/__dbsan_write，把变量 x
的地址传过去，__dbsan_read/__dbsan_write 会将
access_count_map[&x]++。在程序退出时根据 access_count_map
的内容就能给出访问次数最多的变量/地址了。

那么如何实现在每一次 memory access (load, store) 之前都插入一个函数调用
(__dbsan_read, \__dbsan_write) 呢？核心代码其实非常简单：

.. code:: cpp

   SmallVector<Instruction *, 16> LoadsAndStores;
   for (auto &BB : F) {
     for (auto &Inst : BB) {
       if (isa<LoadInst>(Inst) || isa<StoreInst>(Inst))
         LoadsAndStores.push_back(&Inst);
     }
   }

   for (auto *Inst : LoadsAndStores) {
     IRBuilder<> IRB(Inst);
     bool IsWrite;
     Value *Addr = nullptr;
     if (LoadInst *LI = dyn_cast<LoadInst>(I)) {
       IsWrite = false;
       Addr = LI->getPointerOperand();
     } else if (StoreInst *SI = dyn_cast<StoreInst>(I)) {
       IsWrite = true;
       Addr = SI->getPointerOperand();
     }
     if (IsWrite) {
       IRB.CreateCall(DbsanWriteFunc, IRB.CreatePointerCast(Addr, IRB.getInt8PtrTy()));
     } else {
       IRB.CreateCall(DbsanReadFunc, IRB.CreatePointerCast(Addr, IRB.getInt8PtrTy()));
     }
   }

简单解释一下。其实就是遍历 Function F 中的所有指令，收集其中的 LoadInst
和 StoreInst。然后对于每一个保存起来的 LoadInst 或 StoreInst，通过
IRBuilder 在其之前都插入一条 CallInst，被调函数就是 \__dbsan_read 或
\__dbsan_write。函数 \__dbsan_read 或 \__dbsan_write
只有一个参数，该参数就是 LoadInst 或 StoreInst 的
PointerOperand，即读写的 address。

Implementing the runtime library
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

介绍完编译时插桩的关键点后，再来介绍下运行时库的核心实现。

DumbSanitizer 运行时库部分的核心实现见 `DumbSanitizer.patch ·
GitHub <https://gist.github.com/Enna1/f8696072bd9dc36ac236ba63839b7c16>`__
中的：

-  llvm-project/compiler-rt/lib/dbsan/dbsan_interface.h

-  llvm-project/compiler-rt/lib/dbsan/dbsan_interface.cpp

-  llvm-project/compiler-rt/lib/dbsan/dbsan_rtl.h

-  llvm-project/compiler-rt/lib/dbsan/dbsan_rtl.cpp

dbsan_interface.h 和 dbsan_interface.cpp 中是对暴露给外部的函数
\__dbsan_read 和 \__dbsan_write 的实现：

.. code:: cpp

   void __dbsan_read(void *addr) { MemoryAccess((uptr)addr, kAccessRead); }

   void __dbsan_write(void *addr) { MemoryAccess((uptr)addr, kAccessWrite); }

可以看到 \__dbsan_read 和 \__dbsan_write 的实现就是对函数 MemoryAccess
的包装，MemoryAccess 的实现位于 dbsan_rtl.h 和 dbsan_rtl.cpp。

.. code:: cpp

   void MemoryAccess(uptr addr, AccessType typ) {
     ctx->access_count_map[addr]++;
     uptr access_count = ctx->access_count_map[addr];
     if (access_count > ctx->most_frequently_accessed_count) {
       ctx->most_frequently_accessed_count = access_count;
       ctx->most_frequently_accessed_addr = addr;
     }
   }

MemoryAccess 的实现也很简单，就是更新 access_count_map 中 key 为 addr 的
value 值，然后更新访问次数最多的 address。

这里 ctx 是运行时库中维护的一个 Context 类型的全局变量：

.. code:: cpp

   struct Context {
     bool initialized;
     uptr most_frequently_accessed_addr;
     uptr most_frequently_accessed_count;
     __sanitizer::DenseMap<uptr, uptr> access_count_map;
   };

-  most_frequently_accessed_addr 用于记录访问次数最多的地址

-  most_frequently_accessed_count 用于记录最多的访问次数是多少

-  access_count_map 则是记录了每一个地址被访问了多少次

最后讲一下我们是如何做到程序退出时打印访问次数最多的变量的。其实也很简单，就是通过
atexit 来注册程序退出时执行的函数，在该函数中直接打印我们在 Context
中保存的 most_frequently_accessed_addr 和 most_frequently_accessed_count
即可。

.. code:: cpp

   static void dbsan_atexit() {
     __sanitizer::Printf(
         "#Most frequently accessed address: %p, access count: %zd\n",
         (void *)ctx->most_frequently_accessed_addr,
         ctx->most_frequently_accessed_count);
   }

Integrating the sanitizer
~~~~~~~~~~~~~~~~~~~~~~~~~

实现完 DumbSanitizer
的编译时插桩和运行时库这两个核心部分，剩下的就是将我们的 DumbSanitizer
集成在 clang/llvm 的编译流程中，使得能够通过编译选项 -fsanitize=dumb
来启用 DumbSanitizer。

这部分修改的文件多且杂，没有什么需要特别说明的地方。这里只给出所需要修改的文件，详见
`DumbSanitizer.patch ·
GitHub <https://gist.github.com/Enna1/f8696072bd9dc36ac236ba63839b7c16>`__

-  llvm-project/clang/include/clang/Basic/Sanitizers.def，添加
   DumbSanitizer 的定义

-  llvm-project/clang/include/clang/Driver/SanitizerArgs.h，添加是否启用的
   DumbSanitizer 的判断

-  修改 llvm-project/clang/lib/CodeGen/BackendUtil.cpp，将 DumbSanitizer
   的插桩 pass 添加至 pass manager

-  修改
   llvm-project/clang/lib/Driver/ToolChains/CommonArgs.cpp，如果启用了
   DumbSanitizer，则链接 DumbSanitizer 的运行时库

-  修改 llvm-project/clang/lib/Driver/ToolChains/Linux.cpp，定义
   DumbSanitizer 所支持的架构，简单起见我们 DumbSanitizer 只支持 X86_64

Running the dumb sanitizer
~~~~~~~~~~~~~~~~~~~~~~~~~~

本节我们用一个例子来跑下 DumbSanitizer ，看看效果。

.. code:: cpp

   // clang++ -fsanitize=dumb test.cpp -o test
   // DBSAN_OPTIONS='print_frequent_access=1' ./test

   #include <stdio.h>
   int main(int argc, char **argv) {
     int r = 0;
     int i = 1;
     printf("address of `r` is %p\n", &r);
     printf("address of `i` is %p\n", &i);
     for (; i < 2; ++i) {
       r++;
     }
     return i + r;
   }

这里我们在优化等级为 O0 的情况下，开启 DumbSanitizer（注：DumbSanitizer
是在所有的优化 pass 执行后，才执行插桩 pass，即 DumbSanitizer
插桩的是已经优化后的代码，可以尝试改变优化等级查看上述例子程序的输出）。

在执行编译后的二进制时，我们设置了环境变量 DBSAN_OPTIONS，通过
DBSAN_OPTIONS 中的参数 print_frequent_access 为 1 还是 0
来控制在程序退出时是否打印访问次数最多的变量地址是什么。

上述例子的运行结果如下所示：

::

   address of `r` is 0x7fff5817396c
   address of `i` is 0x7fff58173968
   #Most frequently accessed address: 0x7fff58173968, access count: 6

可以看出被访问次数最多的变量是 i，被访问了的 6 次。

感兴趣可以通过 LLVM IR 来分析这是为什么，这里就不再赘述了。

.. code:: llvm

   define dso_local noundef i32 @main(i32 noundef %0, i8** noundef %1) #0 {
     %3 = alloca i32, align 4
     %4 = alloca i32, align 4
     %5 = alloca i8**, align 8
     %6 = alloca i32, align 4 ; address of r
     %7 = alloca i32, align 4 ; address of i
     %8 = bitcast i32* %3 to i8*
     call void @__dbsan_write4(i8* %8)
     store i32 0, i32* %3, align 4
     %9 = bitcast i32* %4 to i8*
     call void @__dbsan_write4(i8* %9)
     store i32 %0, i32* %4, align 4
     %10 = bitcast i8*** %5 to i8*
     call void @__dbsan_write8(i8* %10)
     store i8** %1, i8*** %5, align 8
     %11 = bitcast i32* %6 to i8*
     call void @__dbsan_write4(i8* %11) ; r = 0
     store i32 0, i32* %6, align 4
     %12 = bitcast i32* %7 to i8*
     call void @__dbsan_write4(i8* %12) ; i = 1
     store i32 1, i32* %7, align 4
     %13 = call i32 (i8*, ...) @printf(i8* noundef getelementptr inbounds ([22 x i8], [22 x i8]* @.str, i64 0, i64 0), i32* noundef %6)
     %14 = call i32 (i8*, ...) @printf(i8* noundef getelementptr inbounds ([22 x i8], [22 x i8]* @.str.1, i64 0, i64 0), i32* noundef %7)
     br label %15

   15:                                               ; preds = %24, %2
     %16 = bitcast i32* %7 to i8*
     call void @__dbsan_read4(i8* %16); i < 2
     %17 = load i32, i32* %7, align 4
     %18 = icmp slt i32 %17, 2
     br i1 %18, label %19, label %29

   19:                                               ; preds = %15
     %20 = bitcast i32* %6 to i8*
     call void @__dbsan_read4(i8* %20) ; r' = r (part1 of r++)
     %21 = load i32, i32* %6, align 4
     %22 = add nsw i32 %21, 1
     %23 = bitcast i32* %6 to i8*
     call void @__dbsan_write4(i8* %23) ; r = r' + 1 (part2 of r++)
     store i32 %22, i32* %6, align 4
     br label %24

   24:                                               ; preds = %19
     %25 = bitcast i32* %7 to i8*
     call void @__dbsan_read4(i8* %25) ; i' = i (part1 of ++i)
     %26 = load i32, i32* %7, align 4
     %27 = add nsw i32 %26, 1
     %28 = bitcast i32* %7 to i8*
     call void @__dbsan_write4(i8* %28) ; i = i' + 1 (part2 of ++i)
     store i32 %27, i32* %7, align 4
     br label %15, !llvm.loop !4

   29:                                               ; preds = %15
     %30 = bitcast i32* %7 to i8*
     call void @__dbsan_read4(i8* %30) ; i' = i (part1 of i + r)
     %31 = load i32, i32* %7, align 4
     %32 = bitcast i32* %6 to i8*
     call void @__dbsan_read4(i8* %32) ; r' = r (part2 of i + r)
     %33 = load i32, i32* %6, align 4
     %34 = add nsw i32 %31, %33 ; i' + r' (part 3 of i + r)
     ret i32 %34
   }

References
----------

1. `GitHub - google/sanitizers: AddressSanitizer, ThreadSanitizer,
   MemorySanitizer <https://github.com/google/sanitizers>`__

2. `GitHub - trailofbits/llvm-sanitizer-tutorial: An LLVM sanitizer
   tutorial <https://github.com/trailofbits/llvm-sanitizer-tutorial>`__
