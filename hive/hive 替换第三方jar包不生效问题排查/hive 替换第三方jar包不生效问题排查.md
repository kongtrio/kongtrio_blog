[TOC]

**想直接看结论和解决方案的同学可以直接跳到2.4章节**。

## 一、问题描述  

前几天有用户反馈将hive查询结果以orc导入到hdfs目录时出现异常，sql大概如下:

```sql
insert overwrite directory '/tmp/' stored as orc
select * from x_table;
```

异常显示是在查询结束要写入orc文件时出现NullPointException。

对这个问题感兴趣的同学可以看我的这篇博客：

[hive insert overwrite directory 问题排查](https://blog.csdn.net/u013332124/article/details/86602511)

简单的做了几次模拟，发现有些数据可以，比如说把"*"号改成某些字段。很自然的觉的可能是数据的问题，所以最开始的想法是在orc的出错的源码周围打点日志，顺便判断以为空就跳过不处理同时把那条数据打印出来。**于是从github上下载了orc的源码，开始改代码，然后编译打包，放到测试机器的`hive/lib`下。最后再执行出错语句时，发现异常信息还是一样，连出错的行数都没变化，日志也没输出。这就s说明`orc-core.jar`替换没生效，hive运行时没有加载这个jar**。

所以问题总结就是：

- 替换了hive/lib下面的jar包，重启hive后新的jar包还是没生效的问题。

## 二、排查过程  

**备注：我们的hive是跑在yarn上面的，一般是用spark作为执行引擎**。

### 1. 梳理hive程序的执行流程  

我们都知道，hive在执行sql的时候，会解析语法树，然后构造执行计划，接着做一些优化，最后会生成spark或者mapred job执行任务。所以我们hive任务是在yarn上的各个container上执行的。由于我们的执行引擎设置的是spark，所以最好先了解一下hive是怎么提交生成的spark job到yarn上面执行的。

看了一圈hive源码后，大概理通了spark的提交流程。

如果我们在配置文件中指定了`spark.master=yarn`,hive在真正执行任务的时候会通过`spark-submit`往yarn提交spark job。我们可以在hive.log的日志中看到以下日志:

```
2019-01-22T11:20:54,268  INFO [Thread-23] spark.client.SparkClientImpl:469 Running client driver with argv: /www/spark/bin/spark-submit --executor-cores 2 --executor-memory 6g --properties-file /tmp/spark-submit.7734574653075860835.properties --class org.apache.hive.spark
.client.RemoteDriver /www/apache-hive-2.3.3-bin/lib/hive-exec-2.3.3-yangjb-0121.1.jar --remote-host dn162 --remote-port 13719 --conf hive.spark.client.connect.timeout=10800000 --conf hive.spark.client.server.connect.timeout=36000000 --conf hive.spark.client.channel.log.le
vel=null --conf hive.spark.client.rpc.max.size=52428800 --conf hive.spark.client.rpc.threads=8 --conf hive.spark.client.secret.bits=256 --conf hive.spark.client.rpc.server.address=null
```

了解spark的同学应该很容易看的出来，这就是标准的spark往yarn提交任务的命令。

> 插个题外话，稍微介绍一下hive运行spark job的流程：
>
> 往yarn提交了一个任务执行RemoteDriver这个类的main方法，这个类会在集群中以客户端的方式去连接hive这边启动的服务端rpc。之后hive进程和集群中的RemoteDriver就通过netty rpc的方式来沟通，包括发送执行的spark work等。

### 2. 推测问题产生的原因

知道hive是如何执行的后，就可以开始查找为什么orc.jar包不生效了。既然是在yarn的container上面执行spark job，我们就要看下**orc的jar包是在yarn直接指定的classpath里面，还是提交spark job时直接指定classpath中**。

**我首先先搜了一圈yarn对应的所有classpath，甚至连java的lib都看了，发现并没有orc包**。

接着看spark提交job时执行的配置文件`/tmp/spark-submit.7734574653075860835.properties`。发现有一行配置,应该就是spark job执行时指定的classpath了。

```properties
spark.yarn.jars hdfs://lrtscluster/data/spark2_2_2/*
```

但是去这个hdfs目录下找了一圈，发现也没有对应的orc包。这时候就很蒙圈了，既然都没有orc包，那hive任务运行时是如何加载orc的类的呢。（这里可以百分比确认出错的类是orc包下面的类）

后面又看了几圈yarn和hive、spark的配置，都没有发现有哪个地方可能有这个orc包，所以只能动用工具来排查这个类到底出自哪个jar包的了。

### 3. 通过arthas找出罪魁祸首  

arthas是一款jvm相关问题排查工具，arthas的使用不在这里介绍，不了解的同学可以看我的这篇博客：

[JVM进程诊断利器——arthas介绍](https://mp.csdn.net/postedit/84888074)

主要流程就是观察到spark在某个container上执行时，立马将arthas安装到目标机器上，然后attach到该container进程中排查。

最后通过命令排查到所有已加载的orc类

```shell
sc -d *.orc.impl.* 
```

发现加载这些类的jar包竟然是一个叫/tmp/xxx/\_\_app.jar\_\_的类。**yarn要运行job时，会收集相关的信息到一个临时目录，里面包括跑job要依赖的jar包等**。这个`\_\_app.jar\_\_`包就在这个临时目录下。最后在进入这个目录看，发现这个包竟然是`hive-exec-xxx.jar`的一个软连接。

突然之间恍然大悟，跑去看了下hive的ql模块的pom文件，发现以下这些代码:

```xml
 <plugin>                                                                              
   <groupId>org.apache.maven.plugins</groupId>                                         
   <artifactId>maven-shade-plugin</artifactId>                                         
   <executions>                                                                        
     <execution>                                                                       
       <id>build-exec-bundle</id>                                                      
       <phase>package</phase>                                                          
       <goals>                                                                         
         <goal>shade</goal>                                                            
       </goals>                                                                        
       <configuration>                                                                 
           <!--also see maven-jar-plugin execution.id=core-jar-->                      
         <artifactSet>                                                                 
           <includes>                                                                  
             <!-- order is meant to be the same as the ant build -->                   
             <include>org.apache.hive:hive-common</include>                            
             <include>org.apache.hive:hive-exec</include>                              
             <include>org.apache.hive:hive-serde</include>                             
             <include>org.apache.hive:hive-llap-common</include>                       
             <include>org.apache.hive:hive-llap-client</include>                       
             <include>org.apache.hive:hive-metastore</include>                         
             <include>org.apache.hive:hive-service-rpc</include>                       
             <include>com.esotericsoftware:kryo-shaded</include>                       
	  <include>com.esotericsoftware:minlog</include>                                    
	  <include>org.objenesis:objenesis</include>                                        
             <include>org.apache.parquet:parquet-hadoop-bundle</include>               
             <include>org.apache.thrift:libthrift</include>                            
             <include>org.apache.thrift:libfb303</include>                             
             <include>javax.jdo:jdo-api</include>                                      
             <include>commons-lang:commons-lang</include>                              
             <include>org.apache.commons:commons-lang3</include>                       
             <include>org.jodd:jodd-core</include>                                     
             <include>com.tdunning:json</include>                                      
             <include>org.apache.avro:avro</include>                                   
             <include>org.apache.avro:avro-mapred</include>                            
             <include>org.apache.hive.shims:hive-shims-0.23</include>                  
             <include>org.apache.hive.shims:hive-shims-0.23</include>                  
             <include>org.apache.hive.shims:hive-shims-common</include>                
             <include>com.googlecode.javaewah:JavaEWAH</include>                       
             <include>javolution:javolution</include>                                  
             <include>com.google.protobuf:protobuf-java</include>                      
             <include>io.airlift:aircompressor</include>                               
             <include>org.codehaus.jackson:jackson-core-asl</include>                  
             <include>org.codehaus.jackson:jackson-mapper-asl</include>                
             <include>com.google.guava:guava</include>                                 
             <include>net.sf.opencsv:opencsv</include>                                 
             <include>org.apache.hive:spark-client</include>                           
             <include>org.apache.hive:hive-storage-api</include>                       
             <include>org.apache.orc:orc-core</include>                                
             <include>org.apache.orc:orc-tools</include>                               
             <include>joda-time:joda-time</include>                                    
           </includes>                                                                 
         </artifactSet>                                                                
         <relocations>                                                                 
           <relocation>                                                                
             <pattern>com.esotericsoftware</pattern>                                   
             <shadedPattern>org.apache.hive.com.esotericsoftware</shadedPattern>       
           </relocation>                                                               
           <relocation>                                                                
             <pattern>org.objenesis</pattern>                                          
             <shadedPattern>org.apache.hive.org.objenesis</shadedPattern>              
           </relocation>                                                               
         </relocations>                                                                
       </configuration>                                                                
     </execution>                                                                      
   </executions>                                                                       
 </plugin>                                                                             
```

**上面的代码会将指定的依赖包打进`hive-exec-xx.jar`里面**。

### 4. 问题总结和解决办法  

**hive在打包`hive-exec-xx.jar`时会将一些相关的依赖打进去，也就是FatJar，这就导致我们更换`hive/lib`的一些第三方jar包不生效的问题**。

要解决这个问题，个人认为有两种办法：

1. 如果就涉及一些类的话，直接编译完这些类后，替换jar包里面的指定类。这个通过zip命令可以实现对jar包文件的增删改。
2. 自己编译一个新的版本，然后改一下hive的源码，版本改成自己编译的，之后再编译一遍hive

## 三、个人反思和一些总结  

1. 对hive的了解还不够深入。如果足够熟悉hive的话，这应该是个很简单的问题，要能快速解决才对，花了几个小时排查不太应该
2. 个人认为hive打包的方式有问题，不是说不应该打FatJar，而是既然打了FatJar，又将那些第三方jar包放进lib中，导致开发或运维以为可以直接更换第三方jar包。——其实把这些第三方jar包都删了也不会影响hive运行