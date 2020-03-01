[TOC]

在JVM运行时，我们经常需要获取目标JVM运行时的相关信息。最典型的一个场景就是通过jstack命令输出当前的线程dump。

对于这种场景，java提供了attach机制。通过attach机制，我们可以直接attach到目标JVM进程，然后进行一些操作，比如获取内存dump、线程dump，类信息统计(比如已加载的类以及实例个数等)，动态加载agent，动态设置vm flag(但是并不是所有的flag都可以设置的，因为有些flag是在jvm启动过程中使用的，是一次性的)，打印vm flag，获取系统属性等。

## 一、利用Attach机制实现一个简单的Jstack

在sun 包的`com.sun.tools.attach`下，有一系列和Attach相关的类，要attach到目标的JVM代码也很简单。下面我们利用Attach机制实现一个简单的Jstack：

```Java
    public static void main(String[] args) throws IOException, AttachNotSupportedException {
        VirtualMachine attach = VirtualMachine.attach("pid");
        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        InputStream in = ((HotSpotVirtualMachine) attach).remoteDataDump((Object[]) args);
        byte b[] = new byte[256];
        int n = 0;
        do {
            n = in.read(b);
            if (n > 0) {
                System.out.println(new String(b, 0, n));
            }
        } while (n > 0);
        in.close();
        attach.detach();
    }
```

## 二、Attach实现原理  

attach机制的实现涉及到了进程间的通信。**那么Attach机制是如何让两个JVM进程之间可以正常通信呢**？

经常通过jstack查看线程dump的同学可能会留意到下面这个两个线程：

```
"Attach Listener" #10 daemon prio=9 os_prio=31 tid=0x00007fef2283a000 nid=0x4b03 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
"Signal Dispatcher" #4 daemon prio=9 os_prio=31 tid=0x00007fef22030000 nid=0x3907 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE
```

这两个线程都是JVM线程。其中每个JVM都会有`Signal Dispatcher`线程，用于处理信号。`Attach Listener`线程用于JVM进程间的通信，但是它不一定会启动，启动它有两种方式。

### Attach Listener线程的启动

#### 1. 启动的时候通过jvm参数指定启动该线程。  

主要涉及的参数有  

| JVM参数                | 默认值 |
| ---------------------- | ------ |
| DisableAttachMechanism | false  |
| StartAttachListener    | false  |
| ReduceSignalUsage      | false  |

   JVM启动Attach Listener线程的代码如下  :

   ```c++
   if (!DisableAttachMechanism) {
       if (StartAttachListener || AttachListener::init_at_startup()) {
         AttachListener::init();
       }
   }
   bool AttachListener::init_at_startup() {
     if (ReduceSignalUsage) {
       return true;
     } else {
       return false;
     }
   }
   ```

`java -XX:+StartAttachListener mainClass`即可。 

#### 2. attach目标JVM成功后，目标JVM启动该线程。

如果不在启动jvm的时候启动Attach Listener线程，那只能依靠`Signal Dispatcher`线程来启动了。我们可以看一下VirtualMachine.attach(pid)的实现

```java
    public static VirtualMachine attach(String pid) throws AttachNotSupportedException, IOException {
        if (pid == null) {
            throw new NullPointerException("id cannot be null");
        } else {
            List providerList = AttachProvider.providers();
            if (providerList.size() == 0) {
                throw new AttachNotSupportedException("no providers installed");
            } else {
                AttachNotSupportedException ex = null;
                Iterator iterator = var1.iterator();

                while(iterator.hasNext()) {
                    AttachProvider provider = (AttachProvider)iterator.next();

                    try {
                        return provider.attachVirtualMachine(pid);
                    } catch (AttachNotSupportedException ex2) {
                        ex = ex2;
                    }
                }

                throw ex;
            }
        }
    }
```

上面的代码回去获取所有的AttachProvider，这里包含了各个系统的AttachProvider实现。之后遍历这个providerList，直到匹配上。如果是在linux下，最终会调用LinuxAttachProvider，然后返回一个LinuxVirtualMachine实例。我们看一下LinuxVirtualMachine的构造方法：

