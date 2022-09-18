# How Sanitizer Interceptor Works

我们在前面的文章中提到，所有的 Sanitizer 都由编译时插桩和运行时库两部分组成。

那么 sanitizer 的运行时库中做了哪些事情呢？以 ASan 为例，ASan 运行时库做的最主要的事情就是替换了 malloc/free, new/delete 的实现。这样应用程序的内存分配都由 ASan 实现的 allocator 来做，就能检测像 heap-use-after-free, double-free 这样的堆错误了。

其实不止是 malloc/free, new/delete，ASan 实际上替换非常多的函数实现，如：memcpy, memmove, strcpy, strcat, pthread_create 等等。

那么 sanitizer 是如何做到替换 malloc, free, mempcy 这些函数实现的呢？答案就是 sanitizer 中的 inteceptor 机制。

本文说明在 Linux X86_x64 环境下，sanitizer interceptor 的实现原理。

## Symbol interposition

在讲解 sanitizer interceptor 的实现原理之前，我们先来了解一下前置知识：symbol interposition。

首先我们考虑这样一个问题：如何在我们的应用程序中替换 libc 的 malloc 实现为我们自己实现的版本？

1. 一个最简单的方式就是在我们的应用程序中定义一个同名的 malloc 函数

2. 还有一种方式就是将我们的 malloc 函数实现在 libmymalloc.so 中，然后在运行我们的应用程序之前设置环境变量 `LD_PRELOAD=/path/to/libmymalloc.so`

那么为什么上述两种方式能生效呢？答案是 symbol interposition。

Dynamic loader 在 binding symbol references 时是以一种广度优先搜索的顺序来查找符号的：executable, needed0.so, needed1.so, needed2.so, needed0_of_needed0.so, needed1_of_needed0.so, ...

如果设置了 LD_PRELOAD，那么查找符号的顺序会变为：executable, preload0.so, preload1.so needed0.so, needed1.so, needed2.so, needed0_of_needed0.so, needed1_of_needed0.so, ...

如果一个符号在多个组件（executable 或 shared object）中都存在定义，那么 dynamic loader 会选择它所看到的第一个定义。

这样我们就理解为什么上述两种替换 malloc 的方式能生效了：

- 方式一，在查找符号时 executable 的顺序在 libc.so 之前，因此所有对 malloc 的引用都会绑定到 executable 中 malloc 的实现

- 方式二，在查找符号时 libmymalloc.so 的顺序在 libc.so 之前，因此所有对 malloc 的引用都会绑定到 libmymalloc.so 中 malloc 的实现。

---

实际上 sanitizer 对于 malloc/free 等库函数的替换正是利用了 symbol interposition 这一特性。下面我们以 ASan 为例来验证一下。

考虑如下代码：

```C++
// test.cpp
#include <iostream>
int main() {
    std::cout << "Hello AddressSanitizer!\n";
}
```

我们首先看下 GCC 的行为。

使用 GCC 开启 ASan 编译 test.cpp ，`g++ -fsanitize=address test.cpp -o test-gcc-asan` 得到编译产物 test-gcc-asan。因为 GCC 默认会**动态链接** ASan runtime library，所以我们可以使用 `ldd` 查看 test-gcc-asan 依赖的 shared objects：

```
$ ldd test-gcc-asan
    linux-vdso.so.1 (0x00007ffcfc740000)
    libasan.so.5 => /usr/lib/x86_64-linux-gnu/libasan.so.5 (0x00007f425d487000)
    libstdc++.so.6 => /usr/lib/x86_64-linux-gnu/libstdc++.so.6 (0x00007f425d303000)
    libm.so.6 => /lib/x86_64-linux-gnu/libm.so.6 (0x00007f425d180000)
    libgcc_s.so.1 => /lib/x86_64-linux-gnu/libgcc_s.so.1 (0x00007f425d166000)
    libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f425cfa5000)
    libdl.so.2 => /lib/x86_64-linux-gnu/libdl.so.2 (0x00007f425cfa0000)
    librt.so.1 => /lib/x86_64-linux-gnu/librt.so.1 (0x00007f425cf94000)
    libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007f425cf73000)
    /lib64/ld-linux-x86-64.so.2 (0x00007f425e26d000)
```

可以清楚的看到 libasan.so 的顺序是在 libc.so 之前的。实际上 `-fsanitize=address` 会使得 libasan.so 成为程序的第一个依赖库。

然后我们再通过环境变量 LD_DEBUG 看下 test-gcc-asan 的 symbol bindding 的过程：

