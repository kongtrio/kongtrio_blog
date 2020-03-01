[TOC]

在做文件下载功能时，为了避免下载功能将服务器的带宽打满，从而影响服务器的其他服务。我们可以设计一个限流器来限制下载的速率，从而限制下载服务所占用的带宽。

## 一、算法思路  

定义**一个数据块chunk**(单位 bytes)以及允许的**最大速率 maxRate**(单位 KB/s)。通过maxRate我们可以算出，在maxRate的速率下，通过一个数据块大小的字节流所需要的时间 `timeCostPerChunk`。

之后，在读取/写入字节时，我们维护已经读取/写入的字节量 `bytesWillBeSentOrReceive`。

当`bytesWillBeSentOrReceive`达到一个数据块的大小时，检查期间消耗的时间(nowNanoTime-lastPieceSentOrReceiveTick)

如果期间消耗的时间小于` timeCostPerChunk`的值，说明当前的速率已经超过了 maxRate的速率，这时候就需要休眠一会来限制流量

如果速率没超过或者休眠完后，将 `bytesWillBeSentOrReceive=bytesWillBeSentOrReceive-chunkSize`

之后在读取/写入数据时继续检查。

下面该算法的Java代码实现：

```java
    public synchronized void limitNextBytes(int len) {
        //累计bytesWillBeSentOrReceive
        this.bytesWillBeSentOrReceive += len;
		//如果积累的bytesWillBeSentOrReceive达到一个chunk的大小,就进入语句块操作
        while (this.bytesWillBeSentOrReceive > CHUNK_LENGTH) {
            long nowTick = System.nanoTime();
            //计算积累数据期间消耗的时间
            long passTime = nowTick - this.lastPieceSentOrReceiveTick;
            //timeCostPerChunk表示单个块最多需要多少纳秒
            //如果missedTime大于0，说明此时流量进出的速率已经超过maxRate了，需要休眠来限制流量
            long missedTime = this.timeCostPerChunk - passTime;
            if (missedTime > 0) {
                try {
                    Thread.sleep(missedTime / 1000000, (int) (missedTime % 1000000));
                } catch (InterruptedException e) {
                    LOGGER.error(e.getMessage(), e);
                }
            }
            this.bytesWillBeSentOrReceive -= CHUNK_LENGTH;
            //重置最后一次检查时间
            this.lastPieceSentOrReceiveTick = nowTick + (missedTime > 0 ? missedTime : 0);
        }
    }
```

## 二、限流的完整java代码实现

限流器的实现

```java
public class BandwidthLimiter {

    private static final Logger LOGGER = LoggerFactory.getLogger(BandwidthLimiter.class);
    //KB代表的字节数
    private static final Long KB = 1024L;
    //一个chunk的大小，单位byte。设置一个块的大小为1M
    private static final Long CHUNK_LENGTH = 1024 * 1024L;

    //已经发送/读取的字节数
    private int bytesWillBeSentOrReceive = 0;
    //上一次接收到字节流的时间戳——单位纳秒
    private long lastPieceSentOrReceiveTick = System.nanoTime();
    //允许的最大速率，默认为 1024KB/s
    private int maxRate = 1024;
    //在maxRate的速率下，通过chunk大小的字节流要多少时间（纳秒）
    private long timeCostPerChunk = (1000000000L * CHUNK_LENGTH) / (this.maxRate * KB);

    public BandwidthLimiter(int maxRate) {
        this.setMaxRate(maxRate);
    }

    //动态调整最大速率
    public void setMaxRate(int maxRate) {
        if (maxRate < 0) {
            throw new IllegalArgumentException("maxRate can not less than 0");
        }
        this.maxRate = maxRate;
        if (maxRate == 0) {
            this.timeCostPerChunk = 0;
        } else {
            this.timeCostPerChunk = (1000000000L * CHUNK_LENGTH) / (this.maxRate * KB);
        }
    }

    public synchronized void limitNextBytes() {
        this.limitNextBytes(1);
    }

    public synchronized void limitNextBytes(int len) {
        this.bytesWillBeSentOrReceive += len;

        while (this.bytesWillBeSentOrReceive > CHUNK_LENGTH) {
            long nowTick = System.nanoTime();
            long passTime = nowTick - this.lastPieceSentOrReceiveTick;
            long missedTime = this.timeCostPerChunk - passTime;
            if (missedTime > 0) {
                try {
                    Thread.sleep(missedTime / 1000000, (int) (missedTime % 1000000));
                } catch (InterruptedException e) {
                    LOGGER.error(e.getMessage(), e);
                }
            }
            this.bytesWillBeSentOrReceive -= CHUNK_LENGTH;
            this.lastPieceSentOrReceiveTick = nowTick + (missedTime > 0 ? missedTime : 0);
        }
    }
}
```

