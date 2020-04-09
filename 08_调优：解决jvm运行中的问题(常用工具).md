
### 解决JVM运行中的问题

#### 一个案例理解常用工具

1. 测试代码：

   ```java
   package com.mashibing.jvm.gc;
   
   import java.math.BigDecimal;
   import java.util.ArrayList;
   import java.util.Date;
   import java.util.List;
   import java.util.concurrent.ScheduledThreadPoolExecutor;
   import java.util.concurrent.ThreadPoolExecutor;
   import java.util.concurrent.TimeUnit;
   
   /**
    * 从数据库中读取信用数据，套用模型，并把结果进行记录和传输
    */
   
   public class T15_FullGC_Problem01 {
   
       private static class CardInfo {
           BigDecimal price = new BigDecimal(0.0);
           String name = "张三";
           int age = 5;
           Date birthdate = new Date();
   
           public void m() {}
       }
   
       private static ScheduledThreadPoolExecutor executor = new ScheduledThreadPoolExecutor(50,
               new ThreadPoolExecutor.DiscardOldestPolicy());
   
       public static void main(String[] args) throws Exception {
           executor.setMaximumPoolSize(50);
   
           for (;;){
               modelFit();
               Thread.sleep(100);
           }
       }
   
       private static void modelFit(){
           List<CardInfo> taskList = getAllCardInfo();
           taskList.forEach(info -> {
               // do something
               executor.scheduleWithFixedDelay(() -> {
                   //do sth with info
                   info.m();
   
               }, 2, 3, TimeUnit.SECONDS);
           });
       }
   
       private static List<CardInfo> getAllCardInfo(){
           List<CardInfo> taskList = new ArrayList<>();
   
           for (int i = 0; i < 100; i++) {
               CardInfo ci = new CardInfo();
               taskList.add(ci);
           }
   
           return taskList;
       }
   }
   
   ```
   
    
    2.java -Xms200M -Xmx200M -XX:+PrintGC com.mashibing.jvm.gc.T15_FullGC_Problem01
    命令加上开启JMX协议的配置
    
    3. 一般是运维团队首先受到报警信息（CPU、Memory飙高）
        Ansible软件
    
    4. top命令观察到问题：内存不断增长 CPU占用率居高不下(linux命令)
    
    5. top -Hp 观察进程中的线程，哪个线程CPU和内存占比高(linux命令)
        
        视频1:20:00
    6. jps定位具体java进程（windows也支持）
    
        jstack定位死锁问题，比较方便（windows也支持）
        jstack -l 进程号
    
        ==在使用jstack的时候，显示的是线程号的十六进制
    
       jstack 定位线程状况，重点关注：WAITING BLOCKED
       eg.
       可以定位到代码哪一行
       
       图片803
       
       waiting on <0x0000000088ca3310> (a java.lang.Object)
       
       
       假如有一个进程中100个线程，很多线程都在waiting on <xx> ，一定要找到是哪个线程持有这把锁
       
       怎么找？搜索jstack dump的信息，找<xx> ，看哪个线程持有这把锁RUNNABLE
       
       ==作业：1：写一个死锁程序，用jstack观察 
       ==2 ：写一个程序，一个线程持有锁不释放，其他线程等待
            有10个线程，其中一个线程死循环，
    
    7. 为什么阿里规范里规定，线程的名称（尤其是线程池）都要写有意义的名称
        方便出错时回溯 
    
       怎么样自定义线程池里的线程名称？（自定义ThreadFactory）
            以前讲过
    
    8. jinfo pid 
        进程的虚拟机的详细信息
    
    9. jstat -gc 动态观察gc情况 / 阅读GC日志发现频繁GC / arthas观察 /
        jconsole（界面不好看）/jvisualVM（压测监控）/ Jprofiler（最好用，收费）
       jstat -gc 4655 500 :  4655是线程id 每个500个毫秒打印GC的情况
    
    图片804

