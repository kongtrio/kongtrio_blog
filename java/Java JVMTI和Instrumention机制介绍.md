[TOC]

## 1、JVMTI 介绍 

JVMTI（JVM Tool Interface）是 **Java 虚拟机所提供的 native 编程接口**，是 JVMPI（Java Virtual Machine Profiler Interface）和 JVMDI（Java Virtual Machine Debug Interface）的替代版本。

JVMTI可以用来开发并监控虚拟机，可以查看JVM内部的状态，并控制JVM应用程序的执行。可实现的功能包括但不限于：调试、监控、线程分析、覆盖率分析工具等。

**另外，需要注意的是，并非所有的JVM实现都支持JVMTI。**

JVMTI只是一套接口，**我们要开发JVM工具就需要写一个Agent程序来使用这些接口**。Agent程序其实就是一个C/C++语言编写的动态链接库。这里不详细介绍如何开发一个JVMTI的agent程序。感兴趣的可以点击文章末尾的链接查看。

我们通过JVMTI开发好agent程序后，把程序编译成动态链接库，之后可以在jvm启动时指定加载运行该agent。

```
-agentlib:<agent-lib-name>=<options>
```

之后JVM启动后该agent程序就会开始工作。

### 1.1 Agent的工作形式

agent启动后是和JVM运行在同一个进程，大多agent的工作形式是作为服务端接收来自客户端的请求，然后根据请求命令调用JVMTI的相关接口再返回结果。

很多java监控、诊断工具都是基于这种形式来工作的。如果arthas、jinfo、brace等。

另外，我们熟知的java调试也是其实也是基于这种工作原理。

### 1.2 JDPA 相关介绍 

无论我们在开发调试时，都会用到调试工具。其实我们用的所有调试工具其底层都是基于JVMTI的调用。JVMTI本身就提供了关于调试程序的一系列接口，我们只需要编写agent就可以开发一套调试工具了。

虽然对应的接口已经有了，但是要基于这些接口开发一套完整的调试工具还是有一定工作量的。为了避免重复造轮子，sun公司定义了一套完整独立的调试体系，也就是JDPA。

JDPA由3个模块组成：

1. JVMTI，即底层的相关调试接口调用。sun公司提供了一个 jdwp.dll( jdwp.so)动态链接库，就是我们上面说的agent实现。
2. JDWP（Java Debug Wire Protocol）,定义了agent和调试客户端之间的通讯交互协议。
3. JDI（Java Debug Interface），是由Java语言实现的。有了这套接口，我们就可以直接使用java开发一套自己的调试工具。 

