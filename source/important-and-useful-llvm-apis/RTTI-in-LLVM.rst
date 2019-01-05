RTTI in LLVM
============

不管是阅读LLVM/Clang的源代码，还是基于LLVM/Clang自己动手写一些代码时，最常用到的就是LLVM中的RTTI了，也就是
``isa<>`` , ``cast<>`` 和 ``dyn_cast<>`` ，当然还有 ``cast_or_null<>``
和 ``dyn_cast_or_null<>``\ ，不过我自己好像不怎么常用这两个……

下面给一个使用 ``dyn_cast<>``
的例子，在这个例子中，我们遍历函数中的所有指令，并对其中的 CallInst
指令进行一些处理：

.. code:: c

   for (inst_iterator i = inst_begin(F), e = inst_end(F); i != e; ++i)
   {
       Instruction *I = &(*i);
       if (auto *CI = dyn_cast<CallInst>(I))
       {
           /* do something*/
       }
   }

熟悉 LLVM 的应该知道（不熟悉 LLVM 的，也能从 CallInst类 和 Instruction类
的名称推测出来）CallInst 类是继承自 Instruction 类的。上述代码就是使用
``dyn_cast<>`` 将 Instruction 的对象 cast 成 CallInst 对象，如果一条
Instruction 是 CallInst ，那么 CI 就不是空指针 nullptr ，会执行do
something。

下面进入正文，\ ``isa<>`` , ``cast<>`` 和 ``dyn_cast<>``
到底是怎么实现的。

在 LLVM-5.0.1 中，他们的实现代码位于
llvm-5.0.1.src/include/llvm/Support/Casting.h 中。

注：\ ``isa<>`` , ``cast<>`` 和 ``dyn_cast<>``
的实现中关于模板的代码等我学了模板再填坑。

isa<>的实现
-----------

可以看到 ``isa<>`` 的实现依赖于依赖于 ``classof`` 函数。

.. code:: c

   // The core of the implementation of isa<X> is here; To and From should be
   // the names of classes.  This template can be specialized to customize the
   // implementation of isa<> without rewriting it from scratch.
   template <typename To, typename From, typename Enabler = void>
   struct isa_impl
   {
       static inline bool doit(const From &Val)
       {
           return To::classof(&Val);
       }
   };

我们以 Value 类和 Argument 类为例来进行说明，Argument 类是由 Value
继承而来。

在 Argument 类的头文件( llvm-5.0.1.src/include/llvm/IR/Argument.h
)中，我们可以找到 ``classof`` 函数，可以看到注释， ``classof``
函数就是用于支持 LLVM 中的 RTTI 的。

.. code:: c

   /// Method for support type inquiry through isa, cast, and dyn_cast.
   static bool classof(const Value *V)
   {
       return V->getValueID() == ArgumentVal;
   }

Value 类的实现中与 ``classof`` 函数相关的内容如下：

.. code:: c

   class Value
   {
       const unsigned char SubclassID;  // Subclass identifier (for isa/dyn_cast)
       /// An enumeration for keeping track of the concrete subclass of Value that
       /// is actually instantiated. Values of this enumeration are kept in the
       /// Value classes SubclassID field. They are used for concrete type
       /// identification.
       enum ValueTy
       {
   #define HANDLE_VALUE(Name) Name##Val,
   #include "llvm/IR/Value.def"

       // Markers:
   #define HANDLE_CONSTANT_MARKER(Marker, Constant) Marker = Constant##Val,
   #include "llvm/IR/Value.def"
       };
       /// This is used to implement the classof checks.  This should not be used
       /// for any other purpose, as the values may change as LLVM evolves.  Also,
       /// note that for instructions, the Instruction's opcode is added to
       /// InstructionVal. So this means three things:
       /// # there is no value with code InstructionVal (no opcode==0).
       /// # there are more possible values for the value type than in ValueTy
       /// enum. # the InstructionVal enumerator must be the highest valued
       /// enumerator in
       ///   the ValueTy enum.
       unsigned getValueID() const
       {
           return SubclassID;
       }
       ...
   };

在 Value 类内部定义了一个枚举变量 ValueTy，通过 ``HANDLE_VALUE``
宏和文件 Value.def 配合来定义各种ValueTy
中的枚举常量，我们可以在其中看到枚举常量 ``ArgumentVal`` 的定义方式。