```java
LinuxVirtualMachine(AttachProvider provider, String vmid)
        throws AttachNotSupportedException, IOException
    {
        super(provider, vmid);

        int pid;
        try {
            pid = Integer.parseInt(vmid);
        } catch (NumberFormatException x) {
            throw new AttachNotSupportedException("Invalid process identifier");
        }

		//尝试获取socketFile，如果没获取到，就准备往目标JVM发送信号
        path = findSocketFile(pid);
        if (path == null) {
        	//创建attach文件 /proc/<pid>/cwd/.attach_pid<pid> 
            File f = createAttachFile(pid);
            try {
				//如果是linux，由于linux线程的实现是轻进程，因此需要往它的所有子进程发送信号
				//如果不是linux，直接往目标进程发送sigquit信号即可
                if (isLinuxThreads) {
                    int mpid;
                    try {
                        mpid = getLinuxThreadsManager(pid);
                    } catch (IOException x) {
                        throw new AttachNotSupportedException(x.getMessage());
                    }
                    assert(mpid >= 1);
                    sendQuitToChildrenOf(mpid);
                } else {
                    sendQuitTo(pid);
                }

                int i = 0;
                long delay = 200;
                int retries = (int)(AttachTimeout() / delay);
                do {
                    try {
                        Thread.sleep(delay);
                    } catch (InterruptedException x) { }
                    //开始轮询等待目标JVM建立socketFile
                    path = findSocketFile(pid);
                    i++;
                } while (i <= retries && path == null);
                if (path == null) {
                    throw new AttachNotSupportedException(
                        "Unable to open socket file: target process not responding " +
                        "or HotSpot VM not loaded");
                }
            } finally {
                f.delete();
            }
        }

        //这里需要检查目标JVM创建的socketFile我们是否有权限访问
        checkPermissions(path);

        //检查我们是否有权限连接到目标进程
        int s = socket();
        try {
            connect(s, path);
        } finally {
            close(s);
        }
    }
```

在这个方法里面，会先判断对应目录下有没有socketFile，如果没有，则往目标进程发送sigquit信号。目标JVM进程收到sigquit信号后，主要由`Signal Dispatcher`线程处理。Signal Dispatcher线程判断出信号是sigquit时，就会启动Attach Listener线程。

> 在linux中，线程是用进程的方式来实现的，也就是轻进程。因此，要发sigquit信号到Signal Dispatcher线程，就需要往该JVM进程的所有子进程发送sigquit信号。而JVM里除了vm thread，其他线程都设置了对此信号的屏蔽，因此收不到该信号。

### Attach Listener线程工作原理 

Attach Listener线程启动后，就会创建一个监听套接字，并创建了一个文件/tmp/.java_pid<pid>，**这个就是LinuxVirtualMachine构造函数中一直尝试获取的socketFile**。随着这个socketFile创建，也就意味着客户端那边的attach成功了。

**之后客户端和目标JVM进程就通过这个socketFile进行通信**。客户端可以通过这个socketFile发送相关命令。Attach Listener线程做的事情就是监听这个socketFile，发现有请求就解析，然后根据命令执行不同的方法，最后将结果返回。

> **Unix domain socket 又叫 IPC(inter-process communication 进程间通信) socket，用于实现同一主机上的进程间通信。**socket 原本是为网络通讯设计的，但后来在 socket 的框架上发展出一种 IPC 机制，就是 UNIX domain socket。虽然网络 socket 也可用于同一台主机的进程间通讯(通过 loopback 地址 127.0.0.1)，但是 UNIX domain socket 用于 IPC 更有效率：不需要经过网络协议栈，不需要打包拆包、计算校验和、维护序号和应答等，只是将应用层数据从一个进程拷贝到另一个进程。这是因为，IPC 机制本质上是可靠的通讯，而网络协议是为不可靠的通讯设计的。

## 参考资料

[JVM Attach机制实现](http://ifeve.com/jvm-attach/)

[Unix domain socket维基百科](https://en.wikipedia.org/wiki/Unix_domain_socket)

[Unix domain socket简介](https://www.cnblogs.com/sparkdev/p/8359028.html)