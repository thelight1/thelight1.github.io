---
layout: post
title:  "Log4j2中RollingFile的文件滚动更新机制"
categories: Log4j2
tags: Log4j2 
author: thelight1
mathjax: true
---
* content
{:toc}


## 1.什么是RollingFile
RollingFileAppender是Log4j2中的一种能够实现日志文件滚动更新(rollover)的Appender。<br><br>
rollover的意思是当满足一定条件(如文件达到了指定的大小，达到了指定的时间)后，就重命名原日志文件进行归档，并生成新的日志文件用于log写入。如果还设置了一定时间内允许归档的日志文件的最大数量，将对过旧的日志文件进行删除操作。<br><br>
RollingFile实现日志文件滚动更新，依赖于TriggeringPolicy和RolloverStrategy。<br>
其中，TriggeringPolicy为触发策略，其决定了何时触发日志文件的rollover，即When。<br>
RolloverStrategy为滚动更新策略，其决定了当触发了日志文件的rollover时，如何进行文件的rollover，即How。<br>
Log4j2提供了默认的rollover策略DefaultRolloverStrategy。<br><br>
下面通过一个log4j2.xml文件配置简单了解RollingFile的配置。

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{yyyy-MM-dd HH}.log">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <TimeBasedTriggeringPolicy interval="1"/>
        <SizeBasedTriggeringPolicy size="250MB"/>
      </Policies>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```
上述配置文件中配置了一个RollingFile，日志写入logs/app.log文件中，每经过1小时或者当文件大小到达250M时，按照app-2017-08-01 12.log的格式对app.log进行重命名并归档，并生成新的文件用于写入log。<br><br>
其中，fileName指定日志文件的位置和文件名称（如果文件或文件所在的目录不存在，会创建文件。）<br>
filePattern指定触发rollover时，文件的重命名规则。filePattern中可以指定类似于SimpleDateFormat中的date/time pattern，如yyyy-MM-dd HH，或者%i指定一个整数计数器。<br>
TimeBasedTriggeringPolicy指定了基于时间的触发策略。<br>
SizeBasedTriggeringPolicy指定了基于文件大小的触发策略。
## 2.TriggeringPolicy
RollingFile的触发rollover的策略有CronTriggeringPolicy(Cron表达式触发)、OnStartupTriggeringPolicy(JVM启动时触发)、SizeBasedTriggeringPolicy(基于文件大小)、TimeBasedTriggeringPolicy(基于时间)、CompositeTriggeringPolicy(多个触发策略的混合，如同时基于文件大小和时间)。<br>
其中，SizeBasedTriggeringPolicy(基于日志文件大小)、TimeBasedTriggeringPolicy(基于时间)或同时基于文件大小和时间的混合触发策略最常用。
### SizeBasedTriggeringPolicy
SizeBasedTriggeringPolicy规定了当日志文件达到了指定的size时，触发rollover操作。size参数可以用KB、MB、GB等做后缀来指定具体的字节数，如20MB。
``` xml
<SizeBasedTriggeringPolicy size="250MB"/>
```
### TimeBasedTriggeringPolicy
TimeBasedTriggeringPolicy规定了当日志文件名中的date/time pattern不再符合filePattern中的date/time pattern时，触发rollover操作。
比如，filePattern指定文件重命名规则为app-%d{yyyy-MM-dd HH}.log，文件名为app-2017-08-25 11.log，当时间达到2017年8月25日中午12点（2017-08-25 12），将触发rollover操作。

|参数名|类型|描述|
|-----|---|---|
|interval|integer|此参数需要与filePattern结合使用，规定了触发rollover的频率，默认值为1。假设interval为4，若filePattern的date/time pattern的最小时间粒度为小时(如yyyy-MM-dd HH)，则每4小时触发一次rollover；若filePattern的date/time pattern的最小时间粒度为分钟(如yyyy-MM-dd HH-mm)，则每4分钟触发一次rollover。|
|modulate|boolean|指明是否对interval进行调节，默认为false。若modulate为true，会以0为开始对interval进行偏移计算。例如，最小时间粒度为小时，当前为3:00，interval为4，则以后触发rollover的时间依次为4:00，8:00，12:00，16:00,...。|

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Appenders>
    <RollingFile name="RollingFile" fileName="logs/app.log"
                 filePattern="logs/app-%d{yyyy-MM-dd HH}-%i.log">
      <PatternLayout>
        <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
      </PatternLayout>
      <Policies>
        <TimeBasedTriggeringPolicy />
        <SizeBasedTriggeringPolicy size="250 MB"/>
      </Policies>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```