##### JPS+jvisualVM  windows实验

    1.启动程序
    2.cmd窗口，输入jps
        jstack -l 线程id
        
    3.jstat -gc 线程id    ==该命令看不出什么内容
    
    4.jmap -histo 线程id
    
    5.jmap -dump:format=b,file=d:/test4.dump 线程id
    
    6.jhat -J-mx512M d:/test4.dump

    7.通过http://127.0.0.1:7000访问
    
    8.通过jvisualVM 装入test4.dump,进行分析

    
##### ==OOM定位==
    
    如果面试官问你是怎么定位OOM问题的？如果你回答用图形界面（错误）
    1：已经上线的系统不用图形界面用什么？（cmdline arthas）
    2：图形界面到底用在什么地方？测试！测试的时候进行监控！（压测观察）

    JMX（java management ）: java标准的访问远程服务的协议
    远程服务器把JMX打开
    再利用JMX的客户端连接就可以
    
    JMX服务对服务器性能的影响比较严重
    
    
    10. jmap -histo 4655(线程id) | head -20，查找有多少对象产生
        
        histo？
        jmap - histo对于系统有影响，但是没有那么大，线上可以执行
        
        图片805
        
    11. jmap -dump:format=b,file=xxx pid
        eg:jmap -dump:format=b,file=heap.dump 6428
    
        线上系统，内存特别大，jmap执行期间会对进程产生很大影响，甚至卡顿（电商不适合）
            32G的内存，Jmap时间特别长
            
        怎么解决：
            1：设定了参数HeapDump，OOM的时候会自动产生堆转储文件
            2：=**=很多服务器备份（高可用），停掉这台服务器对其他服务器不影响(推荐说法)，
            
                因为jmap对于在线的机器，影响特别大，
                锁我对这台机器做了隔离，然后对这台机器进行jmap
                
            3：在线定位(一般小点儿公司用不到)--arthas（如果说了，背调会很严）
    
    12. java -Xms20M -Xmx20M -XX:+UseParallelGC -XX:+HeapDumpOnOutOfMemoryError com.mashibing.jvm.gc.T15_FullGC_Problem01
        堆内存中哪个对象特别多，那么相关的代码出问题
        
        如果已经OOM了，设定了参数-XX:+HeapDumpOnOutOfMemoryError，会自动产生堆转储文件
            记住先别重启
    
    13. 使用MAT / jhat /jvisualvm 进行dump文件分析
         https://www.cnblogs.com/baihuitestsoftware/articles/6406271.html 
        
        jhat -J-mx512M xxx.dump
        http://192.168.17.11:7000
        拉到最后：找到对应链接
        可以使用OQL查找特定问题对象

        jvisualvm：文件-装入，dump的文件
            也可以使用OQL语句
            也可以查看多少个最大的对象（jmap - histo 4655(线程id) | head -20）
        
    14. 找到代码的问题（最难的部分，从庞大的项目代码中找到问题）

**==小结==**：

    问题定位（面试重灾区，特别有很多坑，小心回答）
    
    CPU或内存飙高(运维团队) -> top -> JPS(top -Hp) -> jstack -jmap
        
    死锁问题（死循环），jstack就可以定位
    如果飙高的是垃圾回收线程，观察GC日志，发现在频繁GC
    
    如果OOM，jmap -histo | head 20
        图形化界面jvisualVM jconsole一般用于系统上线前的压测
        
        执行jmap dump命令的时候因为对线上系统产生很大影响，甚至卡顿
        所以我先对这台机器进行隔离，隔离以后进行jmap导出，然后观察

        可以在线 histo，不能在线jmap dump

#### jconsole远程连接

1. 程序启动加入参数：

   > ```shell
   > java -Djava.rmi.server.hostname=192.168.17.11 -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.port=11111 -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false XXX
   > ```

    authenticate=false关闭认证

2. 如果遭遇 Local host name unknown：XXX的错误，修改/etc/hosts文件，把XXX加入进去

   > ```java
   > 192.168.17.11 basic localhost localhost.localdomain localhost4 localhost4.localdomain4
   > ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
   > ```

3. 关闭linux防火墙（实战中应该打开对应端口）

   > ```shell
   > service iptables stop
   > chkconfig iptables off #永久关闭
   > ```