```C++
$ LD_DEBUG="bindings" ./test-gcc-asan
   3309213:        binding file /lib/x86_64-linux-gnu/libc.so.6 [0] to /usr/lib/x86_64-linux-gnu/libasan.so.5 [0]: normal symbol `malloc' [GLIBC_2.2.5]
   3309213:        binding file /lib64/ld-linux-x86-64.so.2 [0] to /usr/lib/x86_64-linux-gnu/libasan.so.5 [0]: normal symbol `malloc' [GLIBC_2.2.5]
   3309213:        binding file /usr/lib/x86_64-linux-gnu/libstdc++.so.6 [0] to /usr/lib/x86_64-linux-gnu/libasan.so.5 [0]: normal symbol `malloc' [GLIBC_2.2.5]
```

可以看到 dynamic loader 将 libc.so, ld-linux-x86-64.so.2 和 libstdc++.so 中对 malloc 的引用都 bindding 到了 libasan.so 中的 malloc 实现。

---

下面我们看下 Clang，因为 Clang 默认是静态链接 ASan runtime library，所以我们就不看 test-clang-asan 所依赖的 shared objects 了，直接看 symbol bindding 的过程：

```C++
$ clang++ -fsanitize=address test.cpp -o test-clang-asan
$ LD_DEBUG="bindings" ./test-clang-asan
   3313022:        binding file /lib/x86_64-linux-gnu/libc.so.6 [0] to ./test-clang-asan [0]: normal symbol `malloc' [GLIBC_2.2.5]
   3313022:        binding file /lib64/ld-linux-x86-64.so.2 [0] to ./test-clang-asan [0]: normal symbol `malloc' [GLIBC_2.2.5]
   3313022:        binding file /usr/lib/x86_64-linux-gnu/libstdc++.so.6 [0] to ./test-clang-asan [0]: normal symbol `malloc' [GLIBC_2.2.5]
```

同样可以看到 dynamic loader 将 libc.so, ld-linux-x86-64.so.2 和 libstdc++.so 中对 malloc 的引用都 bindding 到了test-clang-asan 中的 malloc 实现（因为 ASan runtime library 中实现了 malloc，并且 clang 将 ASan runtime libaray 静态链接到 test-clang-asan 中）。

## Interceptor

下面我们来在源码的角度，学习下 sanitizer interceptor 的实现。

阅读学习 LLVM 代码的一个非常有效的方式就是结合对应的测试代码来学习。

