# Runtime Data Area and Instruction Set

jvms 2.4 2.5

## 指令集分类

    1. 基于栈的指令集--JVM，相对简单
    2. 基于寄存器的指令集--汇编，相对复杂，运行速度快
        Hotspot的Local Variable Table局部变量表 = JVM中的寄存器

## Runtime Data Area
![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/501.png)
![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/502.png) 

##### PC 程序计数器(program counter)

    线程私有，存放指令位置
    
    虚拟机的运行，类似于这样的循环：
    while( not end ) {
        取PC中的位置，找到对应位置的指令；
        执行该指令；
        PC ++;
    }

##### Runtime Constant Pool

##### Direct Memory

    JVM可以直接访问的内核空间的内存 (OS 管理的内存)
    NIO（JDK1.4以后增加） ， 提高效率，实现zero copy

##### Heap
    
    线程共享

##### Native Method Stack（NMS）

    线程私有，JNI  
    
##### JVM Stack（VMS）

    线程私有
    1. Frame - 每个方法对应一个栈帧（栈帧在主存main memory中）
       1. Local Variable Table  局部变量表
            通过jclasslib 在methods中可以看到LocalVariableTable
       2. Operand Stack 操作数栈
          对于long的处理（store and load），多数虚拟机的实现都是原子的
          jls 17.7，没必要加volatile
       3. Dynamic Linking 动态连接
           https://blog.csdn.net/qq_41813060/article/details/88379473 
          jvms 2.6.3
          a() -> b()，方法a调用了方法b，b的内容存放在constant_pool常量池中，这个指向常量池的那个符号连接就是动态连接
          
       4. return address 返回地址
          a() -> b()，方法a调用了方法b, b方法的返回值放在什么地方
    
```
public class TestIPulsPlus {
    public static void main(String[] args) {
        int i = 8;
        //i = i++;
        i = ++i;
        System.out.println(i);
    }
}
```


    解析：int i=8; i=i++;
    查看二进制码指令
         0 bipush 8  8压栈
         2 istore_1  出栈，并赋值给局部变量表i，i=8
         3 iload_1  i=8压栈
         4 iinc 1 by 1  局部变量表中的i加1，i=9（i++是在局部变量表中完成的）
         7 istore_1  栈中i(i=8)出栈,并赋值给局部变量表中的i，这时i=8 
         8 getstatic #2 <java/lang/System.out>
        11 iload_1  i压栈(i=8)
        12 invokevirtual #3 <java/io/PrintStream.println> 调用println方法(非静态方法)
        15 return
    

    解析：int i=8; i=++i;
    查看二进制码指令
         0 bipush 8  8压栈
         2 istore_1  出栈，并赋值给局部变量表i（i=8）
         3 iinc 1 by 1  局部变量表中的i加1，i=9（i++是在局部变量表中完成的）
         6 iload_1   i=9压栈
         7 istore_1   i=9出栈
         8 getstatic #2 <java/lang/System.out>
        11 iload_1   i=9压栈
        12 invokevirtual #3 <java/io/PrintStream.println>
        15 return


##### Method Area

    线程共享
    
    方法区是逻辑上的概念
    永久区和元数据是具体的实现，是不同版本方法区的实现
    
    1. Perm Space (<1.8)
       字符串常量位于PermSpace
       不受堆内存管理
       FGC不会清理
       大小启动的时候必须指定，不能变
    2. Meta Space (>=1.8)
       字符串常量位于堆
       会触发FGC清理
       不设定的话，最大就是物理内存

#### 查看代码中的指令

```
public class Hello_01 {
    public static void main(String[] args) {
        int i = 100;
    }

    public void m1() {
        int i = 200;
    }

    public void m2(int k) {
        int i = 300;
    }

    public void add(int a, int b) {
        int c = a + b;
    }

    public void m3() {
        Object o = null;
    }

    public void m4() {
        Object o = new Object();
    }
}

二进制的指令；
main:
    0 bipush 100  100压栈(100小于127，采用byte)
    2 istore_1  出栈，赋值给i
    3 return

m1    
    0 sipush 200  200压栈（200大于127，采用short）
    3 istore_1
    4 return

m2
    0 sipush 300
    3 istore_2  i在局部变量表的第三个位置,第一个位置为this
    4 return
    
    ==非静态的方法，第一个参数是this，接下来才是我们的局部变量
    
add
    0 iload_1 a压栈
    1 iload_2 b压栈
    2 iadd  相加
    3 istore_3  出栈，赋值给c
    4 return
m3    
    0 aconst_null  常量null压栈
    1 astore_1 出栈，赋值给o
    2 return
 
m4
    0 new #2 <java/lang/Object>   新建一个对象
    3 dup  对象指针复制
    4 invokespecial #1 <java/lang/Object.<init>>  执行Object的构造方法
    7 astore_1  出栈，对象指针赋值给o
    8 return

    new  new对象
    dup  new对象后，对象地址会压栈；dup复制该对象地址
    invokespecial <init>  现在有两个对象地址，执行后会弹出dup的对象地址，剩下一个对象地址 执行完构造方法，会把上面dup的对象地址弹出去，这时候对象才算初始化完成
        这时候栈里剩下的对象地址，指向了初始化后的对象
    astore_1  把初始化的对象地址指向h对象
    
    ==这块可以理解指令重排的问题，双重检查装载单例模式，volatile的原因

```



