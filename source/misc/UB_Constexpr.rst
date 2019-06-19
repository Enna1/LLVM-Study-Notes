Exploring C++ Undefined Behavior Using Constexpr
================================================

本文是对 `Exploring Undefined Behavior Using
Constexpr <https://shafik.github.io/c++/undefined%20behavior/2019/05/11/explporing_undefined_behavior_using_constexpr.html>`__
这篇文章的简单翻译学习。

实际上本文的内容和 LLVM/Clang 不是那么的相关。

在 C++ 中有很多 undefined behavior (即未定义行为，常简写为
UB)，一般来讲，我们应该尽可能的避免 UB。常见的 UB 包括 overflow
(溢出)，out-of-bounds memory access (内存越界访问)
等等。现有的很多工具都可以检测出
UB，可以把这些工具分成两类：基于静态分析（编译器警告，clang-tidy
等）和基于动态分析 (基于 LLVM/Clang 的 UndefinedBehaviorSanitizer 和
AddressSanitizer 等)。

作为一个优秀的程序员，我们应该了解 UB，以便在编程时避免 UB，在 code
review 时及时发现 UB。在编写代码时，我们可能觉得某部分代码是
UB，所以我们想 google 来查阅资料来看这部分代码到底是不是
UB，但是通常我们并不知道这种 UB
对应的术语是什么，或者我们找到了一篇相关文章，但文章中没有谈到我们正在处理的这种
UB 的特定情况。

这篇文章中提出了一种方法，通过 C++ 中的 `constant
expressions <https://en.cppreference.com/w/cpp/language/constant_expression>`__
配合 `godbolt <https://godbolt.org/>`__ 来迅速地探究某段代码是否为 UB。

Constant Expressions
--------------------

C++ 中的 constant expressions 有很多需要满足的限制，其中一个就是：一个
constant expression 中不允许出现 UB (`undefined behavior is not allowed
in a constant
expression <https://stackoverflow.com/questions/21319413/why-do-constant-expressions-have-an-exclusion-for-undefined-behavior>`__)。

C++ 标准在 `[expr.const]p4 <http://eel.is/c++draft/expr.const#4>`__ 和
`[expr.const]p4.6 <http://eel.is/c++draft/expr.const#4.6>`__ 对
*constant expressions* and *undefined behavior* 有如下的说明：

   An expression e is a core constant expression unless the evaluation
   of e, following the rules of the abstract machine (6.8.1), would
   evaluate one of the following expressions:

   -  …

   -  an operation that would have undefined behavior as specified in
      [intro] through [cpp] of this document [ Note: including, for
      example, signed integer overflow ([expr.prop]), certain pointer
      arithmetic ([expr.add]), division by zero, or certain shift
      operations — end note ];

所以如果我们有一个操作，这个操作是 UB，那么我们就不能在 constant
expression 的上下文中实现这个操作，根据 C++
标准，编译器必须以警告或者错误的形式报告这种行为，事实上，目前编译器是以报错的形式来报告这种行为的。这样我们就能够让编译器自己告诉我们，什么是
UB，什么不是。

`godbolt <https://godbolt.org/>`__
是一个以网页形式提供服务的交互型编译器，我们可以通过 godbolt
来测试一些简单的代码片段。

由此 godbolt + undefined behavior is not allowed in a constant
expression 为我们提供了一种交互式的快速的探究什么样的代码是
UB，什么样的代码不是 UB 的方法。但是也同样因为 constant expression
需要满足很多限制，所以有一些 UB 不能以这种方式被测试。例如，堆内存分配和
reinterpret_cast 都不能出现在 constant expression
中，因此我们就不能以这种方式来探究 use after free 和 strict aliasing
violations 这类 UB。

需要注意 C++
中有很多特殊的例外情况，通过这种方式得到的结果也不一定是正确的，有很多具体的例子需要深入探究学习。

An Example with Arithmetic Overflow
-----------------------------------

首先以算术溢出为例子，讲一下具体的操作。

.. code:: cpp

   #include <limits>

   void f1()
   {
       unsigned int x1 =
           std::numeric_limits<unsigned int>::max() + 1;  // Overflow one over the max
       unsigned int x2 = 0u - 1u;                         // Wrap one below the min
       int y1 = std::numeric_limits<int>::max() + 1;      // Overflow one over the max
       int y2 = std::numeric_limits<int>::min() - 1;      // Underflow one below the min
   }

