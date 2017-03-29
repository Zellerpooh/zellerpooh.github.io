---
layout: post
title: "奇妙的注解(二)    通过运行时注解实现ViewInject"
date: 2017-03-29
description: "奇妙的注解(二)    通过运行时注解实现ViewInject"
tag: Annotation
---   
> 纸上得来终觉浅，绝知此事要躬行。

[奇妙的注解(一)初识注解](http://www.jianshu.com/p/bb46c6d90032)

上一篇我们对注解有了初步的了解，知道了注解主要分两种运行时注解以及编译时注解，这篇文章我们就一起学习一下XUtils是如何基于运行时注解实现ViewInject的。
XUtils曾经也是个风靡一时的框架，集成了很多功能，作为一个初学者我对它也是爱不释手，对作者也是佩服得五体投地，但是随着学习的深入也逐渐了解到了为人诟病的效率问题(基于运行时注解以及反射)，以及如此庞大的功能与单一职责原则相悖，在实际的开发中已不可能再去使用。但是打开它的源码去一窥究竟一直是一个心愿。今天就先看看他是如何实现setContentView以及ViewInject的。

###  什么是IOC

>  **控制反转**（Inversion of Control，缩写为**IoC**），是[面向对象编程](https://zh.wikipedia.org/wiki/%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%BC%96%E7%A8%8B)中的一种设计原则，可以用来减低计算机代码之间的[耦合度](https://zh.wikipedia.org/wiki/%E8%80%A6%E5%90%88%E5%BA%A6_(%E8%A8%88%E7%AE%97%E6%A9%9F%E7%A7%91%E5%AD%B8))。其中最常见的方式叫做**依赖注入**（Dependency Injection，简称**DI**），还有一种方式叫“依赖查找”（Dependency Lookup）。通过控制反转，对象在被创建的时候，由一个调控系统内所有对象的外界实体，将其所依赖的对象的引用传递给它。也可以说，依赖被注入到对象中。

IOC最主要的实现方式就是依赖注入，依赖注入通俗的讲就是目标类（目标类需要进行依赖初始化的类）中所依赖的其他的类的初始化过程，不是通过手动编码的方式创建，而是通过技术手段可以把其他的类的已经初始化好的实例自动注入到目标类中。
把话再讲得糙一点，就是通过技术手段把初始化好的对象直接注射进目标类中，至于对象是如何来的，目标类并不需要知道。
XUtils中的ViewInject就是这样一种IOC的实现，通过注解的方式直接将View对象注入到Activity类中。

###  实现SetContentView

```
@ContentView(R.layout.main)
public class MyActivity extends FragmentActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        ViewUtils.inject(this);
    }
}
```
我们最终要达到的目的就是像上面这样，通过注解直接setContentView。
```
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface ContentView {
    int value();
}
```
首先定义个ContentView注解，作用的目标是TYPE(类)，有一个int返回值的value()方法用来接收layout资源id。
```
public static void inject(Activity activity) {
         Class<?> handlerType = activity.getClass();
        // inject ContentView
        ContentView contentView = handlerType.getAnnotation(ContentView.class);
        if (contentView != null) {
            try {
                Method setContentViewMethod = handlerType.getMethod("setContentView", int.class);
                setContentViewMethod.invoke(handler, contentView.value());
            } catch (Throwable e) {
                LogUtils.e(e.getMessage(), e);
            }
        }
  }
```
上面的方法也很简单，通过反射调用了setContentView()方法，并将resid从注解上取出后作为参数传递进去。然后你再用ContentView注解去修饰你的Activity,这样简单的setContentView就实现了，一定要动手试一下。
如果对反射一窍不通的话，可以去看看这篇文章
[Java反射机制探秘](http://www.cnblogs.com/dennisit/archive/2013/02/26/2933508.html)

###  实现ViewInject

```
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ViewInject {

    int value();
}
```
与ContentView相似，只是这一次注解的对象是FIELD(成员变量)

```
       Class<?> handlerType = activity.getClass();
// inject view
    //获取到activity中的所有成员变量
        Field[] fields = handlerType.getDeclaredFields();
        if (fields != null && fields.length > 0) {
            //循环成员变量
            for (Field field : fields) {
              //查看该成员变量是否有ViewInject注解
                ViewInject viewInject = field.getAnnotation(ViewInject.class);
                if (viewInject != null) {
                    try {
                    //通过activity的findViewById方法获取到View对象
                        View view = activity.findViewById(viewInject.value());
                        if (view != null) {
                            //将获取到得view对象设置给成员变量
                            field.setAccessible(true);
                            field.set(handler, view);
                        }
                    } catch (Throwable e) {
                        LogUtils.e(e.getMessage(), e);
                    }
                }
        }
```
还是通过反射的方法实现了将find到的View设置给注解了的成员变量，不理解的可以看代码中的注释。
```
    @ViewInject(R.id.btn_change)
    private Button btn_change;
```
这样我们就实现了通过注解实例化了View对象，还是文章开头的那句话 绝知此事要躬行 一定要去试一试，下面是我的代码
[Stride-Annotation](https://github.com/Zellerpooh/Stride-Annotation)
下一篇文章将和大家一起学习如何实现onClick注解

####  感谢您能够看到这里，笔者才疏学浅，如有纰漏，还望指正!

### 参考文章

[控制反转](http://baike.baidu.com/item/%E6%8E%A7%E5%88%B6%E5%8F%8D%E8%BD%AC?fr=aladdin&fromid=5177233&fromtitle=%E4%BE%9D%E8%B5%96%E6%B3%A8%E5%85%A5&type=syn)
[ioc框架（上）ViewInject](http://blog.csdn.net/lmj623565791/article/details/39269193)
