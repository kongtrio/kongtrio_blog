[TOC]

## 一、_SUCCESS的作用和实现

我们在跑完mr或者spark程序时，会发现数据输出目录一般都会有一个\_SUCCESS的空文件。**这个\_SUCCESS空文件用来表示该任务运行成功**。

举个例子，比如我们有两个spark任务：A任务和B任务。B任务依赖于A任务，也就是说B任务要根据A任务的结果再判断是否运行。这时我们就可以根据A任务的输出目录下是否有\_SUCCESS文件来判断A任务是否运行成功。

### 1、 输出 _SUCCESS 文件的代码实现

在hadoop的代码中，有一个抽象类`OutputCommitter`，专门负责执行Job的各个生命周期应该执行的逻辑：

```java
public abstract class OutputCommitter {
  public abstract void setupJob(JobContext jobContext) throws IOException;

  @Deprecated
  public void cleanupJob(JobContext jobContext) throws IOException { }

  public void commitJob(JobContext jobContext) throws IOException {
    cleanupJob(jobContext);
  }

  public void abortJob(JobContext jobContext, JobStatus.State state) 
  throws IOException {
    cleanupJob(jobContext);
  }
  
  public abstract void setupTask(TaskAttemptContext taskContext)
  throws IOException;
  
  public abstract boolean needsTaskCommit(TaskAttemptContext taskContext)
  throws IOException;

  public abstract void commitTask(TaskAttemptContext taskContext)
  throws IOException;
  
  public abstract void abortTask(TaskAttemptContext taskContext)
  throws IOException;

  @Deprecated
  public boolean isRecoverySupported() {
    return false;
  }

  public boolean isCommitJobRepeatable(JobContext jobContext)
      throws IOException {
    return false;
  }

  public boolean isRecoverySupported(JobContext jobContext) throws IOException {
    return isRecoverySupported();
  }

  public void recoverTask(TaskAttemptContext taskContext)
  throws IOException
  {}
}
```

主要包括：

1. 启动一个Job时 —— setupJob()
2. 启动一个Task时 —— setupTask()
3. 停止一个Task时 —— abortTask()
4. 停止一个Job时 —— abortJob()
5. commit 一个Job时(Job运行完之后的操作) —— commitJob()
6. 恢复一个Task —— recoverTask()

我们写mr代码时，可以通过参数来设置这个具体的实现类，从而来操控这些生命周期的具体行为

```java
conf.set("mapred.output.committer.class","自定义实现类");
```

如果没有人为设置，hadoop将默认使用`org.apache.hadoop.mapreduce.lib.output.FileOutputCommitter`作为具体的实现。

因为_SUCCESS文件是在完成时输出的，所以我们可以看一下FileOutputCommitter#commit()方法:

```java
//FileOutputCommitter.java
  public void commitJob(JobContext context) throws IOException {
    int maxAttemptsOnFailure = isCommitJobRepeatable(context) ?
        context.getConfiguration().getInt(FILEOUTPUTCOMMITTER_FAILURE_ATTEMPTS,
            FILEOUTPUTCOMMITTER_FAILURE_ATTEMPTS_DEFAULT) : 1;
    int attempt = 0;
    boolean jobCommitNotFinished = true;
    while (jobCommitNotFinished) {
      try {
        commitJobInternal(context);
        jobCommitNotFinished = false;
      } catch (Exception e) {
        if (++attempt >= maxAttemptsOnFailure) {
          throw e;
        } else {
          LOG.warn("Exception get thrown in job commit, retry (" + attempt +
              ") time.", e);
        }
      }
    }
      
    protected void commitJobInternal(JobContext context) throws IOException {
    if (hasOutputPath()) {
      Path finalOutput = getOutputPath();
      FileSystem fs = finalOutput.getFileSystem(context.getConfiguration());

      if (algorithmVersion == 1) {
        for (FileStatus stat: getAllCommittedTaskPaths(context)) {
          mergePaths(fs, stat, finalOutput);
        }
      }

      cleanupJob(context);
      //判断是否需要输出_SUCCESS
      if (context.getConfiguration().getBoolean(
          SUCCESSFUL_JOB_OUTPUT_DIR_MARKER, true)) {
          //_SUCCESS文件绝对路径
        Path markerPath = new Path(outputPath, SUCCEEDED_FILE_NAME);
        //判断是否可重复提交
        if (isCommitJobRepeatable(context)) {
            //创建_SUCCESS文件
          fs.create(markerPath, true).close();
        } else {
          fs.create(markerPath).close();
        }
      }
    } else {
      LOG.warn("Output Path is null in commitJob()");
    }
  }
```

