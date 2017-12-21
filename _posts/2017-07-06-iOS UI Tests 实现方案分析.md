---
layout:     post
title:      "iOS UI Tests 实现方案分析"
subtitle:   ""
date:       2017-07-06 15:30:00
author:     "荣浩"
header-img: "img/post-bg-apple.jpg"
tags:
- iOS | Test
---

UI Tests是一个可以对UI进行测试的框架。为什么需要UI Tests呢？如果客户端分为基础层和业务层的话，业务层最终都是负责界面展示的，通常是MV(C/P/VM)，(C/P/VM)中的逻辑并不适合用Unit Tests来覆盖。对于具体的界面操作流程，UI Tests是合适的选择。

## 集成UI Tests
下图为在Xcode中为项目添加UI Tests的步骤：
1. 新建Target
![](/assets/images/2017-7-6/UITests_AddBuddle.png)
2. 新建Case Class
![](/assets/images/2017-7-6/UITests_AddCaseClass.png)

![](/assets/images/2017-7-6/UITests_CaseClass.png)

## Writing UI Tests
### Recording
Xcode为我们提供了把整套操作转化为代码的功能，见下图：
![](/assets/images/2017-7-6/UITests_XcodeRecording.png)
几个关键点：
* 方法名必须以**test**开头，所在行的开头才会有一个菱形的标记，标志这个方法可以被测试。
* 满足上一个条件后，把光标放在方法体内，Xcode下方的Debug area的红点就会亮起，点击这个红点就会启动APP，开始录制操作并同步生成代码。

下面是我录制的从打开应用——>打开直播间——>在直播间发言”大家好！”的过程中生成的代码：
```
- (void)testSpeakInLiveRoom {
[XCUIDevice sharedDevice].orientation = UIDeviceOrientationFaceUp;
[XCUIDevice sharedDevice].orientation = UIDeviceOrientationFaceUp;
[XCUIDevice sharedDevice].orientation = UIDeviceOrientationFaceUp;
[XCUIDevice sharedDevice].orientation = UIDeviceOrientationFaceUp;
[XCUIDevice sharedDevice].orientation = UIDeviceOrientationPortrait;

XCUIApplication *app = [[XCUIApplication alloc] init];
[app.tabBars.buttons[@"tab launch"] tap];
[app.buttons[@"\U76f4\U64ad"] tap];
[app.buttons[@"\U5f00\U59cb\U76f4\U64ad"] tap];
[app.buttons[@"mg room btn liao h"] tap];

XCUIApplication *app2 = app;
[app2.buttons[@"\U5927\U5bb6\U597d"] tap];
[app2.buttons[@"\Uff01"] tap];
[[[app childrenMatchingType:XCUIElementTypeWindow] elementBoundByIndex:0].buttons[@"\U53d1\U9001"] tap];
}
```
**Note**: Xcode自动生成的代码中的Unicode为”\Uxxxx”会报错，把”\U”替换为”\u”就可以了。

把其中的Unicode转义一下，并且精简下冗余的代码后，是这样的：
```
- (void)testSpeakInLiveRoom {
[XCUIDevice sharedDevice].orientation = UIDeviceOrientationPortrait;

XCUIApplication *app = [[XCUIApplication alloc] init];
[app.tabBars.buttons[@"tab launch"] tap];
[app.buttons[@"直播"] tap];
[app.buttons[@"开始直播"] tap];
[app.buttons[@"mg room btn liao h"] tap];
[app.buttons[@"大家好"] tap];
[app.buttons[@"！"] tap];
[[[app childrenMatchingType:XCUIElementTypeWindow] elementBoundByIndex:0].buttons[@"发送"] tap];
}
```
**Note**:这段精简的代码已经满足我们正常的阅读了，但要用于生产环境还有很大的优化空间，后面会讲到。

### APIs
通过上述Xcode自动生成并且经过我们简单修改的代码，其实已经一目了然了，启动程序后，通过API获取相应的界面元素集合，并筛选出需要的元素（如何筛选？见下一小节），然后触发一系列操作。下面是UI Tests依赖的三个类：
* XCUIApplication 
* XCUIElement 
* XCUIElementQuery (实现了XCUIElementTypeQueryProvider协议)

