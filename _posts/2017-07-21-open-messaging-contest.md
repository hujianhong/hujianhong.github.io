---
layout:     post
title:      第三届阿里中间件性能挑战赛 初赛总结
date:       2017-07-21
author:     胡建洪
header-img: img/stan-y--g67_YOp6O4-unsplash.jpg
catalog: true
tags:
    - java
    - 消息中间件
---

# 1.	初赛
初赛题目：《基于Open-Messaging实现进程内消息引擎》
## 1.1.	赛题背景分析及理解
赛题描述：支撑阿里双十一的万亿级消息中间件RocketMQ在2016年10月正式进入Apache基金会进行孵化。异步解耦，削峰填谷，让消息中间件在现代软件架构中拥有着举足轻重的地位。天下大势，分久必合，合久必分，软件领域也是如此。市场上消息中间件种类繁多，阿里消息团队在此当口，推出厂商无关的Open-Messaging规范，促进消息领域的进一步发展。本赛题要求参赛选手阅读Open-Messaging规范，了解Message，Topic，Queue，Producer，Consumer等概念，并基于相关语言的接口实现进程内消息引擎。
分析与理解：消息中间件分布式系统的消息传递和通信中具有举足轻重的作用，消息中间件的性能在一定程度上决定了整个系统的性能。消息中间件对于生产者而言，写入消息要快，且消息写入后不能丢失；对于消费者而言，消费顺序不能乱，消息到的消息要准确及时。一般而言，要做到高性能的分布式消息中间件，在架构上和实现上都要高屋建瓴，推出通用的规范和概念，便于沟通和交流，以及软件的发展和升级。目前，阿里具有很大的平台，拥有别人没有的数据和技术优势，但是也面临着别人没有的挑战，比如双十一亿万级消息挑战等。通过这次比赛，我们可以更进一步感受和了解阿里的技术以及面临的挑战，通过拓宽自己的眼界，提升自己的技术水平。
## 1.2.	核心思路
根据题目要求，我们需要实现指定接口，完成消息的生产、存储、消费等功能。其中消息的存储是关键，需要保证消息的正确性和顺序一致性，同时需要持久化到磁盘上，消息持久化的方式和性能影响着生产消息和消费消息的速度。下面是我们的核心思路的具体说明。
### 1.2.1.	整体处理流程
整体处理流程如图1所示，首先启动生产进程生产消息，并进行消息存储，由于消息生产和消息消费是不同进程，需要进行消息持久化，保存到磁盘上，待消息生产结束后，结束生产进程，启动消费进程进行消息消费，消费过程需要保证消费到的消息次序和生产时次序一致，且消息正确。

![图1 整体流程图](/img/2017-07-21/c3b6eb140aa8480fa2f2bfeadb67d336.png)
### 1.2.2.	消息存储方案
在比赛中，我们实现了多种消息存储方案，分别如下：
方案1：这个消息存储方案为一个Topic或者Queue对应一个文件。每个生产者在写入消息到Topic或者Queue时，先获取Topic或者Queue的互斥锁，获取到锁后，然后再写入消息。每个消费者消费时可以同时打开同一个文件进行消费，互不干扰。
优点：管理简单
缺点：需要加锁以保证消息的正确写入，效率不高；
方案2：这个消息存储方案为一个Topic或者Queue对应多个文件。对于一个Topic或者Queue我们以Topic或Queue的名称建立一个目录，每个线程如果要将消息写入到该Topic或者Queue中，则在目录下创建一个独立文件，该线程的要写入该Topic或者Queue的后续消息都将写入到该文件中。当需要消费一个Topic或者Queue中的消息时，消费者只需依次消费该Toipc或者Queue下对应的目录下的所有文件的消息即可。
优点：无需加锁，写入效率较高。
缺点：消息存储时文件数跟线程相关，耦合度相对较高。

对比方案1和方案2，我们最后采用方案2实现消息存储，线上测试时效率也比方案1要高。
### 1.2.3.	消息持久化
消息由头部（headers)、属性（properties)、主体（body)三个部分组成，其中头部和属性为KeyValue结构，消息主体为byte[]数组。

