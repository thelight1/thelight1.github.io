---
layout: post
title:  "Java中的Unsafe类"
date:   2018-09-09 11:40:18 +0800
categories: jekyll
tags: JUC Unsafe 
author: thelight1
mathjax: true
---
# 1.Unsafe类介绍
Unsafe类是在sun.misc包下，不属于Java标准。但是很多Java的基础类库，包括一些被广泛使用的高性能开发库都是基于Unsafe类开发的，比如Netty、Hadoop、Kafka等。

使用Unsafe可用来直接访问系统内存资源并进行自主管理，Unsafe类在提升Java运行效率，增强Java语言底层操作能力方面起了很大的作用。

Unsafe可认为是Java中留下的后门，提供了一些低层次操作，如直接内存访问、线程调度等。

 官方并不建议使用Unsafe。

下面是使用Unsafe的一些例子。

## 1.1实例化私有类
``` java
import java.lang.reflect.Field;  
  
import sun.misc.Unsafe;  
  
public class UnsafePlayer {  
      
    public static void main(String[] args) throws Exception {    
        //通过反射实例化Unsafe  
        Field f = Unsafe.class.getDeclaredField("theUnsafe");  
        f.setAccessible(true);    
        Unsafe unsafe = (Unsafe) f.get(null);    
    
        //实例化Player  
        Player player = (Player) unsafe.allocateInstance(Player.class);   
        player.setName("li lei");  
        System.out.println(player.getName());  
          
    }    
}    
    
class Player{   
      
    private String name;  
      
    private Player(){}
      
    public String getName() {  
        return name;  
    }  
    public void setName(String name) {  
        this.name = name;  
    }  
}
```

## 1.2CAS操作，通过内存偏移地址修改变量值
java并发包中的SynchronousQueue中的TransferStack中使用CAS更新栈顶。

``` java
// Unsafe mechanics
private static final sun.misc.Unsafe UNSAFE;
private static final long headOffset;
static {
    try {
        UNSAFE = sun.misc.Unsafe.getUnsafe();
        Class<?> k = TransferStack.class;
        headOffset = UNSAFE.objectFieldOffset
            (k.getDeclaredField("head"));
    } catch (Exception e) {
        throw new Error(e);
    }
}
//栈顶
volatile SNode head;
//更新栈顶
boolean casHead(SNode h, SNode nh) {
    return h == head &&
        UNSAFE.compareAndSwapObject(this, headOffset, h, nh);
}
```

## 1.3直接内存访问
Unsafe的直接内存访问：用Unsafe开辟的内存空间不占用Heap空间，当然也不具有自动内存回收功能。做到像C一样自由利用系统内存资源。

 

# 2.Unsafe类源码分析
Unsafe的大部分API都是native的方法，主要包括以下几类：

1）Class相关。主要提供Class和它的静态字段的操作方法。

2）Object相关。主要提供Object和它的字段的操作方法。

3）Arrray相关。主要提供数组及其中元素的操作方法。

4）并发相关。主要提供低级别同步原语，如CAS、线程调度、volatile、内存屏障等。

5）Memory相关。提供了直接内存访问方法（绕过Java堆直接操作本地内存），可做到像C一样自由利用系统内存资源。

6）系统相关。主要返回某些低级别的内存信息，如地址大小、内存页大小。

## 2.1Class相关
``` java
  //静态属性的偏移量，用于在对应的Class对象中读写静态属性
    public native long staticFieldOffset(Field f);
​
    public native Object staticFieldBase(Field f);
  //判断是否需要初始化一个类
    public native boolean shouldBeInitialized(Class<?> c);
  //确保类被初始化
    public native void ensureClassInitialized(Class<?> c);
  //定义一个类，可用于动态创建类
    public native Class<?> defineClass(String name, byte[] b, int off, int len,
                                       ClassLoader loader,
                                       ProtectionDomain protectionDomain);
  //定义一个匿名类，可用于动态创建类
    public native Class<?> defineAnonymousClass(Class<?> hostClass, byte[] data, Object[] cpPatches);
```

## 2.2Object相关
Java中的基本类型（boolean、byte、char、short、int、long、float、double）及对象引用类型都有以下方法。