上述代码片段中，有 *unsigned int* 的 overflow (上溢)，有 *unsigned int*
的下溢 (underflow)，有 *signed int* 的上溢，有 *signed int*
的下溢。究竟哪些是 UB 哪些不是 UB？

我们先回过头看一下 *constexpr*\ ，C++ 标准中要求
`[dcl.constexpr]p9 <http://eel.is/c++draft/dcl.constexpr#9>`__\ ：被
*constexpr* 说明符所声明的变量必须是 `literal
types <https://en.cppreference.com/w/cpp/named_req/LiteralType>`__\ ，必须被初始化，并且必须使用
constant expressions 初始化。

   A constexpr specifier used in an object declaration declares the
   object as const. Such an object shall have literal type and shall be
   initialized. In any constexpr variable declaration, the
   full-expression of the initialization shall be a constant expression
   (7.7). …

我们将上面的代码片段加上 *constexpr*
说明符，这样用于初始化被声明的变量的表达式就必须是 constant
expressions。也就是说 ``std::numeric_limits<unsigned int>::max()+1``,
``0u-1u``, ``std::numeric_limits<int>::max()+1``,
``std::numeric_limits<int>::min()-1`` 必须是 constant
expressions。如果它们中存在 UB，那么它们就不是 constant
expressions，编译器就会报错！我们将上述代码复制到 godbolt
中进行编译，查看编译器的输出结果 https://godbolt.org/z/0vx_j6

x86-64 clang (trunk) 的输出如下：

::

   <source>:7:19: error: constexpr variable 'y1' must be initialized by a constant expression
       constexpr int y1=std::numeric_limits<int>::max()+1;
                     ^  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
   <source>:7:53: note: value 2147483648 is outside the range of representable values of type 'int'
       constexpr int y1=std::numeric_limits<int>::max()+1;
                                                       ^
   <source>:8:19: error: constexpr variable 'y2' must be initialized by a constant expression
       constexpr int y2=std::numeric_limits<int>::min()-1;
                     ^  ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
   <source>:8:53: note: value -2147483649 is outside the range of representable values of type 'int'
       constexpr int y2=std::numeric_limits<int>::min()-1;
                                                       ^
   2 errors generated.
   Compiler returned: 1

x86-64 gcc (trunk) 的输出如下：

::

   <source>: In function 'void f()':
   <source>:7:53: warning: integer overflow in expression of type 'int' results in '-2147483648' [-Woverflow]
       7 |     constexpr int y1=std::numeric_limits<int>::max()+1;
         |                      ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^~
   <source>:7:54: error: overflow in constant expression [-fpermissive]
       7 |     constexpr int y1=std::numeric_limits<int>::max()+1;
         |                                                      ^
   <source>:7:54: error: overflow in constant expression [-fpermissive]
   <source>:8:53: warning: integer overflow in expression of type 'int' results in '2147483647' [-Woverflow]
       8 |     constexpr int y2=std::numeric_limits<int>::min()-1;
         |                      ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~^~
   <source>:8:54: error: overflow in constant expression [-fpermissive]
       8 |     constexpr int y2=std::numeric_limits<int>::min()-1;
         |                                                      ^
   <source>:8:54: error: overflow in constant expression [-fpermissive]
   Compiler returned: 1

根据编译器的输出可知：\ **signed int 的 overflow 和 underflow 都是
UB**\ ，\ **unsigned int 的 overflow 和 underflow 都不是 UB**\ 。

C++标准的 `[expr]p4 <http://eel.is/c++draft/expr#pre-4>`__ 和
`[basic.fundamental]p2 <http://eel.is/c++draft/basic.fundamental#2>`__\ 中有相应的说明：

   If during the evaluation of an expression, the result is not
   mathematically defined or not in the range of representable values
   for its type, the behavior is undefined. [ Note: Treatment of
   division by zero, forming a remainder using a zero divisor, and all
   floating-point exceptions vary among machines, and is sometimes
   adjustable by a library function. — end note ]

..

   … The range of representable values for the unsigned type is 0 to
   2^(N−1) (inclusive); arithmetic for the unsigned type is performed
   modulo 2^N. [ Note: Unsigned arithmetic does not overflow. Overflow
   for signed arithmetic yields undefined behavior ([expr.pre]). — end
   note ]