.. code:: c

   HANDLE_VALUE(Argument)

所以在 enum ValueTy 中的内容相当于，省略了 ArgumentVal外的其他枚举常量：

.. code:: c

   enum ValueTy
   {
       ...
       ArgumentVal,
       ...
   };

然后我们看 Argument 类的构造函数的定义(
llvm-5.0.1.src/include/llvm/lib/Argument.cpp ) 和 Value
类构造函数的定义( llvm-5.0.1.src/include/llvm/lib/Value.cpp )

.. code:: c

   Argument::Argument(Type *Ty, const Twine &Name, Function *Par, unsigned ArgNo)
       : Value(Ty, Value::ArgumentVal), Parent(Par), ArgNo(ArgNo)
   {
       ...
   }

   Value::Value(Type *ty, unsigned scid)
       : VTy(checkType(ty)),
         UseList(nullptr),
         SubclassID(scid),
         HasValueHandle(0),
         SubclassOptionalData(0),
         SubclassData(0),
         NumUserOperands(0),
         IsUsedByMD(false),
         HasName(false)
   {
      ...
   }

我们重点关注的是，当构造一个 Argument 类的对象时，会手动调用基类 Value
的构造函数并且传给 Value 构造函数的第二个参数是 Value::ArgumentVal ，而
Value 构造函数会把它成员变量 SubclassID
的值设置为其第二个参数的值。所以如果有一个 Argument
类的对象，然后我们拿到的是指向该 Argument 对象的 Value 类型的指针 V
时，我们我们以该指针作为参数调用 ``isa<Argument>(V)`` 时，会返回
``Argument::classof(V)``
的值，而前面我们看到，\ ``Argument::classof(V)`` 的值就是
``return V->getValueID() == ArgumentVal;`` ，因为在构造该 Argument
对象时，已经将其基类 Value 的 SubclassID 设置为 ArgumentVal
，所以最后会返回true，即指针 V 指向的对象是一个 Argument 类型的对象。

cast<>的实现
------------

.. code:: c

   // cast<X> - Return the argument parameter cast to the specified type.  This
   // casting operator asserts that the type is correct, so it does not return null
   // on failure.  It does not allow a null argument (use cast_or_null for that).
   // It is typically used like this:
   //
   //  cast<Instruction>(myVal)->getParent()
   //
   template <class X, class Y>
   inline typename std::enable_if<!is_simple_type<Y>::value,
                                  typename cast_retty<X, const Y>::ret_type>::type
   cast(const Y &Val)
   {
       assert(isa<X>(Val) && "cast<Ty>() argument of incompatible type!");
       return cast_convert_val<
           X, const Y, typename simplify_type<const Y>::SimpleType>::doit(Val);
   }

   template <class X, class Y>
   inline typename cast_retty<X, Y>::ret_type cast(Y &Val)
   {
       assert(isa<X>(Val) && "cast<Ty>() argument of incompatible type!");
       return cast_convert_val<X, Y, typename simplify_type<Y>::SimpleType>::doit(
           Val);
   }

可以看到 ``cast<>`` 的实现依赖于 ``cast_convert_val::doit``
函数，其定义如下。

.. code:: c

   template <class To, class FromTy>
   struct cast_convert_val<To, FromTy, FromTy>
   {
       // This _is_ a simple type, just cast it.
       static typename cast_retty<To, FromTy>::ret_type doit(const FromTy &Val)
       {
           typename cast_retty<To, FromTy>::ret_type Res2 =
               (typename cast_retty<To, FromTy>::ret_type) const_cast<FromTy &>(
                   Val);
           return Res2;
       }
   };

先使用 C++ ``const_cast`` 然后对 ``const_cast``
的结果进行C风格的强制类型转换。

dyn_cast<>的实现
----------------

.. code:: c

   // dyn_cast<X> - Return the argument parameter cast to the specified type.  This
   // casting operator returns null if the argument is of the wrong type, so it can
   // be used to test for a type as well as cast if successful.  This should be
   // used in the context of an if statement like this:
   //
   //  if (const Instruction *I = dyn_cast<Instruction>(myVal)) { ... }
   //

   template <class X, class Y>
   LLVM_NODISCARD inline
       typename std::enable_if<!is_simple_type<Y>::value,
                               typename cast_retty<X, const Y>::ret_type>::type
       dyn_cast(const Y &Val)
   {
       return isa<X>(Val) ? cast<X>(Val) : nullptr;
   }