4. windows上打开 jconsole远程连接 192.168.17.11:11111

#### jvisualvm远程连接(了解就行)

 https://www.cnblogs.com/liugh/p/7620336.html （简单做法）

#### jprofiler (收费)

#### arthas在线排查工具

    ==官方文档有安装、启动命令
    java -jar arthas-boot.jar

    arthas是挂载到某个线程上，如果某个线程已经OOM了，看不到

    * 为什么需要在线排查？
       在生产上我们经常会碰到一些不好排查的问题，例如线程安全问题，用最简单的threaddump或者heapdump不好查到问题原因。为了排查这些问题，有时我们会临时加一些日志，比如在一些关键的函数里打印出入参，然后重新打包发布，如果打了日志还是没找到问题，继续加日志，重新打包发布。对于上线流程复杂而且审核比较严的公司，从改代码到上线需要层层的流转，会大大影响问题排查的进度。 
    * jvm：观察jvm信息
    * thread 线程ID：定位线程问题（类似jstack -l 线程id）
    * dashboard 观察系统情况（top -Hp 线程id）
    * heapdump + jhat分析（jmap -dump:format=b,file=xxx pid ）
        heapdump /root/20200328.hprof （可以指定文件地址）
        
        
        jhat -J-mx512M xxx.dump
        http://192.168.17.11:7000
        拉到最后：找到对应链接
        可以使用OQL查找特定问题对象
        
        jhat -J-mx512M 20200328.hprof(jhat分析,如果报错提示jhat内存不足，可以指定内存大小)
        如果文件为2G，指定了512M，arthas会慢慢导，
        
        dump file导入文件、resolving分析内容、chasing references追踪引用
        
        jhat结束以后，通过给定的端口（linuxIP+端口）访问分析的结果
        
        最下面有一个：Execute Object Query Language(OQL) query 非常强大
            命令：select s from java.lang.String s
            查看String相关的对象
            
            
        
    * jad反编译
        jad 类（全路径+类名）
        
       动态代理生成类的问题定位
       第三方的类（观察代码）
       版本问题（确定自己最新提交的版本是不是被使用）
       
    * redefine 热替换
        不能随便停服务的情况下，非常有用
        
       目前有些限制条件：只能改方法实现（方法已经运行完成），不能改方法名， 不能改属性
       m() -> mm():不可行
       
       redefine 类.class
       
    * sc  - search class
    * watch  - watch method
    * 没有包含的功能：jmap - histo

##### arthas windows实验

    1.windows也可以安装
    curl -O https://alibaba.github.io/arthas/arthas-boot.jar
    
    2.通过cmd窗口，输入命令启动
    java -jar arthas-boot.jar
    
    3.需要要加载的进程
    [1]: 6428 com.mashibing.jvm.c5_gc.T_GC_waiting
    [2]: 14468 org/netbeans/Main
    [3]: 3892 org.jetbrains.jps.cmdline.Launcher
    [4]: 6036
    [5]: 7864 arthas-boot.jar
    
    输入1
    
    4.thread 1
    
    5.heapdump d:/test.hprof
    
    6.再开一个cmd窗口
        jhat -J-mx512M d:/test.hprof
        
    7.访问报告：http://127.0.0.1:7000
        拉到最后：找到对应链接
        可以使用OQL查找特定问题对象