消息存储需要序列化，消息是一个对象，对象的序列化方案主要有JDK的Serializable ，将对象转换为JSON或者XML格式的字符串，自定义存储格式等。其中JDK的Serializable 在序列化时，需要额外写入对象的Class信息，效率不高，且浪费空间。JSON或者XML则需要将对象转换为字符串，多了一个转换过程，增加了CPU开销。我们根据实际情况，采用自定义存储格式进行消息序列化，具体如下：
对于KeyValue结构的Headers或Properties，我们先写入KeyValue总数，然后对于每个KeyValue，先写入Key的字节数组长度，然后写入key的字节数组，再写入Value的字节数组长度，最后写入Value的字节数组。
对于消息主体的byte[]，我们先写入byte[]的长度，然后写入byte[]。

其中，一条消息序列化后的结构为：Header  + Properties +  body，示意图如图2所示：
![图2 消息序列化结构图](/img/2017-07-21/dce8cb228179413289fd08d0faad8784.png)

### 1.2.4.	数据压缩方案
由于消息数据量大，而IO速度比较慢，因此我们将数据进行压缩后，在进行IO操作，这样做可以减少IO的数据量而提高写入效率。当然这种方案是用CPU开销换IO开销的方案。
对于数据压缩，无损压缩算法主要有Deflater，Snappy，LZ4，QuickLZ等。其中Deflater压缩速度比较慢。在线上测试时，我们使用了Deflater，Snappy，QucikLZ等3个压缩算法，压缩性能依次为：
Snappy > QiuckLZ > Deflater
在数据压缩时，如果每次只压缩一条消息，无疑效率是低下的，因此我们设置消息序列化缓存区，缓存区大小为32KB + 256B，每当缓存区满了，则调用压缩算法将缓存区数据压缩并写入文件。
### 1.3.	关键代码
消息写入关键代码：
```
public class SnappyWriter implements IWriter {
    private RandomAccessFile file;
    public static final int DEFAULT_SIZE = IConstants.CMP_MS;
    private byte[] bytes = new byte[DEFAULT_SIZE + IConstants.MSG_ML];
    private int p;
    private byte[] cmp = new byte[DEFAULT_SIZE + IConstants.MSG_ML];

    public SnappyWriter(String dir) throws IOException {
        file = new RandomAccessFile(dir, "rw");
    }
    private void put(byte a) {
        bytes[p++] = (byte) (a & 0xff);
    }
    private void put(byte[] bs) {
        int a = bs.length;
        if(a < Byte.MAX_VALUE){
            bytes[p++] = (byte) (a & 0xff);
        } else {
            bytes[p++] = (byte) ((a >> 8 & 0xff) | 0x80);
            bytes[p++] = (byte) (a & 0xff);
        }
        System.arraycopy(bs, 0, bytes, p, bs.length);
        p += bs.length;
    }
    public void write(DefaultBytesMessage message) throws IOException {
        DefaultKeyValue h = (DefaultKeyValue)message.headers();
        DefaultKeyValue pro = (DefaultKeyValue)message.properties();
        // 减少一个字节存储头部和属性的长度
        byte tsize = (byte) ((((byte)h.num) << 4 & 0xf0) | (pro.num & 0x0f));
        put(tsize);
        for (int i = 0; i < h.num; i++) {
            put(h.keys[i].getBytes());
            put(h.values[i]);
        }
        for (int i = 0; i < pro.num; i++) {
            put(pro.keys[i].getBytes());
            put(pro.values[i]);
        }
        byte[] body = ((DefaultBytesMessage) message).getBody();
        put(body);
        
        if (p >= DEFAULT_SIZE) {
            int clen = Snappy.compress(bytes, 0,p, cmp, 0);
            file.writeShort(clen);
            file.write(cmp, 0, clen);
            this.p = 0;
        }
    }
    public void close() throws IOException {
        if (this.p > 0) {
            int clen = Snappy.compress(bytes, 0,p, cmp, 0);
            file.writeShort(clen);
            file.write(cmp, 0, clen);
        }
        file.writeShort(0);
        file.close();
    }
}
```

消息读取关键代码：