``` java
  //获得对象的字段偏移量    
  public native long objectFieldOffset(Field f);  
  //获得给定对象地址偏移量的int值
  public native int getInt(Object o, long offset);
  //设置给定对象地址偏移量的int值
    public native void putInt(Object o, long offset, int x);
```

``` java
  //创建对象，但并不会调用其构造方法。如果类未被初始化，将初始化类。
    public native Object allocateInstance(Class<?> cls)
        throws InstantiationException;
```

## 2.3数组相关
``` java
    /**
     * Report the offset of the first element in the storage allocation of a
     * given array class.  If {@link #arrayIndexScale} returns a non-zero value
     * for the same class, you may use that scale factor, together with this
     * base offset, to form new offsets to access elements of arrays of the
     * given class.
     *
     * @see #getInt(Object, long)
     * @see #putInt(Object, long, int)
     */
  //返回数组中第一个元素的偏移地址
    public native int arrayBaseOffset(Class<?> arrayClass);
  //boolean、byte、short、char、int、long、float、double，及对象类型均有以下方法
    /** The value of {@code arrayBaseOffset(boolean[].class)} */
    public static final int ARRAY_BOOLEAN_BASE_OFFSET
            = theUnsafe.arrayBaseOffset(boolean[].class);
​
    /**
     * Report the scale factor for addressing elements in the storage
     * allocation of a given array class.  However, arrays of "narrow" types
     * will generally not work properly with accessors like {@link
     * #getByte(Object, int)}, so the scale factor for such classes is reported
     * as zero.
     *
     * @see #arrayBaseOffset
     * @see #getInt(Object, long)
     * @see #putInt(Object, long, int)
     */
  //返回数组中每一个元素占用的大小
    public native int arrayIndexScale(Class<?> arrayClass);
​
  //boolean、byte、short、char、int、long、float、double，及对象类型均有以下方法
    /** The value of {@code arrayIndexScale(boolean[].class)} */
    public static final int ARRAY_BOOLEAN_INDEX_SCALE
            = theUnsafe.arrayIndexScale(boolean[].class);
```
通过arrayBaseOffset和arrayIndexScale可定位数组中每个元素在内存中的位置。

## 2.4并发相关
### 2.4.1CAS相关
CAS：CompareAndSwap，内存偏移地址offset，预期值expected，新值x。如果变量在当前时刻的值和预期值expected相等，尝试将变量的值更新为x。如果更新成功，返回true；否则，返回false。

``` java
  //更新变量值为x，如果当前值为expected
  //o：对象 offset：偏移量 expected：期望值 x：新值
    public final native boolean compareAndSwapObject(Object o, long offset,
                                                     Object expected,
                                                     Object x);
​
    public final native boolean compareAndSwapInt(Object o, long offset,
                                                  int expected,
                                                  int x);
​
    public final native boolean compareAndSwapLong(Object o, long offset,
                                                   long expected,
                                                   long x);
```
从Java 8开始，Unsafe中提供了以下方法：

``` java
  //增加
  public final int getAndAddInt(Object o, long offset, int delta) {
        int v;
        do {
            v = getIntVolatile(o, offset);
        } while (!compareAndSwapInt(o, offset, v, v + delta));
        return v;
    }
​
    public final long getAndAddLong(Object o, long offset, long delta) {
        long v;
        do {
            v = getLongVolatile(o, offset);
        } while (!compareAndSwapLong(o, offset, v, v + delta));
        return v;
    }
  //设置
    public final int getAndSetInt(Object o, long offset, int newValue) {
        int v;
        do {
            v = getIntVolatile(o, offset);
        } while (!compareAndSwapInt(o, offset, v, newValue));
        return v;
    }
​
    public final long getAndSetLong(Object o, long offset, long newValue) {
        long v;
        do {
            v = getLongVolatile(o, offset);
        } while (!compareAndSwapLong(o, offset, v, newValue));
        return v;
    }
​
    public final Object getAndSetObject(Object o, long offset, Object newValue) {
        Object v;
        do {
            v = getObjectVolatile(o, offset);
        } while (!compareAndSwapObject(o, offset, v, newValue));
        return v;
    }
```