Arithmetic Overflow 的另一种有趣的情况是：

.. code:: cpp

   constexpr int x = std::numeric_limits<int>::min() / -1;

如果我们将上述代码放到 godbolt 中
https://godbolt.org/z/Ecp88m，我们会发现上述代码也是 UB。假设 int 是 64
bit，那么 int 能表示的最大值是 2147483647，而
``std::numeric_limits<int>::min() / -1`` 的结果是 2147483648，超出了 int
所能表示的返回，所以这是一个 UB。

通过这个例子我们可以看到
``undefined behavior is not allowed in a constant expression``
为我们提供了一种强大的方法来探究并识别 UB (
尽管虽然这种方法不能处理所有的 UB )。

下面我们具体分析所有的能用这种方式来捕获的 UB。

Conversions and values that can not be represented
--------------------------------------------------

下面我们看一下，把一个 integral or floating-point type 的变量转换成一个
smaller sized type 是不是 UB。

.. code:: cpp

   constexpr unsigned int u = std::numeric_limits<unsigned int>::max();  // 1
   constexpr int i = u;                                                  // Line 6

   constexpr double d = static_cast<double>(std::numeric_limits<int>::max()) + 1;  // 2
   constexpr int x = d;  // Line 10

   constexpr double d2 = std::numeric_limits<double>::max();  // 3
   constexpr float f = d2;                                    // Line 13

将上述代码放到 godbolt 中 https://godbolt.org/z/2ZfPKt，查看编译输出。

x86-64 clang (trunk) 的输出如下：

::

   <source>:10:16: error: constexpr variable 'x' must be initialized by a constant expression
    constexpr int x = d;
                  ^   ~
   <source>:10:20: note: value 2147483648 is outside the range of representable values of type 'const int'
    constexpr int x = d;
                      ^
   <source>:13:18: error: constexpr variable 'f' must be initialized by a constant expression
    constexpr float f = d2;
                    ^   ~~
   <source>:13:22: note: value 1.797693134862316E+308 is outside the range of representable values of type 'const float'
    constexpr float f = d2;
                        ^
   2 errors generated.
   Compiler returned: 1

根据 x86-64 clang (trunk) 编译器的输出可知：case1 是 well-defined，case2
和 case3 不是 well-defined。值得注意的是 x86-64 gcc (trunk) 只对 case2
报错。

根据 C++ 标准的 Integral conversions 部分
`[conv.integral]p3 <https://timsong-cpp.github.io/cppwp/n4659/conv.integral#3>`__
(this changes in C++20 it modulo 2^N)，case 1 是 impelmentation
defined。

   If the destination type is signed, the value is unchanged if it can
   be represented in the destination type; otherwise, the value is
   implementation-defined.

根据 C++ 标准的 Floating-point conversions 部分
`[conv.dobule]p1 <https://timsong-cpp.github.io/cppwp/n4659/conv.double#1>`__
和 Floating-integral conversions 部分
`[conv.fpint]p1 <https://timsong-cpp.github.io/cppwp/n4659/conv.fpint#1>`__\ ，case2
和 case3 是 UB。

   A prvalue of floating-point type can be converted to a prvalue of
   another floating-point type. If the source value can be exactly
   represented in the destination type, the result of the conversion is
   that exact representation. If the source value is between two
   adjacent destination values, the result of the conversion is an
   implementation-defined choice of either of those values. Otherwise,
   the behavior is undefined.

..

   A prvalue of a floating-point type can be converted to a prvalue of
   an integer type. The conversion truncates; that is, the fractional
   part is discarded. The behavior is undefined if the truncated value
   cannot be represented in the destination type. [ Note: If the
   destination type is bool, see [conv.bool].  — end note ]

Division by zero
----------------

很多人都知道整型变量除以零是 UB，但是对浮点型变量除以零是否为 UB
则不确定。将下面的代码通过 goldbot 编译 https://godbolt.org/z/tRM6oF

.. code:: cpp

   constexpr int x = 1/0;        // Line 2
   constexpr double d = 1.0/0.0; // Line 3

x86-64 clang (trunk) 的输出如下：