从代码中可以看到，hadoop会根据参数`mapreduce.fileoutputcommitter.marksuccessfuljobs`的值来判断是否要生成_SUCCESS文件。因为这个参数的默认值是true，所以我们平常跑的任务基本都会输出\_SUCCESS。

> spark的任务也复用了这个OutputCommitter的机制。所以spark任务输出_SUCCESS文件的流程和上面一样。spark任务默认的实现类也是FileOutputCommitter
>
> 具体的可以看这篇博客：
>
> <https://blog.csdn.net/u013332124/article/details/92001346>

## 二、Hive任务导出数据时没生成_SUCCESS的原因

我们发现通过Hive导出数据时，对应的目录并没有生成_SUCCESS文件:

```sql
# 无论是往目录输出数据还是往某张表输出数据都不会生成_SUCCESS文件
insert overwrite directory '/tmp' select * from test;
insert into test1 select * from test;
```

谷歌搜了下，发现原来是Hive自己实现了一个NullOutputCommitter来作为OutputCommitter的实现类。

> 相关描述：<https://stackoverflow.com/questions/13082606/getting-success-file-for-hive-script>

看了NullOutputCommitter的源码:

```java
  public static class NullOutputCommitter extends OutputCommitter {
    @Override
    public void setupJob(JobContext jobContext) { }
    @Override
    public void cleanupJob(JobContext jobContext) { }

    @Override
    public void setupTask(TaskAttemptContext taskContext) { }
    @Override
    public boolean needsTaskCommit(TaskAttemptContext taskContext) {
      return false;
    }
    @Override
    public void commitTask(TaskAttemptContext taskContext) { }
    @Override
    public void abortTask(TaskAttemptContext taskContext) { }
  }
```

**放空所有方法，等于什么事情都不做，所以当然无法生成_SUCCESS文件了**。

再细跟了下代码，发现不管是mr任务、spark任务还是tez任务，都默认设置了`mapred.output.committer.class`为NullOutputCommitter:

```java
//HiveFileFormatUtils.java
/**
   * Hive uses side effect files exclusively for it's output. It also manages
   * the setup/cleanup/commit of output from the hive client. As a result it does
   * not need support for the same inside the MR framework
   *
   * This routine sets the appropriate options related to bypass setup/cleanup/commit
   * support in the MR framework, but does not set the OutputFormat class.
   */
  public static void prepareJobOutput(JobConf conf) {
    conf.setOutputCommitter(NullOutputCommitter.class);

    //这个参数表示是否要在相应的job生命周期执行setupJob、commitJob等操作
    // option to bypass job setup and cleanup was introduced in hadoop-21 (MAPREDUCE-463)
    // but can be backported. So we disable setup/cleanup in all versions >= 0.19
    conf.setBoolean(MRJobConfig.SETUP_CLEANUP_NEEDED, false);

    //这个参数表示是否要在相应的task生命周期执行setupTask、commitTask等操作
    // option to bypass task cleanup task was introduced in hadoop-23 (MAPREDUCE-2206)
    // but can be backported. So we disable setup/cleanup in all versions >= 0.19
    conf.setBoolean(MRJobConfig.TASK_CLEANUP_NEEDED, false);
  }
```

Hive的注释中也写的比较清楚了。大概就是Hive自己实现了一套管理Job生命周期的方法，所以没必要使用mr的这套OutputCommitter机制了。所以搞了个NullOutputCommitter来让它不做任何工作。

对于一些高版本的hadoop，也可以通过设置`mapreduce.job.committer.setup.cleanup.needed`和`mapreduce.job.committer.task.cleanup.needed`为false来关闭OutputCommitter的机制。这里Hive为了兼容所有的hadoop版本，索性将所有的手段都用上了。

## 三、解决方案 

上面说了，虽然Hive有自己的解决方案，但是看了半天代码，也搜了很多资料，都没发现有什么好的办法可以让Hive输出_SUCCESS文件（在不改Hive源码的情况下）。在Hive的issue界面，搜索关于\_SUCCESS文件的issue，也只找到这个处于Open状态的issue:

<https://issues.apache.org/jira/browse/HIVE-3700>

