---
layout: post
title:  "Log4j2中同步日志与异步日志"
categories: Log4j2
tags: Log4j2 
author: thelight1
mathjax: true
---
* content
{:toc}

# 1.背景
Log4j 2中记录日志的方式有同步日志和异步日志两种方式，其中异步日志又可分为使用AsyncAppender和使用AsyncLogger两种方式。

# 2.Log4j2中的同步日志
所谓同步日志，即当输出日志时，必须等待日志输出语句执行完毕后，才能执行后面的业务逻辑语句。<br><br>
下面通过一个例子来了解Log4j2中的同步日志，并借此来探究整个日志输出过程。<br>
log4j2.xml配置如下：<br>

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="debug" name="MyApp" packages="">
	<!--全局Filter-->
    <ThresholdFilter level="ALL"/>
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n" />
        </Console>
        <RollingFile name="RollingFile" fileName="logs/app.log"
                     filePattern="logs/app-%d{yyyy-MM-dd HH}.log">
			<!--Appender的Filter-->
            <ThresholdFilter level="info" onMatch="ACCEPT" onMismatch="DENY"/>
            <PatternLayout>
                <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
            </PatternLayout>
            <Policies>
                <SizeBasedTriggeringPolicy size="500MB"/>
            </Policies>
        </RollingFile>
    </Appenders>
    <Loggers>
        <Logger name="com.meituan.Main" level="trace" additivity="false">
			<!--Logger的Filter-->
            <ThresholdFilter level="debug"/>
            <appender-ref ref="RollingFile"/>
        </Logger>
        <Root level="debug">
            <AppenderRef ref="Console"/>
        </Root>
    </Loggers>
</Configuration>
```
java代码如下：<br>

``` java
package com.meituan;

import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;

public class Main {
    public static void main(String args[]) {
        Logger logger = LogManager.getLogger(Main.class);
        Person person = new Person("Li", "lei");
        logger.info("hello, {}", person);
    }

    private static class Person {
        private String firstName;
        private String lastName;

        public Person(String firstName, String lastName) {
            this.firstName = firstName;
            this.lastName = lastName;
        }

