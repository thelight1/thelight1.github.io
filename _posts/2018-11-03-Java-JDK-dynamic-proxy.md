---
layout: post
title:  "解析JDK动态代理实现原理"
categories: Java 
tags: Java 
author: thelight1
mathjax: true
---
* content
{:toc}

# JDK动态代理使用实例


代理模式的类图如上。关于静态代理的示例网上有很多，在这里就不讲了。

因为本篇讲述要点是JDK动态代理的实现原理，直接从JDK动态代理实例开始。

首先是Subject接口类。

``` Java
package proxy.pattern;
​
public interface Subject {
    void request() throws Exception;
}
```
接着是RealSubject类。

``` Java
package proxy.pattern;
​
public class RealSubject implements Subject {
​
    public void request() {
        System.out.println("RealSubject execute request()");
    }
}
```
下面是代理对象的InvocationHandler接口实现类。
``` Java
package proxy.dynamic;
​
import proxy.pattern.RealSubject;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
​
public class JdkProxySubject implements InvocationHandler {
​
    private RealSubject realSubject;
​
    public JdkProxySubject(RealSubject realSubject) {
        this.realSubject = realSubject;
    }
​
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("before");
        Object result = null;
        try {
            result = method.invoke(realSubject, args);
        } catch (Exception e) {
            throw e;
        } finally {
            System.out.println("after");
        }
        return result;
    }
}
```
InvocationHandler接口的定义如下。
``` Java
/**
 * {@code InvocationHandler} is the interface implemented by
 * the <i>invocation handler</i> of a proxy instance.
 *
 * <p>Each proxy instance has an associated invocation handler.
 * When a method is invoked on a proxy instance, the method
 * invocation is encoded and dispatched to the {@code invoke}
 * method of its invocation handler.
 */
//每一个代理对象都有一个相应的invocation handler。
//此接口应当由proxy instance的invocation handler实现
​
//当代理对象的方法被调用时，方法调用将会委派为invocation handler的invoke方法
//（这里有个疑问就是：方法委派是如何实现的？
//为什么调用代理对象的方法时，会调用invocation handler的invoke方法。
//答案在生成的代理类的源代码里，可以看最下面对代理类的源代码的分析）
public interface InvocationHandler {
  
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
```

``` Java
package proxy.dynamic;
​
import proxy.pattern.RealSubject;
import proxy.pattern.Subject;
import java.lang.reflect.Proxy;
​
public class Client {
​
    public static void main(String[] args) throws Exception {
        //通过设置参数，将生成的代理类的.class文件保存在本地
        System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
        //通过调用Proxy.newProxyInstance生成代理对象
        //方法参数为：1）classLoader  2）要代理的接口 3）代理对象的InvocationHandler
        //(通过方法参数也可以看出来，JDK代理只能通过代理接口来来实现动态代理)
        Subject subject = (Subject) Proxy.newProxyInstance(Client.class.getClassLoader(),
                new Class[]{Subject.class}, new JdkProxySubject(new RealSubject()));
        //调用代理对象的request方法。
        //（根据InvocationHandler接口的定义，可以知道实际调用的是JdkProxySubject里的invoke方法）
        subject.request();
    }
}
```
运行结果如下。

``` shell
before
RealSubject execute request()
after
```
# JDK动态代理实现原理分析
生成代理对象的关键调用链为Proxy.newProxyInstance()-------->getProxyClass0()------->ProxyClassFactory.apply()-------->ProxyGenerator.generateProxyClass()

## Proxy.newProxyInstance()
newProxyInstance方法的完整代码如下。

``` Java
    @CallerSensitive
    public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h)
        throws IllegalArgumentException
    {
        Objects.requireNonNull(h);
​
        final Class<?>[] intfs = interfaces.clone();
        final SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            checkProxyAccess(Reflection.getCallerClass(), loader, intfs);
        }
​
        /*
         * Look up or generate the designated proxy class.
         */
        //
        Class<?> cl = getProxyClass0(loader, intfs);
​
        /*
         * Invoke its constructor with the designated invocation handler.
         */
        try {
            if (sm != null) {
                checkNewProxyPermission(Reflection.getCallerClass(), cl);
            }
​
            final Constructor<?> cons = cl.getConstructor(constructorParams);
            final InvocationHandler ih = h;
            if (!Modifier.isPublic(cl.getModifiers())) {
                AccessController.doPrivileged(new PrivilegedAction<Void>() {
                    public Void run() {
                        cons.setAccessible(true);
                        return null;
                    }
                });
            }
            return cons.newInstance(new Object[]{h});
        } catch (IllegalAccessException|InstantiationException e) {
            throw new InternalError(e.toString(), e);
        } catch (InvocationTargetException e) {
            Throwable t = e.getCause();
            if (t instanceof RuntimeException) {
                throw (RuntimeException) t;
            } else {
                throw new InternalError(t.toString(), t);
            }
        } catch (NoSuchMethodException e) {
            throw new InternalError(e.toString(), e);
        }
    }
```
我们将错误判断等无用代码去掉后，代码如下。
``` Java
@CallerSensitive
public static Object newProxyInstance(ClassLoader loader,
                                      Class<?>[] interfaces,
                                      InvocationHandler h)
  throws IllegalArgumentException {
        final Class<?>[] intfs = interfaces.clone();
        /*
         * Look up or generate the designated proxy class.
         */
        //根据提供的接口（在我们的例子中，即Subject接口），生成代理类
        Class<?> cl = getProxyClass0(loader, intfs);
​
        /*
         * Invoke its constructor with the designated invocation handler.
         */
        final Constructor<?> cons = cl.getConstructor(constructorParams);
        //调用代理类的构造器，构造器参数为InvocationHandler的接口实现类（在我们的例子中，即JdkProxySubject）
        return cons.newInstance(new Object[]{h});
}
```
下面，我们看看getProxyClass0方法，看看代理类是如何生成的。

