
首先引入一个概念，什么是Java类加载器？


一句话总结：类加载器（class loader）用来加载 Java 类到 Java 虚拟机中。


官方总结：Java类加载器（英语：Java Classloader）是Java运行时环境（Java Runtime Environment）的一部分，负责动态加载Java类到Java虚拟机的内存空间中。类通常是按需加载，即第一次使用该类时才加载。由于有了类加载器，Java运行时系统不需要知道文件与文件系统。


## 类与类加载器


先来看一下JVM中默认的类加载器


### 分类


实现通过类的全限定名获取该类的二进制字节流的代码块叫做类加载器。


类加载器虽然只用于实现类的加载动作，但它在Java程序中起到的作用却远超类加载阶段


对于任意一个类，都必须由加载它的类加载器和这个类本身一起共同确立其在Java虚拟机中的唯一性，每一个类加载器，都拥有一个独立的类名称空间。这句话可以表达得更通俗一些：比较两个类是否“相等”，只有在这两个类是由同一个类加载器加载的前提下才有意义，否则，即使这两个类来源于同一个Class文件，被同一个Java虚拟机加载，只要加载它们的类加载器不同，那这两个类就必定不相等


[![](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202404251622424.gif)](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202404251622424.gif)


为什么要有三个类加载器？一方面是分工，各自负责各自的区块，就如Application Class Loader主要负责加载用户之间开发的代码，另一方面为了实现委托模型。


#### 启动类加载器


引导类加载器属于JVM的一部分，由C\+\+代码实现。


引导类加载器负责加载\\jre\\lib路径下的核心类库，由于安全考虑只加载 包名 java、javax、sun开头的类。


java
```
public class Demo1 {
    public static void main(String[] args) {
        //Bootstrap 引导类加载器
        //打印为null,是因为Bootstrap是C++实现的。
        ClassLoader classLoader = Object.class.getClassLoader();
        System.out.println(classLoader);

        //查看引导类加载器会加载那些jar包
        URL[] urLs = Launcher.getBootstrapClassPath().getURLs();
        for (URL urL : urLs) {
            System.out.println(urL);
        }
    }

}

输出：
null
file:/D:/JavaSoftware/jdk1.8.0_131/jre/lib/resources.jar
file:/D:/JavaSoftware/jdk1.8.0_131/jre/lib/rt.jar
file:/D:/JavaSoftware/jdk1.8.0_131/jre/lib/sunrsasign.jar
file:/D:/JavaSoftware/jdk1.8.0_131/jre/lib/jsse.jar
file:/D:/JavaSoftware/jdk1.8.0_131/jre/lib/jce.jar
file:/D:/JavaSoftware/jdk1.8.0_131/jre/lib/charsets.jar
file:/D:/JavaSoftware/jdk1.8.0_131/jre/lib/jfr.jar
file:/D:/JavaSoftware/jdk1.8.0_131/jre/classes
```

#### 拓展类加载器


全类名:sum.misc.Launch$ExtClassLoader，Java语言实现。


扩展类加载器的父加载器是Bootstrap启动类加载器 (注:不是继承关系)


扩展类加载器负责加载\\jre\\lib\\ext目录下的类库。


[![](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202404251622434.gif)](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202404251622434.gif)


java
```
import com.sun.javafx.webkit.WebPageClientImpl;

public class Demo1 {
    public static void main(String[] args) {
        //ext目录下的类，获取加载器
        ClassLoader classLoader = WebPageClientImpl.class.getClassLoader();
        System.out.println(classLoader);
    }
}

输出:
sun.misc.Launcher$ExtClassLoader@330bedb4
```

#### 应用程序类加载器


全类名: sun.misc.Launcher$AppClassLoader


系统类加载器的父加载器是ExtClassLoader扩展类加载器（注: 不是继承关系）。


系统类加载器负责加载 classpath环境变量所指定的类库，是用户自定义类的默认类加载器。


java
```
public class Demo1 {
    public static void main(String[] args) {
        ClassLoader classLoader = Demo1.class.getClassLoader();
        System.out.println(classLoader);
    }

}

输出：
sun.misc.Launcher$AppClassLoader@18b4aac
```