```
public class Hello_02 {
    public static void main(String[] args) {
        Hello_02 h = new Hello_02();
        h.m1();
    }

    public void m1() {
        int i = 200;
    }

}

二进制指令分析
main
     0 new #2 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/Hello_02>  新建对象，成员变量赋默认值
     3 dup
     4 invokespecial #3 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/Hello_02.<init>>  调用构造方法，成员变量赋初始值
     7 astore_1  出栈，对象指针赋给h
     8 aload_1  h压栈
     9 invokevirtual #4 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/Hello_02.m1> 调用m1方法
    12 return
    
m1
    0 sipush 200
    3 istore_1
    4 return

```

##### 有返回值
```
public class Hello_03 {
    public static void main(String[] args) {
        Hello_03 h = new Hello_03();
//        int i = h.m1();
        h.m1();
    }

    public int m1() {
        return 100;
    }
}

二进制指令分析
main: h.m1()
     0 new #2 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/Hello_03>
     3 dup
     4 invokespecial #3 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/Hello_03.<init>>
     7 astore_1
     8 aload_1
     9 invokevirtual #4 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/Hello_03.m1>
    12 pop  pop:放在栈顶的返回值100，弹出
    13 return


main: int i = h.m1()
     0 new #2 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/Hello_03>
     3 dup
     4 invokespecial #3 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/Hello_03.<init>>
     7 astore_1
     8 aload_1
     9 invokevirtual #4 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/Hello_03.m1>
    12 istore_2   istore_2:放在栈顶的返回值100，弹给变量i
    13 return

```
    
##### 递归
```
public class Hello_04 {
    public static void main(String[] args) {
        Hello_04 h = new Hello_04();
        int i = h.m(3);
    }

    public int m(int n) {
        if(n == 1) return 1;
        return n * m(n-1);
    }
}
二进制指令分析
main
     0 new #2 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/Hello_04>
     3 dup
     4 invokespecial #3 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/Hello_04.<init>>
     7 astore_1
     8 aload_1
     9 iconst_3 常量3压栈
    10 invokevirtual #4  调用m方法 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/Hello_04.m>
    13 istore_2  赋值给i
    14 return
    
m
     0 iload_1 n压栈
     1 iconst_1 常量1压栈
     2 if_icmpne 7 (+5) if判断 compare not equal 不相等跳转到第7条指令，该命令占的字节数比较多，所以把3/4的位置占了
     5 iconst_1  相等，常量1压栈
     6 ireturn 返回结果
     7 iload_1 n压栈
     8 aload_0 this压栈
     9 iload_1 n压栈
    10 iconst_1 常量1压栈
    11 isub n-1
    12 invokevirtual #4 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/Hello_04.m> 执行m(n-1)
    15 imul n*m(n-1)
    16 ireturn 返回结果

```

##### invokeXXX


```
public class T01_InvokeStatic {
    public static void main(String[] args) {
        m();
    }

    public static void m() {}
}

指令分析
main
    0 invokestatic #2 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/T01_InvokeStatic.m>  调静态方法
    3 return

m
    0 return

```


```
public class T02_InvokeVirtual {
    public static void main(String[] args) {
        new T02_InvokeVirtual().m();
    }

    public void m() {}
}
指令分析：
main
     0 new #2 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/T02_InvokeVirtual>
     3 dup
     4 invokespecial #3 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/T02_InvokeVirtual.<init>>   构造方法
     7 invokevirtual #4 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/T02_InvokeVirtual.m>  普通非静态方法
    10 return

```


```
public class T03_InvokeSpecial {
    public static void main(String[] args) {
        T03_InvokeSpecial t = new T03_InvokeSpecial();
        t.m();
        t.n();
    }

    public final void m() {}
    private void n() {}
}
指令分析：
main
     0 new #2 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/T03_InvokeSpecial>
     3 dup
     4 invokespecial #3  构造方法 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/T03_InvokeSpecial.<init>>
     7 astore_1 对象指针赋给t
     8 aload_1
     9 invokevirtual #4 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/T03_InvokeSpecial.m> 调普通方法（final也是）
    12 aload_1
    13 invokespecial #5 <com/mashibing/jvm/c4_RuntimeDataAreaAndInstructionSet/T03_InvokeSpecial.n>  调private方法
    16 return
```


