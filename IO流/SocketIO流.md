# Socket

在现实中，Socket 这个概念没有一个具体的实体，它是描述计算机之间完成相互通信一种抽象定义。

打个比方，可以把 Socket 比作为两个城市之间的交通工具，有了它，就可以在城市之间来回穿梭了。并且，交通工具有多种，每种交通工具也有相应的交通规则。Socket 也一样，也有多种。大部分情况下我们使用的都是基于 TCP/IP 的流套接字，它是一种稳定的通信协议。



## 传输数据

当客户端要与服务端通信时，客户端首先要创建一个 Socket 实例，默认操作系统将为这个 Socket 实例分配一个没有被使用的本地端口号，并创建一个包含本地、远程地址和端口号的套接字数据结构，这个数据结构将一直保存在系统中直到这个连接关闭。

```
客户端->
Socket s = new Socket("127.0.0.1",8080);
BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(s.getOutputStream()));
String str = "helloworld";
bw.write(str);
bw.flush();
bw.close();
```

## 接收数据

与之对应的服务端，也将创建一个 ServerSocket 实例，ServerSocket 创建比较简单，只要指定的端口号没有被占用，一般实例创建都会成功，同时操作系统也会为 ServerSocket 实例创建一个底层数据结构，这个数据结构中包含指定监听的端口号和包含监听地址的通配符，通常情况下都是`*`即监听所有地址。

```
服务端->
ServerSocket s = new ServerSocket(8080);
while(true){  
    Socket socket = s.accept();
    BufferedReader reader = new BufferedReader(new InputStreamReader(socket.getInputStream()));
    String str = reader.readLine();
    Log.d("tag",str);
}
```

当连接已经建立成功，服务端和客户端都会拥有一个 Socket 实例，每个 Socket 实例都有一个 **InputStream** 和 **OutputStream**，正如我们前面所说的，网络 I/O 都是以字节流传输的，Socket 正是通过这两个对象来交换数据。

当 Socket 对象创建时，操作系统将会为 InputStream 和 OutputStream 分别分配一定大小的缓冲区，数据的写入和读取都是通过这个缓存区完成的。

写入端将数据写到 OutputStream 对应的 SendQ 队列中，当队列填满时，数据将被发送到另一端 InputStream 的 RecvQ 队列中，如果这时 RecvQ 已经满了，那么 OutputStream 的 write 方法将会阻塞直到 RecvQ 队列有足够的空间容纳 SendQ 发送的数据。

值得特别注意的是，缓存区的大小以及写入端的速度和读取端的速度非常影响这个连接的数据传输效率，由于可能会发生阻塞，所以网络 I/O 与磁盘 I/O 在数据的写入和读取还要有一个协调的过程，如果两边同时传送数据时可能会产生死锁的问题。

**如何提高网络 IO 传输效率、保证数据传输的可靠，已经成了工程师们急需解决的问题**