        public String toString() {
            return "Person[" + firstName + "," + lastName + "]";
        }
    }
}
```
使用以上的配置，当我们运行程序后，以下log将被添加到logs/app.log中。<br>
**2017-09-13 19:41:00,889 INFO c.m.Main [main] hello, Person[Li,lei]**
####logger.info运行时到底发生了什么？日志信息是如何输出到app.log中的？
![Log4j2中的同步日志](images/log4j2_SyncLogger.png)
Log4j2中日志输出的详细过程如下：<br>
#### 1.首先使用全局Filter对日志事件进行过滤。
Log4j2中的日志Level分为8个级别，优先级从高到低依次为OFF、FATAL、ERROR、WARN、INFO、DEBUG、TRACE、 ALL。<br>
全局Filter的Level为ALL，表示允许输出所有级别的日志。logger.info()请求输出INFO级别的日志，通过。
#### 2.使用Logger的Level对日志事件进行过滤。
Logger的Level为TRACE，表示允许输出TRACE级别及以上级别的日志。logger.info()请求输出INFO级别的日志，通过。
#### 3.生成日志输出内容Message。
使用占位符的方式输出日志，输出语句为logger.info("increase {} from {} to {}", arg1, arg2, arg3)的形式，最终输出时{}占位符处的内容将用arg1,arg2,arg3的字符串填充。<br>
log4j2用Object[]保存参数信息，在这一阶段会将Object[]转换为String[]，生成含有输出模式串"increase {} from {} to {}"和参数数组String[]的Message，为后续日志格式化输出做准备。
#### 4.生成LogEvent。
LogEvent中含有loggerName（日志的输出者），level（日志级别），timeMillis（日志的输出时间），message（日志输出内容），threadName（线程名称）等信息。<br>
在上述程序中，生成的LogEvent的属性值为loggerName=com.meituan.Main，Level=INFO，timeMillis=1505659461759，message为步骤3中创建的Message，threadName=main。
#### 5.使用Logger配置的Filter对日志事件进行过滤。
Logger配置的Filter的Level为DEBUG，表示允许输出DEBUG及以上级别的日志。logger.info()请求输出INFO级别的日志，通过。
#### 6.使用Logger对应的Appender配置的Filter对日志事件进行过滤。
Appender配置的Filter配置的INFO级别日志onMatch=ACCEPT，表示允许输出INFO级别的日志。logger.info()请求输出INFO级别的日志，通过。
#### 7.判断是否需要触发rollover。
**此步骤不是日志输出的必须步骤，如配置的Appender为无需进行rollover的Appender，则无此步骤。**<br>
因为使用RollingFileAppender，且配置了基于文件大小的rollover触发策略，在此阶段会判断是否需要触发rollover。判断方式为当前的文件大小是否达到了指定的size，如果达到了，触发rollover操作。<br>
关于Log4j2中的RollingFileAppender的rollover，可参见[【Log4j2中RollingFile的文件滚动更新机制】](log4j2_RollingFileAppender.md)。
#### 8.PatternLayout对LogEvent进行格式化，生成可输出的字符串。
上述log4j2.xml文件中配置的Pattern及各个参数的意义如下：<br>
**<Pattern>%d %p %c{1.} [%t] %m%n</Pattern>**

|参数|意义|
|----|----|
|%d|日期格式，默认形式为2012-11-02 14:34:02,781|
|%p|日志级别|
|%c{1.}|%c表示Logger名字，{1.}表示精确度。若Logger名字为org.apache.commons.Foo，则输出o.a.c.Foo。|
|%t|处理LogEvent的线程的名字|
|%m|日志内容|
|%n|行分隔符。"\n"或"\r\n"。|

在此步骤，PatternLayout将根据Pattern的模式，利用各种Converter对LogEvent的相关信息进行转换，最终拼接成可输出的日志字符串。如DatePatternConverter对LogEvent的日志输出时间进行格式化转换；LevelPatternConverter对LogEvent的日志级别信息进行格式化转换；LoggerPatternConverter对LogEvent的Logger的名字进行格式化转换；MessagePatternConverter对LogEvent的日志输出内容进行格式化转换等。<br>
经各种Converter转换后，LogEvent的信息被格式化为指定格式的字符串。
![格式化后生成的字符串](images/log4j2_ConvertedString.png)
#### 9.使用OutputStream，将日志输出到文件。
将日志字符串序列化为字节数组，使用字节流OutoutStream将日志输出到文件中。如果配置了immediateFlush为true，打开app.log就可观察到输出的日志了。
# 3.Log4j2中的异步日志
使用log4j2的同步日志进行日志输出，日志输出语句与程序的业务逻辑语句将在同一个线程运行。而使用异步日志进行输出时，日志输出语句与业务逻辑语句并不是在同一个线程中运行，而是有专门的线程用于进行日志输出操作，处理业务逻辑的主线程不用等待即可执行后续业务逻辑。<br><br>
Log4j2中的异步日志实现方式有**AsyncAppender**和**AsyncLogger**两种。<br>
其中，AsyncAppender采用了ArrayBlockingQueue来保存需要异步输出的日志事件；AsyncLogger则使用了Disruptor框架来实现高吞吐。
## 3.1 AsyncAppender

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn">
  <Appenders>
    <RollingFile name="MyFile" fileName="logs/app.log">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
	  <SizeBasedTriggeringPolicy size="500MB"/>
    </RollingFile>
    <Async name="Async">
      <AppenderRef ref="MyFile"/>
    </Async>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="Async"/>
    </Root>
  </Loggers>
</Configuration>
```
上面就是一个使用AsyncAppender的典型配置，配置AsyncAppender后，日志事件写入文件的操作将在单独的线程中执行。<br>
AsyncAppender的常用参数

|参数名|类型|说明|
|-----|---|----|
|name|String|	AsyncAppender的名字。|
|AppenderRef|	String|异步调用的Appender的名字，可以配置多个。|
|blocking|boolean|	默认为true。如果为true，appender将一直等待直到queue中有空闲；如果为false，当队列满的时候，日志事件将被丢弃。(如果配置了error appender，要丢弃的日志事件将由error appender处理)|
|bufferSize|integer|队列中可存储的日志事件的最大数量，默认为128。(源码中为128，Log4j2官网为1024，官网信息有误)|