## getProxyClass0()
``` Java
private static Class<?> getProxyClass0(ClassLoader loader,
                                       Class<?>... interfaces) {
  if (interfaces.length > 65535) {
    throw new IllegalArgumentException("interface limit exceeded");
  }
​
  // If the proxy class defined by the given loader implementing
  // the given interfaces exists, this will simply return the cached copy;
  // otherwise, it will create the proxy class via the ProxyClassFactory
  //已经生成过代理类会放到cache里，没有生成过的话，会由ProxyClassFactory创建。
  //因此，我们可以直接找到ProxyClassFactory，查看代理类是如何创建的。
  return proxyClassCache.get(loader, interfaces);
}
``` 
因此，我们可以继续找到ProxyClassFactory，查看代理类是如何创建的。

## ProxyClassFactory.apply()
ProxyClassFactory类完整的代码如下。

``` Java
private static final class ProxyClassFactory
  implements BiFunction<ClassLoader, Class<?>[], Class<?>>
    {
        // prefix for all proxy class names
        private static final String proxyClassNamePrefix = "$Proxy";
​
        // next number to use for generation of unique proxy class names
        private static final AtomicLong nextUniqueNumber = new AtomicLong();
​
        @Override
        public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
​
            Map<Class<?>, Boolean> interfaceSet = new IdentityHashMap<>(interfaces.length);
            for (Class<?> intf : interfaces) {
                /*
                 * Verify that the class loader resolves the name of this
                 * interface to the same Class object.
                 */
                Class<?> interfaceClass = null;
                try {
                    interfaceClass = Class.forName(intf.getName(), false, loader);
                } catch (ClassNotFoundException e) {
                }
                if (interfaceClass != intf) {
                    throw new IllegalArgumentException(
                        intf + " is not visible from class loader");
                }
                /*
                 * Verify that the Class object actually represents an
                 * interface.
                 */
                if (!interfaceClass.isInterface()) {
                    throw new IllegalArgumentException(
                        interfaceClass.getName() + " is not an interface");
                }
                /*
                 * Verify that this interface is not a duplicate.
                 */
                if (interfaceSet.put(interfaceClass, Boolean.TRUE) != null) {
                    throw new IllegalArgumentException(
                        "repeated interface: " + interfaceClass.getName());
                }
            }
​
            String proxyPkg = null;     // package to define proxy class in
            int accessFlags = Modifier.PUBLIC | Modifier.FINAL;
​
            /*
             * Record the package of a non-public proxy interface so that the
             * proxy class will be defined in the same package.  Verify that
             * all non-public proxy interfaces are in the same package.
             */
            for (Class<?> intf : interfaces) {
                int flags = intf.getModifiers();
                if (!Modifier.isPublic(flags)) {
                    accessFlags = Modifier.FINAL;
                    String name = intf.getName();
                    int n = name.lastIndexOf('.');
                    String pkg = ((n == -1) ? "" : name.substring(0, n + 1));
                    if (proxyPkg == null) {
                        proxyPkg = pkg;
                    } else if (!pkg.equals(proxyPkg)) {
                        throw new IllegalArgumentException(
                            "non-public interfaces from different packages");
                    }
                }
            }
​
            if (proxyPkg == null) {
                // if no non-public proxy interfaces, use com.sun.proxy package
                proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
            }
​
            /*
             * Choose a name for the proxy class to generate.
             */
            long num = nextUniqueNumber.getAndIncrement();
            String proxyName = proxyPkg + proxyClassNamePrefix + num;
​
            /*
             * Generate the specified proxy class.
             */
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
            try {
                return defineClass0(loader, proxyName,
                                    proxyClassFile, 0, proxyClassFile.length);
            } catch (ClassFormatError e) {
                /*
                 * A ClassFormatError here means that (barring bugs in the
                 * proxy class generation code) there was some other
                 * invalid aspect of the arguments supplied to the proxy
                 * class creation (such as virtual machine limitations
                 * exceeded).
                 */
                throw new IllegalArgumentException(e.toString());
            }
        }
    }
``` 
我们将无用的代码去掉后，代码如下。