### 类的加载方式


类加载有三种方式:


1. 命令行启动应用时候由JVM初始化加载
2. 通过Class.forName()方法动态加载
3. 通过ClassLoader.loadClass()方法动态加载


java
```
package com.pdai.jvm.classloader;
public class loaderTest { 
        public static void main(String[] args) throws ClassNotFoundException { 
                ClassLoader loader = HelloWorld.class.getClassLoader(); 
                System.out.println(loader); 
                //使用ClassLoader.loadClass()来加载类，不会执行初始化块 
                loader.loadClass("Test2"); 
                //使用Class.forName()来加载类，默认会执行初始化块 
//                Class.forName("Test2"); 
                //使用Class.forName()来加载类，并指定ClassLoader，初始化时不执行静态块 
//                Class.forName("Test2", false, loader); 
        } 
}

public class Test2 { 
        static { 
                System.out.println("静态初始化块执行了！"); 
        } 
}
```

Class.forName()和ClassLoader.loadClass()的区别：


* Class.forName(): 将类的.class文件加载到jvm中之外，还会对类进行解释，执行类中的static块；
* ClassLoader.loadClass(): 只干一件事情，就是将.class文件加载到jvm中，不会执行static中的内容,只有在newInstance才会去执行static块。
* Class.forName(name, initialize, loader)带参函数也可控制是否加载static块。并且只有调用了newInstance()方法采用调用构造函数，创建类的对象 。


## JVM类加载机制


* 全盘负责：当一个类加载器负责加载某个Class时，该Class所依赖的和引用的其他Class也将由该类加载器负责载入，除非显示使用另外一个类加载器来载入
* 缓存机制：缓存机制将会保证所有加载过的Class都会被缓存，当程序中需要使用某个Class时，类加载器先从缓存区寻找该Class，只有缓存区不存在，系统才会读取该类对应的二进制数据，并将其转换成Class对象，存入缓存区。这就是为什么修改了Class后，必须重启JVM，程序的修改才会生效
* 双亲委派机制：如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把请求委托给父加载器去完成，依次向上，因此，所有的类加载请求最终都应该被传递到顶层的启动类加载器中，只有当父加载器在它的搜索范围中没有找到所需的类时，即无法完成该加载，子加载器才会尝试自己去加载该类。


一个应用程序总是由n多个类组成，Java程序启动时，并不是一次把所有的类全部加载后再运行，它总是先把保证程序运行的基础类一次性加载到JVM中，其它类等到JVM用到的时候再加载。


### 双亲委派模型


一个类加载器收到一个类的加载请求时，它首先不会自己尝试去加载它，而是把这个请求委派给父类加载器去完成，这样层层委派，因此所有的加载请求最终都会传送到顶层的启动类加载器中，只有当父类加载器反馈自己无法完成这个加载请求时，子加载器才会尝试自己去加载。


[![](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202404251622438.gif)](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202404251622438.gif)


双亲委派模型的具体实现代码在 java.lang.ClassLoader 中，此类的 loadClass() 方法运行过程如下：先检查类是否已经加载过，如果没有则让父类加载器去加载。当父类加载器加载失败时抛出ClassNotFoundException ，此时尝试自己去加载。


#### 源码


java
```
protected Class loadClass(String name, boolean resolve) throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {
        // 1.首先查找该类是否已经被该类加载器加载过了
        Class c = findLoadedClass(name);
        //如果没有被加载过
        if (c == null) {
            long t0 = System.nanoTime();
            try {
                if (parent != null) {
                    // 2. 有上级的话，委派上级 loadClass
                    c = parent.loadClass(name, false);
                } else {
                    // 3. 如果没有上级了（ExtClassLoader），则委派BootstrapClassLoader
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                // ClassNotFoundException thrown if class not found
                // from the non-null parent class loader
                //捕获异常，但不做任何处理
            }

            if (c == null) {
                // 4. 每一层找不到，调用 findClass 方法（每个类加载器自己扩展）来加载
                long t1 = System.nanoTime();
                c = findClass(name);

                // 记录时间
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
```

例如:


