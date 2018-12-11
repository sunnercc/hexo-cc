---
title: autoreleasePoll
date: 2018-06-05 11:11:00
tags:

---
苹果的内存管理机制(ARC)中有一个很有趣的东西-autoreleasePoll.可以延迟释放对象.抽出点时间来探究一下, 探究如下内容:

* @autoreleasePoll{}的本质是什么
* autoreleasePoll是通过什么对象来实现的
* AutoreleasePoolPage的数据结构
* AutoreleasePoolPage如何实现autorelease对象的存储
* AutoreleasePoolPage如何释放对象
* POOL_BOUNDARY (哨兵对象)是什么

## @autoreleasePoll{}的本质是什么

首先写一段使用了autoreleasepoll的代码, main.m：

``` objc 
int main(int argc, const char * argv[])
{
  @autoreleasepool {
    NSLog(@"hello world");
  }
  return 0;
}
```

然后使用终端命令编译成c++代码 main.cpp：

``` bash
$ clang -rewrite-objc main.m
```

我在main.cpp中的最后面找到了下面这段代码:

``` cpp
int main(int argc, const char * argv[])
{
  /* @autoreleasepool */ { __AtAutoreleasePool __autoreleasepool; 
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_98_x72bq3s93xb7g9w9qjqnrj8r0000gn_T_main_9f94b6_mi_0);
  }
  return 0;
}
```


``` cpp
struct __AtAutoreleasePool {
  __AtAutoreleasePool() {atautoreleasepoolobj = objc_autoreleasePoolPush();}
  ~__AtAutoreleasePool() {objc_autoreleasePoolPop(atautoreleasepoolobj);}
  void * atautoreleasepoolobj;
};
```

苹果在编译器将代码编译成了如上代码.在大括号开始的时候生成`__AtAutoreleasePool`对象, 会执行初始化方法`__AtAutoreleasePool()`,在大括号结束的时候，会释放 `__AtAutoreleasePool`对象， 会执行析构方法`~__AtAutoreleasePool()`.

## autoreleasePoll是通过什么对象来实现的

通过上述的`__AtAutoreleasePool `的结构体.我们知道:

* objc_autoreleasePoolPush() 生成atautoreleasepoolobj对象.


``` objc 
void *
objc_autoreleasePoolPush(void)
{
  return AutoreleasePoolPage::push();
}
```
* objc_autoreleasePoolPop() 释放atautoreleasepoolobj对象.

``` cpp
void
objc_autoreleasePoolPop(void *ctxt)
{
  AutoreleasePoolPage::pop(ctxt);
}
```

我们知道了`autoreleasePoll `的实现肯定和`AutoreleasePoolPage `有关系

## AutoreleasePoolPage的数据结构
下面我们来看一下`AutoreleasePoolPage `类定义，了解一下他的数据结构是怎样的.

``` cpp
class AutoreleasePoolPage 
{
# define POOL_BOUNDARY nil
  static size_t const SIZE = PAGE_MAX_SIZE
  magic_t const magic;
  id *next;
  pthread_t const thread;
  AutoreleasePoolPage * const parent;
  AutoreleasePoolPage *child;
  uint32_t const depth;
  uint32_t hiwat;
}
```

* POOL_BOUNDARY : 哨兵对象
* SIZE : AutoreleasePoolPage的最大容量 (4096)
* magic : 完整性校验
* thread : 保存了当前page所在的线程
* next :  存储autorelease对象
* parent : 指向父节点的指针
* child : 指向子节点的指针

可以看出，AutoreleasePoolPage是一个双向链表

![](/autoreleasePoll/autoreleasepoll0.png)

## AutoreleasePoolPage如何实现autorelease对象的存储

要想知道如何实现`autorelease `对象的存储，需要知道`AutoreleasePoolPage `的`push`方法的实现:

``` cpp
static inline void *push() 
{
  return autoreleaseFast(POOL_BOUNDARY);
}
```

这里看一下autoreleaseFast这个方法。

``` cpp
static inline id *autoreleaseFast(id obj)
{
  AutoreleasePoolPage *page = hotPage();
  if (page && !page->full()) {
    return page->add(obj);
  } else if (page) {
    return autoreleaseFullPage(obj, page);
  } else {
    return autoreleaseNoPage(obj);
  }
}

```

这里分为了三种情况

* page存在，并且page没有满。 `return page->add(obj);`
* page存在，但是page满了。 `return autoreleaseFullPage(obj, page);`
* page不存在。 `return autoreleaseNoPage(obj);`