### 筛选 With Accessibility
关于Accessibility，以下是apple Developer上的介绍：
```
Accessibility is the core technology that allows disabled users the same rich experience for iOS and macOS that other users receive. It includes a rich set of semantic data about the UI that users can use can use to guide them through using your app. Accessibility is integrated with both UIKit and AppKit and has APIs that allow you to fine-tune behaviors and what is exposed for external use. ~UI testing uses that data to perform its functions.~
```
Accessibility是iOS提供用来服务于残障人士的API，比如一位盲人在使用你的APP，当他点击到一个按钮时，系统会自动播放这个按钮的名称/用途，就是通过accessibilityLabel这个属性实现的。我们可以在编写应用界面时设置这些属性。

同时，UI Tests也会用到Accessibility。上面Xcode自动生成的UI Tests的代码中，从元素集合app.buttons中获取/筛选一个button就是用accessibilityLabel作为下标实现的，但是如果我们自己去写UI Tests，不建议使用accessibilityLabel，而应该使用accessibilityIdentifier，因为上面已经说了accessibilityLabel更倾向用于提供名称，Xcode在UI Testing过程中可能在一个场景中有两个控件的accessibilityLabel是相同的就会报错，而accessibilityIdentifier才是唯一的ID。系统之所以使用前者可能是因为accessibilityIdentifier这个属性默认值是nil。（可以通过Xcode分别定位到这两个属性的定义来查看描述信息和默认值，辅助理解）

### 断言
利用XCTest提供的XCTAssert APIs，编写符合期望的断言。比如在上述代码示例的最后，可以加入一个断言来判断”发送”的内容是否成功显示，就完成了对”发送”这个功能的测试。

这里值得一提的是，有些时候我们并不希望立即进行断言（比如进入直播间后要等上几秒一些功能的网络请求才会返回），这时就会用到XCTestCase提供的`- (void)waitForExpectations:(NSArray<XCTestExpectation *> *)expectations timeout:(NSTimeInterval)seconds;`系列方法，在规定的时间范围内不断轮询检查。

### 查看报告
上文的配图中提到过的在每个测试方法的开头有个菱形标志，在UI Tests跑完后，不论通过与否，你都可以通过右击菱形标志看到”Jump to Report”选项，点击就可以看到本次测试这个方法的具体流程了，查看报告有益于我们了解测试流程以及迅速定位问题所在。

## 实践难点
经过我初步的实践和判断，在目前的项目中集成UI Tests有以下几个难点：
### 1、筛选、筛选、筛选
上一节已经讲了如何筛选，但是想科学筛选还是需要一定的实践。UI Tests无非就是筛选—>触发—>断言，目前来看最麻烦的就是筛选这个环节。通过上述示例代码或者自己动手去录制一段代码，会发现想找到一个指定的元素Xcode会利用accessibilityLabel沿着当前view的层级逐层去定位，跟view层级绑定起来简直是一件很糟的事情，意味着你需要去熟悉凡是牵扯到的功能的view层级还得在UI变动时及时调整筛选代码，好在apple的初衷不是这样的，XCUIElementQuery实现了XCUIElementTypeQueryProvider协议，使得我们可以通过控件的ElementType（XCUIElementTypeQueryProvider中有定义）和accessibilityIdentifier轻松定位到，不再关心所到之处view的层级，见代码：
```
//Xcode生成的
[[[app childrenMatchingType:XCUIElementTypeWindow] elementBoundByIndex:0].buttons[@"发送"] tap];

//改进后
[app.buttons[按钮的accessibilityIdentifier] tap];
```
**但是**：
也有特例，比如系统控件AlertView、ActionSheet等，我们只可以定义按钮名称，想要设置accessibilityIdentifier还要另辟蹊径，这时倒是可以采用上述Xcode的方式（利用accessibilityLabel沿着当前view的层级逐层去定位）来做，见代码：
```
//accessibilityIdentifier定位的方式
app.buttons[@"button的accessibilityIdentifier"];

//由于系统ActionSheet的button并不公开，设置accessibilityIdentifier不可行。
//利用accessibilityLabel沿着当前view的层级逐层去定位的方式
app.sheets.buttons[@"踢出直播间24小时"];
```
也就是说这两个方式都有用武之地，还是要看具体的场景和需求，有待挖掘。

**还有一点值得一提，** 上面说的XCUIElementTypeQueryProvider协议会提供各种类型的XCUIElementQuery，但是使用后我们发现这些类型跟我们UIKit的控件类型并不是一一对应的，比如UIView或者UIImageView，并不能像UIButton那样默认可以在XCUIElementTypeQueryProvider 的`buttons`集合里找到。不用急，Accessibility API里还提供了一个属性`accessibilityTraits`来帮我们完成UIKit控件的归类，比如一个UIImageView，如果在某个场景下可以被点击，我们可以把它的`accessibilityTraits`设置为`UIAccessibilityTraitButton`，就可以在`buttons`里找到了。如果在某个场景下只是用来展示图片的，可以把它的`accessibilityTraits`设置为`UIAccessibilityTraitImage`，就可以在`images`里找到了。如果你不知道应该设置成什么，就用`UIAccessibilityTraitNone`，对应XCUIElementTypeQueryProvider中的`otherElements;`。