上述配置文件中，filePattern中yyyy-MM-dd HH最小时间粒度为小时，TimeBasedTriggeringPolicy中interval使用默认值1，将每1小时触发一次rollover。
若将filePattern改为filePattern=“logs/app-%d{yyyy-MM-dd HH-mm}-%i.log”，yyyy-MM-dd HH-mm最小时间粒度为分钟，将每1分钟触发一次rollover。
### CompositeTriggeringPolicy
将多个TriggeringPolicy放到Policies中,表示使用复合策略

``` xml
<Policies>
    <TimeBasedTriggeringPolicy />
    <SizeBasedTriggeringPolicy size="250MB"/>
</Policies>
```
如上，同时使用了TimeBasedTriggeringPolicy、SizeBasedTriggeringPolicy，有一个条件满足，就会触发rollover。
## 3.DefaultRolloverStrategy
DefaultRolloverStrategy指定了当触发rollover时的默认策略。<br>
DefaultRolloverStrategy是Log4j2提供的默认的rollover策略，即使在log4j2.xml中没有显式指明，也相当于为RollingFile配置下添加了如下语句。DefaultRolloverStrategy默认的max为7。<br>
``` xml
<DefaultRolloverStrategy max="7"/>
```
 <br>max参数指定了计数器的最大值。一旦计数器达到了最大值，过旧的文件将被删除。<br>
**注意：不要认为max参数是需要保留的日志文件的最大数目。**

max参数是与filePattern中的计数器%i配合起作用的，其具体作用方式与filePattern的配置密切相关。

* 如果filePattern中仅含有date/time pattern，每次rollover时，将用当前的日期和时间替换文件中的日期格式对文件进行重命名。max参数将不起作用。
如，filePattern="logs/app-%d{yyyy-MM-dd}.log"
* 如果filePattern中仅含有整数计数器（即%i），每次rollover时，文件重命名时的计数器将每次加1（初始值为1），若达到max的值，将删除旧的文件。
如，filePattern="logs/app-%i.log"
* 如果filePattern中既含有date/time pattern，又含有%i，每次rollover时，计数器将每次加1，若达到max的值，将删除旧的文件，直到data/time pattern不再符合，被替换为当前的日期和时间，计数器再从1开始。如，filePattern="logs/app-%d{yyyy-MM-dd HH-mm}-%i.log"
 
假设fileName为logs/app.log，SizeBasedTriggeringPolicy的size为10KB，DefaultRolloverStrategy的max为3。
根据filePattern配置的不同分为以下几种情况：
#### 情况1：filePattern中仅含有date/time pattern
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="trace" name="MyApp" packages="">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n" />
        </Console>
        <RollingFile name="RollingFile" fileName="logs/app.log"
        filePattern="logs/app-%d{yyyy-MM-dd}.log">
            <PatternLayout>
                <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
            </PatternLayout>
            <Policies>
                <SizeBasedTriggeringPolicy size="10KB"/>
            </Policies>
            <DefaultRolloverStrategy max="3"/>
        </RollingFile>
    </Appenders>
    <Loggers>
        <Root level="trace">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="RollingFile"/>
        </Root>
    </Loggers>
</Configuration>
```
filePattern="logs/app-%d{yyyy-MM-dd}.log"，指定当发生rollover时，将按照app-%d{yyyy-MM-dd}.log的格式对文件进行重命名。<br>
每次触发rollover时，将按照如下方式对文件进行rollover。

|第X次rollover|当前用于写入log的文件|归档的文件|描述|
|------------|------------------|--------|----|
|0|	app.log|-|所有的log都写进app.log中。|
|1|	app.log|app-2017-08-17.log|当app.log的size达到10KB，触发第1次rollover，app.log被重命名为app-2017-08-17.log。新的app.log被创建出来，用于写入log。|
|2|	app.log|app-2017-08-17.log|当app.log的size达到10KB，触发第2次rollover，原来的app-2017-08-17.log将删除。app.log被重命名为app-2017-08-17.log。新的app.log文件被创建出来，用于写入log。|

#### 情况2：filePattern中仅含有整数计数器（%i）
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="trace" name="MyApp" packages="">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n" />
        </Console>
        <RollingFile name="RollingFile" fileName="logs/app.log"
        filePattern="logs/app-%i.log">
            <PatternLayout>
                <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
            </PatternLayout>
            <Policies>
                <SizeBasedTriggeringPolicy size="10KB"/>
            </Policies>
            <DefaultRolloverStrategy max="3"/>
        </RollingFile>
    </Appenders>
    <Loggers>
        <Root level="trace">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="RollingFile"/>
        </Root>
    </Loggers>
</Configuration>
```
filePattern="logs/app-%i.log"，其余配置同上。<br>
每次触发rollover时，将按照如下方式对文件进行rollover。

