零拷贝的“零”是指用户态和内核态间copy数据的次数为零。







### Java BIO

BIO 全称Block-IO 是一种**同步且阻塞**的通信模式。是一个比较传统的通信方式，模式简单，使用方便。但并发处理能力低，通信耗时，依赖网速。

面向流的方式进行数据处理，一个输入流产生一个字节的数据，一个输出流消费一个字节的数据。

### Java NIO

Java NIO，全程 Non-Block IO ，是Java SE 1.4版以后，针对网络传输效能优化的新功能。是一种**非阻塞同步**的通信模式。

面向块的 I/O 系统以块的形式处理数据。每一个操作都在一步中产生或者消费一个数据块。按块处理数据比按(流式的)字节处理数据要快得多。但是面向块的 I/O 缺少一些面向流的 I/O 所具有的优雅性和简单性。



### Java AIO

Java AIO，全程 Asynchronous IO，是**异步非阻塞**的IO。是一种非阻塞异步的通信模式。

在NIO的基础上引入了新的异步通道的概念，并提供了异步文件通道和异步套接字通道的实现。



小结：

**BIO （Blocking I/O）：同步阻塞I/O模式。**

**NIO （New I/O）：同步非阻塞模式。**

**AIO （Asynchronous I/O）：异步非阻塞I/O模型。**



















































































































