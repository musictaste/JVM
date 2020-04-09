# 使用JavaAgent测试Object的大小
![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/401.png)

## 面试题：1.请解释一下对象的创建过程？
    
    1.class loading
    2.class linking(verification、preparation、resolution)
    3.class initializing
        静态变量赋初始值，然后执行静态语句块
    4.申请对象内存
    5.成员变量赋默认值
    6.调用构造方法<init>
        1.成员变量顺序赋初始值
        2.执行构造方法语句

---
    
## 面试题：2.对象在内存中的存储布局？（64位机）
![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/411.png)  

### 观察虚拟机配置

    java -XX:+PrintCommandLineFlags -version
    -XX:+UserCompressedClassPointers  classpointer类型指针的压缩配置
    -XX:+UserCompressedOops  普通对象指针的压缩配置
    
    设置取消指针压缩： -XX:-UserCompressedClassPointers
## 对象大小（64位机）    
### 普通对象

    1. 对象头：markword  8个字节
    2. ClassPointer指针：-XX:+UseCompressedClassPointers 为4字节 不开启为8字节
    3. 实例数据
       1. 引用类型：-XX:+UseCompressedOops 为4字节 不开启为8字节 
          Oop Ordinary Object Pointers
    4. Padding对齐，8的倍数

### 数组对象

    1. 对象头：markword 8
    2. ClassPointer指针同上
    3. 数组长度：4字节
    4. 数组数据
    5. 对齐 8的倍数

下面有程序题

---
        
##### 面试题3.对象头具体包括什么？
    
    hotspot源码，定义在markOop.hpp文件
    
    hashcode部分：
        31位hashcode->System.identityHashCode()
        按原始内容计算的hashcode，重写过的hashcode方法计算的结果不会存在这里
        
        什么时候会产生hashcode？调用未重写的hashcode方法以及System。identityHashCode的时候
        
        如果对象没有重写hashcode方法，那么默认是调用os::random产生hashcode，可以通过System.identityHashCode获取
        
        os::random产生的hashcode的规则为：next_rand=(16807 seed)mod(2*31-1),因此可以使用31位存储，另外一旦生成了hashcode，JVM会将其记录在markword中
        
==当一个对象计算过identityHashCode之后，不能进入偏向锁状态==
        
    
![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/410.png)  
![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/407.png)  
![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/408.png)  
    
    
##### 面试题：4.对象怎么定位？

    •https://blog.csdn.net/clover_lily/article/details/80095580
    1. 句柄池
    2. 直接指针
    
    
    1. 句柄池  间接指针  GC的时候效率高  三色标记的时候效率高
    2. 直接指针 hotspot
    无优劣之分

##### 面试题；5.对象怎么分配？

    GC相关知识

##### 面试题：6.Object o = new Object在内存中占用多少字节？
    
    对象头markword：8个字节
    ClassPointer指针：默认开启压缩，4个字节
    实例数据：为空
    padding：4个字节
    一共16个字节

---

## Hotspot开启内存压缩的规则（64位机）

1. 4G以下，直接砍掉高32位
2. 4G - 32G，默认开启内存压缩 ClassPointers Oops
3. 32G，压缩无效，使用64位
   内存并不是越大越好（^-^）

## IdentityHashCode的问题
![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/021.png)
![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/410.png) 
回答白马非马的问题：

当一个对象计算过identityHashCode之后，不能进入偏向锁状态

https://cloud.tencent.com/developer/article/1480590
 https://cloud.tencent.com/developer/article/1484167
https://cloud.tencent.com/developer/article/1485795
https://cloud.tencent.com/developer/article/1482500


---

## 实验：使用JavaAgent测试Object的大小

1. 新建项目ObjectSize （1.8）
    
    agent代理机制，class文件loading到JVM中，需要先经过agent代理，这个agent可以知道对象的大小，这个agent需要自己实现    
    
2. 创建文件ObjectSizeAgent

   ```java
   package com.mashibing.jvm.agent;
   
   import java.lang.instrument.Instrumentation;
   
   public class ObjectSizeAgent {
       private static Instrumentation inst;//调弦
   
       public static void premain(String agentArgs, Instrumentation _inst) { //premain固定方法 跟main方法一样，格式固定  _inst是虚拟机提供的
           inst = _inst;
       }
   
       public static long sizeOf(Object o) {
           return inst.getObjectSize(o);
       }
   }
   ```

3. src目录下创建META-INF/MANIFEST.MF

   ```java
   Manifest-Version: 1.0
   Created-By: mashibing.com
   Premain-Class: com.mashibing.jvm.agent.ObjectSizeAgent
   ```

   注意Premain-Class这行必须是新的一行（回车 + 换行），确认idea不能有任何错误提示

4. 打包jar文件

5. 在需要使用该Agent Jar的项目中引入该Jar包
   project structure - project settings - library 添加该jar包

6. 运行时需要该Agent Jar的类，加入参数：

   ```java
   -javaagent:C:\work\ijprojects\ObjectSize\out\artifacts\ObjectSize_jar\ObjectSize.jar
   ```

7. 如何使用该类：

   ```java
   ​```java
      package com.mashibing.jvm.c3_jmm;
      
      import com.mashibing.jvm.agent.ObjectSizeAgent;
      
      public class T03_SizeOfAnObject {
          public static void main(String[] args) {
              System.out.println(ObjectSizeAgent.sizeOf(new Object()));
              System.out.println(ObjectSizeAgent.sizeOf(new int[] {}));
              System.out.println(ObjectSizeAgent.sizeOf(new P()));
          }
      
          private static class P {
                              //8 _markword
                              //4 _oop指针
              int id;         //4
              String name;    //4
              int age;        //4
      
              byte b1;        //1
              byte b2;        //1
      
              Object o;       //4
              byte b3;        //1
      
          }
      }
    运行结果：  
      16
      16
      32
      
      分析：
        Object：
            对象头markword：8个字节
            ClassPointer指针：默认开启压缩，4个字节
            实例数据：为空
            padding：4个字节
            一共16个字节
        
        数组：
            对象头markword：8个字节
            ClassPointer指针：默认开启压缩，4个字节
            数组长度：4个字节
            实例数据：0
            padding：0
            一共16个字节
        
        对象：
            对象头：8个字节
            classPointer指针：4个字节
            实例对象：
                int  4个字节
                String  4个字节，本身是8个字节，因为开启了-XX:+UseCompressedOops为4个字节（Oops=ordinary object pointers）
                int  4个字节
                byte 1个字节
                byte 1个字节
                
                Object 4个字节
                byte 1个字节
            补齐：32
                
    取消压缩的运行结果：
     16
     24
     40
   ```