|第X次rollover|当前用于写入log的文件|归档的文件|描述|
|------------|------------------|--------|----|
|0|	app.log|-|所有的log都写进app.log中。|
|1|	app.log|app-1.log|当app.log的size达到10KB，触发第1次rollover，app.log被重命名为app-1.log。新的app.log被创建出来，用于写入log。|
|2|app.log|app-1.log app-2.log|当app.log的size达到10KB，触发第2次rollover，app.log被重命名为app-2.log。新的app.log被创建出来，用于写入log。|
|3|	app.log|app-1.log app-2.log app-3.log|当app.log的size达到10KB，触发第3次rollover，app.log被重命名为app-3.log。新的app.log被创建出来，用于写入log。|
|4|	app.log|app-1.log app-2.log app-3.log|当app.log的size达到10KB，触发第4次rollover，app-1.log被删除(即最初的、最旧的app.log)。app-2.log被重命名为app-1.log，app-3.log被重命名为app-2.log，app.log被重命名为app-3.log。新的app.log被创建出来，用于写入log。|


#### 情况3：如果filePattern中既含有date/time pattern，又含有%i计数器
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="trace" name="MyApp" packages="">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n" />
        </Console>
        <RollingFile name="RollingFile" fileName="logs/app.log"
        filePattern="logs/app-%d{yyyy-MM-dd HH-mm}-%i.log">
            <PatternLayout>
                <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
            </PatternLayout>
            <Policies>
                <TimeBasedTriggeringPolicy />
                <SizeBasedTriggeringPolicy size="10KB"/>
            </Policies>
            <DefaultRolloverStrategy max="3"/>
        </RollingFile>
    </Appenders>
    <Loggers>
        <Root level="trace">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="RollingFile"/>
        </Root>
    </Loggers>
