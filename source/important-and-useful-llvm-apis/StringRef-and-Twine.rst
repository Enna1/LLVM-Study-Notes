StringRef & Twine
=================

在 LLVM 的 API 中为了高效地传递字符串，定义了 StringRef 类和 Twine 类。

StringRef
---------

StringRef 类的定义位于 llvm-5.0.1.src/include/llvm/ADT/StringRef.h

StringRef 类用于表示对常量字符串的一个引用，它支持我们常用的 std::string
所支持的那些操作，但是 StringRef 不需要动态内存分配( heap allocation )

因为 StringRef 类的对象所占用的空间足够小， 所以我们在使用 StringRef
时应该总是使用值传递的方式。

因为 StringRef
类中包含一个指向某个内存空间（常量字符串所在的内存空间）的一个指针，所以只有在保证指向的内存空间不会被释放的情况下，保存一个
StringRef 的对象才是安全的。否则可能会发生UAF ( Use After Free)。

StringRef 类的定义（省略了一些内容）如下：

.. code:: cpp

   class StringRef
   {
   public:
       static const size_t npos = ~size_t(0);

       using iterator = const char *;
       using const_iterator = const char *;
       using size_type = size_t;

   private:
       /// The start of the string, in an external buffer.
       const char *Data = nullptr;

       /// The length of the string.
       size_t Length = 0;

   public:
       ......
       
       /// Construct a string ref from an std::string.
       LLVM_ATTRIBUTE_ALWAYS_INLINE
       /*implicit*/ StringRef(const std::string &Str)
           : Data(Str.data()), Length(Str.length()) {}
       
       /// str - Get the contents as an std::string.
       LLVM_NODISCARD
       std::string str() const
       {
           if (!Data)
               return std::string();
           return std::string(Data, Length);
       }
       
       ......
   };

可以看到，StringRef 类有两个成员变量：Data 和 Length，Data是一个指向
const char 的指针，Length 用于存储字符串的长度。

与 std::string 类似，StringRef 支持 ``data``, ``empty``, ``size``,
``startswith``, ``endswith`` 等常用的函数；当然 StringRef 还支持一些
std::string 中没有的成员函数，如 ``equals`` , ``split``, ``trim`` 等。

StringRef 支持多种构造方式，可以通过 C style null-terminated string ,
std::string 或者通过指定 StringRef 的两个成员变量 Data 和 Length 来构造
StringRef 。

StringRef 还支持一个 ``str`` 函数，该函数返回一个 std::string ( 以
StringRef 的成员变量 Data 和 Length 作为参数调用 std::string
构造函数来得到) 。

在使用 StringRef 时，要注意一下几点限制：

1. 不能直接将一个 StringRef 类型的对象转换为一个 const char \*
   ，因为由于StringRef 成员变量 Length 的存在， StringRef
   所指向的字符串是可以包含“\\0”的，例如: ``StringRef("\0baz", 4)``
2. StringRef 不能控制其指向的常量字符串的生命周期，所以通常不应该以
   StringRef 的对象作为某一个类的成员变量
3. 同样地，如果一个函数的返回值是一个通过计算得到的字符串，那么该函数的返回值类型不应该用
   StringRef ，而应该使用std::string
4. StringRef 不允许对其指向的常量字符串的字节内容进行修改( mutate the
   pointed-to string bytes， insert or remove bytes from the range
   )，对于这样的操作，应使用Twine 与 StringRef 进行配合。

Twine
-----

Twine 类头文件位于
llvm-5.0.1.src/include/llvm/ADT/Twine.h，源文件位于llvm-5.0.1.src/lib/Support/Twine.cpp
。

Twine 类用于高效地表示字符串的拼接操作。比如在 LLVM 的 API
中一种常见范式就是以已有的一条指令的名称再加上一个后缀的方式为一条新的指令命名：

.. code:: cpp

   New = CmpInst::Create(..., SO->getName() + ".cmp");

Twine 类是一个高效且轻量的
`rope <https://en.wikipedia.org/wiki/Rope_(data_structure)>`__\ ，可以由字符串(C-strings,
std::string, StringRef )之间的 + 运算符的结果隐式构造得到(
上面的例子就是由 StringRef 和 C-strings 进行 + 运算后隐式得到 Twine
)。Twine
类只有在字符串之间的拼接结果被实际需要时，才真正执行拼接操作，因此可以避免由于对字符串拼接产生的临时结果进行构造所带来的堆内存分配操作的开销。

例如，下面的代码片段：