::

   <source>:2:18: error: constexpr variable 'x' must be initialized by a constant expression
      constexpr int x = 1/0;
                    ^   ~~~
   <source>:2:23: note: division by zero
      constexpr int x = 1/0;
                         ^
   <source>:3:21: error: constexpr variable 'd' must be initialized by a constant expression
      constexpr double d = 1.0/0.0;
                       ^   ~~~~~~~
   <source>:3:28: note: floating point arithmetic produces an infinity
      constexpr double d = 1.0/0.0;
                              ^
   <source>:2:23: warning: division by zero is undefined [-Wdivision-by-zero]
      constexpr int x = 1/0;
                         ^~

可以看到，整型变量除以零、浮点型变量除以零都是 UB。

Shifty characters
-----------------

关于移位运算，我们可能想知道下面这些操作哪些是 UB：

1. Shifting greater than the bit-width of the type?
2. Shifting by a negative shift?
3. Shifting a negative number?
4. Shifting into the sign bit?

编写对应的测试代码如下：

.. code:: cpp

   void foo()
   {
       static_assert(sizeof(int) == 4 && CHAR_BIT == 8 );
       constexpr int y1 = 1 << 32;   // Shifting greater than the bit-width
       constexpr int y2 = 1 >> 32;   // Shifting greater than the bit-width
       constexpr int y3 = 1 << -1;   // Shifting by a negative amount
       constexpr int y4 = -1 << 12;  // Shifting a negative number
       constexpr int y5 = 1 << 31;   // Shifting into the sign bit
   }

查看编译输出 https://godbolt.org/z/p7onyC 发现：除了 Shifting into the
sign bit 之外，其他的操作都是 UB。

前两种移位操作的情况 (Shifting greater than the bit-width of the type,
Shifting by a negative shift) 在 C++ 标准
`[expr.shift]p1 <https://timsong-cpp.github.io/cppwp/n4659/expr.shift#1>`__
中有相关说明：

   The shift operators << and >> group left-to-right.

   ::

      shift-expression:
        additive-expression
        shift-expression << additive-expression
        shift-expression >> additive-expression

   The operands shall be of integral or unscoped enumeration type and
   integral promotions are performed. The type of the result is that of
   the promoted left operand. **The behavior is undefined if the right
   operand is negative, or greater than or equal to the width of the
   promoted left operand.**