所以，至少目前没有哪个参数设置后可以让Hive输出\_SUCCESS文件。因此，要解决这个问题只能动手改Hive源码了。改的思路有三个：

### 1、自己实现一个OutputCommitter替代NullOutputCommitter(不建议)

最开始的想法是自己实现一个OutputCommitter来取代Hive的NullOutputCommitter。这样就可以自己实现输出_SUCCESS文件的逻辑。大概代码如下：

```java
  public static class OnlyCommitOutputCommitter extends OutputCommitter {
    public static final int FILEOUTPUTCOMMITTER_FAILURE_ATTEMPTS_DEFAULT = 1;
    // Number of attempts when failure happens in commit job
    public static final String FILEOUTPUTCOMMITTER_FAILURE_ATTEMPTS =
            "mapreduce.fileoutputcommitter.failures.attempts";
    public static final String OUTDIR = "mapreduce.output.fileoutputformat.outputdir";
    public static final String SUCCEEDED_FILE_NAME = "_SUCCESS";
    public static final String SUCCESSFUL_JOB_OUTPUT_DIR_MARKER =
            "mapreduce.fileoutputcommitter.marksuccessfuljobs";

    @Override
    public void setupJob(JobContext jobContext) {
      LOG.info("setupJob--------------");
    }
    @Override
    public void cleanupJob(JobContext jobContext) { }

    @Override
    public void commitJob(JobContext context) throws IOException {
      LOG.info("commitJob--------------");
      int maxAttemptsOnFailure = isCommitJobRepeatable(context) ?
              context.getConfiguration().getInt(FILEOUTPUTCOMMITTER_FAILURE_ATTEMPTS,
                      FILEOUTPUTCOMMITTER_FAILURE_ATTEMPTS_DEFAULT) : 1;
      int attempt = 0;
      boolean jobCommitNotFinished = true;
      while (jobCommitNotFinished) {
        try {
          commitJobInternal(context);
          jobCommitNotFinished = false;
        } catch (Exception e) {
          if (++attempt >= maxAttemptsOnFailure) {
            throw e;
          } else {
            LOG.warn("Exception get thrown in job commit, retry (" + attempt +
                    ") time.", e);
          }
        }
      }
    }

    public static Path getOutputPath(JobConf conf) {
      String name = conf.get(org.apache.hadoop.mapreduce.lib.output.
              FileOutputFormat.OUTDIR);
      return name == null ? null: new Path(name);
    }

    protected void commitJobInternal(JobContext context) throws IOException {
      Path finalOutput = getOutputPath(context.getJobConf());
      if (finalOutput != null) {
        FileSystem fs = finalOutput.getFileSystem(context.getConfiguration());

        cleanupJob(context);
        if (context.getConfiguration().getBoolean(
                SUCCESSFUL_JOB_OUTPUT_DIR_MARKER, true)) {
          Path markerPath = new Path(finalOutput, SUCCEEDED_FILE_NAME);
          if (isCommitJobRepeatable(context)) {
            fs.create(markerPath, true).close();
          } else {
            fs.create(markerPath).close();
          }
        }
      } else {
        LOG.warn("Output Path is null in commitJob()");
      }
    }

    @Override
    public void setupTask(TaskAttemptContext taskContext) {
      LOG.info("setupTask--------------");
    }
    @Override
    public boolean needsTaskCommit(TaskAttemptContext taskContext) {
      return false;
    }
    @Override
    public void commitTask(TaskAttemptContext taskContext) {
      LOG.info("commitTask--------------");
    }
    @Override
    public void abortTask(TaskAttemptContext taskContext) { }
  }
```

但是在任务跑起来后发现一个问题，无法从JobConf中获取到对应的outputPath。

我们正常写mr任务时，会通过以下语句设置outputPath：

```java
FileInputFormat.setInputPaths(conf, new Path("/topath"));
```

其实底层是通过设置conf的参数实现的：

```java
    public static void setOutputPath(JobConf conf, Path outputDir) {
        outputDir = new Path(conf.getWorkingDirectory(), outputDir);
        conf.set("mapreduce.output.fileoutputformat.outputdir", outputDir.toString());
    }
```

因此后面我们可以直接通过参数`mapreduce.output.fileoutputformat.outputdir`获取到outputPath。但是在Hive的ExecDriver类中，我们发现Hive并不是这么设置outputPath的。它用的outputFormat也是`HiveOutputFormatImpl.class`，也就是hive实现了自己的一套获取outputPath的方法。