java
```
public class Demo1 {
    public static void main(String[] args) throws ClassNotFoundException {
        Class aClass = Demo1.class.getClassLoader().loadClass("com.seven.jvm.F");
        System.out.println(aClass.getClassLoader());
    }
}
```

执行流程为：


1. sun.misc.Launcher$AppClassLoader //1 处， 开始查看已加载的类，结果没有
2. sun.misc.Launcher$AppClassLoader // 2 处，委派上级
3. sun.misc.Launcher$ExtClassLoader // 1 处，查看已加载的类，结果没有
4. sun.misc.Launcher$ExtClassLoader // 3 处，没有上级了，则委派 BootstrapClassLoader 查找
5. BootstrapClassLoader 是在 JAVA\_HOME/jre/lib 下找 F 这个类，显然没有，就会捕获异常，但是不处理
6. sun.misc.Launcher$ExtClassLoader // 4 处，调用自己的 findClass 方法，是在JAVA\_HOME/jre/lib/ext 下找 F 这个类，显然没有，回到 sun.misc.Launcher $AppClassLoader 的 // 2 处
7. 继续执行到 sun.misc.Launcher$AppClassLoader // 4 处，调用它自己的 findClass 方法，在 classpath 下查找，找到了


#### 双亲委派模型目的


可以防止内存中出现多份同样的字节码。如果没有双亲委派模型而是由各个类加载器自行加载的话，如果用户编写了一个 java.lang.Object 的同名类并放在 ClassPath 中，多个类加载器都去加载这个类到内存中，系统中将会出现多个不同的 Object 类，那么类之间的比较结果及类的唯一性将无法保证。


#### 依赖传递原则


假设类C是由加载器L1定义加载的，那么类C中所依赖的其他类将会通过L1进行加载。


如下：假设Demo2是由L1定义加载的，那么类Demo2中所依赖的类F和接口IDemo将会通过L1进行加载。甚至还用到了String类，String也会由L1加载，但由于双亲委派模型，String类最终会由Bootstrap ClassLoader进行加载。（前提是这些依赖的类尚未加载）


java
```
import com.seven.jvm2.F;
import com.seven.jvm2.IDemo;

public class Demo2 implements IDemo {

    @Override
    public void run() {
        F f = new F();
       String s = f.toString().toLowerCase();
        System.out.println(s);
    }
}
```

### 打破双亲委派模型


什么时候需要打破双亲委派模型？比如类A已经有一个classA，恰好类B也有一个clasA 但是两者内容不一致，如果不打破双亲委派模型，那么类A只会加载一次


只要在加载类的时候，不按照UserCLASSlOADER\-\>Application ClassLoader\-\>Extension ClassLoader\-\>Bootstrap ClassLoader的顺序来加载就算打破打破双亲委派模型了。比如自定义个ClassLoader，重写loadClass方法（不依照往上开始寻找类加载器），那就算是打破双亲委派机制了。


打破双亲委派模型有两种方式：


1. 自定义一个类加载器的类，并覆盖抽象类java.lang.ClassL oader中loadClass..)方法，不再优先委派“父”加载器进行类加载。（比如Tomcat）
2. 主动违背类加载器的依赖传递原则


	* 例如在一个BootstrapClassLoader加载的类中，又通过APPClassLoader来加载所依赖的其它类，这就打破了“双亲委派模型”中的层次结构，逆转了类之间的可见性。
	* 典型的是Java SPI机制，它在类ServiceLoader中，会使用线程上下文类加载器来逆向加载classpath中的第三方厂商提供的Service Provider类。（比如JDBC）


#### Tomcat


在Tomcat部署项目时，是把war包放到tomcat的webapp下，这就意味着一个tomcat可以运行多个Web应用程序。


假设现在有两个Web应用程序，它们都有一个类，叫User，并且它们的类全限定名都一样，比如都是com.yyy.User，但是他们的具体实现是不一样的。那么Tomcat如何保证它们不会冲突呢？


Tomcat给每个 Web 应用创建一个类加载器实例（WebAppClassLoader），该加载器重写了loadClass方法，优先加载当前应用目录下的类，如果当前找不到了，才一层一层往上找，这样就做到了Web应用层级的隔离。


