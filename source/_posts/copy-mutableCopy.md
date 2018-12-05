---
title: copy&mutableCopy
date: 2016-08-05 11:12:26
tags:
---

对于熟悉iOS开发的人的来说，copy和mutableCopy这两个词肯定不陌生。但是区别是什么呢，百度一搜一大堆，但是解释的千奇百怪，毫无真相。

## 首先明白两个概念

* 浅拷贝: 也称为浅复制。不开辟新的内存空间。仅仅是拷贝指向对象的指针。
* 深拷贝: 也称为深复制。开辟新的内存空间。拷贝整个对象内存到新开辟的内存中去。

## 解释一个英文单词

* mutable 可变的

iOS开发中最常见的类就是NSString, 还有一些容器类(NSArray, NSDictionary, NSSet).我会选取经典的NSString以及容器类的代表NSArray来写4个例子。得出实验结论。

## 例子1

针对NSSting 写出下列测试代码：

``` objc
NSString *str1 = @"123";  // 常量区分配
NSString *str2 = [str1 copy];
NSMutableString *str3 = [str1 mutableCopy];
printf("\n str1 -> = %p \n str2 -> = %p \n str3 -> = %p", str1, str2, str3);
```

控制台的打印如下

``` bash
str1 -> = 0x1072d9088
str2 -> = 0x1072d9088
str3 -> = 0x600001c2c000
```

可以看出：

* str1指向的地址和str2指向的地址是同一个地址.str2并没有开辟新的空间,仅仅是指针的拷贝.
* str1指向的地址和str3指向的地址不是同一个地址。str3开辟了新的空间.

## 例子2

针对NSMutableString 写出下列测试代码：

``` objc
NSString *str1 = @"123"; // 常量区分配
NSMutableString *m_str1 = [NSMutableString stringWithString:str1];  // 堆区重新分配
NSString *m_str2 = [m_str1 copy];
NSMutableString *m_str3 = [m_str1 mutableCopy];
printf("\n m_str1 -> = %p \n m_str2 -> = %p \n m_str3 -> = %p", m_str1, m_str2, m_str3);
```

控制台的打印如下

``` bash
m_str1 -> = 0x600003143b40
m_str2 -> = 0xbe9e1c56d14b79e2
m_str3 -> = 0x600003143e70
```

可以看出：

* m_str1指向的地址和m_str2指向的地址不是同一个地址. m_str2开辟了新的空间.
* m_str1指向的地址和m_str3指向的地址不是同一个地址。m_str3开辟了新的空间.

## 例子3

针对容器类NSArray 写出下列测试代码：

``` objc
NSArray *arr1 = @[@1, @2, @3];
NSArray *arr2 = [arr1 copy];
NSMutableArray *arr3 = [arr1 mutableCopy];
printf("\n arr1 -> = %p \n arr2 -> = %p \n arr3 -> = %p", arr1, arr2, arr3);
printf("\n arr1 items:");
for (int i = 0; i < arr1.count; i++) {
    printf("\n %d -> = %p", i, arr1[i]);
}
printf("\n arr2 items:");
for (int i = 0; i < arr2.count; i++) {
    printf("\n %d -> = %p", i, arr2[i]);
}
printf("\n arr3 items:");
for (int i = 0; i < arr3.count; i++) {
    printf("\n %d -> = %p", i, arr3[i]);
}
```

控制台的打印如下

``` bash
arr1 -> = 0x600001490090
arr2 -> = 0x600001490090
arr3 -> = 0x6000014900c0
arr1 items:
0 -> = 0xb690da9c196700a8
1 -> = 0xb690da9c19670098
2 -> = 0xb690da9c19670088
arr2 items:
0 -> = 0xb690da9c196700a8
1 -> = 0xb690da9c19670098
2 -> = 0xb690da9c19670088
arr3 items:
0 -> = 0xb690da9c196700a8
1 -> = 0xb690da9c19670098
2 -> = 0xb690da9c19670088
```

可以看出：

* arr1指向的地址和arr2指向的地址是同一个地址.str2并没有开辟新的空间,仅仅是指针的拷贝.
* arr1中的子元素指向的地址和arr2中的子元素指向的地址是同一个地址, 子元素没有开辟内存，仅仅是指针的拷贝
* arr1指向的地址和str3指向的地址不是同一个地址。arr3开辟了新的空间.
* arr1中的子元素指向的地址和arr3中的子元素指向的地址是同一个地址, 子元素没有开辟内存，仅仅是指针的拷贝

## 例子4

针对容器类NSMutableArray 写出下列测试代码：

``` objc
NSArray *arr1 = @[@1, @2, @3];
NSMutableArray *m_arr1 = [NSMutableArray arrayWithArray:arr1];
NSArray *m_arr2 = [arr1 copy];
NSMutableArray *m_arr3 = [arr1 mutableCopy];
printf("\n m_arr1 -> = %p \n m_arr2 -> = %p \n m_arr3 -> = %p", m_arr1, m_arr2, m_arr3);
printf("\n m_arr1 items:");
for (int i = 0; i < m_arr1.count; i++) {
    printf("\n %d -> = %p", i, m_arr1[i]);
}
printf("\n m_arr2 items:");
for (int i = 0; i < m_arr2.count; i++) {
    printf("\n %d -> = %p", i, m_arr2[i]);
}
printf("\n m_arr3 items:");
for (int i = 0; i < m_arr3.count; i++) {
    printf("\n %d -> = %p", i, m_arr3[i]);
}
```

控制台的打印如下

``` bash
m_arr1 -> = 0x6000014900c0
m_arr2 -> = 0x600001490090
m_arr3 -> = 0x600001490030
m_arr1 items:
0 -> = 0xb690da9c196700a8
1 -> = 0xb690da9c19670098
2 -> = 0xb690da9c19670088
m_arr2 items:
0 -> = 0xb690da9c196700a8
1 -> = 0xb690da9c19670098
2 -> = 0xb690da9c19670088
m_arr3 items:
0 -> = 0xb690da9c196700a8
1 -> = 0xb690da9c19670098
2 -> = 0xb690da9c19670088
```

可以看出

* m_arr1指向的地址和m_str2指向的地址是同一个地址. arr2开辟了新的空间.
* m_arr1中的子元素指向的地址和m_arr2中的子元素指向的地址是同一个地址, 子元素没有开辟内存，仅仅是指针的拷贝
* m_str1指向的地址和m_str3指向的地址不是同一个地址。str3开辟了新的空间
* m_arr1中的子元素指向的地址和m_arr3中的子元素指向的地址是同一个地址, 子元素没有开辟内存，仅仅是指针的拷贝


## 结论

* copy产生不可变类型
 * 不可变类型 -> copy -> 不可变类型 : 不开辟空间
 * 可变类型 -> copy -> 不可变类型 : 开辟空间
 * 容器类中的子元素仅仅指针拷贝,不开辟空间
* mutableCopy产生可变类型
 * 不可变类型 -> mutableCopy -> 可变类型 : 开辟空间
 * 可变类型 -> copy -> 可变类型 : 开辟空间
 * 容器类中的子元素仅仅指针拷贝,不开辟空间



## 代码

[code](https://github.com/sunnercc/Program/tree/master/copy-mutableCopy)



    




