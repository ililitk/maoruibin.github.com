---
layout: post
author: 咕咚
title: "Android 开发中是否应该使用枚举？"
description: 在 Android 开发过程中，这个问题被很多次的提起，不少文章分析枚举的内存占用情况，后来在 Android 官方的内存优化文档中提出，不建议使用枚举，但是现在的官方文档却已将次建议删除，这背后都有哪些值得关心的东西呢？一起看看... 
qrcode_mp: true
tags: Optimize Android Enum
categories: blog 
---


在 Android 官方文档推出[性能优化](https://developer.android.com/topic/performance/)的时候，从一开始有这样一段说明：

> Enums often require more than twice as much memory as static constants. You should strictly avoid using enums on Android.

意思是说在 Android 平台上**避免使用枚举 **，因为枚举类比一般的静态常量多占用两倍的空间。

> 如果你还不了解枚举，参看文章 [枚举介绍以及枚举的本质](../../../2019/11/08/enum-introduce.html)。

由于枚举最终的实现原理还是类，在编译完成后，最终为每一种类型生成一个静态对象，而在内存申请方面，对象需要的内存空间远大于普通的静态常量，而且分析枚举对象的成员变量可知，每一个对象中默认都会有一个字符数组空间的申请，计算下来，枚举需要的空间远大于普通的静态变量。具体分析可见[这篇文章](https://www.liaohuqiu.net/cn/posts/android-enum-memory-usage/)。

所以，照此来看，在 Android 这样对内存寸土必争的平台上，如果只是使用枚举来标记类型，那使用静态常量确实更优，但是现在翻看官方文档发现，这个建议已经被删除了。（

为什么官方会删除？难道是之前的建议有错误吗，或者描述的不够精确？

个人认为，枚举占用空间比普通类型的静态常量大，这是事实，没问题，但是据此就建议不在 Android 中使用时不妥的，具体看 JakeWharton 在 reddit 上的一个[评论](https://www.reddit.com/r/androiddev/comments/7so7ne/you_should_strictly_avoid_using_enums_on_android/?utm_source=share&utm_medium=web2x)

> The fact that enums are full classes often gets overlooked. They can implement interfaces. They can have methods in the enum class and/or in each constant. And in the cases where you aren't doing that, ProGuard turns them back into ints anyway.
>
> The advice was wrong for application developers then. It's remains wrong now.

最重要的一句是

> ProGuard turns them back into ints anyway.

> https://dim.red/2019/01/28/proguard_exploration/

他的意思是 Android 中，在开启 ProGuard 优化的情况下，枚举会被转为 int 类型，所以内存占用问题是可以忽略的。具体可参看 ProGuard 的优化列表页面 [Optimizations Page](http://proguard.sourceforge.net/manual/optimizations.html)，其中就有 enum 默认被优化的一项，如下所示：

> class/unboxing/enum
>
> Simplifies enum types to integer constants, whenever possible.

既然 ProGuard 会把枚举优化为整形，那是不是在 Android 中，就可以继续无所顾忌的使用枚举了呢？😊

**并不是！！！**

ProGuard 只会对简易使用的枚举才会进行整形优化，如下所示

```java
enum Color{
  Red,Black,Green
}
```

或者这样的

```java
enum Date {
  Sunday("星期日"), Monday("星期一"), Tuesday("星期二"), Wednesday("星期三"), Thursday(
    "星期四"), Friday("星期五"), Saturday("星期六");

  public String value;

  private Date(String value) {
    this.value = value;
  }
}
```

都没问题，上面的枚举会在 ProGuard 优化期间被替换为整形，但是如果枚举有下属情况，则 ProGuard 不会对枚举进行优化，如下所示：

1. 枚举实现了自定义接口。并且被调用。
2. 代码中使用了不同签名来存储枚举。
3. 使用 instanceof 指令判断。
4. 在枚举加锁操作。
5. 对枚举强转。
6. 在代码中调用静态方法 valueOf 方法。
7. 定义可以外部访问的方法。

> 参考自：[ProGuard 初探 · dim's blog](https://dim.red/2019/01/28/proguard_exploration/)

所以现在再次解答 Android 是否应该用枚举这一问题，我们就应该结合自己使用枚举的场景，去具体的分析这个问题。

* 一般的，对枚举没有复杂操作的情况下，随意使用枚举，它有更好语义性，代码更好阅读。
* 要确定使用枚举时，确保使用场景不会触发上面列举的情况，具体反推之
  * 不要让枚举实现接口
  * 如果需要自定义构造方法，那么只自定义一个有参的构造方法。
  * 不要对枚举进行 instanceof 判断
  * 不使用 valueOf 方法实例化枚举
  * 等等...

> 思考？是不是可以写一个代码检测工具，检测项目中的枚举是否可以被优化，如果不能，则提示开发者。

至于 Android 为什么会把那条优化建议删掉，我认为官方八成也是考虑到了枚举会被优化为整形这一点，所以才去掉的。而且大部分情况下，使用枚举的场景都比较轻量，并没有那么多复杂情况存在。

但是知其然知其所以然，作为开发者，了解了这些后，再去使用枚举，心里就会更有底。



## 参考链接

* [Android 中的 Enum 到底占多少内存？该如何用？ \| Yet Another Summer Rain](https://www.liaohuqiu.net/cn/posts/android-enum-memory-usage/)
* [关于Java中枚举Enum的深入剖析 \- 技术小黑屋](https://droidyue.com/blog/2016/11/29/dive-into-enum/)
* [should-i-strictly-avoid-using-enums-on-android](https://stackoverflow.com/a/29972028/4318748)
* ["You should strictly avoid using enums on Android" : androiddev](https://www.reddit.com/r/androiddev/comments/7so7ne/you_should_strictly_avoid_using_enums_on_android/)
* [ProGuard 初探 · dim's blog](https://dim.red/2019/01/28/proguard_exploration/)