但是并不是Web应用程序的所有依赖都需要隔离的，比如要用到Redis的话，Redis就可以再Web应用程序之间贡献，没必要每个Web应用程序每个都独自加载一份。因此Tomcat就在WebAppClassLoader上加个父加载器ShareClassLoader，如果WebAppClassLoader没有加载到这个类，就委托给ShareClassLoader去加载。（意思就类似于将需要共享的类放到一个共享目录下）


Web应用程序有类，但是Tomcat本身也有自己的类，为了隔绝这两个类，就用CatalinaClassLoader类加载器进行隔离，CatalinaClassLoader加载Tomcat本身的类


Tomcat与Web应用程序还有类需要共享，那就再用CommonClassLoader作为CatalinaClassLoader和ShareClassLoader的父类加载器，来加载他们之间的共享类


Tomcat加载结构图如下：


[![](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202404251622428.gif)](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202404251622428.gif)


#### JDBC


实际上JDBC定义了接口，具体的实现类是由各个厂商进行实现的(比如MySQL)


类加载有个规则：如果一个类由类加载器A加载，那么这个类的依赖类也是由「相同的类加载器」加载。


而在用JDBC的时候，是使用DriverManager获取Connection的，DriverManager是在java.sql包下的，显然是由BootStrap类加载器进行装载的。当使用DriverManager.getConnection ()时，需要得到的一定是对应厂商（如Mysql）实现的类。这里在去获取Connection的时候，是使用「线程上下文加载器」去加载Connection的，线程上下文加载器会直接指定对应的加载器去加载。也就是说，在BootStrap类加载器利用「线程上下文加载器」指定了对应的类的加载器去加载


[![](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202404251622217.gif)](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202404251622217.gif)


##### 线程上下文加载器


Java 提供了很多服务提供者接口（Service Provider Interface，SPI），允许第三方为这些接口提供实现。常见的 SPI 有 JDBC 。


这些 SPI 的接口由 Java 核心库来提供，而这些 SPI 的实现代码则是作为 Java 应用所依赖的 jar 包被包含进类路径（CLASSPATH）里。SPI接口中的代码经常需要加载具体的实现类。那么问题来了，SPI的接口是Java核心库的一部分，是由启动类加载器来加载的；SPI的实现类是由系统类加载器来加载的。启动类加载器是无法找到 SPI 的实现类的，因为它只加载 Java 的核心库。它也不能委派给系统类加载器，因为它是系统类加载器的祖先类加载器。


线程上下文类加载器正好解决了这个问题。如果不做任何的设置，Java 应用的线程的上下文类加载器默认就是系统上下文类加载器。在 SPI 接口的代码中使用线程上下文类加载器，就可以成功的加载到 SPI 实现的类。线程上下文类加载器在很多 SPI 的实现中都会用到。


线程上下文加载器的一般使用模式（获取 \- 使用 \- 还原）


java
```
ClassLoader calssLoader = Thread.currentThread().getContextClassLoader();
 
try {
    //设置线程上下文类加载器为自定义的加载器
    Thread.currentThread.setContextClassLoader(targetTccl);
    myMethod(); //执行自定义的方法
} finally {
    //还原线程上下文类加载器
    Thread.currentThread().setContextClassLoader(classLoader);
}
```

##### 源码实现


java
```
Connection conn = DriverManager.getConnection(url,username,password);
```

这就是普通的连接数据库的代码，可以直接获取数据库连接进行操作，这段代码没有了加载驱动的代码，那要怎么去确定使用哪个数据库连接的驱动呢？这里就涉及到使用Java的SPI扩展机制来查找相关驱动的东西了，关于驱动的查找其实都在DriverManager中，DriverManager是Java中的实现，用来获取数据库连接，在DriverManager中有一个静态代码块如下：


java
```
static {
    loadInitialDrivers();
    println("JDBC DriverManager initialized");
}
```

可以看到是加载实例化驱动的，接着看loadInitialDrivers方法：


