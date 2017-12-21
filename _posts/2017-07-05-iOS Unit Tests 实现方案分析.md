---
layout:     post
title:      "iOS Unit Tests 实现方案分析"
subtitle:   ""
date:       2017-07-05 15:30:00
author:     "荣浩"
header-img: "img/post-bg-apple.jpg"
tags:
- iOS | Test
---

```
在计算机编程中，单元测试（英语：Unit Testing）又称为模块测试, 是针对程序模块（软件设计的最小单位）来进行正确性检验的测试工作。程序单元是应用的最小可测试部件。在过程化编程中，一个单元就是单个程序、函数、过程等；对于面向对象编程，最小单元就是方法，包括基类（超类）、抽象类、或者派生类（子类）中的方法。
```

## S.O.L.I.D
理解单元测试，首先要理解单元(Unit)：软件设计的最小单位，而在我们面向对象的世界里最小单位就是方法，所以好的单元测试一定很好的覆盖了方法。想覆盖就得有暴露，如果一个类包含了本应该设计成三个类的代码，首先它是不OOP的，其次它也是不可单元测试的（除非暴露私有方法）。所以好的代码结构和好的单元测试是相辅相成，互相依赖的。所以在文章的开始需要强调一下基础，面向对象的五大原则SOLID：
* S - 单一职责原则，类的功能应该单一。
* O - 开放封闭原则，对扩展开放，对修改封闭。
* L - 里氏替换原则，子类可以完全代替父类在程序中运行。
* I - 接口隔离原则，客户端不应去实现它们不需要的接口。
* D - 依赖倒置原则，依赖于抽象而不是具体。

## TDD or BDD ?
### TDD
Test-driven development，测试驱动开发。
在我们设计一个类时，首先需要考虑这个类能提供什么功能以及如何提供，也就是这个类公开的方法。TDD的做法是先为这些公开的方法写测试代码，肯定一开始测试是通不过的，接下来开发人员的目的就是写代码实现类功能来让测试通过，测试通过了，这个类也就实现好了。

### BDD
Behavior-driven development，行为驱动开发。
TDD的目的是测试每个公开方法的正确性，而BDD关注的是行为，BDD的测试过程是以陈述需求的方式进行的（Given..When..Then），所以更友好，也更接近实际需求。

### 举个栗子
在实现直播间内活动挂件按优先级显示的功能时，我们需要维护一个描述view具体信息的model的集合类来保存当前直播间所有view的情况以便调整，它需要满足以下四个功能：
```
//添加
-(void)addModel:(RHShowPriorityModel*)model;
//删除
-(void)removeModel:(RHShowPriorityModel*)model;
//通过view索引model
-(RHShowPriorityModel*)modelForView:(UIView*)view;
//通过position索引model
-(NSArray<RHShowPriorityModel*>*)modelsForPosition:(RHShowPriorityPosition)position;
```


**以TDD的思想**编写测试的话，就是聚焦到每一个公开的方法，设法进行全面的测试，我们使用官方的XCTest框架进行演练，代码如下：
```
- (void)setUp {
    [super setUp];
    
    self.modelA = [[RHShowPriorityModel alloc] init];
    self.viewA = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 100, 100)];
    _modelA.view = _viewA;
    _modelA.position = RHShowPriorityPositionLeftUp;
    
    self.modelB = [[RHShowPriorityModel alloc] init];
    self.viewB = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 200, 200)];
    _modelB.view = _viewB;
    _modelB.position = RHShowPriorityPositionLeftUp;
    
    self.collection = [[RHShowPriorityCollection alloc] initWithModels:[NSArray arrayWithObjects:_modelA, _modelB, nil]];
}

- (void)tearDown {
    [super tearDown];
}


-(void)testModelForView {
    XCTAssertEqual([_collection modelForView:_viewA], _modelA);
    XCTAssertEqual([_collection modelForView:_viewB], _modelB);
}

-(void)testModelsForPosition {
    NSArray *array = [_collection modelsForPosition:RHShowPriorityPositionLeftUp];
    XCTAssertTrue([array isKindOfClass:[NSArray class]]);
    XCTAssertEqual(array[0], _modelA);
    XCTAssertEqual(array[1], _modelB);
}

-(void)testAddAndRemoveModel {
    RHShowPriorityModel *addModel = [[RHShowPriorityModel alloc] init];
    UIView *addView = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 150, 150)];
    addModel.view = addView;
    XCTAssertNil([_collection modelForView:addView]);
    
    //test addModel
    [_collection addModel:addModel];
    XCTAssertEqual([_collection modelForView:addView], addModel);
    
    //test removeModel
    [_collection removeModel:addModel];
    XCTAssertNil([_collection modelForView:addView]);
}
```