可以看到 ``dyn_cast`` 的是通过三元运算符实现的，如果 ``isa<X>(val)``
返回 true (val是 X 类的一个对象)，则将 val ``cast`` 为 X
类后返回，否则返回空指针 nullptr 。

让LLVM-style RTTI支持自己的编写的类
-----------------------------------

假设要编写如下继承关系的类

::

   | Shape
     | Square
       | SpecialSquare
     | Circle

了解了 LLVM 中 RTTI 的实现，我们想要让其支持自己编写的类，模仿 Value 类
和 Argument
类的写法，声明一个枚举变量，在子类构造函数中显示调用父类构造函数并传递给父类构造函数一个表示子类类型的枚举常量，还有需要定义
``classof`` 函数。

具体的实现代码如下：

.. code:: cpp

   #include "llvm/Support/Casting.h"
   #include <iostream>
   #include <vector>
   using namespace llvm;

   class Shape
   {
   public:
       // 类似class Value 中 enum ValueTy的定义
       enum ShapeKind
       {
           /* Square Kind Begin */
           SK_SQUARE,
           SK_SEPCIALSQUARE,
           /* Square Kind end */
           SK_CIRCLE,
       };

   private:
       const ShapeKind kind_;

   public:
       Shape(ShapeKind kind) : kind_(kind) {}

       ShapeKind getKind() const
       {
           return kind_;
       }

       virtual double computeArea() = 0;
   };

   class Square : public Shape
   {
   public:
       double side_length_;

   public:
       Square(double side_length) : Shape(SK_SQUARE), side_length_(side_length) {}

       Square(ShapeKind kind, double side_length)
           : Shape(kind), side_length_(side_length)
       {
       }

       double computeArea() override
       {
           return side_length_ * side_length_;
       }

       static bool classof(const Shape *s)
       {
           return s->getKind() >= SK_SQUARE && s->getKind() <= SK_SEPCIALSQUARE;
       }
   };

   class SepcialSquare : public Square
   {
   public:
       double another_side_length_;

   public:
       SepcialSquare(double side_length, double another_side_length)
           : Square(SK_SEPCIALSQUARE, side_length),
             another_side_length_(another_side_length)
       {
       }

       double computeArea() override
       {
           return side_length_ * another_side_length_;
       }

       static bool classof(const Shape *s)
       {
           return s->getKind() == SK_SEPCIALSQUARE;
       }
   };

   class Circle : public Shape
   {
   public:
       double radius_;

   public:
       Circle(double radius) : Shape(SK_CIRCLE), radius_(radius) {}

       double computeArea() override
       {
           return 3.14 * radius_ * radius_;
       }

       static bool classof(const Shape *s)
       {
           return s->getKind() == SK_CIRCLE;
       }
   };

   int main()
   {
       Square s1(1);
       SepcialSquare s2(1, 2);
       Circle s3(3);
       std::vector<Shape *> v{ &s1, &s2, &s3 };
       for (auto i : v)
       {
           if (auto *S = dyn_cast<Square>(i))
           {
               std::cout << "This is a Square object\n";
               std::cout << "Area is : " << S->computeArea() << "\n";
           }
           if (auto *SS = dyn_cast<SepcialSquare>(i))
           {
               std::cout << "This is a SepcialSquare object\n";
               std::cout << "Area is : " << SS->computeArea() << "\n";
           }
           if (auto *C = dyn_cast<Circle>(i))
           {
               std::cout << "This is a Circle object\n";
               std::cout << "Area is : " << C->computeArea() << "\n";
           }
           std::cout << "-----\n";
       }

       return 0;
   }

参考链接：

1. http://llvm.org/docs/ProgrammersManual.html#the-isa-cast-and-dyn-cast-templates
2. http://llvm.org/docs/HowToSetUpLLVMStyleRTTI.html
3. https://stackoverflow.com/questions/6038330/how-is-llvm-isa-implemented
