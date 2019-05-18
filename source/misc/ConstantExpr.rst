ConstantExpr
============

Constant
--------

在 LLVM IR 中有这样一种值: constants，这种值在 LLVM IR
中能独立于基本块和函数存在，包括数、全局变量、常量字符数组等。我们拿
LLVM IR 中的一条指令为例来简单说明以下：

``llvm ir call void @llvm.memset.p0i8.i64(i8* nonnull align 8 %some, i8 0, i64 40, i1 false)``

``llvm.memset.*`` 是 LLVM 中的一类 intrinsics 指令，LLVM 为 C
标准库中的一些函数提供了 intrinsics 实现，有了这些 intrinsics
函数，就允许编译器前端 (如 clang)
将有关指针对齐的相关信息传给代码生成器，能够使代码生成这一过程变得更加高效。

回到 constants 上来，注意上述 callinst 的后三个参数
``i8 0, i64 40, i1 false``\ ，倒数第一个参数 ``i1 false`` 是 boolean
constants，实际上 ``true`` 和 ``false`` 都是为 i1 type 的
constants。倒数第二个参数和倒数第三个参数 ``i8 0``, ``i64 40`` 也同样是
constant，具体来说是 integer constant。类似的还有 floating-point
constants, null pointer constants ( ‘null’ in LLVM IR ) 等。
上面提到的都是 simple constants，同样也有 complex constants，比如
structure constants, array constants (包括字符数组), vector constants
等。 下面就是一个 character array constants:

::

   @.str.123 = private unnamed_addr constant [5 x i8] c"YES!\00", align 1

Constants 除了包括上面提到的 simple constants 和 complex constants，还有
Global Variable and Function Addresses，Undefined Values，Poison
Values，Addresses of Basic Blocks，Constant Expressions。其实上面
``@.str.123`` 就是一个 global variable。我们本文主要讨论 ConstantExpr
(即 Constant Expressions)。

.. _constantexpr-1:

ConstantExpr
------------

ConstantExpr is a constant value that is initialized with an expression
using other constant values，也就是说如果一个表达式的所有操作数都是
constant 的话，那么这个表达式就是一个 constant
experssion，当然它同样也是一个 constant。

一个具体的例子：

.. code:: bash

   $ cat a.c
   int a;
   int main()
   {
       return 5+(long)(&a);
   }

将 a.c 通过 clang 编译得到 LLVM IR 和 可执行程序：

.. code:: bash

   $ clang -S -O2 -emit-llvm a.c -o a.o2.ll
   $ clang -O2 a.c -o a.o2.out

``a.o2.ll`` 的部分内容如下：

.. code:: llvm

   @a = common dso_local global i32 0, align 4

   ; Function Attrs: norecurse nounwind readnone uwtable
   define dso_local i32 @main() local_unnamed_addr #0 {
   entry:
     ret i32 trunc (i64 add (i64 ptrtoint (i32* @a to i64), i64 5) to i32)
   }

这里 ``trunc (i64 add (i64 ptrtoint (i32* @a to i64), i64 5) to i32)``
整个表达式是一个 ConstantExpr。

使用 objdump 查看 main 函数对应的汇编代码：

::

   mov    $0x601039,%eax
   retq

可见 在 LLVM IR 中的 ContantExpr 在汇编代码中实际上是一个 constant
value。实际上 ContantExpr 经过很多个阶段的处理后 (some in the backend,
some by the linker, some by the dynamic loader)
，最终在程序被加载时成为一个 constant value。

BreakConstantExpr
-----------------

每一种 ContantExpr 都对应一种 Instruction，在 LLVM
中经常可以看到这样的实现 ``visitConstantExpr(ConstantExpr *CE)``
函数的写法：

.. code:: cpp

   void visitConstantExpr(ConstantExpr *CE)
   {
       switch (CE->getOpcode())
       {    
       case Instruction::Trunc:
       case Instruction::ZExt:
       case Instruction::SExt:
       case Instruction::FPTrunc:
       case Instruction::FPExt:
       case Instruction::FPToUI:
       case Instruction::FPToSI:
       case Instruction::UIToFP:
       case Instruction::SIToFP:
       case Instruction::PtrToInt:
       case Instruction::IntToPtr:
       case Instruction::BitCast:
       case Instruction::AddrSpaceCast:
       case Instruction::GetElementPtr:
       case Instruction::Select:
       case Instruction::ICmp:
       case Instruction::FCmp:
       case Instruction::ExtractElement:
       case Instruction::InsertElement:
       case Instruction::ShuffleVector:
       case Instruction::ExtractValue:
       case Instruction::InsertValue:    
       case Instruction::Add:
       case Instruction::Sub:
       case Instruction::FSub:
       case Instruction::Mul:
       case Instruction::FMul:
       case Instruction::UDiv:
       case Instruction::SDiv:
       case Instruction::FDiv:
       case Instruction::URem:
       case Instruction::SRem:
       case Instruction::FRem:
       case Instruction::And:
       case Instruction::Or:
       case Instruction::Xor:
       case Instruction::Shl:
       case Instruction::LShr:
       case Instruction::AShr:    
       default:
           llvm_unreachable("Unknown constantexpr type encountered!");
       }
   }

``visitConstantExpr`` 的用处就是先判断某条 Instruction 的操作数是否为
ConstantExpr，如果是则调用该函数进行处理。

但是在我接触的一些基于 LLVM IR 做程序分析的开源的工具中
(如SVF)，发现这些工具通常会对 LLVM IR 进行处理：如果一条 Instruction
的某个操作数是 ConstantExpr ，那么将该 ConstantExpr 转换为对应
Instruction 并插入到使用该 ConstantExpr 的 Instruction 之前，将该
ConstantExpr 的所有使用的地方替换为新插入的转换后的
Instruction。该功能的实现代码可以参考
https://github.com/Enna1/LLVM-Clang-Examples/tree/master/break-constantexpr
。

将前面提到的 LLVM IR 文件 ``a.o2.ll`` 经过 BreakConstantExpr
处理后，得到的 LLVM IR 如下，可以看到已经将ConstantExpr 转换为了
Instruction 。

.. code:: llvm

   @a = common dso_local global i32 0, align 4

   ; Function Attrs: norecurse nounwind readnone uwtable
   define dso_local i32 @main() local_unnamed_addr #0 {
   entry:
     %0 = ptrtoint i32* @a to i64
     %1 = add i64 %0, 5
     %2 = trunc i64 %1 to i32
     ret i32 %2
   }

参考链接：
----------

1. http://llvm.org/docs/LangRef.html#constant-expressions
2. http://lists.llvm.org/pipermail/llvm-dev/2017-March/110885.html
