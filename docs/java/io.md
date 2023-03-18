---
title: I/O
layout: default
tags: java
permalink: /io
parent: Java
---
# I/O

![image-20230318164307451](/assets/images/java_io_xmind.png)

1. 输入输出字节流（InputStream、OutStream）、字符流（Writer、Reader），分别处理字节和字符

2. InputStrem 输入字节流：从文件中读取字节信息到内存中

   - read() 方法，返回输入流中下一个字节的数据，结果范围 0～255，返回-1时表示文件结束

   - read(byte[] b)

   - close()

   - 通常配合 BufferedInputStream 使用，字节缓冲输入流

   - DataInputStream 用于读取指定类型的数据，允许应用程序以与机器无关方式从底层输入流中读取基本Java 数据类型

   - ObjectInputSteam 用于从输入流中读取 Java 对象（反序列化）

3. OutputStream 字节输出流：用于将数据写入到文件中

   - write(int b)

   - write(byte [] b)

   - flush()

   - close()

   - BufferedOutputStream 字节缓冲输出流

   - ObjectOutputStream 将对象写入输出流中，序列化

4. 字符流，便于对字符、音频、图片等媒体文件操作，乱码问题，字符流默认采用的是 `Unicode` 编码，我们可以通过构造方法自定义编码。顺便分享一下之前遇到的笔试题：常用字符编码所占字节数？`utf8`:英文占 1 字节，中文占 3 字节，`unicode`：任何字符都占 2 个字节，`gbk`：英文占 1 字节，中文占 2 字节

5. Reader 字符输入流

   - read()

   - read(char[] c)

   - close()

6. Writer 字符输出流

   - write(int c)

   - write(char[] c)

   - flush()

   - close()

7. BufferedInputStream BufferedOutputStream 字节缓冲流：避免IO频繁操作，提高流的传输效率

8. BufferedReader BufferedWriter 字符缓冲流

9. PrintStream 继承 OutStream 标准的输出流。字节打印流，PrintWriter 字符打印流

10. RandomAccessFile 随机访问流，支持随机跳转到文件的任意位置进行读写

11. 一个进程的地址空间被划分为用户空间和内核空间

12. 应用程序发起 IO 操作：内核等待 IO 设备准备数据，内核将数据从内核空间拷贝到用户空间

13. 常见的 IO 模型

    - 同步阻塞 IO，对应 Java 中 BIO，客户段连接数不高的情况下可行，当连接数过大时则服务器响应时间将缓慢

    - 同步非阻塞 IO 线程仍然是阻塞的，，通过轮询操作，避免一直阻塞 IO，频繁的轮询操作会耗费大量的 CPU 资源

    - IO 多路复用 对应 Java 中 NIO，通过减少无效系统调用，减少对 CPU 资源的消耗，IO 多路复用模型中，线程会先发起 select 调用，询问内核数据是否已准备，用户线程再发起 read 指令，read 过程数据从内核空间到用户空间是阻塞的

      - select：内核提供的系统调用，它支持一次查询多个系统调用的可用状态
        - 监听的文件描述符数量有限制
        - 通过轮询完成

      - epoll：基于 os 支持的 I/O 通知机制，优化了IO的执行效率
        - 没有最大文件描述符限制
        - 通过事件回调通知完成

    - 信号驱动 IO

    - 异步 IO

14. Java 中的 NIO 多路复用，一个线程管理多个客户端连接，通过选择器管理通道，通道 + 缓冲区 + 客户端

15. AIO 基于 NIO 的改进版本，基于事件回调完成

16. IO 中使用的装饰器模式：

    - 读取一个 txt 文件

      ```java
      InputStream in = new FileInputStream("/test.txt");
      InputStream bin = new BufferedInputStream(in);
      byte[] data = new byte[128];
      while (bin.read(data) != -1) 
      { 
        //...
      }
      ```

      InputStream 是一个抽象类，FileInputStream 是专门用来读取文件流的子类。BufferedInputStream 是一个支持带缓存功能的数据读取类，可以提高数据读取的效率。

    - 为什么不直接使用 BufferedInputStream 读取文件

      - 基于继承的实现方案

        若 InputStream 只有一个子类 FileInputStream ，在 FileInputStream 基础之上，设计一个 FileInputStream 子类 BufferedFileInputStream，也算是可以接受的，毕竟继承结构还算简单。但实际上继承 InputStream 的子类有很多。我们需要给每一个 InputStream 的子类，再继续派生支持缓存读取的子类，最后导致类无限增多，并且继承结构将变得复杂难以维护。

      - 基于装饰器模式的设计方案，设计原则：组合优先于继承

    - IO 中装饰器模式的特殊点

      - 装饰器类和原始类继承同样的父类，这样可以对原始类嵌套多个装饰器类
      - 装饰器类是对功能的增强，这也是装饰器模式应用场景的一个重要特点

    - FilterInputStream 类的作用，因类似 BufferedInputStream 装饰类有多种，若直接继承 InputStream 则必须实现其抽象方法，但具体的装饰类只需实现它们关心的方法即可，其余方法无需进行装饰增强，所以 FilterInputStream 的作用就是提供 InputStream 中抽象方法的默认实现，避免多个装饰类重复实现

    - 装饰器模式与代理模式的区别

      - 代理模式：增加附加功能，如鉴权、日志统计、限流等
      - 装饰器模式：解决继承关系过于复杂的问题，给原始类增强功能，
