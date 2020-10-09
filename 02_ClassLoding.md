## 3：类加载-初始化
![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/023.png) 

### 一. 加载过程
#### 1. Loading
    
##### 1.1类加载器

![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/024.png)   
    
    一个class文件，load到内存中
        1.将class文件放入metaspace中；
        2.在堆内存中创建一个class对象
    
    类加载器
        1.Bootstrap
            jre/lib/rt.jar(runtime) charset.jar等核心类
            返回null，因为这些类是C++实现，java中没有对应的class
        2.Extension
            jre/lib/ext/*.jar
            或由-Djava.ext.dirs指定
            
        3.App
            加载class.path指定内容
        
        4.Custom
        
==注意：类加载器，bootstrap、ext、app、custome是父子关系，但是请记住他们之间不存在继承关系==

==父加载器不是“类加载器的加载器”！！！！也不是“类加载器的父类加载器”==         
    所以一般说：custome找不到，去它的父加载器App，而不是父类加载器
    
==下图是语法上的关系，实际加载过程是按bootstrap、ext、app、custome来进行加载==

![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/025.png)


```
public class T002_ClassLoaderLevel {
    public static void main(String[] args) {
        System.out.println(String.class.getClassLoader());//Bootstrap
        System.out.println(sun.awt.HKSCS.class.getClassLoader());//Bootstrap
        System.out.println(sun.net.spi.nameservice.dns.DNSNameService.class.getClassLoader());//Extension
        System.out.println(T002_ClassLoaderLevel.class.getClassLoader());//app

        System.out.println(sun.net.spi.nameservice.dns.DNSNameService.class.getClassLoader().getClass().getClassLoader());//bootstrap
        System.out.println(T002_ClassLoaderLevel.class.getClassLoader().getClass().getClassLoader());//bootstrap

        System.out.println(new T006_MSBClassLoader().getParent());//custome--app
        System.out.println(ClassLoader.getSystemClassLoader());//app
    }
}

运行结果：
null
null
sun.misc.Launcher$ExtClassLoader@677327b6
sun.misc.Launcher$AppClassLoader@18b4aac2
null
null
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$AppClassLoader@18b4aac2
```

##### 1.2 双亲委派机制，主要出于安全来考虑

**找不到抛异常：ClassNotFoundExecuption**

![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/026.png)
==面试题：为什么类加载器要用双亲委派？==

    1.主要是为了安全来考虑
    用反正法来验证，自定义了一个java.lang.String，然后由Custom ClassLoader加载到内存中，覆盖掉Bootstrap加载器加载的String对象
    2.次要：避免资源浪费，不需要重复加载
    
==面试题：java.lang.String类由自定义类加载器加载行不行？==

    不行，不安全，jvm采用了双亲委派

```
public class T004_ParentAndChild {
    public static void main(String[] args) {
        System.out.println(T004_ParentAndChild.class.getClassLoader());//app
        System.out.println(T004_ParentAndChild.class.getClassLoader().getClass().getClassLoader());//null
        System.out.println(T004_ParentAndChild.class.getClassLoader().getParent());//exe
        System.out.println(T004_ParentAndChild.class.getClassLoader().getParent().getParent());//null
        System.out.println(T004_ParentAndChild.class.getClassLoader().getParent().getParent().getParent());//ClassNotFoundExecuption

    }
}    
运行结果
sun.misc.Launcher$AppClassLoader@18b4aac2
null
sun.misc.Launcher$ExtClassLoader@1b6d3586
null
Exception in thread "main" java.lang.NullPointerException
	at com.mashibing.jvm.c2_classloader.T004_ParentAndChild.main(T004_ParentAndChild.java:9)
```
      
##### 1.3类加载范围：

> 来自Launcher源码
> 
> BootstrapClassLoader加载路径：sun.boot.class.path
> 
> ExtensionClassLoader加载路径：java.ext.dirs
> 
> AppClassLoader加载路径：java.class.path

![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/027.png)

==注意：app下会加载项目路径==

```
public class T003_ClassLoaderScope {
    public static void main(String[] args) {
        String pathBoot = System.getProperty("sun.boot.class.path");
        System.out.println(pathBoot.replaceAll(";", System.lineSeparator()));

        System.out.println("--------------------");
        String pathExt = System.getProperty("java.ext.dirs");
        System.out.println(pathExt.replaceAll(";", System.lineSeparator()));

        System.out.println("--------------------");
        String pathApp = System.getProperty("java.class.path");
        System.out.println(pathApp.replaceAll(";", System.lineSeparator()));
    }
}

运行结果：
C:\Program Files\Java\jdk1.8.0_231\jre\lib\resources.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\rt.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\sunrsasign.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\jsse.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\jce.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\charsets.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\jfr.jar
C:\Program Files\Java\jdk1.8.0_231\jre\classes
--------------------
C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext
C:\WINDOWS\Sun\Java\lib\ext
--------------------
C:\Program Files\Java\jdk1.8.0_231\jre\lib\charsets.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\deploy.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\access-bridge-64.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\cldrdata.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\dnsns.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\jaccess.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\jfxrt.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\localedata.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\nashorn.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\sunec.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\sunjce_provider.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\sunmscapi.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\sunpkcs11.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\ext\zipfs.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\javaws.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\jce.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\jfr.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\jfxswt.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\jsse.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\management-agent.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\plugin.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\resources.jar
C:\Program Files\Java\jdk1.8.0_231\jre\lib\rt.jar
D:\ideaWorkspace\mashibing\JVM-master\out\production\JVM-master//注意是项目路径
D:\software\IntelliJ IDEA 2019.3.1\lib\idea_rt.jar
```
        
##### 1.4Launcher
    
    classLoader的包装类
    三个加载器是ClassLauncher的内部类
![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/029.png) 

Launcher源码
```
static class AppClassLoader extends URLClassLoader {
        final URLClassPath ucp = SharedSecrets.getJavaNetAccess().getURLClassPath(this);

        public static ClassLoader getAppClassLoader(final ClassLoader var0) throws IOException {
            final String var1 = System.getProperty("java.class.path");//加载路径
            final File[] var2 = var1 == null ? new File[0] : Launcher.getClassPath(var1);
            return (ClassLoader)AccessController.doPrivileged(new PrivilegedAction<Launcher.AppClassLoader>() {
                public Launcher.AppClassLoader run() {
                    URL[] var1x = var1 == null ? new URL[0] : Launcher.pathToURLs(var2);
                    return new Launcher.AppClassLoader(var1x, var0);
                }
            });
        }


static class ExtClassLoader extends URLClassLoader {
        private static volatile Launcher.ExtClassLoader instance;
        
private static File[] getExtDirs() {
            String var0 = System.getProperty("java.ext.dirs");//加载路径
            File[] var1;
            if (var0 != null) {
                StringTokenizer var2 = new StringTokenizer(var0, File.pathSeparator);
                int var3 = var2.countTokens();
                var1 = new File[var3];

                for(int var4 = 0; var4 < var3; ++var4) {
                    var1[var4] = new File(var2.nextToken());
                }
            } else {
                var1 = new File[0];
            }

            return var1;
        }
        
        
private static class BootClassPathHolder {
        static final URLClassPath bcp;

        private BootClassPathHolder() {
        }

        static {
            URL[] var0;
            if (Launcher.bootClassPath != null) {
                var0 = (URL[])AccessController.doPrivileged(new PrivilegedAction<URL[]>() {
                    public URL[] run() {
                        File[] var1 = Launcher.getClassPath(Launcher.bootClassPath);
                        int var2 = var1.length;
                        HashSet var3 = new HashSet();

                        for(int var4 = 0; var4 < var2; ++var4) {
                            File var5 = var1[var4];
                            if (!var5.isDirectory()) {
                                var5 = var5.getParentFile();
                            }

                            if (var5 != null && var3.add(var5)) {
                                MetaIndex.registerDirectory(var5);
                            }
                        }

                        return Launcher.pathToURLs(var1);
                    }
                });
            } else {
                var0 = new URL[0];
            }

            bcp = new URLClassPath(var0, Launcher.factory, (AccessControlContext)null);
            bcp.initLookupCache((ClassLoader)null);
        }
    }
    private static String bootClassPath = System.getProperty("sun.boot.class.path");//加载路径
```
        

##### 1.5. LazyLoading 五种情况（很复杂，面试会考）
    
    严格讲应该叫LazyInitialing
      
==面试题：JVM规范并没有规定何时加载==
      
==但是严格规定了什么时候必须初始化（现在会考）==
     
    –new getstatic putstatic invokestatic指令、访问final变量除外（访问非静态的方法或属性）
  
    –java.lang.reflect对类进行反射调用时
  
    –初始化子类的时候，父类首先初始化
  
    –虚拟机启动时，被执行的主类必须初始化
  
    –动态语言支持java.lang.invoke.MethodHandle解析的结果为REF_getstatic、
    REF_putstatic、REF_invokestatic的方法句柄时，该类必须初始化(不了解)
    

```
public class T008_LazyLoading { //严格讲应该叫lazy initialzing，因为java虚拟机规范并没有严格规定什么时候必须loading,但严格规定了什么时候initialzing
    static {
        System.out.println("主类加载");// 加载--主类初始化
    }

    public static void main(String[] args) throws Exception {
//        P p; //不加载
//        X x = new X(); //加载---父子类初始化
//        System.out.println(P.i); //不加载
//        System.out.println(P.j); //加载---非静态属性初始化
        Class.forName("com.mashibing.jvm.c2_classloader.T008_LazyLoading$P"); //加载---反射初始化 T008_LazyLoading$P是加载内部类P

    }

    public static class P {
        final static int i = 8;
        static int j = 9;
        static {
            System.out.println("P");
        }
    }

    public static class X extends P {
        static {
            System.out.println("X");
        }
    }
}
```

      
##### 1.6. ClassLoader的源码
      
==**findInCache -> parent.loadClass -> findClass()**==

==迭代==

    1.loadClass将加载到内存中的class对象返回
    
    2.那么如何自定义一个ClassLoader？
        实现findClass方法就可以了
        
        ==历史上JDK自己破坏过双亲委派，可百度 
    
    3.采用的设计模式：钩子函数、模板方法
    
面试题：用过哪些设计模式？举例

    模板方法：ClassLoader  loadClass的时候
    
ClassLoader源码   
```
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // First, check if the class has already been loaded
        Class<?> c = findLoadedClass(name);//findInCache
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    c = parent.loadClass(name, false);//迭代操作,也是调用loadClass方法，自底向上
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
            }

            if (c == null) {
                // If still not found, then invoke findClass in order
                // to find the class.
                long t1 = System.nanoTime();
                c = findClass(name);//自己找

                // this is the defining class loader; record the stats
                sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                sun.misc.PerfCounter.getFindClasses().increment();
            }
        }
        if (resolve) {
            resolveClass(c);
        }
        return c;
    }
}

private final ClassLoader parent;

protected final Class<?> findLoadedClass(String name) {
    if (!checkName(name))
        return null;
    return findLoadedClass0(name);
}

private native final Class<?> findLoadedClass0(String name);

protected Class<?> findClass(String name) throws ClassNotFoundException {
    throw new ClassNotFoundException(name);
}

URLClassLoader类
protected Class<?> findClass(final String name)
        throws ClassNotFoundException
    {
        final Class<?> result;
        try {
            result = AccessController.doPrivileged(
                new PrivilegedExceptionAction<Class<?>>() {
                    public Class<?> run() throws ClassNotFoundException {
                        String path = name.replace('.', '/').concat(".class");
                        Resource res = ucp.getResource(path, false);
                        if (res != null) {
                            try {
                                return defineClass(name, res);
                            } catch (IOException e) {
                                throw new ClassNotFoundException(name, e);
                            }
                        } else {
                            return null;
                        }
                    }
                }, acc);
        } catch (java.security.PrivilegedActionException pae) {
            throw (ClassNotFoundException) pae.getException();
        }
        if (result == null) {
            throw new ClassNotFoundException(name);
        }
        return result;
    }
    
AppClassLoader类   
public Class<?> loadClass(String var1, boolean var2) throws ClassNotFoundException {
            int var3 = var1.lastIndexOf(46);
            if (var3 != -1) {
                SecurityManager var4 = System.getSecurityManager();
                if (var4 != null) {
                    var4.checkPackageAccess(var1.substring(0, var3));
                }
            }

            if (this.ucp.knownToNotExist(var1)) {
                Class var5 = this.findLoadedClass(var1);
                if (var5 != null) {
                    if (var2) {
                        this.resolveClass(var5);
                    }

                    return var5;
                } else {
                    throw new ClassNotFoundException(var1);
                }
            } else {
                return super.loadClass(var1, var2);
            }
        }
```
      
##### 1.7. 自定义类加载器
    
    利用类加载器加载资源：代码T005_LoadClassByHand
    
    ClassLoader跟反射的关系
        ClassLoader是反射的基石，ClassLoader获取的class对象，再进行反射
    
    什么时候需要加载一个类？
        Spring的动态代理
        热部署Jrebel
    
==Spring、tomcat都定义了自己的ClassLoader==


==自定义加载器的步骤==
1. extends ClassLoader  
2. overwrite findClass() -> defineClass(byte[] -> Class clazz)  
3. 加密  


4. <font color=red>parent是如何指定的</font>  
    用super(parent)指定  

代码：
```
public class T005_LoadClassByHand {
    public static void main(String[] args) throws ClassNotFoundException {
        Class clazz = T005_LoadClassByHand.class.getClassLoader().loadClass("com.mashibing.jvm.c2_classloader.T002_ClassLoaderLevel");
        System.out.println(clazz.getName());

        //利用类加载器加载资源，参考坦克图片的加载
        //T005_LoadClassByHand.class.getClassLoader().getResourceAsStream("");
    }
}
运行结果：
com.mashibing.jvm.c2_classloader.T002_ClassLoaderLevel

```


```
public class Hello {
    public void m() {
        System.out.println("Hello JVM!");
    }
}

public class T006_MSBClassLoader extends ClassLoader {

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        File f = new File("D:/ideaWorkspace/mashibing/JVM/out/production/JVM-projec", name.replace(".", "/").concat(".class"));
        try {
            FileInputStream fis = new FileInputStream(f);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            int b = 0;

            while ((b=fis.read()) !=0) {
                baos.write(b);
            }

            byte[] bytes = baos.toByteArray();
            baos.close();
            fis.close();//可以写更加严谨

            return defineClass(name, bytes, 0, bytes.length);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return super.findClass(name); //throws ClassNotFoundException
    }

    public static void main(String[] args) throws Exception {
        ClassLoader l = new T006_MSBClassLoader();
        Class clazz = l.loadClass("com.mashibing.jvm.Hello");
        Class clazz1 = l.loadClass("com.mashibing.jvm.Hello");

        System.out.println(clazz == clazz1);

        Hello h = (Hello)clazz.newInstance();
        h.m();

        System.out.println(l.getClass().getClassLoader());//app
        System.out.println(l.getParent());//app

        System.out.println(getSystemClassLoader());//app
    }
}

运行结果：
true
Hello JVM!
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$AppClassLoader@18b4aac2
```


```
加密
public class T007_MSBClassLoaderWithEncription extends ClassLoader {

    public static int seed = 0B10110110;

    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        File f = new File("D:/ideaWorkspace/mashibing/JVM/out/production/JVM-project", name.replace('.', '/').concat(".msbclass"));

        try {
            FileInputStream fis = new FileInputStream(f);
            ByteArrayOutputStream baos = new ByteArrayOutputStream();
            int b = 0;

            while ((b=fis.read()) !=0) {
                baos.write(b ^ seed);//二进制异或解密
            }

            byte[] bytes = baos.toByteArray();
            baos.close();
            fis.close();//可以写的更加严谨

            return defineClass(name, bytes, 0, bytes.length);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return super.findClass(name); //throws ClassNotFoundException
    }

    public static void main(String[] args) throws Exception {

        encFile("com.mashibing.jvm.hello");

        ClassLoader l = new T007_MSBClassLoaderWithEncription();
        Class clazz = l.loadClass("com.mashibing.jvm.Hello");
        Hello h = (Hello)clazz.newInstance();
        h.m();

        System.out.println(l.getClass().getClassLoader());
        System.out.println(l.getParent());
    }

    private static void encFile(String name) throws Exception {
        File f = new File("D:/ideaWorkspace/mashibing/JVM/out/production/JVM-projectr", name.replace('.', '/').concat(".class"));
        FileInputStream fis = new FileInputStream(f);
        FileOutputStream fos = new FileOutputStream(new File("D:/ideaWorkspace/mashibing/JVM/out/production/JVM-project", name.replaceAll(".", "/").concat(".msbclass")));
        int b = 0;

        while((b = fis.read()) != -1) {
            fos.write(b ^ seed);//异或加密
        }

        fis.close();
        fos.close();
    }
}
运行结果：
Hello JVM!
sun.misc.Launcher$AppClassLoader@18b4aac2
sun.misc.Launcher$AppClassLoader@18b4aac2
```


```
public class T010_Parent {
    private static T006_MSBClassLoader parent = new T006_MSBClassLoader();

    private static class MyLoader extends ClassLoader {
        public MyLoader() {
            super(parent);
        }
    }
}
```


```
源码：获取ClassLoader的parent
protected ClassLoader() {//构造方法
    this(checkCreateClassLoader(), getSystemClassLoader());
}

@CallerSensitive
public static ClassLoader getSystemClassLoader() {//获取默认的ClassLoader
    initSystemClassLoader();
    if (scl == null) {
        return null;
    }
    SecurityManager sm = System.getSecurityManager();
    if (sm != null) {
        checkClassLoaderPermission(scl, Reflection.getCallerClass());
    }
    return scl;
}
    
private static synchronized void initSystemClassLoader() {
    if (!sclSet) {
        if (scl != null)
            throw new IllegalStateException("recursive invocation");
        sun.misc.Launcher l = sun.misc.Launcher.getLauncher();
        if (l != null) {
            Throwable oops = null;
            scl = l.getClassLoader();//通过launcher获取ClassLoader
            try {
                scl = AccessController.doPrivileged(
                    new SystemClassLoaderAction(scl));
            } catch (PrivilegedActionException pae) {
                oops = pae.getCause();
                if (oops instanceof InvocationTargetException) {
                    oops = oops.getCause();
                }
            }
            if (oops != null) {
                if (oops instanceof Error) {
                    throw (Error) oops;
                } else {
                    // wrap the exception
                    throw new Error(oops);
                }
            }
        }
        sclSet = true;
    }
}
    
public Launcher() {//launcher的构造方法
    Launcher.ExtClassLoader var1;
    try {
        var1 = Launcher.ExtClassLoader.getExtClassLoader();
    } catch (IOException var10) {
        throw new InternalError("Could not create extension class loader", var10);
    }

    try {
        this.loader = Launcher.AppClassLoader.getAppClassLoader(var1);//指定了APP为默认的ClassLoader
    } catch (IOException var9) {
        throw new InternalError("Could not create application class loader", var9);
    }

    Thread.currentThread().setContextClassLoader(this.loader);
    String var2 = System.getProperty("java.security.manager");
    if (var2 != null) {
        SecurityManager var3 = null;
        if (!"".equals(var2) && !"default".equals(var2)) {
            try {
                var3 = (SecurityManager)this.loader.loadClass(var2).newInstance();
            } catch (IllegalAccessException var5) {
            } catch (InstantiationException var6) {
            } catch (ClassNotFoundException var7) {
            } catch (ClassCastException var8) {
            }
        } else {
            var3 = new SecurityManager();
        }

        if (var3 == null) {
            throw new InternalError("Could not create SecurityManager: " + var2);
        }

        System.setSecurityManager(var3);
    }

}

    
```

==作业：了解CompareAPI==

##### ==1.8 双亲委派的打破(知识点扩展，有面试问到)==
       
    1. 如何打破：重写loadClass（）
    2. 何时打破过？
        1. JDK1.2之前，自定义ClassLoader都必须重写loadClass()
        2. ThreadContextClassLoader可以实现基础类调用实现类代码，通过thread.setContextClassLoader指定
        3. 热启动，热部署
             1. osgi tomcat 都有自己的模块指定classloader（可以加载同一类库的不同版本）
            tomcat热部署，因为tomcat重写了classLoader(),热部署的时候会将之前的ClassLoader干掉，再重新new一个ClassLoader
            
==待办：公开课：连老师，tomcat源码==

```
//不能实现热部署 
public class T011_ClassReloading1 {
    public static void main(String[] args) throws Exception {
        T006_MSBClassLoader msbClassLoader = new T006_MSBClassLoader();
        Class clazz = msbClassLoader.loadClass("com.mashibing.jvm.Hello");

        msbClassLoader = null;
        System.out.println(clazz.hashCode());

        msbClassLoader = null;

        msbClassLoader = new T006_MSBClassLoader();
        Class clazz1 = msbClassLoader.loadClass("com.mashibing.jvm.Hello");
        System.out.println(clazz1.hashCode());

        System.out.println(clazz == clazz1);
    }
}

运行结果：
1163157884
1163157884
true
```

```
//热部署
public class T012_ClassReloading2 {
    private static class MyLoader extends ClassLoader {
        @Override
        public Class<?> loadClass(String name) throws ClassNotFoundException {

            File f = new File("D:/ideaWorkspace/mashibing/JVM-master/out/production/JVM-master" + name.replace(".", "/").concat(".class"));

            if(!f.exists()) return super.loadClass(name);

            try {

                InputStream is = new FileInputStream(f);

                byte[] b = new byte[is.available()];
                is.read(b);
                return defineClass(name, b, 0, b.length);
            } catch (IOException e) {
                e.printStackTrace();
            }

            return super.loadClass(name);
        }
    }

    public static void main(String[] args) throws Exception {
        MyLoader m = new MyLoader();
        Class clazz = m.loadClass("com.mashibing.jvm.Hello");

        m = new MyLoader();
        Class clazzNew = m.loadClass("com.mashibing.jvm.Hello");

        System.out.println(clazz == clazzNew);
    }
}
运行结果：
true
```

         
##### 1.9 混合模式         
        
面试题：JVM为什么不直接编译呢？

    1.现在解释器执行也很快
    2.如果直接编译的，启动过程会特别长
    

---
    解释器（bytecode intepreter）\
    JIT（Just in-Time compiler）
        
    通过命令设置模式
        -Xmixed
        -Xint：启动快，执行慢
        -Xcomp：启动慢，执行快
    
    热点代码检测：
        多次被调用的方法（方法计数器）
        多次被调用的循环(循环计数器)
        
        检测热点代码：-XX:CompileThreshold=10000
        
![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/028.png)        

```
public class T009_WayToRun {
    public static void main(String[] args) {
        for(int i=0; i<10_0000; i++)
            m();

        long start = System.currentTimeMillis();
        for(int i=0; i<10_0000; i++) {
            m();
        }
        long end = System.currentTimeMillis();
        System.out.println(end - start);
    }

    public static void m() {
        for(long i=0; i<10_0000L; i++) {
            long j = i%3;
        }
    }
}

tomcat配置：
    -Xmixed :4000
    -Xint：时间太长，没有等结束
    -Xcomp：3954

使用JMH进行测试

```

### 小结：

1. 加载过程
   1. Loading
      
      1. 类加载器以及加载范围
      
      2. 双亲委派，主要出于安全来考虑
      
      3. LazyLoading 五种情况
      
            –new getstatic putstatic invokestatic指令，访问final变量除外
      
            –java.lang.reflect对类进行反射调用时
      
            –初始化子类的时候，父类首先初始化
      
            –虚拟机启动时，被执行的主类必须初始化
      
            –动态语言支持java.lang.invoke.MethodHandle解析的结果为REF_getstatic REF_putstatic REF_invokestatic的方法句柄时，该类必须初始化
      
      3. ClassLoader的源码
      
         1. findInCache -> parent.loadClass -> findClass()
      
      4. 自定义类加载器
      
         1. extends ClassLoader
         2. overwrite findClass() -> defineClass(byte[] -> Class clazz)
         3. 加密
         4. <font color=red>parent是如何指定的，打破双亲委派</font>
            1. 用super(parent)指定
            2. 双亲委派的打破
               1. 如何打破：重写loadClass（）
               2. 何时打破过？
                  1. JDK1.2之前，自定义ClassLoader都必须重写loadClass()
                  2. ThreadContextClassLoader可以实现基础类调用实现类代码，通过thread.setContextClassLoader指定
                  3. 热启动，热部署
                     1. osgi tomcat 都有自己的模块指定classloader（可以加载同一类库的不同版本）
                    
                     OSGI（Open Service Gateway Initiative，直译为“开放服务网关”）
                     OSGi联盟给出的最新OSGi定义是The Dynamic Module System for Java，即面向Java的动态模块化系统。
      5. 混合执行 编译执行 解释执行
      
         1. 检测热点代码：-XX:CompileThreshold = 10000
         2. 方法计数器
         3. 循环计数器



---
---
## 小结：类加载-初始化
![image](https://raw.githubusercontent.com/musictaste/JVM/master/image/023.png) 
1. 加载过程
   1. Loading
      
      1. 双亲委派，主要出于安全来考虑
      
      2. LazyLoading 五种情况
      
         1. –new getstatic putstatic invokestatic指令，访问final变量除外
      
            –java.lang.reflect对类进行反射调用时
      
            –初始化子类的时候，父类首先初始化
      
            –虚拟机启动时，被执行的主类必须初始化
      
            –动态语言支持java.lang.invoke.MethodHandle解析的结果为REF_getstatic REF_putstatic REF_invokestatic的方法句柄时，该类必须初始化
      
      3. ClassLoader的源码
      
         1. findInCache -> parent.loadClass -> findClass()
      
      4. 自定义类加载器
      
         1. extends ClassLoader
         2. overwrite findClass() -> defineClass(byte[] -> Class clazz)
         3. 加密
         4. <font color=red>第一节课遗留问题：parent是如何指定的，打破双亲委派，学生问题桌面图片</font>
            1. 用super(parent)指定
            2. 双亲委派的打破
               1. 如何打破：重写loadClass（）
               2. 何时打破过？
                  1. JDK1.2之前，自定义ClassLoader都必须重写loadClass()
                  2. ThreadContextClassLoader可以实现基础类调用实现类代码，通过thread.setContextClassLoader指定
                  3. 热启动，热部署
                     1. osgi tomcat 都有自己的模块指定classloader（可以加载同一类库的不同版本）
      
      5. 混合执行 编译执行 解释执行
      
         1. 检测热点代码：-XX:CompileThreshold = 10000
      
   2. Linking 
      1. Verification
         1. 验证文件是否符合JVM规定
      2. Preparation（面试常考）
         1. 静态成员变量赋默认值
      3. Resolution
         1. 将类、方法、属性等符号引用解析为直接引用
            常量池中的各种符号引用解析为指针、偏移量等内存地址的直接引用
      
   3. Initializing
   
      1. 调用类初始化代码 <clinit>，给静态成员变量赋初始值
   
2. 小总结：

   1. load - 默认值 - 初始值
   2. new - 申请内存 - 默认值 - 初始值    