### 2.4.2线程调度相关
``` java
    //取消阻塞线程
  public native void unpark(Object thread);
  //阻塞线程
    public native void park(boolean isAbsolute, long time); 
  //获得对象锁
    public native void monitorEnter(Object o);
  //释放对象锁
    public native void monitorExit(Object o);
  //尝试获取对象锁，返回true或false表示是否获取成功
    public native boolean tryMonitorEnter(Object o);
```

### 2.4.3volatile相关读写
Java中的基本类型（boolean、byte、char、short、int、long、float、double）及对象引用类型都有以下方法。

``` java
  //从对象的指定偏移量处获取变量的引用，使用volatile的加载语义
  //相当于getObject(Object, long)的volatile版本
    public native Object getObjectVolatile(Object o, long offset);
​
  //存储变量的引用到对象的指定的偏移量处，使用volatile的存储语义
  //相当于putObject(Object, long, Object)的volatile版本
    public native void    putObjectVolatile(Object o, long offset, Object x);
```

``` java
    /**
     * Version of {@link #putObjectVolatile(Object, long, Object)}
     * that does not guarantee immediate visibility of the store to
     * other threads. This method is generally only useful if the
     * underlying field is a Java volatile (or if an array cell, one
     * that is otherwise only accessed using volatile accesses).
     */
    public native void    putOrderedObject(Object o, long offset, Object x);
​
    /** Ordered/Lazy version of {@link #putIntVolatile(Object, long, int)}  */
    public native void    putOrderedInt(Object o, long offset, int x);
​
    /** Ordered/Lazy version of {@link #putLongVolatile(Object, long, long)} */
    public native void    putOrderedLong(Object o, long offset, long x);
``` 
### 2.4.4内存屏障相关
Java 8引入 ，用于定义内存屏障，避免代码重排序。

``` java
  //内存屏障，禁止load操作重排序，即屏障前的load操作不能被重排序到屏障后，屏障后的load操作不能被重排序到屏障前
    public native void loadFence();
  //内存屏障，禁止store操作重排序，即屏障前的store操作不能被重排序到屏障后，屏障后的store操作不能被重排序到屏障前
    public native void storeFence();
  //内存屏障，禁止load、store操作重排序
    public native void fullFence();
```

## 2.5直接内存访问（非堆内存）
allocateMemory所分配的内存需要手动free（不被GC回收）

``` java
  //（boolean、byte、char、short、int、long、float、double)都有以下get、put两个方法。 
  //获得给定地址上的int值
  public native int getInt(long address);
    //设置给定地址上的int值
    public native void putInt(long address, int x); 
  //获得本地指针
    public native long getAddress(long address);
  //存储本地指针到给定的内存地址
    public native void putAddress(long address, long x);
  
  //分配内存
    public native long allocateMemory(long bytes);
  //重新分配内存
    public native long reallocateMemory(long address, long bytes);
  //初始化内存内容
    public native void setMemory(Object o, long offset, long bytes, byte value);
  //初始化内存内容
    public void setMemory(long address, long bytes, byte value) {
        setMemory(null, address, bytes, value);
    }
  //内存内容拷贝
    public native void copyMemory(Object srcBase, long srcOffset,
                                  Object destBase, long destOffset,
                                  long bytes);
  //内存内容拷贝
    public void copyMemory(long srcAddress, long destAddress, long bytes) {
        copyMemory(null, srcAddress, null, destAddress, bytes);
    }
  //释放内存
    public native void freeMemory(long address);

## 2.6系统相关。

``` java
  //返回指针的大小。返回值为4或8。
    public native int addressSize();

    /** The value of {@code addressSize()} */
    public static final int ADDRESS_SIZE = theUnsafe.addressSize();

  //内存页的大小。
    public native int pageSize();
``` 

 

# 3.参考资料
https://www.cnblogs.com/pkufork/p/java_unsafe.html 说一说Java中的Unsafe类

https://www.cnblogs.com/suxuan/p/4948608.html java魔法类：sun.misc.Unsafe