``` Java
    private static final class ProxyClassFactory
        implements BiFunction<ClassLoader, Class<?>[], Class<?>>
    {
        // prefix for all proxy class names
        private static final String proxyClassNamePrefix = "$Proxy";
​
        // next number to use for generation of unique proxy class names
        private static final AtomicLong nextUniqueNumber = new AtomicLong();
​
        @Override
        public Class<?> apply(ClassLoader loader, Class<?>[] interfaces) {
            //第一步：确定包名，如果没有非public的代理接口，包名使用com.sun.proxy
            String proxyPkg = null;     // package to define proxy class in
            int accessFlags = Modifier.PUBLIC | Modifier.FINAL;
            if (proxyPkg == null) {
                // if no non-public proxy interfaces, use com.sun.proxy package
                proxyPkg = ReflectUtil.PROXY_PACKAGE + ".";
            }
            //第二步：确定类名。（在我们上述例子中，生成的代理类名为com.sun.proxy.$Proxy0）
            /*
             * Choose a name for the proxy class to generate.
             */
            long num = nextUniqueNumber.getAndIncrement();
            String proxyName = proxyPkg + proxyClassNamePrefix + num;
            //第三步：根据类名、代理接口等信息，生成.class文件，就是javac编译后的.class文件。
            /*
             * Generate the specified proxy class.
             */
            byte[] proxyClassFile = ProxyGenerator.generateProxyClass(
                proxyName, interfaces, accessFlags);
           //第四步：根据生成的.class文件，返回一个代理类Class。此方法为native方法
           return defineClass0(loader, proxyName,
                                    proxyClassFile, 0, proxyClassFile.length);
            
        }
    }
```
## ProxyGenerator.generateProxyClass()
ProxyGenerator.generateProxyClass生成.class文件的过程其实就是根据.class文件格式来生成对应字节数组
由于代码太长了，不贴了，可以看这个http://hg.openjdk.java.net/jdk7/jdk7/jdk/file/f097ca2434b1/src/share/classes/sun/misc/ProxyGenerator.java

## 生成的代理对象的.class文件
在创建代理对象前，可以通过下面语句，可以将生成的代理类的.class保存在本地。

``` Java
System.getProperties().put("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");
``` 
生成的代理类的.class文件发编译后，是下面这个样子的。
``` Java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//
​
package com.sun.proxy;
​
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Method;
import java.lang.reflect.Proxy;
import java.lang.reflect.UndeclaredThrowableException;
import proxy.pattern.Subject;
​
public final class $Proxy0 extends Proxy implements Subject {
    private static Method m1;
    private static Method m2;
    private static Method m3;
    private static Method m0;
​
    public $Proxy0(InvocationHandler h) throws  {
        super(h);//通过这句，将invocationHandler实例传给了父类构造器。
    }
​
    public final boolean equals(Object var1) throws  {
        try {
            return ((Boolean)super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }
​
    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
​
    //$Proxy0继承了Proxy，且将InvocationHandler h在构造时传给了Proxy。
    //因此，super.h.invoke(this, m3, (Object[])null)其实就是调用的invoker handler的invoke方法。
    //也正是像InvocationHandler定义中所说的，【当proxy instance的方法被调用时，方法调用将会委派为invocation handler的invoke方法】。
    public final void request() throws Exception {
        try { 
            super.h.invoke(this, m3, (Object[])null);
        } catch (Exception | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
​
    public final int hashCode() throws  {
        try {
            return ((Integer)super.h.invoke(this, m0, (Object[])null)).intValue();
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }
​
    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", new Class[]{Class.forName("java.lang.Object")});
            m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
            m3 = Class.forName("proxy.pattern.Subject").getMethod("request", new Class[0]);
            m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```​
``` Java
public class Proxy implements java.io.Serializable {
​
    /**
     * the invocation handler for this proxy instance.
     * @serial
     */
    protected InvocationHandler h;
    /**
     * Constructs a new {@code Proxy} instance from a subclass
     * (typically, a dynamic proxy class) with the specified value
     * for its invocation handler.
     *
     * @param  h the invocation handler for this proxy instance
     *
     * @throws NullPointerException if the given invocation handler, {@code h},
     *         is {@code null}.
     */
    protected Proxy(InvocationHandler h) {
        Objects.requireNonNull(h);
        this.h = h;
    }
```