Sanitizer interceptor 存在一个测试文件 interception_linux_test.cpp，[llvm-project/interception_linux_test.cpp at main · llvm/llvm-project · GitHub](https://github.com/llvm/llvm-project/blob/main/compiler-rt/lib/interception/tests/interception_linux_test.cpp)

```C++
#include "interception/interception.h"
#include "gtest/gtest.h"

static int InterceptorFunctionCalled;

DECLARE_REAL(int, isdigit, int);

INTERCEPTOR(int, isdigit, int d) {
  ++InterceptorFunctionCalled;
  return d >= '0' && d <= '9';
}

namespace __interception {

TEST(Interception, Basic) {
  EXPECT_TRUE(INTERCEPT_FUNCTION(isdigit));

  // After interception, the counter should be incremented.
  InterceptorFunctionCalled = 0;
  EXPECT_NE(0, isdigit('1'));
  EXPECT_EQ(1, InterceptorFunctionCalled);
  EXPECT_EQ(0, isdigit('a'));
  EXPECT_EQ(2, InterceptorFunctionCalled);

  // Calling the REAL function should not affect the counter.
  InterceptorFunctionCalled = 0;
  EXPECT_NE(0, REAL(isdigit)('1'));
  EXPECT_EQ(0, REAL(isdigit)('a'));
  EXPECT_EQ(0, InterceptorFunctionCalled);
}

}  // namespace __interception
```

这段测试代码基于 sanitizer 的 interceptor 机制替换了 `isdigit` 函数的实现，在我们实现的 `isdigit` 函数中，每次 `isdigit` 函数被调用时都将变量 `InterceptorFunctionCalled` 自增 1。然后通过检验变量 `InterceptorFunctionCalled` 的值来测试 interceptor 机制的实现是否正确，通过 `REAL(isdigit)` 来调用真正的 `isdigit` 函数实现。

上述测试文件 interception_linux_test.cpp 中实现替换 `isdigit` 函数的核心部分是如下代码片段：

```C++
DECLARE_REAL(int, isdigit, int);

INTERCEPTOR(int, isdigit, int d) {
  ++InterceptorFunctionCalled;
  return d >= '0' && d <= '9';
}

INTERCEPT_FUNCTION(isdigit);
```

这部分代码在宏展开后的内容如下：

```C++
// DECLARE_REAL(int, isdigit, int) 宏展开
typedef int (*isdigit_type)(int);
namespace __interception { extern isdigit_type real_isdigit; };

// INTERCEPTOR(int, isdigit, int d) 宏展开
typedef int (*isdigit_type)(int d);
namespace __interception { isdigit_type real_isdigit; } 
extern "C" int isdigit(int d) __attribute__((weak, alias("__interceptor_isdigit"), visibility("default")));
extern "C" __attribute__((visibility("default"))) int __interceptor_isdigit(int d) {
  ++InterceptorFunctionCalled;
  return d >= '0' && d <= '9';
}

// INTERCEPT_FUNCTION(isdigit) 宏展开
::__interception::InterceptFunction(
    "isdigit",
    (::__interception::uptr *) & __interception::real_isdigit,
    (::__interception::uptr) & (isdigit),
    (::__interception::uptr) & __interceptor_isdigit);

// REAL(isdigit)('1') 宏展开
__interception::real_isdigit('1');
```

- 我们首先看下 INTERCEPTOR 宏做了哪些事情
  
  - 首先在 __interception namespace 中定义了一个函数指针 real_isdigit，该函数指针实际上后续会被设置为指向真正的 `isdigit` 函数地址。
  
  - 然后将 `isdigit` 函数设置为 weak，并且将 `isdigit` 设置成 `__interceptor_isdigit` 的 alias 别名
  
  - 最后将我们自己版本的 `isdigit` 函数逻辑实现在 `__interceptor_isdigit` 函数中
  
  根据 symbol interposition 这一节的内容，我们知道：要想替换 libc.so 中 某个函数的实现（不妨把该函数称作 `foo`），只需要在 sanitizer runtime library 中定义同名 `foo` 函数，然后让 dynamic loader 在查找符号时 sanitizer runtime library 的顺序先于 libc.so 即可。
  
  那为什么这里要将我们的 `isdigit` 函数逻辑实现在函数 `__interceptor_isdigit` 中，并且将 `isdigit` 设置成 `__interceptor_isdigit` 的 alias 别名呢？
  
  考虑如下场景：假设用户代码中也替换了 `isdigit` 函数的实现，添加了自己的逻辑，那么最终 dynamic loader 选择的是用户代码中的 `isdigit` 的实现，而不是 sanitizer runtime library 中的 `isdigit` 的实现，这样的话 sanitizer 的功能就不能正常工作了。（实际上 sanitizer runtime library 中并没有替换 `isdigit` 的实现，这里只是用 `isdigit` 举例子便于说明）。
  
  但是如果我们在 sanitizer runtime library 中将 `isdigit` 设置成 `__interceptor_isdigit` 的 alias 别名，那么在用户代码中自己替换 `isdigit` 实现时就可以显示调用 `__interceptor_isdigit` 了。这样既不影响用户自行替换库函数，也不影响 sanitizer 功能的正确运行 ：
  
  ```cpp
  extern "C" int __interceptor_isdigit(int d);
  extern "C" int isdigit(int d) {
    fprintf(stderr, "my_isdigit_interceptor\n");
    return __interceptor_isdigit(d);
  }
  ```
  
  那在 sanitizer runtime library 中为什么将被替换的函数设置为 weak 呢？
  
  这是因为如果不设置为 weak ，在静态链接 sanitizer runtime library 时就会因为 multiple definition 而链接失败。

- 接着我们看下 INTERCEPT_FUNCTION 宏做了哪些事情
  
  INTERCEPT_FUNCTION 宏展开后就是对 __interception::InterceptFunction 函数的调用。`InterceptFunction` 函数的定义在 https://github.com/llvm/llvm-project/blob/main/compiler-rt/lib/interception/interception_linux.cpp：
  
  ```C++
  namespace __interception {
  static void *GetFuncAddr(const char *name, uptr wrapper_addr) {
    void *addr = dlsym(RTLD_NEXT, name);
    if (!addr) {
      // If the lookup using RTLD_NEXT failed, the sanitizer runtime library is
      // later in the library search order than the DSO that we are trying to
      // intercept, which means that we cannot intercept this function. We still
      // want the address of the real definition, though, so look it up using
      // RTLD_DEFAULT.
      addr = dlsym(RTLD_DEFAULT, name);
  
      // In case `name' is not loaded, dlsym ends up finding the actual wrapper.
      // We don't want to intercept the wrapper and have it point to itself.
      if ((uptr)addr == wrapper_addr)
        addr = nullptr;
    }
    return addr;
  }
  
  bool InterceptFunction(const char *name, uptr *ptr_to_real, uptr func,
                         uptr wrapper) {
    void *addr = GetFuncAddr(name, wrapper);
    *ptr_to_real = (uptr)addr;
    return addr && (func == wrapper);
  }
  }  // namespace __interception
  ```
  
  其实 `InterceptFunction` 函数的实现很简单：首先通过函数 `GetFuncAddr` 获得原本的名为 name 的函数地址，然后将该地址保存至指针 `ptr_to_real` 指向的内存。
  
  函数 `GetFuncAddr` 的代码实现也很简单，核心就是 [dlsym](https://man7.org/linux/man-pages/man3/dlsym.3.html)：
  
      RTLD_DEFAULT
          Find the first occurrence of the desired symbol using the
          default shared object search order.  The search will
          include global symbols in the executable and its
          dependencies, as well as symbols in shared objects that
          were dynamically loaded with the RTLD_GLOBAL flag.
      
      RTLD_NEXT
          Find the next occurrence of the desired symbol in the
          search order after the current object.  This allows one to
          provide a wrapper around a function in another shared
          object, so that, for example, the definition of a function
          in a preloaded shared object (see LD_PRELOAD in ld.so(8))
          can find and invoke the "real" function provided in
          another shared object (or for that matter, the "next"
          definition of the function in cases where there are
          multiple layers of preloading).
  
  这也是为什么在函数 `GetFuncAddr` 中，先用 `dlsym(RTLD_NEXT, name)` 去寻找被 intercepted 函数的真实地址，因为 sanitizer runtime library 是先于 name 函数真正所在的 shared object。

- 最后我们看下 DECLARE_REAL 宏 和 REAL 宏做了哪些事情
  
  DECLARE_REAL 展开后就是声明了在 __interception namespace 中存在一个指向被替换函数真正实现的函数指针，REAL 宏就是通过这个函数指针来调用被替换函数的真正实现。
  
  例如，在测试用例中，`DECLARE_REAL(int, isdigit, int);` 就是在声明 __interception namespace 中存在一个函数指针 `real_isdigit`，该函数指针指向真正的 `isdigit` 函数地址，通过 `REAL(isdigit)` 来调用真正的 `isdigit` 函数。

## P.S.

`__attribute__((alias))` 很有意思：

> Where a function is defined in the current translation unit, the alias call is replaced by a call to the function, and the alias is emitted alongside the original name. Where a function is not defined in the current translation unit, the alias call is replaced by a call to the real function. Where a function is defined as static, the function name is replaced by the alias name and the function is declared external if the alias name is declared external.

在 ASan runtime library 中 malloc 是 weak 符号，并且 malloc 和 __interceptor_malloc 实际指向同一个地址。

也就是说 `extern "C" void *malloc(size_t size) __attribute__((weak, alias("__interceptor_malloc"), visibility("default")));` 使得在 ASan runtime library 中造了一个弱符号 malloc，然后指向的和 __interceptor_malloc 是同一个地址。

```
$ readelf -sW --dyn-syms $(clang -print-file-name=libclang_rt.asan-x86_64.a) | grep malloc
  ...
  99: 0000000000001150   606 FUNC    GLOBAL DEFAULT    3 __interceptor_malloc
  102: 0000000000001150   606 FUNC    WEAK   DEFAULT    3 malloc

$ readelf -sW --dyn-syms $(clang -print-file-name=libclang_rt.asan-x86_64.so) | grep malloc
  ...
  3008: 00000000000fd600   606 FUNC    WEAK   DEFAULT   12 malloc
  4519: 00000000000fd600   606 FUNC    GLOBAL DEFAULT   12 __interceptor_malloc
```

## References

1. [ELF interposition and -Bsymbolic | MaskRay](https://maskray.me/blog/2021-05-16-elf-interposition-and-bsymbolic)

2. [dlsym(3) - Linux manual pagedlsym(3) - Linux manual page](https://man7.org/linux/man-pages/man3/dlsym.3.html)

3. [asan/tsan: weak interceptors · llvm/llvm-project@7fb7330 · GitHub](https://github.com/llvm/llvm-project/commit/7fb7330469af52ae1313b2b47c273e62c61a4dd5)
