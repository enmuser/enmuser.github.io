---
title: Java之NIO
date: 2020-01-19 00:10:24
tags: NIO
categories: java
---
##### NIO和IO的区别

- IO是面向的流的编程，是阻塞的

- NIO是面向缓存区的编程，是非阻塞的，通道(channel)负责连接IO的两端，缓冲器(buffer)负责存储数据

##### 缓冲区

缓存区的实质就是数组，不同的类型的数组，存储不同类型的数据(boolean 类型除外)

`ByteBuffer` `CharBuffer` `ShortBuffer` `IntBuffer` `LongBuffer` `FloatBuffer` `DoubleBuffer`

通过allocate(缓冲区大小)分配缓冲区

存取数据 

put() 将数据存入缓冲区

get() 获取缓冲区的数据

##### 缓冲区中的四个核心概念

`capacity`:容量，表示缓冲区中的最大的存储数据的容量。一旦声明不能改变

`limit`：限制，当前缓冲区实际的数据量(limit后得数据不能被读写)

`position`：位置，表当前缓冲正在操作数据的位置

`mark`:标记，记录当前position的位置，可以通过reset() 恢复mark记录的position

关系：`0` <= `mark `<= `position` <= `limit` <= `capacity`

##### 直接缓存区和非直接缓存区的区别

- 非直接缓存区：通过allocate() 方法分配缓冲区，将缓存区建立在JVM的内存中
- 直接缓冲区：通过allocateDirect() 方法分配直接缓冲区 将缓存区直接开辟在物理内存中 

##### 缓冲区的操作

```java
// 获取非直接缓存区
ByteBuffer buf = ByteBuffer.allocate(1024);
// 获取直接缓存区 
ByteBuffer buf = ByteBuffer.allocateDirect(1024);
//在缓冲区放入数据
String str = "abcdefg";
buf.put(str.getBytes());
//切换读取数据模式 flip快速切换
buf.flip();
//存入数据的byte数组
byte[] dst = new byte[buf.limit()];
//从缓冲区获取数据
buf.get(dst);
//重新可以读取刚才的数据 重置position=0 rewind倒带
buf.rewind();
//清空缓冲区,但是缓冲区的数据依然存在，但是出于“被遗忘”的状态
buf.clear();
//判断缓冲区是否还有剩余的数据
if(buf.hasRemaining()){
    //获取缓冲区还可操作的数据量
    buf.remaining(); 
}

```

##### 通道

通道(channel):用于源节点和目标节点的链接，Java NIO 负责缓冲区中数据的传输。Channel本身不存储数据，需要配合缓冲区进行传输数据

通道在`java.nio.channels.Channel`接口

`FileChannel` (本地文件)

`SocketChannel `(网络TCP)

`ServerSocketChannel`(网络TCP)

`DatagramChannel`(UDP数据报)

##### 获取通道

1. **通过getChannel方法获取，以下类可以获取**

   本地IO:  `FileInputStream`/`FileOutputStream `/`RandomAccessFile`

   网络IO:  `Socket` /` ServerSocket`/` DatagramSocket`

2. **JDK1.7 在NIO.2 针对各通道提供的静态方法open()**
3. **JDK1.7 在NIO.2 的Files工具类中的newByteChannel()**

