## 背景
随着时代的发展，在当今互联网时代，大家对时间的敏感度越来越高。因而对程序的处理能力要求也越来越高，各种程序优化层出不穷。其中在文件IO处理上，大部分人都采用了内存映射的概念（毕竟程序操作内存的速度比操作磁盘的速度快不是一丁半点）。

既然使用到了内存映射，在Java中，就不得不认识一下MappedByteBuffer了。


## MappedByteBuffer
我们先用最简单的例子来看看 MappedByteBuffer 的使用：


```
  // 通过 RandomAccessFile 创建对应的文件操作类，第二个参数 rw 代表该操作类可对其做读写操作
  RandomAccessFile raf = new RandomAccessFile("fileName", "rw");

  // 获取操作文件的通道
  FileChannel fc = raf.getChannel();

  // 也可以通过FileChannel的open来打开对应的fc
  // FileChannel fc = FileChannel.open(Paths.get("/usr/local/test.txt"),StandardOpenOption.WRITE);


  // 把文件映射到内存
  MappedByteBuffer mbb = fc.map(FileChannel.MapMode.READ_WRITE, 0, (int)    fc.size());

  // 读写文件
  mbb.putInt(4);
  mbb.put("test".getBytes());
  mbb.force();

  mbb.position(0);
  mbb.getInt();
  mbb.get(new byte[test.getBytes().size()]);
  ........
```
https://blog.csdn.net/wujumei1962/article/details/42919383

https://www.ibm.com/developerworks/cn/java/j-lo-javasecurity/
至此
