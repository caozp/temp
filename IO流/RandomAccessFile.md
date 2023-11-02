# RandomAccessFile

Java除了File类之外，还提供了专门处理文件的类，即RandomAccessFile（随机访问文件）类。该类是Java语言中功能最为丰富的文件访问类，它提供了众多的文件访问方法。RandomAccessFile类支持“随机访问”方式，这里“随机”是指可以跳转到文件的任意位置处读写数据。在访问一个文件的时候，不必把文件从头读到尾，而是希望像访问一个数据库一样“随心所欲”地访问一个文件的某个部分，这时使用RandomAccessFile类就是最佳选择


RandomAccessFile对象类有个位置指示器，指向当前读写处的位置，当前读写n个字节后，文件指示器将指向这n个字节后面的下一个字节处。刚打开文件时，文件指示器指向文件的开头处，可以移动文件指示器到新的位置，随后的读写操作将从新的位置开始。RandomAccessFile类在数据等长记录格式文件的随机（相对顺序而言）读取时有很大的优势，但该类仅限于操作文件，不能访问其他的I/O设备，如网络、内存映像等

**RandomAccessFile的一个重要使用场景就是网络请求中的多线程下载及断点续传**



## 构造方法

> RandomAccessFile ra  = new RandomAccessFile("d:\\employee.txt" , "rw")

除了文件名之外还需要访问模式

* "r": 以只读方式打开。调用结果对象的任何 write 方法都将导致抛出 IOException。
* "rw": 打开以便读取和写入。
* "rws": 打开以便读取和写入。相对于 "rw"，"rws" 还要求对“文件的内容”或“元数据”的每个更新都同步写入到基础存储设备。
* "rwd" : 打开以便读取和写入，相对于 "rw"，"rwd" 还要求对“文件的内容”的每个更新都同步写入

## 重要方法

**long getFilePointer( )：**返回文件记录指针的当前位置

**void seek(long pos )：**将文件指针定位到pos位置

```
public static void main(String[] args)
{
    try
    {
        insert("d:/out.txt",5,"插入的内容");
    }
    catch (IOException e)
    {
        e.printStackTrace();
    }
}

private static void insert(String fileName,long pos,String content) throws IOException
{
    //创建临时空文件
    File tempFile = File.createTempFile("temp",null);
    //在虚拟机终止时，请求删除此抽象路径名表示的文件或目录
    tempFile.deleteOnExit();
    FileOutputStream fos = new FileOutputStream(tempFile);

    RandomAccessFile raf = new RandomAccessFile(fileName,"rw");
    raf.seek(pos);
    byte[] buffer = new byte[4];
    int num = 0;
    while(-1 != (num = raf.read(buffer)))
    {
        fos.write(buffer,0,num);
    }
    raf.seek(pos);
    raf.write(content.getBytes());
    FileInputStream fis = new FileInputStream(tempFile);
    while(-1 != (num = fis.read(buffer)))
    {
        raf.write(buffer,0,num);
    }
}  
```

## 断点续传

原理知道了，功能实现起来也简单。每当线程停止时就把已下载的数据长度写入记录文件，当重新下载时，从记录文件读取已经下载了的长度。而这个长度就是所需要的断点。
续传的实现也简单，可以通过设置网络请求参数，请求服务器从指定的位置开始读取数据。
而要实现这两个功能只需要使用到httpURLconnection里面的**setRequestProperty**方法便可以实现

## 与FileInputStream的差别

RandomAccessFile使用随机访问的方式，而FileInputStream及FileOutputStream使用的是流式访问的方式。