java
```
private static void loadInitialDrivers() {
    String drivers;
    try {
        drivers = AccessController.doPrivileged(new PrivilegedAction() {
            public String run() {
                return System.getProperty("jdbc.drivers");
            }
        });
    } catch (Exception ex) {
        drivers = null;
    }

    AccessController.doPrivileged(new PrivilegedAction() {
        public Void run() {
            //使用SPI的ServiceLoader来加载接口的实现
            ServiceLoader loadedDrivers = ServiceLoader.load(Driver.class);
            Iterator driversIterator = loadedDrivers.iterator();
            try{
                while(driversIterator.hasNext()) {
                    driversIterator.next();
                }
            } catch(Throwable t) {
            // Do nothing
            }
            return null;
        }
    });

    println("DriverManager.initialize: jdbc.drivers = " + drivers);

    if (drivers == null || drivers.equals("")) {
        return;
    }
    String[] driversList = drivers.split(":");
    println("number of Drivers:" + driversList.length);
    for (String aDriver : driversList) {
        try {
            println("DriverManager.Initialize: loading " + aDriver);
            Class.forName(aDriver, true,
                    ClassLoader.getSystemClassLoader());
        } catch (Exception ex) {
            println("DriverManager.Initialize: load failed: " + ex);
        }
    }
}
```

上面的代码主要步骤是：


1. 从系统变量中获取有关驱动的定义。
2. 使用SPI来获取驱动的实现。
3. 遍历使用SPI获取到的具体实现，实例化各个实现类。
4. 根据第一步获取到的驱动列表来实例化具体实现类。


主要关注2,3步，这两步是SPI的用法，首先看第二步，使用SPI来获取驱动的实现，对应的代码是：


java
```
ServiceLoader loadedDrivers = ServiceLoader.load(Driver.class);
```

这里没有去META\-INF/services目录下查找配置文件，也没有加载具体实现类，做的事情就是封装了我们的接口类型和类加载器，并初始化了一个迭代器。


接着看第三步，遍历使用SPI获取到的具体实现，实例化各个实现类，对应的代码如下：


java
```
//获取迭代器
Iterator driversIterator = loadedDrivers.iterator();
//遍历所有的驱动实现
while(driversIterator.hasNext()) {
    driversIterator.next();
}
```

在遍历的时候，首先调用driversIterator.hasNext()方法，这里会搜索classpath下以及jar包中所有的META\-INF/services目录下的java.sql.Driver文件，并找到文件中的实现类的名字，此时并没有实例化具体的实现类（ServiceLoader具体的源码实现在下面）。


然后是调用driversIterator.next();方法，此时就会根据驱动名字具体实例化各个实现类了。现在驱动就被找到并实例化了。


可以看下截图，我在测试项目中添加了两个jar包，mysql\-connector\-java\-6\.0\.6\.jar和postgresql\-42\.0\.0\.0\.jar，跟踪到DriverManager中之后：


[![](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202404251622245.jpg)](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202404251622245.jpg)


可以看到此时迭代器中有两个驱动，mysql和postgresql的都被加载了。


## 自定义类加载器加载 java.lang.String


很多人都有个误区：双亲委派机制不能被打破，不能使用自定义类加载器加载java.lang.String


但是事实上并不是，只要重写ClassLoader的loadClass()方法，就能打破了。


java
```
import java.io.File;
import java.net.URL;
import java.net.URLClassLoader;

public class MyClassLoader extends URLClassLoader {

    public MyClassLoader(URL[] urls) {
        super(urls);
    }
    
    @Override
    public Class loadClass(String name) throws ClassNotFoundException {
        //只对MyClassLoader和String使用自定义的加载，其他的还是走双亲委派
        if(name.equals("MyClassLoader") || name.equals("java.lang.String")) {
            return super.findClass(name);
        } else {
            return getParent().loadClass(name);
        }
    }

    public static void main(String[] args) throws Exception {
        //urls指定自定义类加载器的加载路径
        URL url = new File("J:/apps/demo/target/classes/").toURI().toURL();
        URL url3 = new File("C:/Program Files/Java/jdk1.8.0_191/jre/lib/rt.jar").toURI().toURL();
        URL[] urls = {
                url
                , url3
        };
        MyClassLoader myClassLoader = new MyClassLoader(urls);

        Class c1 = MyClassLoader.class.getClassLoader().loadClass("MyClassLoader");
        Class c2 = myClassLoader.loadClass("MyClassLoader");
        System.out.println(c1 == c2); //false
        System.out.println(c1.getClassLoader()); //AppClassLoader
        System.out.println(c2.getClassLoader()); //MyClassLoader

        System.out.println(myClassLoader.loadClass("java.lang.String")); //Exception 
    }

}
```