**以BDD的思想**进行测试的话，应该聚焦到需求，可以试着以Given..When..Then的方式陈述一下我们对于这个集合类的需求，比如`给你一个Coloection类，当它被初始化后，应该包含0个元素`。我们使用比较热门的Kiwi框架[GitHub - kiwi-bdd/Kiwi: Simple BDD for iOS](https://github.com/kiwi-bdd/Kiwi) 进行演练，代码如下：
```
describe(@"RHShowPriorityCollection", ^{
    context(@"当用初始数据modelA和modelB进行初始化后", ^{
        __block RHShowPriorityCollection *collection = nil;
        RHShowPriorityModel *modelA = [[RHShowPriorityModel alloc] init];
        UIView *viewA = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 100, 100)];
        modelA.view = viewA;
        modelA.position = RHShowPriorityPositionLeftUp;
        
        RHShowPriorityModel *modelB = [[RHShowPriorityModel alloc] init];
        UIView *viewB = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 200, 200)];
        modelB.view = viewB;
        modelB.position = RHShowPriorityPositionLeftUp;
        
        beforeEach(^{
            collection = [[RHShowPriorityCollection alloc] initWithModels:[NSArray arrayWithObjects:modelA, modelB, nil]];
        });
        
        afterEach(^{
            collection = nil;
        });
        
        
        it(@"通过viewA可以得到modelA，通过viewB可以得到modelB。", ^{
            [[[collection modelForView:viewA] should] equal:modelA];
            [[[collection modelForView:viewB] should] equal:modelB];
        });
        
        it(@"通过一个位置可以得到一个数组，包含所有在这个位置的view的model。", ^{
            NSArray *array = [collection modelsForPosition:RHShowPriorityPositionLeftUp];
            [[array should] beKindOfClass:[NSArray class]];
            [[array[0] should] equal:modelA];
            [[array[1] should] equal:modelB];
        });
        
        it(@"可以成功添加一个model。", ^{
            RHShowPriorityModel *addModel = [[RHShowPriorityModel alloc] init];
            UIView *addView = [[UIView alloc] initWithFrame:CGRectMake(0, 0, 150, 150)];
            addModel.view = addView;
            [[[collection modelForView:addView] should] beNil];
            [collection addModel:addModel];
            [[[collection modelForView:addView] should] equal:addModel];
        });
        
        it(@"也可以成功删除一个model。", ^{
            [[[collection modelForView:viewB] shouldNot] beNil];
            [collection removeModel:modelB];
            [[[collection modelForView:viewB] should] beNil];
        });
        
    });
});
```
惊不惊喜？反正在写的过程中我的内心是很舒服的，这段代码可以说是一目了然，清晰的陈述了我们对于这个类的要求，如果一位新同学要接手这块也是轻而易举。


**最后**回到我们的项目我认为：

对于像业务层的网络请求方法，只牵扯到基本的请求返回逻辑，测试也就只需要关注返回数据的正确与否，可以选择使用TDD的思想来做，也就是用官方的XCTest框架。

对于像上文中描述的场景，承载一个功能的类，包含比较丰富的逻辑在里面，使用BDD的思想更合适，写起来舒服，读起来清晰。

## Stub、Mock
未完待续...