</Configuration>
```
filePattern="logs/app-%d{yyyy-MM-dd HH-mm}-%i.log",同时指定了TimeBasedTriggeringPolicy和SizeBasedTriggeringPolicy的触发策略，每1分钟或者文件大小达到10KB，将触发rollover。<br>
每次触发rollover时，将按照如下方式对文件进行rollover。

|第X次rollover|当前用于写入log的文件|归档的文件|描述|
|------------|------------------|--------|----|
|0|app.log|-|	所有的log都写进app.log中。|
|1|	app.log|app-2017-08-17 20-52-1.log|当app.log的size达到10KB，触发第1次rollover，app.log被重命名为app-2017-08-17 20-52-1.log。新的app.log被创建出来，用于写入log。|
|2|	app.log|app-2017-08-17 20-52-1.log app-2017-08-17 20-52-2.log|当app.log的size达到10KB，触发第2次rollover，app.log被重命名为app-2017-08-17 20-52-2.log。新的app.log被创建出来，用于写入log。|
|3|	app.log|app-2017-08-17 20-52-1.log app-2017-08-17 20-52-2.log app-2017-08-17 20-52-3.log|当app.log的size达到10KB，触发第3次rollover，app.log被重命名为app-2017-08-17 20-52-3.log.log。新的app.log被创建出来，用于写入log。|
|4|	app.log|app-2017-08-17 20-52-1.log app-2017-08-17 20-52-2.log app-2017-08-17 20-52-3.log|当app.log的size达到10KB，触发第4次rollover，因计数器的值到达max值，app-2017-08-17 20-52-1.log被删除(即最初的、最旧的app.log)。app-2017-08-17 20-52-2.log被重命名为app-2017-08-17 20-52-1.log，app-2017-08-17 20-52-3.log被重命名为app-2017-08-17 20-52-2.log，app.log被重命名为app-2017-08-17 20-52-3.log。新的app.log被创建出来，用于写入log。|
|5|	app.log|app-2017-08-17 20-52-1.log app-2017-08-17 20-52-2.log app-2017-08-17 20-52-3.log|当前时间变为app-2017-08-17 20-53，触发第5次rollover，app-2017-08-17 20-52-1.log被删除。app-2017-08-17 20-52-2.log被重命名为app-2017-08-17 20-52-1.log，app-2017-08-17 20-52-3.log被重命名为app-2017-08-17 20-52-2.log，app.log被重命名为app-2017-08-17 20-52-3.log。新的app.log被创建出来，用于写入log。|
|6|	app.log|app-2017-08-17 20-52-1.log app-2017-08-17 20-52-2.log app-2017-08-17 20-52-3.log app-2017-08-17 20-53-1.log|当app.log的size达到10KB，触发第6次rollover，app.log被重命名为app-2017-08-17 20-53-1.log。新的app.log被创建出来，用于写入log。|

总结：<br>
1.max参数是与filePattern中的计数器%i配合起作用的，若filePattern为filePattern="logs/app-%d{yyyy-MM-dd}.log">，由于没有设置%i计数器，max参数将不起作用。<br>
2.max参数不是需要保留的文件的最大个数。如情况3，日志文件date/time pattern不再符合filePattern时，计算器将被重置为1，日志总个数超过了max的指定值。<br>
可认为max参数规定了一定时间范围内归档文件的最大个数。
## 4.DeleteAction
DefaultRolloverStrategy制定了默认的rollover策略，通过max参数可控制一定时间范围内归档的日志文件的最大个数。<br>
Log4j 2.5 引入了DeleteAction，使用户可以自己控制删除哪些文件，而不仅仅是通过DefaultRolloverStrategy的默认策略。<br>
**注意：通过DeleteAction可以删除任何文件，而不仅仅像DefaultRolloverStrategy那样，删除最旧的文件，所以使用的时候需要谨慎！**

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="warn" name="MyApp" packages="">
  <Properties>
    <Property name="baseDir">logs</Property>
  </Properties>
  <Appenders>
    <RollingFile name="RollingFile" fileName="${baseDir}/app.log"
          filePattern="${baseDir}/app-%d{yyyy-MM-dd}.log.gz">
      <PatternLayout pattern="%d %p %c{1.} [%t] %m%n" />
      <CronTriggeringPolicy schedule="0 0 0 * * ?"/>
      <DefaultRolloverStrategy>
        <Delete basePath="${baseDir}" maxDepth="2">
          <IfFileName glob="*/app-*.log.gz" />
          <IfLastModified age="60d" />
        </Delete>
      </DefaultRolloverStrategy>
    </RollingFile>
  </Appenders>
  <Loggers>
    <Root level="error">
      <AppenderRef ref="RollingFile"/>
    </Root>
  </Loggers>
</Configuration>
```
上述配置文件中，Delete部分便是配置DeleteAction的删除策略，指定了当触发rollover时，删除baseDir文件夹或其子文件下面的文件名符合app-*.log.gz且距离最后的修改日期超过60天的文件。<br>
其中，basePath指定了扫描开始路径，为baseDir文件夹。maxDepth指定了目录扫描深度，为2表示扫描baseDir文件夹及其子文件夹。
IfFileName指定了文件名需满足的条件，IfLastModified指定了文件修改时间需要满足的条件。<br>
DeleteAction常用参数如下：