加载同一个类MyClassLoader，使用的类加载器不同，说明这里是打破了双亲委派机制的，但是尝试加载String类的时候报错了


[![](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202404251622279.gif)](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202404251622279.gif)


看代码是ClassLoader类里面的限制，**只要加载java开头的包就会报错**。所以真正原因是**JVM安全机制**，并不是因为双亲委派。


那么既然是ClassLoader里面的代码做的限制，那把ClassLoader.class修改了不就好了吗。


写了个java.lang.ClassLoader，把preDefineClass()方法里那段if直接删掉，再用编译后的class替换rt.jar里面的,直接通过命令jar uvf rt.jar java/lang/ClassLoader/class即可。


不过事与愿违，修改之后还是报错:


java
```
Exception in thread "main" java.lang.SecurityException: Prohibited package name: java.lang
    at java.lang.ClassLoader.defineClass1(Native Method)
    at java.lang.ClassLoader.defineClass(ClassLoader.java:756)
    at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:142)
    at java.net.URLClassLoader.defineClass(URLClassLoader.java:468)
    at java.net.URLClassLoader.access$100(URLClassLoader.java:74)
    at java.net.URLClassLoader$1.run(URLClassLoader.java:369)
    at java.net.URLClassLoader$1.run(URLClassLoader.java:363)
    at java.security.AccessController.doPrivileged(Native Method)
    at java.net.URLClassLoader.findClass(URLClassLoader.java:362)
    at com.example.demo.mini.test.MyClassLoader.loadClass(MyClassLoader.java:17)
    at com.example.demo.mini.test.MyClassLoader.main(MyClassLoader.java:31)
```

仔细看报错和之前的不一样了，这次是native方法报错了。这就比较难整了，看来要自己重新编译个JVM才行了。理论上来说，编译JVM的时候把校验的代码去掉就行了。


结论：**自定义类加载器加载java.lang.String，必须修改jdk的源码，自己重新编译个JVM才行**。


## 总结


1. JDK中有三个默认类加载器：AppClassLoader、ExtClassLoader、BootStrapClassLoader。AppClassLoader的父加载器为Ext ClassLoader、Ext ClassLoader的父加载器为BootStrap ClassLoader。
2. 什么是双亲委派机制：加载器在加载过程中，先把类交由父加载器进行加载，父加载器没找到才由自身加载。
3. 双亲委派机制目的：为了防止内存中存在多份同样的字节码（安全）
4. 依赖传递原则：如果一个类由类加载器A加载，那么这个类的依赖类也是由「相同的类加载器」加载。
5. 什么时候需要打破双亲委派模型？比如类A已经有一个classA，恰好类B也有一个clasA 但是两者内容不一致，如果不打破双亲委派模型，那么类A只会加载一次
6. 如何打破双亲委派机制：


	* 自定义ClassLoader，重写loadClass方法（只要不依次往上交给父加载器进行加载，就算是打破双亲委派机制）（Tomcat）
	* 主动违背类加载器的依赖传递原则（JDBC）
7. 打破双亲委派机制案例：Tomcat


	* 为了Web应用程序类之间隔离，为每个应用程序创建WebAppClassLoader类加载器
	* 为了Web应用程序类之间共享，把ShareClassLoader作为WebAppClassLoader的父类加载器，如果WebAppClassLoader加载器找不到，则尝试用ShareClassLoader进行加载
	* 为了Tomcat本身与Web应用程序类隔离，用CatalinaClassLoader类加载器进行隔离，CatalinaClassLoader加载Tomcat本身的类
	* 为了Tomcat与Web应用程序类共享，用CommonClassLoader作为CatalinaClassLoader和ShareClassLoader的父类加载器
