# JVM

## 1：JVM基础知识

1. 什么是JVM
    
    JVM是一种规范

        -java virtual machine specifications
        -https://docs.oracle.com/en/java/javase/13
        -https://docs.oracle.com/javase/specs/index.html
        文档下载
    
    虚构出来的一台计算机
    
        -字节码指令集（汇编指令）
        -内存管理：堆、栈、方法区等

    JDK = JRE+ development kit
    
    JRE = JDK + core lib
 
![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/008.png)
![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/009.png)  

2. 常见的JVM
    
        Zing性能很高，费用也很贵
        Oracel吸收了Zing的垃圾回收算法推出了ZGC

        视频：Java要收钱，我该怎么办？
            不对开发人员收费
            oracle的hotspot收费
            OpenJDK开源版本不收费
            当然还有TaobaoJVM也不收费
        
    
![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/010.png)  

## 2：ClassFileFormat

- 二进制字节流
- 数据类型：u1 u2 u4 u8和_info（表类型）
- - _info的来源是hotspot源码中的写法
- 查看16进制格式的ClassFile
- - subme/notepad/
- - ==IDEA插件-BinEd==
- 有很多可以观察ByteCode的方法：
- - javap
    javap -v class文件路径
- - JBE-可以直接修改
- - ==JClassLib-IDEA插件之一==
    光标在类里面-->view-->show Bytecode with jclasslib
- classfile构成
>     classFile{
>         u4 magic;
>         u2 minor_version;
>         u2 major_version;
>         u2 constant_pool_count;
>         cp_info constant_pool[constant_pool_count-1];
>         access_flags
>         this_class
>         super_class
>         interfaces_count
>         interfaces
>         fields_count
>         fiels
>         methods_count
>         methods
>         attributes_count -u2
>     }


![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/011.png) 

![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/%E9%A9%AC%E5%A3%AB%E5%85%B5%E6%95%99%E8%82%B2%20java1.8%E7%B1%BB%E6%96%87%E4%BB%B6%E6%A0%BC%E5%BC%8F%E7%AC%AC%E4%B8%80%E7%89%88.png) 

![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/013.png)  


- General information

        constant_pool_count:下标从1开始，0做了预留，代表没有任何引用
            00 10 代表有16-1=15个常量
            
        access_flags 0x0021 是0x0001和0x0020按位与的结果
            代表ACC_PUBLIC 和 ACC_SUPER
        
        This_class:  cp_info_#2 代表存放在常量池的第二个位置
        super_class: cp_info_#3 代表常量池第三个位置

- Constant Pool


## 3：类编译-加载-初始化

    hashcode
    锁的信息（2位 四种组合）
    GC信息（年龄）
    如果是数组，数组的长度

## 4：JMM

    new Cat()
    pointer -> Cat.class
    寻找方法的信息

## 5：对象

    1：句柄池 （指针池）间接指针，节省内存
    2：直接指针，访问速度快

## 6：GC基础知识

    栈上分配
    TLAB（Thread Local Allocation Buffer）
    Old
    Eden
    老不死 - > Old

## 7：GC常用垃圾回收器

    new Object()
    markword          8个字节
    类型指针           8个字节
    实例变量           0
    补齐                  0		
    16字节（压缩 非压缩）
    Object o
    8个字节 
    JVM参数指定压缩或非压缩

