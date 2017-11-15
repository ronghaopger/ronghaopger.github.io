---
layout:     post
title:      "类似逐帧动画或者tableView列表中有大量图片展示需求的场景我们可以做哪些优化？"
subtitle:   ""
date:       2017-09-25 15:30:00
author:     "荣浩"
header-img: "img/post-bg-apple.jpg"
tags:
- iOS | 优化
---

## 背景
最近公司在调研后打算用SenseTime的产品代替之前的FaceMagic，据说在识别率和性能上做的更好一些。但是在集成后实际压测发现并没有什么提升，为了继续优化性能SenseTime还临时加入了隔帧检测技术，理论上隔一帧相当于少检测了一半的帧，性能是不是应该翻倍，然而实际压测的结果却还是没有什么提升。

所以我们不得不从全局重新思考，因为面部/手势识别礼物是由识别+播放环节组成，那如果识别做的很好了，是不是问题出在之前忽略的播放环节呢？果然在测试后发现是因为礼物资源太重导致的，不同于FaceMagic采用的骨骼动画，SenseTime采用传统的逐帧动画，那么一小段动画可能都需要上百张图片来顺序播放。想到这里我们开始尝试删减不必要的帧并尽量减小每帧图片的大小来优化，优化完资源包后由SDK的同事先行压测发现CPU占比大幅下降，大家喜上眉梢赶紧集成进项目，在直播间中开始实际压测，发现有提升但是并不明显啊！没有SDK同事用demo压测的效果那么大啊。

一头雾水时，在我跟SDK同事的交流中发现了问题，他们的压测demo就只是在开始时set一次资源（SenseTime的API要求set一个压缩资源包），然后开始循环识别+播放动画进行压测。但是我们直播间是每次礼物消息来的时候set一次资源（意味着要进行一次解压缩操作），然后开始进入识别+播放动画状态（我推测：而且由于直播时APP的内存吃紧，图片在内存中的缓存很容易被干掉，从而造成重复的文件IO和图片解码）。

那么思路捋到这里我们可以总结出来两个至关重要的优化点：
1. 避免频繁的解压资源包操作，每次解压绝对是件耗费的操作。这个是比较低级但在俯瞰全局时容易忽略的点，本篇不做讨论。
2. 逐帧动画的图片数量比较大质量比较高，能否把PNG/JPEG图片解码成位图后保存到本地，避免CPU进行重复频繁的解码图片操作？基于这一点我们开始本篇的探讨。