### 2、accessibilityIdentifier管理
整个APP中的每个控件都需要一个唯一的accessibilityIdentifier（或者至少是同一个ElementType的accessibilityIdentifier不能相同），这个需要制定规范统一管理。我们目前采取的方案是用控件所在类的类名+控件变量名作为accessibilityIdentifier。

### 3、逻辑复用、多人协作
一个大型APP一定是多团队多人协作，并且很多功能的测试都依赖一些公共的逻辑，你可能只是想为直播间内一个新功能（比如小活动展示功能）添加UI Tests，但是需要涉及从打开应用到进入直播间的整段逻辑，这里就需要统一的封装，比如开直播这个功能可以统一封装，所有需要开直播的流程都可以调用。 

对于第2、3两点我在项目中的实践：
![](/assets/images/2017-7-6/UITests_封装.png)

### 4、UI频繁变动
几个星期一个版本迭代，意味着UI一定是在不断变化，想要完善的UI Tests，就需要开发人员养成在UI变动时及时调整UI Tests的习惯。

## 总结
上面说了这么多，总结下来，就诞生了这段优化之后的示例代码（用的还是accessibilityLabel请忽略）
```
-(void)testSpeakInLiveRoom {
[XCUIDevice sharedDevice].orientation = UIDeviceOrientationPortrait;
XCUIApplication *app = [[XCUIApplication alloc] init];
[self _openLive:app];
[app.buttons[@"mg room btn liao h"] tap];
NSString *text = [NSString stringWithFormat:@"%f : 大家好！", [[NSDate date] timeIntervalSince1970]];
[app.textFields[@"和大家说点什么"] typeText:text];
[app.buttons[@"发送"] tap];

NSString *showText = [NSString stringWithFormat:@"Hulk Rong:%@",text];
XCTAssertTrue(app.tables.staticTexts[showText].exists);
}

- (void)_openLive:(XCUIApplication*)app {
[app.buttons[@"tab launch"] tap];
[app.buttons[@"直播"] tap];
[app.buttons[@"开始直播"] tap];
}
```
**Tips**:来自apple Developer的UI Tests编写思路：
* Use an XCUIElementQuery to find an XCUIElement.
* Synthesize an event and send it to the XCUIElement.
* Use an assertion to compare the state of the XCUIElement against an expected reference state.

## 抛砖引玉
说了这么多，基本把写一个简单的UI Tests需要知道的都说了。接下来我写几个自己想到的例子，代码在我们的工程中都有，算是抛砖引玉。
1. 测试直播间发言是否成功，也就是本篇的示例代码。步骤是打开应用—>开直播—>发言—>判断公聊区是否有发言内容。
2. 测试直播间商业红包步骤是否正确，这个我已经在项目中实现。步骤是打开应用—>开直播—>（手动触发直播间红包）—>判断红包是否展示—>判断跟红包位置冲突的活动轮播图是否已经隐藏—>点击红包判断是否弹出提示弹窗—>......—>红包倒计时结束后是否显示”发放中”—>........等
3. 测试直播间是否有热门火箭。步骤是打开应用—>开直播—>判断直播间是否有小火箭的view。（这块完整的测试还应该包括火箭被点击前后的位置变化是否正确、使用火箭的流程中各个view的出现时机和位置是否正确、使用后的提示是否正确）

## 畅想
如果一个功能有全面合理的UI Tests覆盖，那么假如基础架构进行了重构，假如某一块业务进行了重构，假如…，你所担心的，都只需跑一遍UI Tests心里就有个大概了，如果再辅以Unit Tests，再也不怕改变带来的未知恐惧了，还给测试大大减少了负担。惊喜不惊喜，意外不意外？

## 参考：
[User Interface Testing](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/testing_with_xcode/chapters/09-ui_testing.html#//apple_ref/doc/uid/TP40014132-CH13-SW1)

[WWDC15 Session笔记 - Xcode 7 UI 测试初窥](https://onevcat.com/2015/09/ui-testing/)

[Adding UI Testing to an existing iOS App](https://medium.com/imgur-engineering/adding-ui-testing-to-an-existing-ios-app-e0e440ca213d)