```
public class T04_InvokeInterface {
    public static void main(String[] args) {
        List<String> list = new ArrayList<String>();
        list.add("hello");

        ArrayList<String> list2 = new ArrayList<>();
        list2.add("hello2");
    }
}

指令分析：
 0 new #2 <java/util/ArrayList>
 3 dup
 4 invokespecial #3 <java/util/ArrayList.<init>>
 7 astore_1
 8 aload_1
 9 ldc #4 <hello>
11 invokeinterface #5 <java/util/List.add> count 2   list的add方法为接口
16 pop
17 new #2 <java/util/ArrayList>
20 dup
21 invokespecial #3 <java/util/ArrayList.<init>>
24 astore_2
25 aload_2
26 ldc #6 <hello2>
28 invokevirtual #7 <java/util/ArrayList.add>  ArrayList的add方法为普通非静态方法
31 pop
32 return
```


```
public class T05_InvokeDynamic {
    public static void main(String[] args) {
        I i = C::n;
        I i2 = C::n;
        I i3 = C::n;
        I i4 = () -> {
            C.n();
        };
        System.out.println(i.getClass());
        System.out.println(i2.getClass());
        System.out.println(i3.getClass());
        //for(;;) {I j = C::n;} //MethodArea <1.8 Perm Space (FGC不回收)
    }

    @FunctionalInterface
    public interface I {
        void m();
    }

    public static class C {
        static void n() {
            System.out.println("hello");
        }
    }
}

运行结果：
class com.mashibing.jvm.c4_RuntimeDataAreaAndInstructionSet.T05_InvokeDynamic$$Lambda$1/1096979270
class com.mashibing.jvm.c4_RuntimeDataAreaAndInstructionSet.T05_InvokeDynamic$$Lambda$2/1747585824
class com.mashibing.jvm.c4_RuntimeDataAreaAndInstructionSet.T05_InvokeDynamic$$Lambda$3/1023892928

指令分析：
main
     0 invokedynamic #2 <m, BootstrapMethods #0>
     5 astore_1
     6 invokedynamic #2 <m, BootstrapMethods #0>
    11 astore_2
    12 invokedynamic #2 <m, BootstrapMethods #0>
    17 astore_3
    18 invokedynamic #3 <m, BootstrapMethods #1>
    23 astore 4
    25 getstatic #4 <java/lang/System.out>
    28 aload_1
    29 invokevirtual #5 <java/lang/Object.getClass>
    32 invokevirtual #6 <java/io/PrintStream.println>
    35 getstatic #4 <java/lang/System.out>
    38 aload_2
    39 invokevirtual #5 <java/lang/Object.getClass>
    42 invokevirtual #6 <java/io/PrintStream.println>
    45 getstatic #4 <java/lang/System.out>
    48 aload_3
    49 invokevirtual #5 <java/lang/Object.getClass>
    52 invokevirtual #6 <java/io/PrintStream.println>
    55 return
```


##### 思考：

    如何证明1.7字符串常量位于Perm，而1.8位于Heap？
    提示：结合GC， 一直创建字符串常量，观察堆，和Metaspace


## 常用指令

    <clinit> 类的静态语句块
    <init> 构造方法
    store
    load
    pop
    mul
    sub
    invoke
    
    1. InvokeStatic  调静态方法
    2. InvokeVirtual  调普通非静态方法，该命令自带多态
        final方法  普通非静态方法
    3. InvokeInterface
    4. InovkeSpecial
       可以直接定位，不需要多态的方法
       private方法 ， 构造方法
    5. InvokeDynamic
       JVM最难的指令
       lambda表达式或者反射或者其他动态语言（scala kotlin，或者CGLib 或ASM），动态产生的class，会用到的指令

        会产生一个lambda表达式的匿名内部类

==面试题：==
        
    for(;;){
        I i=C::n;
    }
    这段代码会产生什么问题？
    
    分析：死循环，因为lambda表达会产生匿名内部类，匿名内部类会放在method Area中，
                JDK1.8之前，是Perm space,  FGC不会清理，并造成OOM
                JDK1.8之后，是MetaSpace，FGC会清理，触发FGC，不会造成OOM；【但是如果FGC以后，内存空间还不够，会造成OOM】
    
    答案：  
        JDK1.8之前，是Perm space,  FGC不会清理，并造成OOM
        JDK1.8之后，是MetaSpace，FGC会清理，触发FGC，不会造成OOM；【但是如果FGC以后，内存空间还不够，会造成OOM】
                
                
    java Visual VM查看JVM内存空间的工具