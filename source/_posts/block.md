---
title: block
date: 2018-12-12 15:21:24
tags:
---

本篇简单探究一下block的一些底层原理：

* block的本质是什么
* __block的作用
* block的3种类型

## block的本质是什么
首先简单写一段block代码

``` cpp
int main(int argc, const char * argv[])
{
  void (^block)(void) = ^{
    printf("hello block");
  };
  block();
}
```

使用clang指令进行代码转化:

``` bash
$ clang -rewrite-objc test.m
```
转化成了这样:

``` cpp
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int flags=0) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

``` cpp
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {

  printf("hello block");
}
```

``` cpp
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
```

``` cpp
int main(int argc, const char * argv[])
{
  void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA));
  ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
}
```

看上去好乱，不着急，我们先去掉强制类型转化的括号.



``` cpp
int main(int argc, const char * argv[])
{
  void (*block)(void) = &__main_block_impl_0(__main_block_func_0, &__main_block_desc_0_DATA);
  (((__block_impl *)block)->FuncPtr)((__block_impl *)block);
}
```

这下清晰多了。我们可以看出一些东西:

* `block`就是一个结构体指针, 指向`__main_block_impl_0 `这个结构体
*  这个结构体的初始化需要一些参数
   * 代码块`__main_block_func_0 `函数的地址 传递给 `Funcptr`
   * 对`block`描述的`__main_block_desc_0`结构体类型的地址 传递给 `Desc`

我们来看一下是如何调用的。
首先将`block`强制转化为`__block_impl *`类型。我们知道结构体`__main_block_impl_0`的内存布局的第一个位置是`__block_impl`. 所以转化之后是没有任何变化.只会影响指针的步长. 我们来看一下`__block_impl`的结构体是什么样的。

``` cpp
struct __block_impl {
  void *isa;
  int Flags;
  int Reserved;
  void *FuncPtr;
};
```
从`__main_block_impl_0`的构造函数中，我们知道代码块`__main_block_func_0 `函数的地址传递给了 `FuncPtr`
所以我们拿到`FuncPtr`直接执行，就完成了block的调用.

## __block的作用
这里我们写几个例子实验一下.
### 基本数据类型 

``` cpp
int main(int argc, const char * argv[])
{
  int a = 0;
  void (^block)(void) = ^{
    printf("%d", a);
  };
  block();
}
```

clang指令之后:

``` cpp

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int a;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int _a, int flags=0) : a(_a) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

```

``` cpp
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int a = __cself->a; // bound by copy

  printf("%d", a);
}
```

``` cpp
static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};

```

``` cpp
int main(int argc, const char * argv[])
{
  int a = 0;
  void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, a));
  ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
}
```

注意一下代码块`__main_block_func_0 `函数中的执行:

* 传递参数`__main_block_impl_0 `结构体指针
* 后去`__main_block_impl_0 `中的成员变量a

在看一下`__main_block_impl_0 `结构体的变化

* 多了一个成员变量a.

所以我们知道基本数据类型在block中是值传递.

### 指针类型 

``` cpp
int main(int argc, const char * argv[])
{
  int* a = 0;
  void (^block)(void) = ^{
    printf("%d", a);
  };
  block();
}
```

clang指令之后:

代码块`__main_block_func_0 `函数

``` cpp
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int *a = __cself->a; // bound by copy

  printf("%d", a);
}

```

和`__main_block_impl_0 `结构体

``` cpp
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int *a;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_a, int flags=0) : a(_a) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```

我们可以看出指针类型是指针传递.

### __block

``` cpp
int main(int argc, const char * argv[])
{
  __block int a = 0;
  void (^block)(void) = ^{
    printf("%d", a);
  };
  block();
}
```

clang指令之后,这就精彩了

``` cpp

struct __Block_byref_a_0 {
  void *__isa;
  __Block_byref_a_0 *__forwarding;
  int __flags;
  int __size;
  int a;
};

```

``` cpp

struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  __Block_byref_a_0 *a; // by ref
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, __Block_byref_a_0 *_a, int flags=0) : a(_a->__forwarding) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};

```

``` cpp

static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  __Block_byref_a_0 *a = __cself->a; // bound by ref

  printf("%d", (a->__forwarding->a));
}

```

``` cpp

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};

```

``` cpp

int main(int argc, const char * argv[])
{
  __attribute__((__blocks__(byref))) __Block_byref_a_0 a = {(void*)0,(__Block_byref_a_0 *)&a, 0, sizeof(__Block_byref_a_0), 0};
  void (*block)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, (__Block_byref_a_0 *)&a, 570425344));
  ((void (*)(__block_impl *))((__block_impl *)block)->FuncPtr)((__block_impl *)block);
}

```

我们可以看出

* `__block`修饰的变量`a`被包装成了一个`__Block_byref_a_0 `结构体.
* `__Block_byref_a_0 `中的`__forwarding `指向`a`的地址.
* `__main_block_impl_0 `结构体多了一个变量`__Block_byref_a_0 `的指针.

结论:

* __block 可以将修饰的对象包装成结构体进行指针传递
* __block 是基本数据类型拥有指针类型传递的特性.


## block的3种类型


* StackBlock 栈区
* GlobalBlock 全局区
* MallocBlock 堆区










