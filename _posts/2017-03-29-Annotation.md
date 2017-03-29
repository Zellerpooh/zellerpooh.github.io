---
layout: post
title: "奇妙的注解(一)  初识注解"
date: 2017-03-29
description: "奇妙的注解(一)  初识注解"
tag: Annotation
---  
> 雄关漫道真如铁，而今迈步从头越。


 注解作为Java的一个基础知识点非常重要，Android很多的框架都是基于注解搭建的，但是自己一直不够重视，从来没认真看过注解相关的知识点，所以每次信誓旦旦地要看看XXX的源码，大都无功而返。从哪里跌倒还是要从哪里爬起来，今天就和大家一起学习下注解相关的知识。主要学习下下面这些内容：
* 什么是注解，如何实现自己的注解
* 基于运行时注解的IOC框架
* 利用编译时注解生成java文件
* 开源代码的分析(ButterKnife,Retrofit)

***
### 注解是什么
> 注解（也被成为元数据）为我们在代码中添加信息提供了一种形式化的方法，使我们可以在稍后某个时刻非常方便地使用这些数据。 ——————摘自《Thinking in Java》

上面这个解释非常的通俗易懂，我就不班门弄斧了。注解就是在稍后的某个时刻让我们更方便的使用这些数据，至于是啥时候呢，主要有两个一个是运行时(Runtime)，另外一个就是编译时(Compile)。

还是举个栗子吧，像@Override   就是我们常见的一个注解，我们应该天天都能见到它，那它到底是什么呢，点进源码可以看到
```
/**
 * Indicates that a method declaration is intended to override a
 * method declaration in a supertype. If a method is annotated with
 * this annotation type compilers are required to generate an error
 * message unless at least one of the following conditions hold:
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.SOURCE)
public @interface Override {
}
```
解释的非常清楚，这个注解主要用来提示这个方法是重写的方法，那我们需要怎么样才能实现一个自己的注解呢，接着往下看。

###  实现自己的注解

看上面Override的源码我们不难发现，它用到了3样东西和我们平常定义一个类时使用的不太一样，@interface，@Target，@Retention
#### @interface
首先来看@interface，它就是一个关键字，告诉你这是注解类型的，所以在设计annotations的时候必须把一个类型定义为@interface。

#### @Target
```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Target {
    /**
     * Returns an array of the kinds of elements an annotation type
     * can be applied to.
     * @return an array of the kinds of elements an annotation type
     * can be applied to
     */
    ElementType[] value();
}
```
@Target是可以点进去看源码的，我们可以看到除了常规的那3个，Target中多了一个返回值是 ElementType[] 的value()方法，那我们就去看看什么是ElementType。

```
public enum ElementType {
    /** Class, interface (including annotation type), or enum declaration */
    TYPE,

    /** Field declaration (includes enum constants) */
    FIELD,

    /** Method declaration */
    METHOD,

    /** Formal parameter declaration */
    PARAMETER,

    /** Constructor declaration */
    CONSTRUCTOR,

    /** Local variable declaration */
    LOCAL_VARIABLE,

    /** Annotation type declaration */
    ANNOTATION_TYPE,

    /** Package declaration */
    PACKAGE,

    /**
     * Type parameter declaration
     *
     * @since 1.8
     */
    TYPE_PARAMETER,

    /**
     * Use of a type
     *
     * @since 1.8
     */
    TYPE_USE
}
```
这个枚举里面解释得很清楚@Target其实就是告诉我们这个注解的作用对象是什么，如@Target(ElementType.METHOD) 修饰的注解表示该注解只能用来修饰在方法上。 其他同理。

#### @Retention
```
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.ANNOTATION_TYPE)
public @interface Retention {
    /**
     * Returns the retention policy.
     * @return the retention policy
     */
    RetentionPolicy value();
}
```
与@Target相似的是@Retention有一个返回值为RetentionPolicy的value()方法
```
public enum RetentionPolicy {
    /**
     * Annotations are to be discarded by the compiler.
     */
    SOURCE,

    /**
     * Annotations are to be recorded in the class file by the compiler
     * but need not be retained by the VM at run time.  This is the default
     * behavior.
     */
    CLASS,

    /**
     * Annotations are to be recorded in the class file by the compiler and
     * retained by the VM at run time, so they may be read reflectively.
     *
     * @see java.lang.reflect.AnnotatedElement
     */
    RUNTIME
}
```
看了上面的注释应该不难理解@Retention 表示了要注释具有注释类型的注释的保留时间。
> Indicates how long annotations with the annotated type are to be retained.

