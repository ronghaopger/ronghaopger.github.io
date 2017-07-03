---
layout:     post
title:      "对[foreach]的浅究到发现[yield]"
subtitle:   ""
date:       2014-06-11 12:00:00
author:     "荣浩"
header-img: "img/post-bg-microsoft.jpg"
tags:
    - C# | .NET
---

<p>　　闲来无事，翻了翻以前的代码，做点总结，菜鸟从这里起航，呵呵。</p>
<h3>一、List的foreach遍历</h3>
<p>　　先上代码段[1]：</p>

``` csharp
 class Program
 {
 	static void Main(string[] args)
   		{
			List<string> dayList = new List<string> { "Sun", "Mon", "Tue", "Wed", "Thr", "Fri", "Sat" };
			foreach (var day in dayList)
          	{
              	Console.WriteLine(day);
            }
            Console.ReadKey();
         }
     }
 }
```

<p>　　这是我们经常用的，简单明了，这里就不赘述了。</p>
<h3>二、对List的浅究</h3>
<p>　　接着我就产生了疑问，List具有怎样的特性才使得foreach可以对它进行遍历呢？这个遍历是如何实现的？</p>
<p>　　下面就来浅究，再上代码段[2]：</p>

``` csharp
public class DaysList<T> : System.Collections.IEnumerable
{
	T[] daysArry;
	public DaysList(T[] days)
	{
		daysArry = days;
	}
	public System.Collections.IEnumerator GetEnumerator()
	{
		for (int i = 0; i < daysArry.Length; i++)
		{
			yield return daysArry[i];
		}
	}
}

class Program
{
	static void Main(string[] args)
	{
		string[] days = { "Sun", "Mon", "Tue", "Wed", "Thr", "Fri", "Sat" };
		var daysList = new DaysList<string>(days);
		foreach (string day in daysList)
		{
			System.Console.WriteLine(day);
		}
		Console.ReadKey();
	}
}
```

<p>　　通过查阅我们发现LIst是通过实现System.Collections.IEnumerable接口来达到可以被遍历的能力，而实现System.Collections.IEnumerable接口必须实现它里面的GetEnumerator()方法，用来<span>返回一个循环访问集合的枚举器</span>，代码段[2]中就有我对GetEnumerator()方法的实现，其中有个关键字[yield]不知大家注意到没。</p>
<p>　　我的理解是：与其说是foreach遍历List，不如说是foreach遍历的是List中的GetEnumerator()方法返回的枚举器，注意这个枚举器实现了IEnumerator 接口，（插句话，IEnumerable接口标识某个类具备被遍历的能力，而IEnumerator 接口则使某个类真正具备这个能力！）。而当foreach对List进行循环遍历时，每个循环就是通过[yield]来分隔的。</p>
<h3>三、对foreach的浅究</h3>
<p>　　通过标题二，我们大概对List进行了了解，但不清楚，下面看看foreach。</p>
<p>　　依旧代码段[3]：</p>

``` csharp
//注意：其中类DaysList<T>的实现同代码段[2]一样
//这里只展示foreach的实现。    
class Program
{
	static void Main(string[] args)
	{
		string[] days = { "Sun", "Mon", "Tue", "Wed", "Thr", "Fri", "Sat" };
		var daysList = new DaysList<string>(days);
 		System.Collections.IEnumerator enumertor = daysList.GetEnumerator();
		while (enumertor.MoveNext())
		{             
			System.Console.WriteLine(enumertor.Current);
		}
		Console.ReadLine();
	}
}
```

<p>　　就像上面说的，foreach其实遍历的是List中的GetEnumerator()方法返回的枚举器enumertor，而这个枚举器所实现的接口IEnumerator中定义了只读的Current属性（用来获取枚举器当前的所指的集合中的元素）、MoveNext方法（将枚举器推进到集合中的下一个元素，返回值代表是否到了集合末尾）、Reset方法（使枚举器指到集合第一个元素之前，也就是重置枚举器），明白了这些，我们就可以像代码段[3]中一样通过[while]语法来实现跟foreach一样的功能了，而上文中的[yield]关键字浅显的理解就是用来划分要遍历的集合中的每个元素的。</p>
<p>　　最后，本来还想分析分析这个实现了IEnumerator接口的枚举器是怎么生成的，想象一下它的内部实现，应该很有意思！</p>
<p>　　就这样吧，下班了，大家共勉！</p>