---
layout:     post
title:      "对[yield]的浅究到发现[async][await]"
subtitle:   ""
date:       2014-06-12 12:00:00
author:     "荣浩"
header-img: "img/post-bg-microsoft.jpg"
tags:
    - C# | .NET
---

<p>　　<span style="font-size: 15px;">上篇<a href="http://www.cnblogs.com/such/p/3782512.html" target="_blank">对[foreach]的浅究到发现[yield]</a>写完后，觉得对[yield]还没有理解清楚，想起曾经看过一位大牛的帖子讲的很深刻（<a href="http://www.cnblogs.com/yuyijq/archive/2011/02/24/1963326.html" target="_blank">链接在此</a>），回顾了下，在这里写出自己的理解，与各位分享。</span></p>
<h3><span style="font-size: 15px;">一、通常的异步</span></h3>
<p><span style="font-size: 15px;">　　现在我们假设一种平时经常遇到的情况，现有三个方法，其中funcOne和funcTwo比较耗时需要异步执行，而且他们的逻辑是必须在funcOne执行完后才可以执行funcTwo，同理funcTwo执行完后才能执行funcThree。</span></p>
<p><span style="font-size: 15px;">　　按照这样的设定，通常的做法请看</span><span style="font-size: 15px;">代码段[1]:</span></p>

``` csharp
public class Program
{
	public delegate void CallBack(string nextName);
	public void funcOne(CallBack callback)
	{
		ThreadPool.QueueUserWorkItem((state1) =>
		{
			Console.WriteLine("[One] async Continue!");
			Console.WriteLine("[One] do something!");
			callback("Called Two");
		});
	}
	public void funcTwo(CallBack callback)
	{
		ThreadPool.QueueUserWorkItem((state2) =>
		{
			Console.WriteLine("[Two] async Continue!");
			Console.WriteLine("[Two] do something!");
			callback("Called Three");
			});
	}
     public void funcThree(CallBack callback)
     {
         Console.WriteLine("[Three] do something!");
         callback("Called ...");
     }
     static void Main()
     {
         Program p = new Program();
         p.funcOne((name1) =>
         {
             Console.WriteLine(name1);
             p.funcTwo((name2) =>
             {
                 Console.WriteLine(name2);
                 //执行funcThree
                 p.funcThree((name3) =>
                 {
                     Console.WriteLine(name3);
                     //当然还有可能继续嵌套
                     Console.WriteLine("End!");
                 });
             });
         });
         Console.Read();
     }
 }
```

<p><span style="font-size: 15px;">　　</span><span style="font-size: 15px; font-family: verdana, Arial, Helvetica, sans-serif; line-height: 1.5;">相信看完代码后我们的感觉是一样的，好繁琐，就是不断的嵌套！那有没有方法可以避免这样呢，也就是说用同步的写法来写异步程序。</span></p>
<h3><span style="font-size: 15px;">二、改进后的异步</span></h3>
<p><span style="font-size: 15px;">　　该[yield]粉墨登场了，先看代码段[2]：</span></p>

``` csharp
//三个方法以及委托CallBack的定义不变，此处不再列出。
//新增了静态的全局变量enumerator，和静态方法funcList.
public static System.Collections.IEnumerator enumerator = funcList();
public static System.Collections.IEnumerator funcList()
{
    Program p = new Program();
    p.funcOne((name1) =>
    {
        enumerator.MoveNext();
    });
    yield return 1;
    Console.WriteLine("Called Two");
    p.funcTwo((name2) =>
    {
        enumerator.MoveNext();
    });
    yield return 2;
    Console.WriteLine("Called Three");
    p.funcThree((name3) =>
    {
        //当然还有可能继续嵌套
    });
    Console.WriteLine("Called ...");
    Console.WriteLine("End!");
    yield return 3;
}

//变化后的Main函数
static void Main()
{
    enumerator.MoveNext();
    Console.Read();
}    
```

<p><span style="font-size: 15px;">　　</span><span style="font-size: 15px; font-family: verdana, Arial, Helvetica, sans-serif; line-height: 1.5;">现在看看，是不是清爽了一些，没有无止尽的嵌套。代码中我们只需要定义一个迭代器，在迭代器中调用需要同步执行的方法，用[yield]来分隔，各方法通过在callback里调用迭代器的MoveNext()方法来保持同步。</span></p>
<p><span style="font-size: 15px;">　　通过这样的实践，我们可以理解为每当[yield return ..]，程序会把迭代器的[上下文环境]暂时保存下来，等到MoveNext()时，再调出来继续执行到下一个[yield return ..]。</span></p>
<h3><span style="font-size: 15px; line-height: 22.5px;">三、是我想太多</span></h3>
<p><span style="font-size: 15px; line-height: 22.5px;">　　以上纯属瞎扯，在.net 4.5之后，我们有了神器：async/await，下面看看多么简洁吧。</span></p>
<p><span style="font-size: 15px; line-height: 22.5px;">　　代码段[3]：</span></p>

``` csharp
public class Program
{
    public async Task funcOne()
    {
        Console.WriteLine("[One] async Continue!");
        Console.WriteLine("[One] do something!");
        //await ...
    }
    public async Task funcTwo()
    {
        Console.WriteLine("[Two] async Continue!");
        Console.WriteLine("[Two] do something!");
        //await ...
    }
    public void funcThree()
    {
        Console.WriteLine("[Three] do something!");
    }
    public static async Task funcList()
    {
        Program p = new Program();
        await p.funcOne();
        await p.funcTwo();
        p.funcThree();
        //无尽的嵌套可以继续await下去。
        Console.WriteLine("End!");
    }
    static void Main()
    {
        funcList();
        Console.Read();
    }
}
```

<p>　　<span style="font-size: 15px;">我已经感觉到了您的惊叹之情。</span></p>
<p>　　<span style="font-size: 15px;">async修饰符将方法指定为异步方法（注意！：异步不异步，并不是async说了算，如果这个方法中没有await语句，就算用了async修饰符，它依然是同步执行，因为它就没有异步基因）。</span></p>
<p><span style="font-size: 15px;">　　被指定为异步方法后，方法的返回值只能为Task、Task&lt;TResult&gt;或者void，返回的Task对象用来表示这个异步方法，你可以对这个Task对象进行控制来操作这个异步方法，详细的可以参考msdn中给出的<a href="http://msdn.microsoft.com/zh-cn/library/system.threading.tasks.task.aspx" target="_blank">Task解释与示例代码</a>。</span></p>
<p><span style="font-size: 15px;">　　await用于挂起主线程(这里的主线程是相对的)，等待这个异步方法执行完成并返回，接着才继续执行主线程的方法。用了await，主线程才能得到异步方法返回的Task对象，以便于在主线程中对它进行操作。如果一个异步方法调用时没有用await，那么它就会去异步执行，主线程不会等待，就像我们代码段[3]中Main方法里对funcList方法的调用那样。</span></p>
<p>&nbsp;</p>
<p><span style="font-size: 15px;">　　最后就分析到这儿吧，希望各位兄弟博友能一起讨论，共同进步。</span></p>
<p><span style="font-size: 15px;">　　当然，别吝啬您的[赞]哦 :)</span></p>
<p>&nbsp;</p>
<p>&nbsp;<span style="color: #ff0000; font-size: 16px;">更新：</span>现在大家看到的是改进之后的文章，在此谢谢文章下面几位高手的讨论，让我学会了不少。抱拳！<span style="font-size: 15px;"><br /></span></p>
<p>&nbsp;</p>
<p><span style="font-size: 15px;">　　</span></p>
<p>&nbsp;</p>