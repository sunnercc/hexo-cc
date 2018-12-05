---
title: swift-protocol-extentsion
date: 2017-08-05 11:13:17
tags:
---

协议在我的理解中就是一个定义好的规范，相当于C++的抽象类，Java的接口。定义好这个规范之后，其他类可以去遵循这个协议，去实现协议中的方法和属性。但是如果有很多类对协议方法的方法体都一样，这样就会写大量重复代码，浪费实现和精力，这个时候协议的extension就起作用了，extension可以给协议扩展一个协议方法的默认实现，这样其他类就可以不用实现了，使用extension中的默认实现就可以了。


``` swift
import UIKit

protocol Motion {
    
    /// 重力感应 透视
    func motion(effectOffset: CGFloat, target: UIView)
}

extension Motion {
    
    /// 重力感应 透视
    func motion(effectOffset: CGFloat, target: UIView) {
        
        let motionX = UIInterpolatingMotionEffect(keyPath: "center.x", type: .tiltAlongHorizontalAxis)
        motionX.maximumRelativeValue = effectOffset
        motionX.minimumRelativeValue = -effectOffset
        
        let motionY = UIInterpolatingMotionEffect(keyPath: "center.y", type: .tiltAlongVerticalAxis)
        motionY.maximumRelativeValue = effectOffset
        motionY.minimumRelativeValue = -effectOffset
        
        let motionGroup = UIMotionEffectGroup()
        motionGroup.motionEffects = [motionX, motionY]
        
        target.addMotionEffect(motionGroup)
    }
}

```