有了限流器后，现在我们要对下载功能做限流。因为java的io流的设计是装饰器模式，因此我们可以方便的封装一个我们自己的InputStream

```java
public class LimitInputStream extends InputStream {

    private InputStream inputStream;
    private BandwidthLimiter bandwidthLimiter;

    public LimitInputStream(InputStream inputStream, BandwidthLimiter bandwidthLimiter) {
        this.inputStream = inputStream;
        this.bandwidthLimiter = bandwidthLimiter;
    }

    @Override
    public int read() throws IOException {
        if (bandwidthLimiter != null) {
            bandwidthLimiter.limitNextBytes();
        }
        return inputStream.read();
    }

    @Override
    public int read(byte[] b, int off, int len) throws IOException {
        if (bandwidthLimiter != null) {
            bandwidthLimiter.limitNextBytes(len);
        }
        return inputStream.read(b, off, len);
    }

    @Override
    public int read(byte[] b) throws IOException {
        if (bandwidthLimiter != null && b.length > 0) {
            bandwidthLimiter.limitNextBytes(b.length);
        }
        return inputStream.read(b);
    }
}
```

后面我们使用这个LimitInputStream来读取文件，每次读取一块数据，限流器都会检查当前的速率是否超过指定的最大速率。这样就能间接的达到限制下载速率的目的了。

附上SpringMVC的一个下载限流的demo：

```java
    @GetMapping("/limit")
    public void limitDownloadFile(String file, HttpServletResponse response) throws IOException {
        LOGGER.info("download file");
        if (file == null) {
            file = "/tmp/test.txt";
        }
        File downloadFile = new File(file);
        FileInputStream fileInputStream = new FileInputStream(downloadFile);

        response.setContentType("application/x-msdownload;");
        response.setHeader("Content-disposition", "attachment; filename=" + new String(downloadFile.getName()
                .getBytes("utf-8"), "ISO8859-1"));
        response.setHeader("Content-Length", String.valueOf(downloadFile.length()));
        ServletOutputStream outputStream = null;
        try {
            LimitInputStream limitInputStream = new LimitInputStream(fileInputStream, new BandwidthLimiter(1024));

            long beginTime = System.currentTimeMillis();
            outputStream = response.getOutputStream();
            byte[] bytes = new byte[1024];
            int read = limitInputStream.read(bytes, 0, 1024);
            while (read != -1) {
                outputStream.write(bytes);
                read = limitInputStream.read(bytes, 0, 1024);
            }
            LOGGER.info("download use {} ms", System.currentTimeMillis() - beginTime);
        } finally {
            fileInputStream.close();
            if (outputStream != null) {
                outputStream.close();
            }
            LOGGER.info("download success!");
        }
    }
```

## 三、注意点

使用这个算法要注意一个问题，就是chunk的块大小不能设置的太小，即CHUNK_LENGTH不能设置的太小。**否则容易造成明明maxRate设置的很大，但是实际下载速率却很小的问题**。

假设CHUNK_LENGTH就设置为1024 bytes，每次读取的块大小也是1024 bytes，maxRate 为 64M/s。那么我们可以计算出timeCostPerChunk约等于**15258**纳秒。

再如果真正的速率是100M/s，也就是每秒差不多会调用limitNextBytes方法100000次，由于每次读取消耗的时间极短，因此每次进入该方法都要sleep `15258`纳秒之后再读取下一个块的数据。**如果没有算上线程调度的时间，就算1秒内休眠100000次也完全没什么问题。**但是线程的休眠和唤醒都需要内核来进行，线程上下文切换的时间应该远大于`15258`纳秒，这时候频繁的休眠就会导致线程暂停运行的时间和我们预期的不符。由于休眠时间过长，最终导致实际的下载速率大大的低于maxRate。

因此，我们需要调大CHUNK_LENGTH，尽量让timeCostPerChunk的值**远大于**线程调度的时间，减少线程调度对限流造成的影响。

## 四、具体demo的github地址

https://github.com/kongtrio/download-limit