### 案例汇总

    OOM产生的原因多种多样，有些程序未必产生OOM，不断FGC(CPU飙高，但内存回收特别少) （上面案例）
    
    1. 硬件升级系统反而卡顿的问题（见上）
    
    2. 线程池不当运用产生OOM问题（见上）
       不断的往List里加对象（实在太LOW）
    
    3. smile jira问题
       实际系统不断重启
       解决问题 加内存 + 更换垃圾回收器 G1
       真正问题在哪儿？不知道
    
    4. tomcat http-header-size过大问题（Hector）
    
        ==不知道哪个哥们改了这个参数，有可能是网管（**）
        
        server.port=9898
        server.max-http-header-size=10000000
        
        原来为4096 bytes,4K
        max-http-header-size:每个请求过来占用的内存
        
        http相关的对象占用内存过大
        
        用jmeter模拟发现每个请求都创建headerBufferSize+8192K
        
        org.apache.coyote.http11.Http11OutputBuffer 这个对象占用内存过大
        
    
    5. lambda表达式导致方法区溢出问题(MethodArea / Perm Metaspace)--不建议说这个案例
    
        lambda表达式，会创建新的class，分配在MethodArea
        
        caused by: java.lang.OutOfMemoryError : compressed class space
        
        LambdaGC.java     -XX:MaxMetaspaceSize=9M -XX:+PrintGCDetails

   ```java
   "C:\Program Files\Java\jdk1.8.0_181\bin\java.exe" -XX:MaxMetaspaceSize=9M -XX:+PrintGCDetails "-javaagent:C:\Program Files\JetBrains\IntelliJ IDEA Community Edition 2019.1\lib\idea_rt.jar=49316:C:\Program Files\JetBrains\IntelliJ IDEA Community Edition 2019.1\bin" -Dfile.encoding=UTF-8 -classpath "C:\Program Files\Java\jdk1.8.0_181\jre\lib\charsets.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\deploy.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\access-bridge-64.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\cldrdata.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\dnsns.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\jaccess.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\jfxrt.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\localedata.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\nashorn.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\sunec.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\sunjce_provider.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\sunmscapi.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\sunpkcs11.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\ext\zipfs.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\javaws.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\jce.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\jfr.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\jfxswt.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\jsse.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\management-agent.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\plugin.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\resources.jar;C:\Program Files\Java\jdk1.8.0_181\jre\lib\rt.jar;C:\work\ijprojects\JVM\out\production\JVM;C:\work\ijprojects\ObjectSize\out\artifacts\ObjectSize_jar\ObjectSize.jar" com.mashibing.jvm.gc.LambdaGC
   [GC (Metadata GC Threshold) [PSYoungGen: 11341K->1880K(38400K)] 11341K->1888K(125952K), 0.0022190 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
   [Full GC (Metadata GC Threshold) [PSYoungGen: 1880K->0K(38400K)] [ParOldGen: 8K->1777K(35328K)] 1888K->1777K(73728K), [Metaspace: 8164K->8164K(1056768K)], 0.0100681 secs] [Times: user=0.02 sys=0.00, real=0.01 secs] 
   [GC (Last ditch collection) [PSYoungGen: 0K->0K(38400K)] 1777K->1777K(73728K), 0.0005698 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
   [Full GC (Last ditch collection) [PSYoungGen: 0K->0K(38400K)] [ParOldGen: 1777K->1629K(67584K)] 1777K->1629K(105984K), [Metaspace: 8164K->8156K(1056768K)], 0.0124299 secs] [Times: user=0.06 sys=0.00, real=0.01 secs] 
   java.lang.reflect.InvocationTargetException
   	at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
   	at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62)
   	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
   	at java.lang.reflect.Method.invoke(Method.java:498)
   	at sun.instrument.InstrumentationImpl.loadClassAndStartAgent(InstrumentationImpl.java:388)
   	at sun.instrument.InstrumentationImpl.loadClassAndCallAgentmain(InstrumentationImpl.java:411)
   Caused by: java.lang.OutOfMemoryError: Compressed class space
   	at sun.misc.Unsafe.defineClass(Native Method)
   	at sun.reflect.ClassDefiner.defineClass(ClassDefiner.java:63)
   	at sun.reflect.MethodAccessorGenerator$1.run(MethodAccessorGenerator.java:399)
   	at sun.reflect.MethodAccessorGenerator$1.run(MethodAccessorGenerator.java:394)
   	at java.security.AccessController.doPrivileged(Native Method)
   	at sun.reflect.MethodAccessorGenerator.generate(MethodAccessorGenerator.java:393)
   	at sun.reflect.MethodAccessorGenerator.generateSerializationConstructor(MethodAccessorGenerator.java:112)
   	at sun.reflect.ReflectionFactory.generateConstructor(ReflectionFactory.java:398)
   	at sun.reflect.ReflectionFactory.newConstructorForSerialization(ReflectionFactory.java:360)
   	at java.io.ObjectStreamClass.getSerializableConstructor(ObjectStreamClass.java:1574)
   	at java.io.ObjectStreamClass.access$1500(ObjectStreamClass.java:79)
   	at java.io.ObjectStreamClass$3.run(ObjectStreamClass.java:519)
   	at java.io.ObjectStreamClass$3.run(ObjectStreamClass.java:494)
   	at java.security.AccessController.doPrivileged(Native Method)
   	at java.io.ObjectStreamClass.<init>(ObjectStreamClass.java:494)
   	at java.io.ObjectStreamClass.lookup(ObjectStreamClass.java:391)
   	at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1134)
   	at java.io.ObjectOutputStream.defaultWriteFields(ObjectOutputStream.java:1548)
   	at java.io.ObjectOutputStream.writeSerialData(ObjectOutputStream.java:1509)
   	at java.io.ObjectOutputStream.writeOrdinaryObject(ObjectOutputStream.java:1432)
   	at java.io.ObjectOutputStream.writeObject0(ObjectOutputStream.java:1178)
   	at java.io.ObjectOutputStream.writeObject(ObjectOutputStream.java:348)
   	at javax.management.remote.rmi.RMIConnectorServer.encodeJRMPStub(RMIConnectorServer.java:727)
   	at javax.management.remote.rmi.RMIConnectorServer.encodeStub(RMIConnectorServer.java:719)
   	at javax.management.remote.rmi.RMIConnectorServer.encodeStubInAddress(RMIConnectorServer.java:690)
   	at javax.management.remote.rmi.RMIConnectorServer.start(RMIConnectorServer.java:439)
   	at sun.management.jmxremote.ConnectorBootstrap.startLocalConnectorServer(ConnectorBootstrap.java:550)
   	at sun.management.Agent.startLocalManagementAgent(Agent.java:137)
   
   ```

    6. 直接内存溢出问题（少见）---不建议说
       ==《深入理解Java虚拟机》P59，使用Unsafe分配直接内存，或者使用NIO的问题

    7. 栈溢出问题---不建议说
       -Xss设定太小
    
        一个方法调用别的方法，深度特别深
        代码：StackOverFlow

    8. 比较一下这两段程序的异同，分析哪一个是更优的写法：

   ```java 
   Object o = null;
   for(int i=0; i<100; i++) {
       o = new Object();
       //业务处理
   }
   ```

   ```java
   for(int i=0; i<100; i++) {
       Object o = new Object();
   }
   ```
         第一种方法：只有一个对象，第二次循环，第一次的new Object,会被垃圾回收期回收掉， 当然更优

    9. 重写finalize引发频繁GC
       小米云，HBase同步系统，系统通过nginx访问超时报警，最后排查，C++程序员重写finalize引发频繁GC问题
        为什么C++程序员会重写finalize？（new delete）
            需要手动回收内存，调用delete方法会默认调用析构函数；
            调用new方法，会默认调用构造方法
        
        finalize耗时比较长（200ms），创建了大量对象，对象回收时，每个对象都需要耗时200ms，造成内存飙高
       
    10. 如果有一个系统，内存一直消耗不超过10%，但是观察GC日志，发现FGC总是频繁产生，会是什么引起的？---不建议说
    
        有人显示调用了System.gc() (这个比较Low)
        设置参数：-XX:+DisableExplictGC，System.gc()不管用
    
    11. Distuptor有个可以设置链的长度，如果过大，然后对象大，消费完不主动释放，会溢出 (来自 死物风情)
    
    12. 用jvm都会溢出，mycat用崩过，1.6.5某个临时版本解析sql子查询算法有问题，9个exists的联合sql就导致生成几百万的对象（来自 死物风情）
    
    13. new 大量线程，会产生 native thread OOM，（low）应该用线程池，
        解决方案：减少堆空间（太TMlow了）,预留更多内存产生native thread
        JVM内存占物理内存比例 50% - 80%