```java
FileInputStream fis = null;
        FileOutputStream fos = null;
        FileChannel inChannel = null;
        FileChannel outChannel = null;
        try
        {
            //FileInputStream FileOutputStream 获取通道
            fis = new FileInputStream(new File("1.jpg"));
            fos = new FileOutputStream((new File("2.jpg")));
            //获取FileChannel
            inChannel = fis.getChannel();
            outChannel = fos.getChannel();
            //准备缓冲区(非直接缓冲区)
            ByteBuffer buf = ByteBuffer.allocate(1024);
            //通道读取数据到缓冲区(写入缓冲区)
            while(inChannel.read(buf) != -1)
            {
                buf.flip();//转换到读的模式
                outChannel.write(buf);//将缓存的数据写入通道
                buf.clear();//清空缓冲区为下一次读入准备
            }
        }
        catch(IOException e)
        {
            e.printStackTrace();
        }
        finally
        {
            if(outChannel!=null){
                try
                {
                    outChannel.close();
                }
                catch(IOException e)
                {
                    e.printStackTrace();
                }
            }

            if(inChannel!=null){
                try
                {
                    inChannel.close();
                }
                catch(IOException e)
                {
                    e.printStackTrace();
                }
            }

            if(fos!=null){
                try
                {
                    fos.close();
                }
                catch(IOException e)
                {
                    e.printStackTrace();
                }
            }
            if(fis!=null){
                try
                {
                    fis.close();
                }
                catch(IOException e)
                {
                    e.printStackTrace();
                }
            }

        }
```

```java
//用Open方式获取通道
FileChannel inChannel = FileChannel.open(Paths.get("1.jpg"), StandardOpenOption.READ);
FileChannel outChannel =                               FileChannel.open(Paths.get("3.jpg"),StandardOpenOption.WRITE,StandardOpenOption.READ,StandardOpenOption.CREATE);
//通过直接内存映射方式获取缓冲区
MappedByteBuffer inbuf = inChannel.map(FileChannel.MapMode.READ_ONLY,0,inChannel.size());
MappedByteBuffer outbuf = outChannel.map(FileChannel.MapMode.READ_WRITE,0,inChannel.size());
//直接对缓冲区进行读写
byte[] bytes = new byte[inbuf.limit()];
inbuf.get(bytes);
outbuf.put(bytes);
inChannel.close();
outChannel.close();
```

##### 通道之间的数据传输

`transferFrom()  transferTo()`

```java
//用Open方式获取通道
FileChannel inChannel = FileChannel.open(Paths.get("1.jpg"), StandardOpenOption.READ);
FileChannel outChannel = FileChannel.open(Paths.get("3.jpg"),StandardOpenOption.WRITE,StandardOpenOption.READ,StandardOpenOption.CREATE);
//通道直接的数据传输
inChannel.transferTo(0,inChannel.size(),outChannel);
//等价于
outChannel.transferFrom(inChannel,0,inChannel.size());
inChannel.close();
outChannel.close();
```

##### 分散(Scatter)与聚集(Gather)

- 分散读取(Scattering Reads) 将通道中的数据分散到多个缓冲区

- 聚集写入(Gathering Writes) 将多个缓冲区中的数据聚集到通道中 
```java
RandomAccessFile raf = new RandomAccessFile("1.txt","rw");
FileChannel channel1 = raf.getChannel();

ByteBuffer buf1 = ByteBuffer.allocate(500);
ByteBuffer buf2 = ByteBuffer.allocate(500);
ByteBuffer[] byteBuffers = {buf1,buf2};
//分散读取
channel1.read(byteBuffers);
for(ByteBuffer buf : byteBuffers)
{
   buf.flip();
}
/** System.out.println(new String(byteBuffers[0].array(),0,byteBuffers[0].limit()));
System.out.println("--------------------------------------------------------");
System.out.println(new String(byteBuffers[1].array(),0,byteBuffers[0].limit()));*/

        //聚集写入
        RandomAccessFile raf2 = new RandomAccessFile("2.txt","rw");
        FileChannel channel2 = raf2.getChannel();
        channel2.write(byteBuffers);
```

##### 编码与解码

```java
Charset cset = Charset.forName("GBK");
//字符到字节数组 编码
CharsetEncoder encoder = cset.newEncoder();
//字节数组到字符 解码
CharsetDecoder decoder = cset.newDecoder();
CharBuffer cbuf = CharBuffer.allocate(1024);
cbuf.put("张家口检索");
cbuf.flip();
ByteBuffer buf = encoder.encode(cbuf);
//buf.flip();
CharBuffer cbuf1 = decoder.decode(buf);
System.out.println(cbuf1.toString());
```