|参数名|类型|描述|
|-----|---|----|
|basePath|String|必填。目录扫描开始路径。|
|maxDepth|int|扫描的最大目录深度。0表示basePath指定的文件自身。Integer.MAX_VALUE表示扫描所有的目录层。默认值为1，表示仅扫描basePath下的文件。|
|testMode|boolean|如果为true，实际的文件不会被删除，删除文件的信息会打印到log4j2的INFO级别的log中。可使用此参数测试配置是否符合预测。默认为false。|
|pathConditions|	PathCondition[]|删除文件的过滤条件，满足指定条件的文件将会被删除，可以指定一个或多个。如果指定多个pathCondition，需要同时满足。<br>Conditions可以嵌套，当嵌套配置时，只有当满足了外部的contion时，才能对内部的condition进行判断。如果Conditions不是嵌套的，会可能以任意顺序进行判断。<br>Conditions也可以通过使用IfAll，IfAny，IfNot等类似于AND，OR，NOT的condition，实现复杂的condition。![DeleteAction](https://blogimages-1258086356.cos.ap-beijing.myqcloud.com/20181117202726.png)|

## 5.程序测试demo
``` java
public class HelloWorld {
 
    public static void main(String[] args) {
        Logger logger = LogManager.getLogger(LogManager.ROOT_LOGGER_NAME);
        try{
            //通过打印i，日志文件中数字越小代表越老
            for(int i = 0; i < 50000; i++) {
                logger.info("{}", i);
                logger.info("logger.info\n");
                Thread.sleep(100);//为了防止50000条很快跑完，sleep一段时间
            }
        } catch (InterruptedException e) {}
    }
}
```
### 5.1测试基于时间触发
filePattern最小时间粒度为秒，将每5秒触发一次rollover

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="trace" name="MyApp" packages="">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n" />
        </Console>
        <!--<RollingFile name="RollingFile" fileName="logs/app.log"-->
                     <!--filePattern="logs/app-%d{yyyy-MM-dd HH}-%i.log">-->
        <RollingFile name="RollingFile" fileName="logs/app.log"
                     filePattern="logs/app-%d{yyyy-MM-dd HH-mm-ss}.log">
            <PatternLayout>
                <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
            </PatternLayout>
            <Policies>
                <!--当经过了interval时间后，将根据filePattern对文件进行重命名，并生成新的文件用于日志写入-->
                <TimeBasedTriggeringPolicy interval="5"/>
                <!--当日志文件大小大于size时，将根据filepattern对文件进行重命名，并生成新的文件用于日志写入-->
                <!--<SizeBasedTriggeringPolicy size="30KB"/>-->
            </Policies>
            <DefaultRolloverStrategy max="3"/>
        </RollingFile>
    </Appenders>
    <Loggers>
        <Root level="trace">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="RollingFile"/>
        </Root>
    </Loggers>
</Configuration>
```
### 5.2测试基于文件大小的触发
日志文件达到5KB，将触发rollover

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="trace" name="MyApp" packages="">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n" />
        </Console>
        <!--<RollingFile name="RollingFile" fileName="logs/app.log"-->
                     <!--filePattern="logs/app-%d{yyyy-MM-dd HH}-%i.log">-->
        <RollingFile name="RollingFile" fileName="logs/app.log"
                     filePattern="logs/app-%d{yyyy-MM-dd HH-mm-ss}.log">
            <PatternLayout>
                <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
            </PatternLayout>
            <Policies>
                <!--当经过了interval时间后，将根据filePattern对文件进行重命名，并生成新的文件用于日志写入-->
                <!--<TimeBasedTriggeringPolicy interval="5"/>-->
                <!--当日志文件大小大于size时，将根据filepattern对文件进行重命名，并生成新的文件用于日志写入-->
                <SizeBasedTriggeringPolicy size="5KB"/>
            </Policies>
            <DefaultRolloverStrategy max="3"/>
        </RollingFile>
    </Appenders>
    <Loggers>
        <Root level="trace">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="RollingFile"/>
        </Root>
    </Loggers>
</Configuration>
```
### 5.3测试DefaultRolloverStrategy的max参数和%i计数器的搭配使用
注意filePattern最小时间粒度为分钟，且含%i计数器

``` xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="trace" name="MyApp" packages="">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} [%t] %-5level %logger{36} - %msg%n" />
        </Console>
        <RollingFile name="RollingFile" fileName="logs/app.log"
                     filePattern="logs/app-%d{yyyy-MM-dd HH-mm}-%i.log">
        <!--<RollingFile name="RollingFile" fileName="logs/app.log"-->
                     <!--filePattern="logs/app-%d{yyyy-MM-dd HH-mm-ss}.log">-->
            <PatternLayout>
                <Pattern>%d %p %c{1.} [%t] %m%n</Pattern>
            </PatternLayout>
            <Policies>
                <!--当经过了interval时间后，将根据filePattern对文件进行重命名，并生成新的文件用于日志写入-->
                <!--<TimeBasedTriggeringPolicy interval="5"/>-->
                <!--当日志文件大小大于size时，将根据filepattern对文件进行重命名，并生成新的文件用于日志写入-->
                <SizeBasedTriggeringPolicy size="5KB"/>
            </Policies>
            <DefaultRolloverStrategy max="3"/>
        </RollingFile>
    </Appenders>
    <Loggers>
        <Root level="trace">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="RollingFile"/>
        </Root>
    </Loggers>
</Configuration>
```
## 6.参考资料
http://logging.apache.org/log4j/2.x/manual/appenders.html  RollingFileAppender部分