---
title: swift循环引用
date: 2017-10-05 11:13:29
tags:
---

### weak
在我理解中，weak和oc中的weak没有任何区别，弱引用，引用基数不会加1，当引用的对象释放的时候，指针会被置为nil、避免僵尸对象的产生
### unowned
这个很有意思，相当于oc中unsafe-unretain，也就是不安全不retain, 不安全就是当引用的对象释放的时候，指针不会被置为nil，会产生也僵尸对象。不retain也就是弱引用，引用计数不会加1.

苹果有一个customer和card的例子,一个customer不一定拥有card，所以customer使用可选类型的strong指针引用card，一个card一定联系一个customer，所以card使用unowned引用customer。
> “The relationship between Customer and CreditCard is slightly different from the relationship between Apartment and Person seen in the weak reference example above. In this data model, a customer may or may not have a credit card, but a credit card will always be associated with a customer. A CreditCard instance never outlives the Customer that it refers to. To represent this, the Customer class has an optional card property, but the CreditCard class has an unowned (and nonoptional) customer property.”

``` swift
class Customer {
    let name: String
    var card: CreditCard?
    init(name: String) {
        self.name = name
    }
    deinit { print("\(name) is being deinitialized") }
}
 
class CreditCard {
    let number: UInt64
    unowned let customer: Customer
    init(number: UInt64, customer: Customer) {
        self.number = number
        self.customer = customer
    }
    deinit { print("Card #\(number) is being deinitialized") }
}

```

![](swift循环引用/swiftUnown.png)