.. code:: cpp

   void foo(const Twine &T);
   ...
   StringRef X = ...
   unsigned i = ...
   foo(X + "." + Twine(i));

函数 ``foo`` 是由多个字符串拼接而来，假设 StringRef 指向的常量字符串是
“arg”, unsigned i 为 123，此拼接并不会构造出临时的中间字符串 “arg” 或者
“arg.”，而是只产生 “arg.123” 来作为函数 ``foo`` 的参数。

需要注意的是，因为 Twine
的内部结点（Twine是以二叉树实现的）是构造在栈上的，在该条语句（构造
Twine的那条语句）结束之后，Twine
对象就会被销毁，通常Twine只应该被用作函数的参数，而且应该以
``const Twine &`` 的方式被使用，如上面的示例代码。

下面的使用方式是错误的!!!：

.. code:: cpp

   void foo(const Twine &T);
   ...
   StringRef X = ...
   unsigned i = ...
   const Twine &Tmp = X + "." + Twine(i);
   foo(Tmp);

因为在Tmp作为函数 ``foo`` 的参数之前，已经被销毁。

关于Twine的源码实现。

首先是 Twine 的构造函数，Twine
有很多的构造函数，其中包含了支持隐式类型转换的构造函数：

.. code:: cpp

   /*implicit*/ Twine(const char *Str);
   /*implicit*/ Twine(const std::string &Str);
   /*implicit*/ Twine(const StringRef &Str);
   ...... //省略

Twine是以二叉树实现的，在Twine的内部使用枚举变量 enum
NodeKind来表示结点的可能的类型，因为结点可能类型有很多，所以使用 union
作为结点的值的类型。

.. code:: cpp

   class Twine
   {
       enum NodeKind : unsigned char
       {
           NullKind,
           EmptyKind,
           TwineKind,
           CStringKind,
           StdStringKind,
           StringRefKind,
           SmallStringKind,
           FormatvObjectKind,
           CharKind,
           DecUIKind,
           DecIKind,
           DecULKind,
           DecLKind,
           DecULLKind,
           DecLLKind,
           UHexKind
       };

       union Child {
           const Twine *twine;
           const char *cString;
           const std::string *stdString;
           const StringRef *stringRef;
           const SmallVectorImpl<char> *smallString;
           const formatv_object_base *formatvObject;
           char character;
           unsigned int decUI;
           int decI;
           const unsigned long *decUL;
           const long *decL;
           const unsigned long long *decULL;
           const long long *decLL;
           const uint64_t *uHex;
       };

       Child LHS;
       Child RHS;
       NodeKind LHSKind;
       NodeKind RHSKind;
       ...... // 省略
   };

我们重点关注一下，关于拼接的实现

.. code:: cpp

   inline Twine Twine::concat(const Twine &Suffix) const
   {
       // Concatenation with null is null.
       if (isNull() || Suffix.isNull())
           return Twine(NullKind);

       // Concatenation with empty yields the other side.
       if (isEmpty())
           return Suffix;
       if (Suffix.isEmpty())
           return *this;

       // Otherwise we need to create a new node, taking care to fold in unary
       // twines.
       Child NewLHS, NewRHS;
       NewLHS.twine = this;
       NewRHS.twine = &Suffix;
       NodeKind NewLHSKind = TwineKind, NewRHSKind = TwineKind;
       if (isUnary())
       {
           NewLHS = LHS;
           NewLHSKind = getLHSKind();
       }
       if (Suffix.isUnary())
       {
           NewRHS = Suffix.LHS;
           NewRHSKind = Suffix.getLHSKind();
       }

       return Twine(NewLHS, NewLHSKind, NewRHS, NewRHSKind);
   }

   inline Twine operator+(const Twine &LHS, const Twine &RHS)
   {
       return LHS.concat(RHS);
   }

实现拼接的是成员函数
``concat``\ ，很简单，就是将左操作数和右操作数分别作为新的 Twine
对象的左结点和右结点来构造一个新的 Twine
对象，对左操作数、左操作数只含有一个结点的情况做了特别处理。函数
``concat`` 只是构造了拼接后的字符串的 Twine 表示，并没有生成 std::string
。

如果要得到拼接后的字符串 std::string ，需要调用函数
``std::string Twine::str() const``
，该函数通过递归遍历左结点和右结点来产生实际的拼接结果 std::string。

参考链接：http://llvm.org/docs/ProgrammersManual.html#passing-strings-the-stringref-and-twine-classes
