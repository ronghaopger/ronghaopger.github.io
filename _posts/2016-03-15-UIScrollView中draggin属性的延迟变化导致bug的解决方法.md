---
layout:     post
title:      "UIScrollView中draggin属性的延迟变化导致bug的解决方法"
subtitle:   ""
date:       2016-03-15 15:30:00
author:     "荣浩"
header-img: "img/post-bg-apple.jpg"
tags:
    - Swift | iOS
---

最近开始用Swift写项目，就先自己封装了下拉上拉的库[EasyPull](https://github.com/ronghaopger/EasyPull)上传到GitHub，期间跟网友交流解决了一些没考虑到的bug，现在库已经基本稳定了，我已经应用到生产中了。

但是，在我不断的测试中，发现一个很隐晦的bug：在拉动到**临界位置**左右释放后，有时不能触发刷新。随即我看了看手机上的常用应用，发现天猫、美团、今日头条等都存在这个问题，好奇之心涌上心头，一番debug后，发现罪魁祸首是UIScrollView的`draggin`属性，官方的API有介绍：

``` swift
public var dragging: Bool { get } // returns YES if user has started scrolling. this may require some time and or distance to move to initiate dragging
```

也就是说`draggin`的变化有延迟，这就说明我们在用KVO对`contentOffset`进行观察时，由于`draggin`的延迟，会有异常出现，前面的那个bug就是这个延迟导致的。

发现问题了，如何解决呢？

首先，我想到了去实现UIScrollViewDelegate，通过`scrollViewDidEndDragging`方法来做，事实证明它没有延迟，这样做是可以达到预期效果的，但这样又是不行的！因为作为第三方库去实现UIScrollViewDelegate会跟用户的实现冲突，导致总有一方不起作用。所以在写第三方库时这个思路是不可以的，如果不使用第三方库的话，这个思路还是可以的。

到手的鸟飞了，不甘心，再加思索，能不能让我的库向系统的`scrollViewDidEndDragging`方法中注入一些我想要的逻辑呢？

自然OC的动态运行时该出场了。代码是这样的：

``` swift
extension UIViewController {
    private struct AssociatedKeys {
        static var OnceToken = 0
    }
    
    public override class func initialize() {
        dispatch_once(&AssociatedKeys.OnceToken, {
            let originalSelector = #selector(UIScrollViewDelegate.scrollViewDidEndDragging(_:willDecelerate:))
            let swizzledSelector = #selector(UIViewController.easy_scrollViewDidEndDragging(_:willDecelerate:))
            
            let originalMethod = class_getInstanceMethod(self.classForCoder(), originalSelector)
            let swizzledMethod = class_getInstanceMethod(self.classForCoder(), swizzledSelector)
            
            let didAddMethod = class_addMethod(self.classForCoder(), originalSelector, method_getImplementation(swizzledMethod), method_getTypeEncoding(swizzledMethod))
            
            if didAddMethod {
                class_replaceMethod(self.classForCoder(), swizzledSelector, method_getImplementation(originalMethod), method_getTypeEncoding(originalMethod))
            } else {
                method_exchangeImplementations(originalMethod, swizzledMethod);
            }
        })
    }
    
    // MARK: - Method Swizzling
    public func easy_scrollViewDidEndDragging(scrollView: UIScrollView, willDecelerate decelerate: Bool) {
        //self.easy_scrollViewDidEndDragging(scrollView, willDecelerate: decelerate) 
        //因为originalMethod不存在，所以swizzledSelector的方法交换就是失败的，再调用这句的结果就是死循环。
        
        //TODO: 在符合特定条件的情况下，调用第三方库释放刷新的操作逻辑。保证操作能及时执行！
    }
}
```

这个思路看似不错，逼格也高，但是认真看代码你会不会发现这个用法跟平时不太一样呢，我们交换的是UIScrollViewDelegate.scrollViewDidEndDragging(_:willDecelerate:)的实现，UIScrollViewDelegate根本就没有实现，所以这段代码我们只是做到了给UIViewController增加了scrollViewDidEndDragging的实现。当用户在viewContrller里自己实现scrollViewDidEndDragging时，这个方法就失效了。

还是不行，继续思考，发现就只有一个方法能做到最方便了，用户在自己的baseViewController里定义scrollViewDidEndDragging方法，就搞定。当然子类在需要时可以重写，但是记得super. scrollViewDidEndDragging。

``` swift
class BaseViewController: UIViewController {
	.........
	.........

    func scrollViewDidEndDragging(scrollView: UIScrollView, willDecelerate decelerate: Bool) {
        //TODO: 在符合特定条件的情况下，调用第三方库释放刷新的操作逻辑。保证操作能及时执行！
    }
}
```

搞定！