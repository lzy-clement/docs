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
至此一个简单的Java通过内存操作文件的例子就完成了。但是可以看到上面的程序并没有关闭资源。这简单，加上关闭代码就好了
```
  // 将内存中内容刷盘
  mbb.force();
  // 关闭资源
  fc.close();
  raf.close();
```
很简单的代码。但是仅仅认为就这样就真的too young too simple了。

你会发现在你关闭了所有资源以后，程序还是占用着文件没有释放。即使你将mbb，fc，raf都设置为空，一样无济于事。。。这就有点尴尬了。如果程序中要定时删除对应过期的文件，由于这一直持有其文件句柄，我还无法删除了。

找了很多资料，发现这个Java关于mmap的一个[bug](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=4715154)。由于FileChannel调用了map方法做内存映射，但是没提供对应的unmap方法释放内存，导致内存一直占用该文件。实际unmap方法在FileChannelImpl中私有方法中，在finalize时，unmap无法调用导致内存没释放。

同样找了很多的解决方案，最终这2个比较靠谱的：
- 手动执行unmap方法
```
// 在关闭资源时执行以下代码释放内存
Method m = FileChannelImpl.class.getDeclaredMethod("unmap", MappedByteBuffer.class);
m.setAccessible(true);
m.invoke(FileChannelImpl.class, buffer);
```

- 让MappedByteBuffer自己释放本身持有的内存
```
AccessController.doPrivileged(new PrivilegedAction() {
    public Object run() {
      try {
        Method getCleanerMethod = buffer.getClass().getMethod("cleaner", new Class[0]);
        getCleanerMethod.setAccessible(true);
        sun.misc.Cleaner cleaner = (sun.misc.Cleaner)
        getCleanerMethod.invoke(byteBuffer, new Object[0]);
        cleaner.clean();
      } catch (Exception e) {
        e.printStackTrace();
      }
      return null;
    }
});
```

实际上面两个方法都调用了Cleaner类的clean方法释放，参考unmap代码
```
private static void unmap(MappedByteBuffer bb) {
    Cleaner cl = ((DirectBuffer)bb).cleaner();
    if (cl != null)
        cl.clean();
}
```

其实讲到这里该问题的解决办法已然清晰明了了。就是在删除索引文件的同时还取消对应的内存映射，删除mapped对象。 不过令人遗憾的是，Java并没有特别好的解决方案——令人有些惊讶的是，Java没有为MappedByteBuffer提供unmap的方法， 该方法甚至要等到Java 10才会被引入 ,DirectByteBufferR类是不是一个公有类  class DirectByteBufferR extends DirectByteBuffer implements DirectBuffer 使用默认访问修饰符 不过Java倒是提供了内部的“临时”解决方案——DirectByteBufferR.cleaner().clean() 切记这只是临时方法，毕竟该类在Java9中就正式被隐藏了，而且也不是所有JVM厂商都有这个类。 还有一个解决办法就是显式调用System.gc()，让gc赶在cache失效前就进行回收。 不过坦率地说，这个方法弊端更多：首先显式调用GC是强烈不被推荐使用的， 其次很多生产环境甚至禁用了显式GC调用，所以这个办法最终没有被当做这个bug的解决方案。 



问题是解决了。但是可以看到第二个方法中，使用了AccessController，这个又是一个新的东西，我们下一期来再来研究看看这个到底是什么。