这样，我们要复用hadoop的OutputCommitter机制来输出_SUCCESS文件就要改动大量的代码去兼容Hive。

**改的越多，出问题的概率也就越大，因此，不建议使用该方案**。

### 2、改写FileSinkOperator(不建议)

hive输出数据到文件的逻辑操作都在FileSinkOperator代码中。FileSinkOperator#process()会处理一行一行的数据，之后在数据全部处理完后，会调用FileSinkOperator#jobCloseOp()方法。jobCloseOp()方法中，会将数据从一个临时目录(\_tmp.-ext-10000)移到另外一个临时目录(-ext-10000)

因此，我们可以在jobCloseOp()的方法中加上一段往最终输出的临时目录创建_SUCCESS文件的代码:

```java
  @Override
    public void jobCloseOp(Configuration hconf, boolean success)
            throws HiveException {
        try {
            if ((conf != null) && isNativeTable) {
                Path specPath = conf.getDirName();
                DynamicPartitionCtx dpCtx = conf.getDynPartCtx();
                if (conf.isLinkedFileSink() && (dpCtx != null)) {
                    specPath = conf.getParentDir();
                }
                Utilities.mvFileToFinalPath(specPath, hconf, success, LOG, dpCtx, conf,
                        reporter);
				//以下是添加代码块
                FileSystem fs = specPath.getFileSystem(hconf);
                Path successFilePath = new Path(specPath, "_SUCCESS");
                if (!fs.exists(successFilePath)) {
                    fs.createNewFile(successFilePath);
                }
                //添加代码块结束
            }
        } catch (IOException e) {
            throw new HiveException(e);
        }
        super.jobCloseOp(hconf, success);
    }
```

虽然理论上这个方案看似可行，但是编译部署后，发现输出目录还是没有_SUCCESS原因。又追了一下代码，发现hive每次进行文件move时，只会挑符合命名规范的文件move:

```java
//FileUtils.java
public static final PathFilter HIDDEN_FILES_PATH_FILTER = new PathFilter() {
    @Override
    public boolean accept(Path p) {
      String name = p.getName();
      return !name.startsWith("_") && !name.startsWith(".");
    }
  };
```

所以即使我们创建了_SUCCESS文件在move的过程中也会被丢弃掉。

**因此这个思路也不可用**

### 3、改写MoveTask

FileSinkOperator是在往任务的临时目录输出数据文件时调用的。真正我们在导出数据时，都会调用MoveTask将临时目录的数据移到真正的输出目录上（也只有导出数据会使用到MoveTask）。

因此我们只要修改MoveTask的相关代码，让它在move完数据后再创建一个_SUCCESS文件即可。

```java
//MoveTask.java
    private void moveFile(Path sourcePath, Path targetPath, boolean isDfsDir)
            throws HiveException {
        try {
            String mesg = "Moving data to " + (isDfsDir ? "" : "local ") + "directory "
                    + targetPath.toString();
            String mesg_detail = " from " + sourcePath.toString();
            console.printInfo(mesg, mesg_detail);

            FileSystem fs = sourcePath.getFileSystem(conf);
            //获取输出目录的fileSystem
            FileSystem dstFs = fs;
            if (isDfsDir) {
                moveFileInDfs(sourcePath, targetPath, fs);
            } else {
                // This is a local file
                dstFs = FileSystem.getLocal(conf);
                moveFileFromDfsToLocal(sourcePath, targetPath, fs, dstFs);
            }
			//判断是否需要输出_SUCCESS文件,默认先设置成false，和之前保持一致
            if (conf.getBoolean("mapreduce.fileoutputcommitter.marksuccessfuljobs", false)) {
                //判断之前有没有_SUCCESS文件，没有才输出
                Path successPath = new Path(targetPath, "_SUCCESS");
                if (!dstFs.exists(successPath)) {
                    dstFs.createNewFile(successPath);
                }
            }
        } catch (Exception e) {
            throw new HiveException("Unable to move source " + sourcePath + " to destination "
                    + targetPath, e);
        }
    }
```

因为MoveTask只会在真正往输出目录输出数据时才调用，因此我们可以保证不会影响到其他的任务执行。

另外，在这个基础上，我们添加了支持hadoop的mapreduce.fileoutputcommitter.marksuccessfuljobs参数的代码，可以用来控制是否要输出_SUCCESS文件，使其更加灵活。