## 图片优化思路的来源和延伸
为了提高APP界面流畅度引出的优化方法有很多，在图片优化方面有一个思路，就是提前异步解码图片的思路而不是官方UIKit采用的在图片提交给GPU之前由主线程同步解码，这个思路的鼻祖应该就是[Facebook的ASDK](https://github.com/facebookarchive/AsyncDisplayKit)了，国内的[ibireme大神的YYImage](https://github.com/ibireme/YYImage)也借鉴了这个思路。（文末的附录1是我从他们这里学到的强制解码图片的思路）

但是面对逐帧动画这个变态的操作，我觉得提前异步解码并不能解决痛点，因为我们直播间的瓶颈在CPU，提前异步解码终归还是要每次解码，那能不能让解码的操作只做一次，把解码得到的位图缓存到本地，以后再用了就直接加载位图呢？在这之前，还是先来了解下iOS展示图片的过程吧。

## iOS展示图片的过程
我们写代码时，用给定的图片初始化一个UIImage对象并赋值给UIImageView.image，就在屏幕上呈现出该图片了，那么这个展示过程的原理是怎样的呢？
1. `self.testImageView.image = [UIImage imageNamed:@“xxx.png”];` 这时系统会去应用的mainBundle下寻找xxx.png，并生成一个CGImageRef指针指向它。
2. RunLoop的CATransaction捕捉到testImageView属性的变化，准备把数据提交给GPU来渲染。
3. 在把模型图层树(modelLayer tree)的数据提交给渲染图层树(renderLayer tree)之前，CPU会加载PNG图片到内存，并把PNG解码成位图。
4. 在OpenGL的驱动下，GPU把位图数据渲染到屏幕上。

**Note 1**: 第1步并不会加载图片到内存，更不会解码，我是通过以下代码验证的：
```
    NSMutableArray *array = [NSMutableArray array];
    for (NSInteger i = 0; i < 100; i++) {
        [array addObject:[UIImage imageNamed:[NSString stringWithFormat:@"xiongbao_%ld.png", (long)i]]];
    }
    self.imageView.animationImages = [array copy];
```
这段代码执行后通过Xcode没有观察到内存的变化，只有在真正要展示的时候（本例就是`[self.imageView startAnimating];`），才会加载并解码。

**Note 2**: 关于modelLayer tree、presentationLayer tree和renderLayer tree，Apple官方有文档解释，网上也有些解释。我的理解是我们平时通过UIView直接操作的就是modelLayer。在一个动画过程中真正起作用的是presentationLayer，这时modelLayer的属性值仅代表的是这个动画的终点值，所以在动画过程中我们通过`CALayer.presentationLayer`才能获得正在动画中的layer。而renderLayer是负责渲染的，不论是modelLayer还是presentationLayer在真正需要渲染展示的时候需要把数据提交给renderLayer。

## PNG？位图？
首先我们需要搞清楚一个概念，所谓PNG/JPEG都是在描述一种特定的文件格式，比如PNG它有自己的表示方法和压缩方法，图片格式的出现应该就是为了便于传输。不讨论利用硬件解码，GPU可以看作是一个只会并行高速渲染像素的傻子，它可搞不懂什么PNG格式JPEG格式，所以传输给GPU的应该是像素化后的数据。

值得一提的是，一张在硬盘里100K大小的PNG图片，解码成位图后可能有1M多，位图的大小 = 图片分辨率的宽 x 图片分辨率的高 x 单个像素点所占的空间大小。

## 文件IO VS CPU解码
那么问题来了：
* 使用PNG图片 = 从本地读取100K数据 + CPU解码这100K数据。
* 使用位图 = 从本地读取1+M的数据

其实就是文件IO与CPU运算的一次平衡。

目前为止我并没有想到什么有效的方法可以精确测量iPhone手机应用读取1M数据的损耗和CPU解码100K数据的损耗，但是说到平衡就是在强调我们应用的瓶颈在哪里，目前我们的直播间在各种功能效果都打开的情况下CPU是瓶颈，CPU的高速运转带来了手机发烫和CPU主频下调，紧接着就是掉帧，所以能为CPU减负是当务之急。

## ASDK之后的又一个惊喜
世界这么大，我们能想到的，往往已经有先行者。在我尝试做这次优化时，机缘巧合碰到了[FastImageCache](https://github.com/path/FastImageCache)，它带来的惊喜不亚于第一次碰到ASDK。所以接下来的优化思路会有借鉴FastImageCache的地方，膜拜学习吧。

## Color Copied Images 
这是Xcode提供的检测工具Instruments中Core Animation工具下的一个检测点，字如其意：复制图片。FastImageCache并没有涉及这项优化，但在我看它的README时自然想到了这点，什么意思呢？

上文中提到过，屏幕上的图片展示最终还是要靠GPU去渲染，具体如何去渲染牵扯到我们iOS框架的底层封装以及专门的图像渲染领域我们不做深究。
iOS渲染图像是通过OpenGL驱动GPU来做的，OpenGL有它所支持的颜色空间[Color Space](https://en.wikipedia.org/wiki/Color_space)如下：
> For color formats, there are more possibilities. GL_RED, GL_GREEN, and GL_BLUE represent transferring data for those specific components (GL_ALPHA cannot be used). GL_RG represents two components, R and G, in that order. GL_RGB and GL_BGR represent those three components, with GL_BGR being in reverse order. GL_RGBA and GL_BGRA represent those components; the latter reverses the order of the first three components. These are the only color formats supported (note that there are ways around that).  
> 摘自OpenGL Wiki: https://www.khronos.org/opengl/wiki/Pixel_Transfer  

可见OpenGL ES支持RGB BGR RGBA BGRA等颜色空间，那么当遇到它不支持的颜色空间时，就需要麻烦CPU先行转换成它所支持的颜色空间（转换过程必然带来开辟空间并写入数据的操作，也就是复制操作），就给CPU带来额外的负担，也就是Color Copied Images所在意的关键点。

**Note 1**: 在探究这一点时，我去检测了我们的APP，发现项目中的一些图标的颜色空间采用了Gray即[灰度图像 - 维基百科，自由的百科全书](https://zh.wikipedia.org/wiki/%E7%81%B0%E5%BA%A6%E5%9B%BE%E5%83%8F)，导致产生了复制图片的操作。这点我跟我们的设计同学确认过，他的回复是这种图片可能是早期用PS生成的，现在Sketch生成的图片都是RGB的，即使是黑白的图片。我觉得这个说法没有说服力，从原理上推测应该是为了控制图片的大小，因为灰度图像每个像素只占8bits，图片会小很多。

**Note 2**:  我们一直在说OpenGL，其实严格的说在移动端设备上使用的应该是OpenGL ES，即OpenGL for Embedded Systems。OpenGL的嵌入式版本，算是OpenGL的一个子集。

## 像素对齐
这是FastImageCache关注的一个点（其实在Instruments中也有体现：Color Misaligned Images），大意就是比如CPU每次读取固定的8 bytes的数据，现在有一个41 bytes大小的文件，那么CPU就需要读取6次，并且第6次读取的8 bytes还需要做多余的处理（只取第1个byte），那么自然会给CPU带来了额外的负担。解决的方法就是把这个文件填充到48 bytes，同时填充位不能影响原文件内容。

接下来我努力以相对专业的角度解释下，
1. 为什么CPU要每次读取8 bytes的数据呢？这个牵扯到CPU的cache line size，这篇[Data structure alignment - Wikipedia](https://en.wikipedia.org/wiki/Data_structure_alignment)可以解决部分疑惑，也可以自行google。通过FastImageCache源码发现它针对iPhone给出的对齐值是64，在Apple官方我并没有找到A系列处理器的cache line size的信息，但看到过有讨论说是A9的cache line size为64。下面贴上FastImageCache这块的源码：
![](/assets/images/2017-9-25/FastImageCache.png)
2. 位图数据用矩阵来表示，以二维数组的形式存储在内存中。注意看附录中那段强制解码图片的代码，CGBitmapContextCreate(…)方法有个参数就是bytesPerRow，即每行的字节数。FastImageCache做到了bytesPerRow的大小是64的整数倍，具体一点用它的原文来说应该是：`A properly aligned bytes-per-row value must be a multiple of 8 pixels × bytes per pixel. `，不仅是8的整数倍，还是bytesPerPixel的整数倍，可以推理这么做的目的是可以保证每行都存放了完整的像素，不会让一个像素数据跨行存储，也是优化的一个点。
3. 可能大家也会有个疑问，为什么是以图片的每行（即bytesPerRow）为单位要求对齐，而不是图片数据的整体呢？那是因为GPU渲染图片是以行为单位进行的，并且是多行并行渲染。

## MMAP
除了像素对齐，MMAP[MMAP - Wikipedia](https://en.wikipedia.org/wiki/Mmap)也是FastImageCache关注的一个点。

这个优化点是什么意思呢？通常系统加载一张图片（或者别的文件），会先把图片从硬盘拷贝到内存中的内核空间，某个进程需要时再拷贝到该进程的用户空间，也就是进程加载一张图片的过程中发生了**两次拷贝**，并且在另一个进程也需要访问该图片时还会从内核空间**再拷贝**一份到它的用户空间（没有进程间共享内存）。

而MMAP会把硬盘中的图片地址直接映射到进程的用户空间中，在进程访问图片数据时触发缺页中断[Demand paging - Wikipedia](https://en.wikipedia.org/wiki/Demand_paging)，才会把硬盘中的图片数据直接拷贝到内存中并更新页表，这样就只进行了**一次拷贝**，并且在另一个进程也需要访问该图片触发缺页中断时，可以加载内存中已经更新的页表，实现**内存共享**，同时可以想到的是内存共享带来了写入时需要加锁处理。

那么可以得出结论，MMAP相较于传统的文件读取，减少了数据拷贝的次数，减小了内存消耗。尤其对于展示图片这种只读不写的场景尤为适用。

## 总结
回到开始，对于有展示大量图片需求的场景，有几个优化点：
1. 直接使用位图。
2. 避免使用OpenGL不支持的颜色空间。
3. 保证图片数据内存对齐。
4. 用MMAP代替传统的文件读取。

最后还是要膜拜一下FastImageCache的库作者，感叹作为一名计算机工程师所拥有的知识面和基础功底的重要性，决定了我们在写代码时会去关注哪个层面的东西。

共勉！

- - - -
### 附录1
为了膜拜这些领路人，我先贴一段从他们那里学来的iOS强制解码图片的思路，就是利用Core Graphics把图片绘制到一块开辟好的context上并保存成位图：
```
- (void)createBitmap {
    UIImage *pngImage = [UIImage imageNamed:@"xiongbao_7.png"];
    CGFloat width = pngImage.size.width;
    CGFloat height = pngImage.size.height;
    CGImageRef pngImageRef = pngImage.CGImage;
    CGColorSpaceRef colorSpace = CGColorSpaceCreateDeviceRGB();
    //解码
    CGContextRef context = CGBitmapContextCreate(NULL, width, height, 8, width * 4, colorSpace, kCGImageAlphaNoneSkipLast);
    CGContextDrawImage(context, CGRectMake(0, 0, width, height), pngImageRef); // decode
    CGImageRef bitmapImageRef = CGBitmapContextCreateImage(context);
    //保存到本地
    NSString *path = [NSString stringWithFormat:@"%@/xiongbao_7.bitmap", [[NSBundle mainBundle] bundlePath]];
    CFURLRef url = CFURLCreateWithFileSystemPath(kCFAllocatorDefault,  (__bridge CFStringRef)path, kCFURLPOSIXPathStyle, false);
    CFStringRef type = kUTTypeBMP;
    CGImageDestinationRef dest = CGImageDestinationCreateWithURL(url, type, 1, 0);
    CGImageDestinationAddImage(dest, bitmapImageRef, 0);
    //释放资源
    CGContextRelease(context);
    CGColorSpaceRelease(colorSpace);
    CGImageDestinationFinalize(dest);
}
```


- - - -
### 参考
[iOS图片加载速度极限优化—FastImageCache解析 «  bang’s blog](http://blog.cnbang.net/tech/2578/)

http://www.cairuitao.com/fastimagecache源码分析/

http://blog.csdn.net/mg0832058/article/details/5890688

[linux进程间通信之共享内存篇 - Gordon0918 - 博客园](http://www.cnblogs.com/gordon0918/p/5280589.html)

[osx - CGImageRef width doesn’t agree with bytes-per-row - Stack Overflow](https://stackoverflow.com/questions/25706397/cgimageref-width-doesnt-agree-with-bytes-per-row)