![](http://assets.processon.com/chart_image/5c832caae4b0ed6b42fa2018.png?_=1552100770444)

其实有了jdwp Agent以及知道了交互的消息协议格式，我们就可以基于这些开发一套调试工具了。但是相对还是比较费时费力，所以才有了JDI的诞生，JDI是一套JAVA API。这样对于不熟悉C/C++的java程序员也能开发自己的调试工具了。

> **另外，JDI 不仅能帮助开发人员格式化 JDWP 数据，而且还能为 JDWP 数据传输提供队列、缓存等优化服务**

再回头看一下启动JVM debug时需要带上的参数:

```shell
java -Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=8000 -jar test.jar 
```

 jdwp.dll作为一个jvm内置的agent，不需要上文说的-agentlib来启动agent。这里通过`-Xrunjdwp`来启动该agent。后面还指定了一些参数：

- transport=dt_socket，表示用监听socket端口的方式来建立连接，这里也可以选择dt_shmem共享内存方式，但限于windows机器，并且服务端和客户端位于一台机器上
- server=y 表示当前是调试服务端，=n表示当前是调试客户端
- suspend=n 表示启动时不中断（如果启动时中断，一般用于调试启动不了的问题）
- address=8000 表示本地监听8000端口

## 2、Instrumention 机制

虽然java提供了JVMTI，但是对应的agent需要用C/C++开发，对java开发者而言并不是非常友好。因此在Java SE 5的新特性中加入了Instrumentation机制。有了 Instrumentation，开发者可以构建一个基于Java编写的Agent来监控或者操作JVM了，比如替换或者修改某些类的定义等。

### 2.1 Instrumention支持的功能

Instrumention支持的功能都在`java.lang.instrument.Instrumentation`接口中体现:

```java
public interface Instrumentation {
    //添加一个ClassFileTransformer
    //之后类加载时都会经过这个ClassFileTransformer转换
    void addTransformer(ClassFileTransformer transformer, boolean canRetransform);

    void addTransformer(ClassFileTransformer transformer);
	//移除ClassFileTransformer
    boolean removeTransformer(ClassFileTransformer transformer);

    boolean isRetransformClassesSupported();
	//将一些已经加载过的类重新拿出来经过注册好的ClassFileTransformer转换
    //retransformation可以修改方法体，但是不能变更方法签名、增加和删除方法/类的成员属性
    void retransformClasses(Class<?>... classes) throws UnmodifiableClassException;

    boolean isRedefineClassesSupported();

    //重新定义某个类
    void redefineClasses(ClassDefinition... definitions)
        throws  ClassNotFoundException, UnmodifiableClassException;

    boolean isModifiableClass(Class<?> theClass);

    @SuppressWarnings("rawtypes")
    Class[] getAllLoadedClasses();

    @SuppressWarnings("rawtypes")
    Class[] getInitiatedClasses(ClassLoader loader);

    long getObjectSize(Object objectToSize);

    void appendToBootstrapClassLoaderSearch(JarFile jarfile);

    void appendToSystemClassLoaderSearch(JarFile jarfile);

    boolean isNativeMethodPrefixSupported();

    void setNativeMethodPrefix(ClassFileTransformer transformer, String prefix);
}
```

我们通过addTransformer方法注册了一个ClassFileTransformer，后面类加载的时候都会经过这个Transformer处理。**对于已加载过的类，可以调用retransformClasses来重新触发这个Transformer的转换**。

ClassFileTransformer可以判断是否需要修改类定义并根据自己的代码规则修改类定义然后返回给JVM。利用这个Transformer类，我们可以很好的实现虚拟机层面的AOP。

> redefineClasses 和 retransformClasses 的区别：
>
> 1. transform是对类的byte流进行读取转换的过程，需要先获取类的byte流然后做修改。而redefineClasses更简单粗暴一些，**它需要直接给出新的类byte流，然后替换旧的**。
> 2. transform可以添加很多个，retransformClasses 可以让指定的类重新经过这些transform做转换。

### 2.2 基于Instrumention开发一个Agent

利用java.lang.instrument包下面的相关类，我们可以开发一个自己的Agent程序。

#### 2.2.1 编写premain函数

编写一个java类，不用继承或者实现任何类，直接实现下面两个方法中的任一方法:

```java
//agentArgs是一个字符串，会随着jvm启动设置的参数得到
//inst就是我们需要的Instrumention实例了，由JVM传入。我们可以拿到这个实例后进行各种操作
public static void premain(String agentArgs, Instrumentation inst);  [1]
public static void premain(String agentArgs); [2]
```

其中，**[1] 的优先级比 [2] 高，将会被优先执行，[1] 和 [2] 同时存在时，[2] 被忽略。**

编写一个PreMain:

```java
public class PreMain {

    public static void premain(String agentArgs, Instrumentation inst) throws ClassNotFoundException,
            UnmodifiableClassException {
        inst.addTransformer(new MyTransform());
    }
}
```

MyTransform是我们自己定义的一个ClassFileTransformer实现类，这个类遇到`com/yjb/Test`类，就会进行类定义转换。

```java
public class MyTransform implements ClassFileTransformer {

    public static final String classNumberReturns2 = "/tmp/Test.class";

    public static byte[] getBytesFromFile(String fileName) {
        try {
            // precondition
            File file = new File(fileName);
            InputStream is = new FileInputStream(file);
            long length = file.length();
            byte[] bytes = new byte[(int) length];

            // Read in the bytes
            int offset = 0;
            int numRead = 0;
            while (offset < bytes.length
                    && (numRead = is.read(bytes, offset, bytes.length - offset)) >= 0) {
                offset += numRead;
            }

            if (offset < bytes.length) {
                throw new IOException("Could not completely read file "
                        + file.getName());
            }
            is.close();
            return bytes;
        } catch (Exception e) {
            System.out.println("error occurs in _ClassTransformer!"
                    + e.getClass().getName());
            return null;
        }
    }

    /**
     * 参数：
     * loader - 定义要转换的类加载器；如果是引导加载器，则为 null
     * className - 完全限定类内部形式的类名称和 The Java Virtual Machine Specification 中定义的接口名称。例如，"java/util/List"。
     * classBeingRedefined - 如果是被重定义或重转换触发，则为重定义或重转换的类；如果是类加载，则为 null
     * protectionDomain - 要定义或重定义的类的保护域
     * classfileBuffer - 类文件格式的输入字节缓冲区（不得修改）
     * 返回：
     * 一个格式良好的类文件缓冲区（转换的结果），如果未执行转换,则返回 null。
     * 抛出：
     * IllegalClassFormatException - 如果输入不表示一个格式良好的类文件
     */
    public byte[] transform(ClassLoader l, String className, Class<?> c,
                            ProtectionDomain pd, byte[] b) throws IllegalClassFormatException {
        System.out.println("transform class-------" + className);
        if (!className.equals("com/yjb/Test")) {
            return null;
        }
        return getBytesFromFile(targetClassPath);
    }
}
```

#### 2.2.2 打成jar包

之后我们把上面两个类打成一个jar包，并在其中的**META-INF/MAINIFEST.MF属性当中加入” Premain-Class”来指定成**上面的PreMain类。

我们可以用maven插件来做到自动打包并写MAINIFEST.MF:

```xml
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <executions>
                    <execution>
                        <goals>
                            <goal>single</goal>
                        </goals>
                        <phase>package</phase>

                        <configuration>
                            <descriptorRefs>
                                <descriptorRef>jar-with-dependencies</descriptorRef>
                            </descriptorRefs>
                            <archive>
                                <manifestEntries>
                                    <Premain-Class>com.yjb.PreMain</Premain-Class>
                                    <Can-Redefine-Classes>true</Can-Redefine-Classes>
                                    <Can-Retransform-Classes>true</Can-Retransform-Classes>
                                    <Specification-Title>${project.name}</Specification-Title>
                                    <Specification-Version>${project.version}</Specification-Version>
                                    <Implementation-Title>${project.name}</Implementation-Title>
                                    <Implementation-Version>${project.version}</Implementation-Version>
                                </manifestEntries>
                            </archive>
                        </configuration>
                    </execution>
                </executions>
            </plugin>

```

#### 2.2.3 编写测试类 

上面的agent会转换`com/yjb/Test`类，我们就编写一个Test类进行测试。

```java
public class Test {

    public void print() {
        System.out.println("A");
    }
}
```

先编译这个类，然后把Test.class 放到 /tmp 下。

之后再修改这个类：

```java
public class Test {

    public void print() {
        System.out.println("B");
    }
    
    public static void main(String[] args) throws InterruptedException {
        new Test().print();
    }
}
```

**之后运行时指定加上JVM参数 `-javaagent:/toPath/agent-jar-with-dependencies.jar` 就会发现Test已经被转换了**。

### 2.3 如何在运行时加载agent

上面开发的agent需要启动就必须在jvm启动时设置参数，但很多时候我们想要在程序运行时中途插入一个agent运行。在Java 6的新特性中，就可以通过Attach的方式去加载一个agent了。

关于Attach的机制原理可以看我的这篇博客：

https://blog.csdn.net/u013332124/article/details/88362317

使用这种方式加载的agent启动类需要实现这两种方法中的一种：

```java
public static void agentmain (String agentArgs, Instrumentation inst); [1] 
public static void agentmain (String agentArgs);[2]
```

和premain一样，[1] 比 [2] 的优先级高。

之后要在**META-INF/MAINIFEST.MF属性当中加入” AgentMain-Class”来指定目标启动类**。

我们可以在上面的agent项目中加入一个AgentMain类

```java
public class AgentMain {

    public static void agentmain(String agentArgs, Instrumentation inst) throws ClassNotFoundException,
            UnmodifiableClassException, InterruptedException {
        //这里的Transform还是使用上面定义的那个
        inst.addTransformer(new MyTransform(), true);
        //由于是在运行中才加入了Transform，因此需要重新retransformClasses一下
        Class<?> aClass = Class.forName("com.yjb.Test");
        inst.retransformClasses(aClass);
        System.out.println("Agent Main Done");
    }
}
```

还是把项目打包成`agent-jar-with-dependencies.jar`。

之后再编写一个类去attach目标进程并加载这个agent

```java
public class AgentMainStarter {

    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException,
            AgentInitializationException {
                //这个pid填写具体要attach的目标进程
        VirtualMachine attach = VirtualMachine.attach("pid");
        attach.loadAgent("/toPath/agent-jar-with-dependencies.jar");
        attach.detach();
        System.out.println("over");
    }
}
```

之后修改一下Test类，让他不断运行下去

```java
public class Test {

    private void print() {
        System.out.println("1111");
    }

    public static void main(String[] args) throws InterruptedException {
        Test test = new Test();
        while (true) {
            test.print();
            Thread.sleep(1000L);
        }
    }
}
```

运行Test一段时间后，再运行AgentMainStarter类，会发现输出变成了最早编译的那个/tmp/Test.class下面的"A"了。说明我们的agent进程已经在目标JVM成功运行。

## 3、参考资料

[Java Attach机制简介](https://blog.csdn.net/u013332124/article/details/88362317)

[基于Java Instrument的Agent实现](https://www.jianshu.com/p/b72f66da679f)

[IBM: Instrumentation 新功能](https://www.ibm.com/developerworks/cn/java/j-lo-jse61/index.html)

[Instrumentation 中redefineClasses 和 retransformClasses 的区别](https://stackoverflow.com/questions/19009583/difference-between-redefine-and-retransform-in-javaagent)

[JVMTI开发文档](http://blog.caoxudong.info/blog/2017/12/07/jvmti_reference)

[JVMTI oracle 官方文档](https://docs.oracle.com/javase/8/docs/platform/jvmti/jvmti.html#writingAgents)

[JVMTI和JDPA介绍](https://www.jianshu.com/p/90ad6a8960fe)