```
public class SnappyReader implements IReader {
    private int cnt = 0;
    private MappedByteBuffer[] mBuffers;
    
    public SnappyReader(String dir) throws IOException {
        File dirFile = new File(dir);
        if (!dirFile.exists()) {
            this.complete = true;
            return;
        }
        File[] files = dirFile.listFiles();
        if (files.length <= 0) {
            this.complete = true;
            return;
        }
        mBuffers = new MappedByteBuffer[files.length];
        for (int i = 0; i < files.length; i++) {
            @SuppressWarnings("resource")
            FileChannel channel = new FileInputStream(files[i])
                    .getChannel();
            mBuffers[i] = channel.map(MapMode.READ_ONLY, 0, channel.size());
        }
        mBuffer = mBuffers[cnt++];
    }
    
    public BytesMessage read() throws Exception {
        if (complete) {
            return null;
        }
        if(p < limit){
            return fromBuffer();
        }
        int len = 0;
        while ((len = getLen()) == 0) {
            if (cnt == mBuffers.length) {
                this.complete = true;
                return null;
            }
            mBuffer = mBuffers[cnt++];
        }
        mBuffer.get(cmp, 0, len);
        limit = Snappy.uncompress(cmp, 0, len, bytes, 0);
        p = 0;
        return fromBuffer();
    }
    private int getLen(){
        byte b1 = mBuffer.get();
        byte b2 = mBuffer.get();
        return ((((b1 & 0xff) << 8) | (b2 & 0xff))) & 0x7fffffff;
    }
    
    private byte[] cmp = new byte[IConstants.CMP_MS + IConstants.MSG_ML];
    private byte[] bytes = new byte[IConstants.CMP_MS + IConstants.MSG_ML];
    private int p;
    private int limit;
    
    private ByteBuffer mBuffer;
    private boolean complete = false;
    
    private QingBytesMessage msg = new QingBytesMessage();
    private QingKeyValue hds = new QingKeyValue(2);
    private QingKeyValue pros = new QingKeyValue(4);
    
    private static final String TOPIC = MessageHeader.TOPIC;
    private static final String QUEUE = MessageHeader.QUEUE;
    
    private BytesMessage fromBuffer() {
        byte tsize = getByte();
        int keySize = (tsize >> 4 & 0x0f);
        int pSize = (tsize & 0x0f);
        hds.clear();
        for (int i = 0; i < keySize; i++) {
            String key = getString();
            switch (key) {
            case "T":
                hds.put(TOPIC, getBytes());
                break;
            case "Q":
                hds.put(QUEUE, getBytes());
                break;
            default:
                hds.put(key, getBytes());
                break;
            }
        }
        msg.setHeaders(hds);
        if(pSize > 0){
            pros.clear();
            for (int i = 0; i < pSize; i++) {
                pros.put(getString(), getBytes());
            }
            msg.setProperties(pros);
        } else {
            msg.setProperties(null);
        }
        msg.setBody(getBytes());
        return msg;
    }
    public int getShort() {
        return (((bytes[p++] & 0xff) << 8) | ((bytes[p++] & 0xff)));
    }

    public byte getByte() {
        return (byte) (bytes[p++] & 0xff);
    }
    public String getString() {
        int a = (bytes[p++] & 0xff);
        String s = new String(bytes, p, a);
        p += a;
        return s;
    }
    public byte[] getBytes() {
        int a = (((bytes[p++] & 0xff) << 8) | ((bytes[p++] & 0xff)));
        byte[] b = new byte[a];
        System.arraycopy(bytes, p, b, 0, a);
        p += a;
        return b;
    }
}
```
1.4.	总结
尽量实现对象复用，避免频繁创建对象，避免频繁GC。
能够避免加锁就避免，如果无法避免，则将加锁范围控制在最小范围内。
减少中间数据拷贝过程，尽量直接将数据拷贝志最终目标处，能极大的提高效率。
代码越优化越简单，往往简单的代码比复杂的代码可读性高，性能也相对来说比较高。

代码已经开源在GitHub上了，需要的同学可以参考下：[初赛代码](https://github.com/hujianhong/open-messaging-contest.git "初赛代码")