用@Retention(RetentionPolicy.CLASS)修饰的注解，表示注解的信息被保留在class文件(字节码文件)中当程序编译时，但不会被虚拟机读取在运行的时候；
用@Retention(RetentionPolicy.SOURCE )修饰的注解,表示注解的信息会被编译器抛弃，不会留在class文件中，注解的信息只会留在源文件中；
用@Retention(RetentionPolicy.RUNTIME )修饰的注解，表示注解的信息被保留在class文件(字节码文件)中当程序编译时，会被虚拟机保留在运行时，所以他们可以用反射的方式读取。
RetentionPolicy.RUNTIME 可以让你从JVM中读取Annotation注解的信息，以便在分析程序的时候使用.

@interface是关键字，而@Retention和@Target则是元注解，元注解的作用就是负责注解其他注解，除了这两个还有@Documented和@Inherited，有兴趣的可以去了解下，那么接下来我们就可以动手实现自己的注解了。

```
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface MyAnnotation {
    String value();
}
```
我们首先定义我们的注解，将RetentionPolicy设为RUNTIME，表示一直持续到运行时，ElementType设为Type表示这个注解作用的对象是类。这里最重要的一点就是Annotation类型里面的参数该怎么设定:
 1. 只能用public或默认(default)这两个访问权修饰.例如,String value();这里把方法设为defaul默认类型.
 2. 参数成员只能用基本类型byte,short,char,int,long,float,double,boolean八种基本数据类型和String,Enum,Class,annotations等数据类型,以及这一些类型的数组.例如,String value();这里的参数成员就为String.
 3.  如果只有一个参数成员,最好把参数名称设为"value",后加小括号.例:上面的例子就只有一个参数成员.

```
@MyAnnotation(value = "Hello Annotation")
public class Test {
}
```
然后写一个Test.java用我们刚定义的 注解去修饰。

```
public class Anntest {
    public static void main(String[] args) throws ClassNotFoundException {
        String className = "com.annotation.Test";
        Class test = Class.forName(className);
        boolean isAnnotation = test.isAnnotationPresent(MyAnnotation.class);
        if (isAnnotation) {
            MyAnnotation annotation = (MyAnnotation) test.getAnnotation(MyAnnotation.class);
            System.out.println("MyAnnotation:" + annotation.value());
        }
    }
}
```
最后我们通过一个main函数进行测试，首先我们通过反射拿到Test类的Class对象，然后判断它是否被我们自定义的注解类MyAnnatation修饰，是的话打印出注解的value()值。
最后运行代码得到如下的结果
>  MyAnnotation:Hello Annotation

这样我们就实现了自定义的运行时注解，我们还可以修饰更多的对象像方法属性等等，也可以定义更多返回值类型的参数，大家可以自行查看更多Annotation相关的API进行学习，这里就不一一举例了。

但是你还会发现这样自定义的注解似乎并没有什么实际的用途，在实际的开发中运行时注解大多同反射联系在一起实现一些巧妙的功能，而编译时的注解则通过动态生成java代码降低开发的难度，下一篇我们就一起学习一下通过运行时注解以及反射的一些IOC框架的实现。

代码(包括之后的)都上传到了github，欢迎查看和Star,
[Stride-Annotation](https://github.com/Zellerpooh/Stride-Annotation)

#### 感谢您能够看到这里，笔者才疏学浅，如有纰漏，还望指正!

### 参考文章

[实战篇：设计自己的Annotation](https://sites.google.com/site/yeyanbojava/Home/annotation)

[深入浅出Java Annotation(元注解和自定义注解)](http://josh-persistence.iteye.com/blog/2226493)
