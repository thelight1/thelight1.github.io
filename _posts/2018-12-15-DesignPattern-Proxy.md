---
layout: post
title:  "【设计模式】之代理模式"
categories: DesignPattern 
tags: DesignPattern 
author: thelight1
mathjax: true
---
* content
{:toc}

# 1.模式动机与定义

代理模式定义：为其他对象提供一种代理以控制对象的访问。

# 2.模式结构与分析

``` Java
/**
 * 定义了RealSubject和Proxy的共同接口，使得在任何使用RealSubject的地方都可以使用Proxy
 */
public interface Subject {
​
    void request();
}
```

``` Java
/**
 * 定义了Proxy所代表的真正实体
 */
public class RealSubject implements Subject {
​
    @Override
    public void request() {
        System.out.println("RealSubject invoke request() method");
    }
}
```

``` Java
/**
 * 持有真实实体RealSubject的引用，使得Proxy可以访问真实实体
 * 并实现了Subject接口，这样代理就可以代替真实实体
 */
public class Proxy implements Subject{
​
    private RealSubject subject;
​
    public Proxy() {
        this.subject = new RealSubject();
    }
​
    @Override
    public void request() {
        this.subject.request();
    }
}
```

``` Java
public class Client {
​
    public static void main(String[] args) {
        Subject proxy = new Proxy();
        proxy.request();
    }
}
```

``` Java
RealSubject invoke request() method
```

# 3.模式实例与解析
模式实例：
学校里男同学A想要女同学B做自己的女朋友，想送女同学B礼物来追求女同学B，但是自己不好意思直接送，于是委托中间人来帮自己送礼物。

``` Java
/**
 * 校园中的漂亮女同学
 */
public class SchoolGirl {
​
    private String name;
​
    public SchoolGirl(String name) {
        this.name = name;
    }
​
    public String getName() {
        return name;
    }
​
    public void setName(String name) {
        this.name = name;
    }
}
```
GiveGift接口对应于代理模式类图中的Subject。

``` Java
public interface GiveGift {
    void giveDolls();
    void giveFlowers();
    void giveChocolate();
}
```
下面的Pursuer对应于类图中的RealSubject，是真实实体对象。

``` Java
/**
 * 女同学的追求者
 */
public class Pursuer implements GiveGift {
​
    private SchoolGirl girl;
​
    public Pursuer(SchoolGirl girl) {
        this.girl = girl;
    }
​
    @Override
    public void giveDolls() {
        System.out.println("[追求者]送给[" + this.girl.getName() + "]洋娃娃");
    }
​
    @Override
    public void giveFlowers() {
        System.out.println("[追求者]送给[" + this.girl.getName() + "]鲜花");
    }
​
    @Override
    public void giveChocolate() {
        System.out.println("[追求者]送给[" + this.girl.getName() + "]巧克力");
    }
}
```
Proxy对应于类图中的Proxy，为代理对象，其实现了与真实实体对象相同的接口GiveGift，并持有真实实体Pursuer实例的引用。

``` Java
/**
 * 女同学的追求者不好意思直接给女同学送礼物
 * 于是，追求者委托Proxy这个中间人来给女同学送礼物
 */
public class Proxy implements GiveGift {
​
    private Pursuer pursuer;
​
    public Proxy(SchoolGirl girl) {
        this.pursuer = new Pursuer(girl);
    }
​
    @Override
    public void giveDolls() {
        this.pursuer.giveDolls();
    }
​
    @Override
    public void giveFlowers() {
        this.pursuer.giveFlowers();
    }
​
    @Override
    public void giveChocolate() {
        this.pursuer.giveChocolate();
    }
}
```
客户端代码如下。

``` Java
public class Client {
​
    public static void main(String[] args) {
        SchoolGirl girl = new SchoolGirl("小静");
        GiveGift proxy = new Proxy(girl);
        proxy.giveDolls();
        proxy.giveFlowers();
        proxy.giveChocolate();
    }
}
/**
 运行输出如下：
 [追求者]送给[小静]洋娃娃
 [追求者]送给[小静]鲜花
 [追求者]送给[小静]巧克力
 */
 ```
# 4.模式效果与应用
代理模式的使用场景如下
1）远程代理。为一个对象在不同的地址空间提供局部代表，从而隐瞒一个对象存在于不同地址空间的事实。

举例：在RPC调用中，会在client端创建一个server端接口的代理对象，这样client端调用远程server端的接口就像调用本地方法一样。

2）虚拟代理。对象实例化需要花费很长时间，可以临时用虚拟代理代替。

举例：我们访问一些网页时，常含有一些图片，如果图片很大，文本渲染完了，图片还没从服务器下载下来，这时，可以在网页上显示一个图片框，等到图片下载完了在显示真正的图片。

3）安全代理。用于控制对象的访问权限。

4）智能指引。调用真实的对象时，代理添加一些额外的操作。

举例：Spring AOP。

一句话总结：代理模式就是在访问对象时，引入了一定程度的间接性。