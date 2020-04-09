
#### 2. Linking 
##### 2.1. Verification校验

    验证文件是否符合JVM规定
        
##### 2.2. Preparation（面试常考）准备

    静态成员变量赋默认值
         
##### 2.3. Resolution解析

    将类、方法、属性等符号引用解析为直接引用
    常量池中的各种符号引用解析为指针、偏移量等内存地址的直接引用
      
#### 3. Initializing（面试常考）
   
    调用类初始化代码 <clinit>，给静态成员变量赋初始值


   
面试题：T001_ClassLoadingProcedure
```
//类加载
public class T001_ClassLoadingProcedure {
    public static void main(String[] args) {
        System.out.println(T.count);
    }
}

class T {
    public static T t = new T(); // null
    public static int count = 2; //0

    //private int m = 8;

    private T() {
        count ++;
        System.out.println("--" + count);
    }
}

运行情况分析：
3的情况
public static int count=2;//0----2
public static T t= new T();//null---count++

private T(){
    count++
}

2的情况
public static T t= new T();//null---count++->1
public static int count=2;//0---2

private T(){
    count++
}

```

```
//new 对象
代码
public class TT {
    int m = 8;

    public static void main(String[] args) {
        TT t = new TT();
    }
}

class指令
    0 new #3 <com/mashibing/jvm/c0_basic/TT>  内存已经申请
    3 dup
    4 invokespecial #4 <com/mashibing/jvm/c0_basic/TT.<init>>  初始化
    7 astore_1   赋值为8
    8 return
    
```

==静态变量是类加载过程中分为两步，preparation 和initializing==
    
==new 对象是：1.先分配内存，成员变量赋默认值；2.构造方法，赋值为初始值==


### 二. 小总结

   1. load类 - 默认值 - 初始值
   2. new对象 - 申请内存 - 默认值 - 初始值
   3. 单例模式的DCL，INSTANCE需要加volatile，防止指令重排  
        INSTANCE = new Mgr06();  
        new对象分两步，在第一步赋默认值后被别的线程读取，在没有赋初始值之前，从默认值开始进行count++


  