关于AsyncAppender的其他参数，可参考Log4j2官网对AsyncAppender的详细介绍。
![AsyncAppender](images/log4j2_AsyncAppender.png)
每个AsyncAppender，内部维护了一个ArrayBlockingQueue，并将创建一个线程用于输出日志事件，如果配置了多个AppenderRef，将分别使用对应的Appender进行日志输出。
## 3.2 AsyncLogger
Log4j2中的AsyncLogger的内部使用了Disruptor框架。
### Disruptor简介
Disruptor是英国外汇交易公司LMAX开发的一个高性能队列，基于Disruptor开发的系统单线程能支撑每秒600万订单。<br>
目前，包括Apache Strom、Log4j2在内的很多知名项目都应用了Disruptor来获取高性能。<br>
Disruptor框架内部核心数据结构为RingBuffer，其为无锁环形队列。<br>
![RingBuffer](images/ringbuffer.png)
#### 单线程每秒能够处理600万订单，Disruptor为什么这么快？
#### a.lock-free-使用了CAS来实现线程安全
ArrayBlockingQueue使用锁实现并发控制，当get或put时，当前访问线程将上锁，当多生产者、多消费者的大量并发情形下，由于锁竞争、线程切换等，会有性能损失。Disruptor通过CAS实现多生产者、多消费者对RingBuffer的并发访问。CAS相当于乐观锁，其性能优于Lock的性能。
#### b.使用缓存行填充解决伪共享问题
计算机体系结构中，内存的访问速度远远低于CPU的运行速度，在内存和CPU之间，加入Cache，CPU首先访问Cache中的数据，CaChe未命中，才访问内存中的数据。<br>
伪共享：Cache是以缓存行（cache line）为单位存储的，当多个线程修改互相独立的变量时，如果这些变量共享同一个缓存行，就会无意中影响彼此的性能。

![FalseSharing](images/False-Sharing.png)<br>

关于伪共享的深度分析，可参考[《伪共享，并发编程的性能杀手》](http://www.cnblogs.com/cyfonly/p/5800758.html)这篇文章。
### AsyncLogger
Log4j2异步日志如何进行日志输出，我们同样从一个例子出发来探究Log4j2的异步日志。<br>
log4j2.xml配置如下：<br>

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="debug" name="MyApp" packages="">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n" />
        </Console>
        <RollingFile name="RollingFile" fileName="logs/app.log"
                     filePattern="logs/app-%d{yyyy-MM-dd HH}.log">
            <PatternLayout>
                <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
            </PatternLayout>
            <Policies>
                <SizeBasedTriggeringPolicy size="500MB"/>
            </Policies>
        </RollingFile>
        <RollingFile name="RollingFile2" fileName="logs/app2.log"
                     filePattern="logs/app2-%d{yyyy-MM-dd HH}.log">
            <PatternLayout>
                <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
            </PatternLayout>
            <Policies>
                <SizeBasedTriggeringPolicy size="500MB"/>
            </Policies>
        </RollingFile>
    </Appenders>
    <Loggers>
        <AsyncLogger name="com.meituan.Main" level="trace" additivity="false">
            <appender-ref ref="RollingFile"/>
        </AsyncLogger>
        <AsyncLogger name="RollingFile2" level="trace" additivity="false">
            <appender-ref ref="RollingFile2"/>
        </AsyncLogger>
        <Root level="debug">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="RollingFile"/>
        </Root>
    </Loggers>
</Configuration>
```
java代码如下：<br>

``` java
public class Main {
    public static void main(String args[]) {
        Logger logger = LogManager.getLogger(Main.class);
        Logger logger2 = LogManager.getLogger("RollingFile2");

        Person person = new Person("Li", "lei");
        logger.info("hello, {}", person);
        logger2.info("good bye, {}", person);
}
```
上述log4j2.xml中配置了两个AsyncLogger，名字分别为com.meituan.Main和RollingFile2。
并且，在main方法中分别使用两个logger来输出两条日志。

在加载log4j2.xml的启动阶段，如果检测到配置了AsyncRoot或AsyncLogger，将启动一个disruptor实例。
![AsyncLogger](images/log4j2_AsyncLogger.png)
上述程序中，main线程作为生产者，EventProcessor线程作为消费者。<br>
#### 生产者生产消息
当运行到类似于logger.info、logger.debug的输出语句时，将生成的LogEvent放入RingBuffer中。
#### 消费者消费消息
* 如果RingBuffer中有LogEvent需要处理，EventProcessor线程从RingBuffer中取出LogEvent，调用Logger相关联的Appender输出LogEvent（具体输出过程与同步过程相同，同样需要过滤器过滤、PatternLayout格式化等步骤）。
* 如果RingBuffer中没有LogEvent需要处理，EventProcessor线程将处于等待阻塞状态（默认策略）。

需要注意的是，虽然在log4j2.xml中配置了多个AsyncLogger，但是并不是每个AsyncLogger对应着一个处理线程，而是仅仅有一个EventProcessor线程进行日志的异步处理。

# 4.总结

||日志输出方式|
|---|-------|
|sync|同步打印日志，日志输出与业务逻辑在同一线程内，当日志输出完毕，才能进行后续业务逻辑 操作|
|AsyncAppender|异步打印日志，内部采用ArrayBlockingQueue，对每个AsyncAppender创建一个线程用于处理日志输出。|
|AsyncLogger|异步打印日志，采用了高性能并发框架Disruptor，创建一个线程用于处理日志输出。|