8. 线程上下文加载器：由于类加载的规则，很可能导致父加载器加载时依赖子加载器的类，导致无法加载成功（BootStrap ClassLoader无法加载第三方库的类），所以存在「线程上下文加载器」来进行加载。
9. 打破双亲委派机制案例：JDBC


	* JDBC的接口的具体的实现类是由各个厂商进行实现的(比如MySQL)，因此在用JDBC的时候，需要得到厂商实现的类。这里就使用「线程上下文加载器」，线程上下文加载器直接指定对应的加载器去加载对应厂商的具体的实现类。
10. 自定义类加载器加载java.lang.String，必须修改jdk的源码，自己重新编译个JVM才行。


## 面试题专栏


[Java面试题专栏](https://github.com)已上线，欢迎访问。


* 如果你不知道简历怎么写，简历项目不知道怎么包装；
* 如果简历中有些内容你不知道该不该写上去；
* 如果有些综合性问题你不知道怎么答；


那么可以私信我，我会尽我所能帮助你。


  * [类与类加载器](#%E7%B1%BB%E4%B8%8E%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8)
* [分类](#%E5%88%86%E7%B1%BB)
* [启动类加载器](#%E5%90%AF%E5%8A%A8%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8)
* [拓展类加载器](#%E6%8B%93%E5%B1%95%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8)
* [应用程序类加载器](#%E5%BA%94%E7%94%A8%E7%A8%8B%E5%BA%8F%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8)
* [类的加载方式](#%E7%B1%BB%E7%9A%84%E5%8A%A0%E8%BD%BD%E6%96%B9%E5%BC%8F)
* [JVM类加载机制](#jvm%E7%B1%BB%E5%8A%A0%E8%BD%BD%E6%9C%BA%E5%88%B6)
* [双亲委派模型](#%E5%8F%8C%E4%BA%B2%E5%A7%94%E6%B4%BE%E6%A8%A1%E5%9E%8B):[FlowerCloud机场](https://hushicha.org)
* [源码](#%E6%BA%90%E7%A0%81)
* [双亲委派模型目的](#%E5%8F%8C%E4%BA%B2%E5%A7%94%E6%B4%BE%E6%A8%A1%E5%9E%8B%E7%9B%AE%E7%9A%84)
* [依赖传递原则](#%E4%BE%9D%E8%B5%96%E4%BC%A0%E9%80%92%E5%8E%9F%E5%88%99)
* [打破双亲委派模型](#%E6%89%93%E7%A0%B4%E5%8F%8C%E4%BA%B2%E5%A7%94%E6%B4%BE%E6%A8%A1%E5%9E%8B)
* [Tomcat](#tomcat)
* [JDBC](#jdbc)
* [线程上下文加载器](#%E7%BA%BF%E7%A8%8B%E4%B8%8A%E4%B8%8B%E6%96%87%E5%8A%A0%E8%BD%BD%E5%99%A8)
* [源码实现](#%E6%BA%90%E7%A0%81%E5%AE%9E%E7%8E%B0)
* [自定义类加载器加载 java.lang.String](#%E8%87%AA%E5%AE%9A%E4%B9%89%E7%B1%BB%E5%8A%A0%E8%BD%BD%E5%99%A8%E5%8A%A0%E8%BD%BD-javalangstring)
* [总结](#%E6%80%BB%E7%BB%93)
* [面试题专栏](#%E9%9D%A2%E8%AF%95%E9%A2%98%E4%B8%93%E6%A0%8F)

   \_\_EOF\_\_

   ![](https://github.com/seven97-top)Seven  - **本文链接：** [https://github.com/seven97\-top/p/18523063](https://github.com)
 - **关于博主：** Seven的菜鸟成长之路
 - **版权声明：** 本博客所有文章除特别声明外，均采用 [BY\-NC\-SA](https://github.com "BY-NC-SA") 许可协议。转载请注明出处！
 - **声援博主：** 如果您觉得文章对您有帮助，可以点击文章右下角**【[推荐](javascript:void(0);)】**一下。
     