第三种移位操作的情况 (Shifting a negative number) 在 C++20 之前是 UB
[`expr.shift]p2 <https://timsong-cpp.github.io/cppwp/n4659/expr.shift#2>`__\ ：

   The value of E1 << E2 is E1 left-shifted E2 bit positions; vacated
   bits are zero-filled. If E1 has an unsigned type, the value of the
   result is E1×(2^E2), reduced modulo one more than the maximum value
   representable in the result type. Otherwise, **if E1 has a signed
   type and non-negative value**, and E1×(2^E2) is representable in the
   corresponding unsigned type of the result type, then that value,
   converted to the result type, is the resulting value; **otherwise,
   the behavior is undefined**.

在
`p0907r4 <http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2018/p0907r4.html>`__
中被规定为 well-defined.

Everyones favorite pointer, nullptr
-----------------------------------

关于 nullptr，一些人可能很简单地认为：只要是涉及到 nullptr 的操作都是
UB。

下面是一段简单的示例代码：

.. code:: cpp

   constexpr int bar()
   {
       constexpr int* p = nullptr;
       return *p;  // Unconditional UB
   }

   constexpr void foo()
   {
       constexpr int x = bar();
   }

上述代码的编译输出 https://godbolt.org/z/cyiVq9
与我们所预想的一致，确实是 UB。

::

   <source>:1:15: error: constexpr function never produces a constant expression [-Winvalid-constexpr]
   constexpr int bar() {
                 ^
   <source>:3:12: note: read of dereferenced null pointer is not allowed in a constant expression
       return *p;        // Unconditional UB
              ^
   <source>:7:19: error: constexpr variable 'x' must be initialized by a constant expression
       constexpr int x = bar();
                     ^   ~~~~~
   <source>:3:12: note: read of dereferenced null pointer is not allowed in a constant expression
       return *p;        // Unconditional UB
              ^
   <source>:7:23: note: in call to 'bar()'
       constexpr int x = bar();

下面，我们看一些比较复杂的情况。

通过 nullptr 来访问类的非静态成员，通过 nullptr
来访问类的静态成员，是否都是 UB？

.. code:: cpp

   struct A
   {
       constexpr int g() { return 0;}
       constexpr static int f(){ return 1;}
   };

   static constexpr A* a=nullptr;

   void foo()
   {
       constexpr int x = a->f(); // case1
       constexpr int y = a->g(); // case2
   }

case1 是通过 nullptr 访问类的静态成员，case2是通过 nullptr
访问类的非静态成员。提交到 godbolt 编译后，发现 x86-64 clang(trunk) 和
x86-64 gcc(trunk) 的编译输出结果不一致
https://godbolt.org/z/7NrcFD。x86-64 clang(trunk) 认为 case2 是 UB，而
gcc 对 case1 和 case2 都没有报错。

事实上，虽然 case1 是 well-defined，但是 `CWG defect report 315: Is call
of static member function through null pointer
undefined? <http://www.open-std.org/jtc1/sc22/wg21/docs/cwg_closed.html#315>`__
告诉我们，当通过 nullptr 访问静态成员时，没有 lvalue-to-rvalue 的转换。

More pointer fun
----------------

Incrementing pointer out of bounds
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果一个指针越界了，但是我们不使用该指针、不对该指针解引用，那么是否为
UB 呢？

.. code:: cpp

   static const int arrs[10]{};

   void foo()
   {
       constexpr const int* y = arrs + 11;
   }

根据 https://godbolt.org/z/-E06pt，x86-64 clang(trunk)
报告了该错误，但是 x86-64 gcc(trunk)
并没有捕获到该错误。可以看到如果一个指针越界了，不过该指针后续是否用于其他操作中，都是
UB。但是，需要注意的是一个例外，如果一个指针只越界了一个元素，那么则不是
UB，如 std::end
就是用指向最后一个元素的后一个元素的指针来表示容器或者数组的结尾，https://en.cppreference.com/w/cpp/iterator/end。

Incrementing out of bounds but coming back in
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. code:: cpp

   constexpr int foo(const int *p)
   {
       return *((p + 12) - 5); // ?
   }

   constexpr void bar()
   {
       constexpr int arr[10]{};
       constexpr int x = foo(arr);
   }

虽然中间的表达式 p + 12 越界，(p + 12) - 5 没有越界，但是这也是
UB。与上一种情况类似 x86-64 gcc(trunk) 同样没有捕获到该错误
https://godbolt.org/z/D4uayd。

C++ 标准 `[expr.add]p4 <http://eel.is/c++draft/expr.add#4>`__
中告诉我们什么样的索引是可接受的。

   When an expression J that has integral type is added to or subtracted
   from an expression P of pointer type, the result has the type of P. -
   If P evaluates to a null pointer value and J evaluates to 0, the
   result is a null pointer value. - Otherwise, if P points to an array
   element i of an array object x with n elements, the expressions P + J
   and J + P (where J has the value j) point to the
   (possibly-hypothetical) array element i + j of x if 0 ≤ i + j ≤ n and
   the expression P - J points to the (possibly-hypothetical) array
   element i - j of x if 0 ≤ i − j ≤ n. - Otherwise, the behavior is
   undefined.

`Footnote 80 <http://eel.is/c++draft/expr.add#footnote-80>`__\ ：

   … A pointer past the last element of an array x of n elements is
   considered to be equivalent to a pointer to a hypothetical element
   x[n] for this purpose; see [basic.compound].

上述标准涵盖了 incrementing out of bounds 和 incrementing out of bounds
during an intermediate step 这两种情况。

在 `[basic.compound]p3 <http://eel.is/c++draft/basic.compound#3>`__
说明了指向最后一个元素的后一个元素的指针是合法的 (one past the end is a
valid pointer) ：

   … Every value of pointer type is one of the following:

   -  a pointer to an object or function (the pointer is said to point
      to the object or function), or
   -  a pointer past the end of an object ([expr.add]), or

   …

Out of bounds access
~~~~~~~~~~~~~~~~~~~~

没有什么好说的，越界访问是 UB。

.. code:: cpp

   constexpr int foo(const int *p)
   {
       return *(p + 12);
   }

   constexpr void bar()
   {
       constexpr int arr[10]{};
       constexpr int x = foo(arr);
   }

https://godbolt.org/z/O2GqaX，clang 和 gcc 都报告除了该错误。

End of life
-----------

某个变量在其生命周期结束后被使用是一种很难被检测的 UB。因 constant
expressions
中不允许内存分配，但是允许使用引用，所以下面用了一个函数返回对局部变量的引用的例子。

.. code:: cpp

   constexpr int& foo()
   {
       int x = 0;

       return x;  // x will soon be out of scope
                  // but we return it by reference
   }  // bye bye x

   constexpr int bar()
   {
       constexpr int x = foo();
       return x;
   }

https://godbolt.org/z/hM607D gcc 和 clang 都报告了该 UB，clang
的错误报告可读性更好 (read of variable whose lifetime has ended) 。

Flowing off the end of a value returning function
-------------------------------------------------

.. code:: cpp

   constexpr int foo(int x)
   {
       if (x)
           return 1;
       // Oppps we forgot the return 0;
   }

   void bar()
   {
       constexpr int x = foo(0);
   }

在函数 foo 中有两条可能被执行的路径，但只有一条路径上有 return
语句，如果 x == 0 则会发生未定义行为。

https://godbolt.org/z/AALpj5 gcc 和 clang 都报告了该 UB。

C++ 标准
`[stmt.return]p2 <http://eel.is/c++draft/stmt.return#2.sentence-8>`__
对这种情况进行了说明：

   … Flowing off the end of a constructor, a destructor, or a
   non-coroutine function with a cv void return type is equivalent to a
   return with no operand. Otherwise, flowing off the end of a function
   other than main or a coroutine ([dcl.fct.def.coroutine]) results in
   undefined behavior.

Modifying a constant object
---------------------------

尝试修改一个 constant 对象是 UB。下面的代码通过 const_cast 去除了变量的
const 属性，然后对变量进行了修改。

.. code:: cpp

   struct B
   {
       int i;
       double d;
   };

   constexpr B bar()
   {
       constexpr B b = { 10, 10.10 };
       B *p = const_cast<B *>(&b);

       p->i = 11;
       p->d = 11.11;

       return *p;
   }

   void foo()
   {
       constexpr B y = bar();
   }

编译器的输出结果：https://godbolt.org/z/AYulw0 ，clang 捕获到了该
UB，gcc 又没有…

C++标准在 `[dcl.type.cv]p4 <http://eel.is/c++draft/dcl.type.cv#4>`__
进行了说明：

   Except that any class member declared mutable ([dcl.stc]) can be
   modified, any attempt to modify ([expr.ass], [expr.post.incr],
   [expr.pre.incr]) a const object ([basic.type.qualifier]) during its
   lifetime ([basic.life]) results in undefined behavior …

值得注意的是，如果我们通过 const_case 去除了变量的 const
属性，但是并不修改变量，这种行为是 well-defined，例如下面的代码：

.. code:: cpp

   struct B
   {
       int i;
       double d;
   };

   constexpr B bar()
   {
       constexpr B b = { 10, 10.10 };
       B *p = const_cast<B *>(&b);

       int x = p->i;

       return *p;
   }

   void foo()
   {
       constexpr B y = bar();
   }

https://godbolt.org/z/PaADef clang 和 gcc 都没有报错。

Accessing a non-active union member
-----------------------------------

.. code:: cpp

   union Y { float f; int k; };
   void g() {
     constexpr Y y = { 1.0f }; // OK, y.x is active union member (10.3)
     constexpr int n = y.k;    // Line 4
   }

https://godbolt.org/ clang 和 gcc 都报告了该 UB。

C++ 标准中与此相关的说明
`[class.union]p1 <http://eel.is/c++draft/class.union#1>`__\ ：

   … a non-static data member is active if its name refers to an object
   whose lifetime has begun and has not ended ([basic.life]). At most
   one of the non-static data members of an object of union type can be
   active at any time, that is, the value of at most one of the
   non-static data members can be stored in a union at any time …

Casting int to enum outside its range
-------------------------------------

C++ 标准中的相关说明：

`[dcl.enum]p8 <http://eel.is/c++draft/dcl.enum#8>`__\ ：

   For an enumeration whose underlying type is fixed, the values of the
   enumeration are the values of the underlying type. Otherwise, the
   values of the enumeration are the values representable by a
   hypothetical integer type with minimal width M such that all
   enumerators can be represented. The width of the smallest bit-field
   large enough to hold all the values of the enumeration type is M. It
   is possible to define an enumeration that has values not defined by
   any of its enumerators. …

`[expr.static.cast]p10 <http://eel.is/c++draft/expr.static.cast#10>`__\ ：

   A value of integral or enumeration type can be explicitly converted
   to a complete enumeration type. If the enumeration type has a fixed
   underlying type, the value is first converted to that type by
   integral conversion, if necessary, and then to the enumeration type.
   If the enumeration type does not have a fixed underlying type, the
   value is unchanged if the original value is within the range of the
   enumeration values ([dcl.enum]), and otherwise, the behavior is
   undefined …

`defect report
2338 <http://www.open-std.org/jtc1/sc22/wg21/docs/cwg_defects.html#2338>`__
应该是当前标准这么规定的原因。

在下面的例子中，enum A 需要 1 bit 来表示其所有的枚举变量，同时 enum A
没有 fixed underlying type，所以将任意需要超过 1 bit 来表示的数 cast 为
enum A 都是 UB。

.. code:: cpp

   enum A
   {
       e1 = 0,
       e2
   };

   constexpr int foo()
   {
       constexpr A a1 = static_cast<A>(4); // 4 requires 2 bit
       return a1;
   }

   constexpr int bar()
   {
       constexpr int x = foo();
       return x;
   }

   int main()
   {
       return bar();
   }

查看 godbolt 的结果 https://godbolt.org/z/ZWI2xb，发现 gcc, clang, msvc
都没有报告编译错误，也就是没有捕获到该 UB。clang 通过
UBSan，能检测到与上例类似的一种情况
https://wandbox.org/permlink/s6judERNBKYIgspG，但是也不能检测到上例中的
UB。

Multiple unsequenced modifications
----------------------------------

Stack Overflow 上的一个 infamous 的问题 `Why are these constructs using
pre and post-increment undefined
behavior? <https://stackoverflow.com/q/949433/1708801>`__

国内谭浩强的教材习题中也有类似的问题。

下面代码是该问题的简化版本，其中 x
被修改两次，而加法运算符的两个操作数的求值是无顺序的。

.. code:: cpp

   constexpr int f(int x)
   {
       return x++ + x++;
   }

   int main()
   {
       constexpr int x = 2;
       constexpr int y = f(x);
   }

通过 https://godbolt.org/z/Uai2P9，我们发现 clang, gcc, msvc
都没有报错。而事实上该行为是 UB。

One More Inconsistency, Guaranteed Copy Elision
-----------------------------------------------

TODO 见原文 ……

`[class.copy.elision]p1 <http://eel.is/c++draft/class.copy.elision#1>`__
中指出：

   …Copy elision is not permitted where an expression is evaluated in a
   context requiring a constant expression ([expr.const]) and in
   constant initialization ([basic.start.static]). [ Note: Copy elision
   might be performed if the same expression is evaluated in another
   context.— end note ]

原文中举了一个 `Richard Smith <https://twitter.com/zygoloid>`__
分享的例子来说明：

.. code:: cpp

   struct B {B* self=this;};
   extern const B b;
   constexpr B f() {
       B b;                              // Line 4
       if(&b == &::b) return B();        // Line 5
       else return b;                    // Line 6
   }
   constexpr B b=f(); // is b.self == b  // Line 8

An Example, A Strong Integer Type
---------------------------------

在原文的最后，作者举了一个例子说明本文提供的方法怎样被应用。该例演示了如何构造一个简单的强
int 类型，当该类在 constexpr上下文中被使用时，它将捕获我们在使用 int
时可能遇到的所有常见的 UB。

.. code:: cpp

   struct Integer {
      constexpr Integer(int v){value = v;}
      constexpr Integer(double d){value = d;}
      constexpr Integer(const Integer&) = default;

      int Value() const {return value;}

      constexpr Integer operator+(Integer y) const { 
          return {value + y.value};
      }

      constexpr Integer operator-(Integer y) const { 
          return {value - y.value};
      }

      constexpr Integer operator*(Integer y) const {
          return {value*y.value};
      }

      constexpr Integer operator/(Integer y) const {
          return {value/y.value};
      }

      constexpr Integer operator<<(Integer shift) const {
          return {value << shift.value};
      }

      constexpr Integer operator>>(Integer shift) const {
          return {value >> shift.value};
      }

      int value{};
   };

一些本文中已讨论过的 UB 的操作：

.. code:: cpp

     constexpr Integer i_int_max{INT_MAX};
     constexpr Integer i_int_max_plus_one{i_int_max+1}; // Overflow
     constexpr Integer i_one{1};
     constexpr Integer i_zero{0};
     constexpr Integer i_divide_by_zero = i_one/i_zero;  // Divide by zero
     constexpr Integer i_double_max{DBL_MAX}; // double value outside of range representable by int
     constexpr Integer i_int_min{INT_MIN};
     constexpr Integer i_minus_one{-1};
     constexpr Integer i_overflow_division = i_int_min/i_minus_one;  // Overflow
     constexpr Integer i_shift_ub1 = i_one << 32;
     constexpr Integer i_shift_ub2 = i_minus_one << 1;
     constexpr Integer i_shift_ub3 = i_one << -1;

https://godbolt.org/z/ScpyN1 说明上述 UB 均被捕获到了。

Conclusion
----------

本文中我们了解了常量表达式，并且学习到了在常量表达式上下文中是禁止未定义行为的。我们可以利用constexpr
来捕获和探究未定义的行为，本文中已经探究了可以在常量表达式上下文中被研究的大部分未定义行为。

需要注意的是此方法依靠编译器为我们捕获未定义行为，但是编译器是有 bug
的，事实上在本文中也已经看到了一些编译器结果与标准不一致的情况，因此我们应尽可能使用多个编译器进行测试，以避免漏报和误报。

另一个需要注意的是 Guaranteed Copy Elision，\ *TODO*\ 。

P.S.
----

Integral Promotions
~~~~~~~~~~~~~~~~~~~

   The implicit conversions that preserve values are commonly referred
   to as promotions. Before an arithmetic operation is performed,
   integral promotion is used to create int s out of shorter integer
   types. Similarly, floating-point promotion is used to create double s
   out of float s. Note that these promotions will not promote to long
   (unless the operand is a char16_t, char32_t, wchar_t, or a plain
   enumeration that is already larger than an int ) or long double. This
   reflects the original purpose of these promotions in C: to bring
   operands to the ‘‘natural’’ size for arithmetic operations. The
   integral promotions are:

   -  A char, signed char, unsigned char, short int, or unsigned short
      int is converted to an int if int can represent all the values of
      the source type; otherwise, it is converted to an unsigned int.
   -  A char16_t, char32_t, wchar_t, or a plain enumeration type is
      converted to the first of the following types that can represent
      all the values of its underlying type: int, unsigned int, long,
      unsigned long, or unsigned long long.
   -  A bit-field is converted to an int if int can represent all the
      values of the bit-field; otherwise, it is converted to unsigned
      int if unsigned int can represent all the values of the bit-field.
      Otherwise, no integral promotion applies to it.
   -  A bool is converted to an int ; false becomes 0 and true becomes
      1. Promotions are used as part of the usual arithmetic
      conversions.

Usual Arithmetic Conversions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

   These conversions are performed on the operands of a binary operator
   to bring them to a common type, which is then used as the type of the
   result: 1. If either operand is of type long double, the other is
   converted to long double. - Otherwise, if either operand is double,
   the other is converted to double. - Otherwise, if either operand is
   float, the other is converted to float. - Otherwise, integral
   promotions (§10.5.1) are performed on both operands. 2. Otherwise, if
   either operand is unsigned long long, the other is converted to
   unsigned long long. - Otherwise, if one operand is a long long int
   and the other is an unsigned long int, then if a long long int can
   represent all the values of an unsigned long int, the unsigned long
   int is converted to a long long int ; otherwise, both operands are
   converted to unsigned long long int. Otherwise, if either operand is
   unsigned long long, the other is converted to unsigned long long. -
   Otherwise, if one operand is a long int and the other is an unsigned
   int, then if a long int can represent all the values of an unsigned
   int, the unsigned int is converted to a long int ; otherwise, both
   operands are converted to unsigned long int. - Otherwise, if either
   operand is long, the other is converted to long. - Otherwise, if
   either operand is unsigned, the other is converted to unsigned. -
   Otherwise, both operands are int. These rules make the result of
   converting an unsigned integer to a signed one of possibly larger
   size implementation-defined. That is yet another reason to avoid
   mixing unsigned and signed integers.
