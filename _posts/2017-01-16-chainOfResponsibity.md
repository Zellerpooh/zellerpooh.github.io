---
layout: post
title: "《Android源码设计模式》笔记(十)——责任链模式"
date: 2017-01-16
description: "《Android源码设计模式》笔记(十)——责任链模式"
tag: Design Pattern
---   
#### 文章为阅读《Android源码设计模式解析与实战》的一些感悟与笔记，有理解错误的地方，欢迎指正，要领略精髓还是要自己去阅读原书。

### 原书作者的博客

### [Mr.Simple](http://blog.csdn.net/bboyfeiyu)

### [aigestudio](http://blog.csdn.net/aigestudio)

### 定义

  一个请求沿着一条“链”传递，直到该“链”上的某个处理者处理它为止。
![责任链模式](http://upload-images.jianshu.io/upload_images/1859111-b74d057a030c6886.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 责任链模式结构

*  抽象处理者（Handler）:声明一个请求处理的方法，并在其中保持一个对下一个处理节点的Handler对象的引用。
*  具体处理者（ConcreteHandler）:对请求进行处理，如果不能处理则将该请求转发给下一个节点上的请求对象。

### 具体实现


![具体实现](http://upload-images.jianshu.io/upload_images/1859111-dfa89b29e8a3aaae.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 代码实现

* 抽象处理者（AbstractHandler）：可以清楚的看到，抽象处理者中持有对下一个处理节点的Handler对象的引用，handleRequest方法中实现了处理逻辑，当该处理这能处理时调用handle()方法处理，没有能力处理时，调用下一个处理者的handle()方法进行处理，实现了解耦。
```bash

public abstract class AbstractHandler {
    protected AbstractHandler nextHandler;

    public final void handleRequest(AbstractRequest request) {
        if (getHandleLevel()==request.getRequestLevel()){
            handle(request);
        }else {
            if (nextHandler!=null){
                nextHandler.handle(request);
            }else {
                System.out.println("All of handler can not handle the request");
            }
        }
    }

    protected abstract int getHandleLevel();

    protected abstract void handle(AbstractRequest request);
}

```

* 抽象请求（AbstractRequest）：抽象请求的逻辑很简单，封装了一个与处理者中对应的处理等级，以及一个对象。

```bash

public abstract class AbstractRequest {
    private Object obj;

    public AbstractRequest(Object obj) {
        this.obj = obj;
    }

    public Object getContent() {
        return obj;
    }

    public abstract int getRequestLevel();
}

```

* 具体的处理者和请求：处理者中实现了一个返回请求等级和一个处理请求的方法，而具体的请求只是实现了构造方法及返回处理等级的方法。

```bash

public class Handler1 extends AbstractHandler {
    @Override
    protected int getHandleLevel() {
        return 1;
    }

    @Override
    protected void handle(AbstractRequest request) {
        System.out.println("Handler1 handle request:" + request.getRequestLevel());
    }
}  

  public class Request1 extends AbstractRequest {
    public Request1(Object obj) {
        super(obj);
    }

    @Override
    public int getRequestLevel() {
        return 1;
    }
}

```

* 客户端：客户端创建了3个处理者和3各请求，分别从第一级开始处理三个请求

```bash

public class Client {
    public static void main(String[] args) {
        AbstractHandler handler1 = new Handler1();
        AbstractHandler handler2 = new Handler2();
        AbstractHandler handler3 = new Handler3();

        handler1.nextHandler = handler2;
        handler2.nextHandler = handler3;

        AbstractRequest request1 = new Request1("Request1");
        AbstractRequest request2 = new Request2("Request2");
        AbstractRequest request3 = new Request3("Request3");

        handler1.handleRequest(request1);
        handler1.handleRequest(request2);
        handler1.handleRequest(request3);
    }
}

```

结果显而易见，handler1直接处理了request1,request2被handler2处理，request直到handler3才被处理。

### 总结

  责任链模式能将请求者和处理者关系进行解耦，提高代码的灵活性。而缺点也显而易见，对链中请求处理者的遍历，如果处理者太多那么遍历必定会影响性能，特别是一些递归调用。

<br>
### 相关推荐

[事件分发机制原理](https://github.com/GcsSloop/AndroidNote/blob/master/CustomView/Advance/%5B12%5DDispatch-TouchEvent-Theory.md)
