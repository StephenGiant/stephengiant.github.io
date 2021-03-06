---
layout:     post
title:      kotlin的lambda与高阶函数
subtitle:   kotlin高阶函数用法
date:       2017-05-05
author:     Booker
header-img: img/rookie.jpg
catalog: true
tags:
    - Android
    - Kotlin
    - Lambda
---


> 本文首次发布于 [BookerBlog](http://stephengiant.github.io), 作者 [@Booker](http://github.com/StephenGiant) ,转载请保留原文链接.

# 前言

kotlin语言也使用了很多语言都会使用的高阶函数以及Lambda表达式来简化代码。

众所周知，Java是没有高阶函数概念的，在1.8开始才支持Lambda表达式。在Java里，我们要扩展一个方法，必须自定义一个接口，定义一个抽象方法，再将接口作为形参

`public interface A{`

​			`void method();`

`}`



这对于程序员来说 是比较麻烦的一个方式，有时候我们只是想扩展个方法而已，而当我们需要扩展一系列方法的时候，还是适合使用接口。kotlin的高阶函数就为我们解决了这个问题。

# 正文

先讲一讲如何在kotlin中使用高阶函数

高阶函数和Lambda表达式是密不可分的，下面举一个不用Lambda的写法示例

```kotlin
iv_photo.setOnClickListener(object : View.OnClickListener {
    override fun onClick(v: View?) {
        //dosomething
    }

})
```

这里我们对一个ImageView添加了点击事件 需要创建一个onclick事件监听再重写他的方法，基本和JAVA的写法是一样的，甚至还多了个object: 来声明匿名对象

一样的代码，现在使用Lambda表达式来写

```kotlin
iv_photo.setOnClickListener{it->
    
}
```

这里就省略了很多代码 这个it指的就是我们需要重写的方法的参数 这和Java的Lambda差不多，但是在kotlin中，如果只有一个参数，甚至连it都可以省略

```kotlin
iv_photo.setOnClickListener{
    //dosomething
}
```

而这三者的写法的运行结果就出来了是一样的

让我们仔细观察下最后一个写法，看上去好像我们传给setOnclickLIstener的参数是一个方法，而且是参数为View的方法，其实这就是相当于高阶函数了。我们用高阶函数也一样可以达到一个监听的效果。

```kotlin
class MyView {
     val text = "我是字段"
   private lateinit var click:MyView.()->Unit
    fun setOnclickListener(click:MyView.()->Unit){
        this.click = click
    }
    fun performClick(){
      click()
    }
}
```

在这里 我们不再使用接口 不用重新在类的内部定义一个接口，这无形中会减小JVM的压力（虽然很小但也比没有好）可以看出在kotlin中，函数是可以抽象为一个对象的，在设置了监听后，click事件就可以调用click方法，而内部到底怎么做，就完全是开放的，符合了程序设计的原则，内部不可修改，外部可扩展。

此Lambda表达式的第一个部分 click 是我们可以自己命名的这个高阶函数的名字,:来表示他的调用者类型MyView.()指的是调用他的是MyView对象，并且在高阶函数里可以使用这个MyView对象。（）表示这个高阶函数的参数，由于我们模仿的是View 已经直接直接可以使用了所以此处不传参数也是一样的 如果要写成传参的形式，直接在括号里加上参数类型即可。（注意，高阶函数不支持可变参数）

附上调用示例

```kotlin
   fun main(){
            MyView().setOnclickListener {
                //我想被点击
                println(text)
            }
        }
```

> # 在Android项目中使用高阶函数优化
>

在Android项目中，Retrofit已经成为了主流的网络请求封装框架，很多人会把Call转化成一个背压式的网络请求对象（即Observable）使用Observable去操作的时候，如果配置CallAdapter的时候用的是默认的Rxjava2CallAdapterFactory，那么，默认是同步的，所以使用的时候如果要拿回掉，就必须去指定subscrib的线程和observa的线程

```kotlin
serviceApi.run {
    getForString(ApiConfig.HotCreditCardList,map).subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread()).subscribe ({
                
            },{
                
            } 
            )

}
```

如果每次调用的时候要写这么多东西，其实和用Java是差不多的，利用kotlin的类扩展的话，我们就能优化不少。

类扩展这里就不详细说了，首先我们创建一个Extension.kt,然后在这个文件的顶层（必须是顶层），创建我们的扩展方法

```kotlin
fun Observable<String>.fetchData(comp: CompositeDisposable, onSucess: Consumer<JsonElement>.(e:JsonElement)->Unit, onError: Consumer<Throwable>.(e:Throwable)->Unit){
   comp.add(this.subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread()).flatMap<JsonObject>(Function {

        Observable.just(JsonParser().parse(it).asJsonObject)
    })
            .compose(RxjavaUtils.handleResult()).subscribe(object :Consumer<JsonElement>{
                override fun accept(t: JsonElement?) {
                    t?.let { onSucess(it) }
                }
            },
                    object :Consumer<Throwable>{
                        override fun accept(t: Throwable?) {
                            t?.let { onError(it) }
                        }

                    } ))
}
```

在这个扩展方法中，我们加入了很多的东西，订阅统一管理，订阅流的转化，线程的指定，以及回调的执行，其中我们把回调的部分抽成了两个高阶函数 一个叫onSuccess，一个叫onError，这样我们就完成了一个回调执行代码可灵活传入的带统一订阅管理的一个API调用。

```kotlin
serviceApi.run {
    getForString(ApiConfig.HotCreditCardList,map).fetchData(getCompositeDisposable(),{
        OnHotCardGet(it)
    },{
        onHotCardGetFail(it)
    })
```

CompositeDisposable对象是我封装在activity基类中的一个实例，可以在界面销毁的时候统一结束掉当前界面的所有api请求，调用过程是不是很简洁而且逻辑清晰？高阶函数还有很多实用的地方，让我们一起来探索吧！