###第一种情况:

``` cpp
id *add(id obj)
{
  unprotect();
  id *ret = next;    
  *next++ = obj;
  protect();
  return ret;
}
```

其中`*next++ = obj` 这句话的意思是: next指针指向obj,然后next指针+1.说明苹果是通过这种方式将autorelease对象存储的。
###第二种情况:

``` cpp
static __attribute__((noinline))
id *autoreleaseFullPage(id obj, AutoreleasePoolPage *page)
{
  do {
  if (page->child) page = page->child;
    else page = new AutoreleasePoolPage(page);
  } while (page->full());

  setHotPage(page);
  return page->add(obj);
}
```

这段代码的大概意思是: 通过一个接一个的节点来寻找没有满的page。如果page满了就新建一个page。把新建的page设置为hotpage, 然后将autorelase对象存储page中。

###第三种情况:

``` cpp
static __attribute__((noinline))
id *autoreleaseNoPage(id obj)
{

  bool pushExtraBoundary = false;
  if (haveEmptyPoolPlaceholder()) {
    pushExtraBoundary = true;
  }
  else if (obj != POOL_BOUNDARY  &&  DebugMissingPools) {
    // We are pushing an object with no pool in place, 
    // and no-pool debugging was requested by environment.
    _objc_inform("MISSING POOLS: (%p) Object %p of class %s "
    "autoreleased with no pool in place - "
    "objc_autoreleaseNoPool() to debug", 
    pthread_self(), (void*)obj, object_getClassName(obj));
    objc_autoreleaseNoPool(obj);
    return nil;
  }
  else if (obj == POOL_BOUNDARY  &&  !DebugPoolAllocation) {
    // We are pushing a pool with no pool in place,
    // and alloc-per-pool debugging was not requested.
    // Install and return the empty pool placeholder.
    return setEmptyPoolPlaceholder();
  }

  AutoreleasePoolPage *page = new AutoreleasePoolPage(nil);
  setHotPage(page);

  if (pushExtraBoundary) {
    page->add(POOL_BOUNDARY);
  }

  return page->add(obj);
}
```
这段代码的意思是新建一个page，将page设置为hotpage，然后将autorelease对象存入page中。

通过对`push`方法的分析，我们了解了 `AutoreleasePoolPage `是通过next指针来实现`autorelease`对象的存储.

`AutoreleasePoolPage `的初始状态是这样的

![](/autoreleasePoll/autoreleasepoll1.png)

加入一个`autorelease`对象

![](/autoreleasePoll/autoreleasepoll2.png)


## AutoreleasePoolPage如何释放对象

下面我们来看看objc_autoreleasePoolPop的过程.

``` cpp
void
objc_autoreleasePoolPop(void *ctxt)
{
  AutoreleasePoolPage::pop(ctxt);
}
```
这里我们知道objc_autoreleasePoolPop是通过AutoreleasePoolPage类的pop方法实现的. 传入的参数就是需要释放的AutoreleasePoolPage对象.

``` cpp
static inline void pop(void *token) 
{
  AutoreleasePoolPage *page;
  id *stop;
  stop = (id *)token;
  page->releaseUntil(stop);
}
```
这里我们关注一下 `page->releaseUntil(stop);`

``` cpp
void releaseUntil(id *stop) 
{
  while (this->next != stop) {

    AutoreleasePoolPage *page = hotPage();
    while (page->empty()) {
      page = page->parent;
      setHotPage(page);
    }

    page->unprotect();
    id obj = *--page->next;
    memset((void*)page->next, SCRIBBLE, sizeof(*page->next));
    page->protect();

    if (obj != POOL_BOUNDARY) {
      objc_release(obj);
    }
  }
  setHotPage(this);
}
```

这段代码的意思是，从page中一个一个的对autorelase对象发送release消息。终止的临界条件是POOL_BOUNDARY.

![](/autoreleasePoll/autoreleasepoll3.png)

## POOL_BOUNDARY (哨兵对象)是什么

我们知道`AutoreleasePoolPage `的类定义中有一段代码:

``` cpp
#define POOL_BOUNDARY nil
```
* `AutoreleasePoolPage `调用`push`方法的时候，会把`POOL_BOUNDARY `放入自动自动释放池的栈顶.

* `AutoreleasePoolPage `调用`pop`方法的时候，会对realease对象发送消息直到`POOL_BOUNDARY `.

所以: `POOL_BOUNDARY `是每一次@autoreleasepoll{}的标记.











