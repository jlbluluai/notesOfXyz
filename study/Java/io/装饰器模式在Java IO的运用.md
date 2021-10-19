# Table of Contents

* [装饰器模式在Java IO的运用](#装饰器模式在java-io的运用)
    * [介绍](#介绍)
    * [准备](#准备)
    * [分析](#分析)
    * [实战理解](#实战理解)


# 装饰器模式在Java IO的运用


## 介绍

本篇以`InputStream`为例，其他诸如`OutputStream`、`Reader`之类的都是类似的。

本篇也不会详细叙述`InputStream`家族的所有东西，只是举出其中的经典佐证装饰器，让理解一目了然。

具体装饰器模式还不知道是啥的，[先去了解下](https://www.runoob.com/design-pattern/decorator-pattern.html)。


## 准备

本篇会涉及以下几个类（可以先用`idea`打开备好）：

- `InputStream`
- `FileInputStream`
- `FilterInputStream`
- `BufferedInputStream`

他们之前的关系大概如下：

![](http://img.yelizi.top/83891475-7eca-4f18-a352-f3de6ab5cd37.jpg$xyz)


## 分析

首先`InputStream`是字节输入流的最基础的抽象组件，所有针对字节的操作都是继承它。

`FileInputStream`是继承自`InputStream`专注于文件字节输入流操作。


`FilterInputStream`就是装饰器模式的核心了，可以看到它定义了`InputStream`对象，就是用来装饰的对象。


`BufferedInputStream`就是一个继承自`FilterInputStream`的具体的装饰器类了，主要是提供缓冲功能，那装饰到`FileInputStream`上就让文件字节输入流具备了缓冲功能（这里不具体去分析各个类实现）。


## 实战理解

我们自定义一个简单的装饰器类，记录每次读取耗时并打印日志。

```java
@Slf4j
public class TimingInputStream extends FilterInputStream {

    protected TimingInputStream(InputStream in) {
        super(in);
    }

    @Override
    public int read() throws IOException {
        long start = System.currentTimeMillis();
        try {
            return super.read();
        } finally {
            log.info("本次耗时 = {}ms", System.currentTimeMillis() - start);
        }
    }

    @Override
    public int read(byte[] b) throws IOException {
        long start = System.currentTimeMillis();
        try {
            return super.read(b);
        } finally {
            log.info("本次耗时 = {}ms", System.currentTimeMillis() - start);
        }
    }

    @Override
    public int read(byte[] b, int off, int len) throws IOException {
        long start = System.currentTimeMillis();
        try {
            return super.read(b, off, len);
        } finally {
            log.info("本次耗时 = {}ms", System.currentTimeMillis() - start);
        }
    }
}
```

测试用例（文件100M，读取设定每次10m）

```java
public static void main(String[] args) throws Exception {
        File file = new File("/Users/zhuweijie/Downloads/test.zip");
        try (FileInputStream fis = new FileInputStream(file);
        TimingInputStream tis = new TimingInputStream(fis);
        FileOutputStream fos = new FileOutputStream("/Users/zhuweijie/Downloads/test_copy.zip")) {
        byte[] buffer = new byte[10 * 1024 * 1024];
        int cnt;

        while ((cnt = tis.read(buffer, 0, buffer.length)) != -1) {
        fos.write(buffer, 0, cnt);
        }
        }
        }
```

日志

```

20:36:31.387 [main] INFO com.xyz.bu.utils.TimingInputStream - 本次耗时 = 8ms
20:36:31.402 [main] INFO com.xyz.bu.utils.TimingInputStream - 本次耗时 = 8ms
20:36:31.412 [main] INFO com.xyz.bu.utils.TimingInputStream - 本次耗时 = 5ms
20:36:31.424 [main] INFO com.xyz.bu.utils.TimingInputStream - 本次耗时 = 5ms
20:36:31.435 [main] INFO com.xyz.bu.utils.TimingInputStream - 本次耗时 = 7ms
20:36:31.446 [main] INFO com.xyz.bu.utils.TimingInputStream - 本次耗时 = 4ms
20:36:31.455 [main] INFO com.xyz.bu.utils.TimingInputStream - 本次耗时 = 5ms
20:36:31.464 [main] INFO com.xyz.bu.utils.TimingInputStream - 本次耗时 = 5ms
20:36:31.472 [main] INFO com.xyz.bu.utils.TimingInputStream - 本次耗时 = 4ms
20:36:31.481 [main] INFO com.xyz.bu.utils.TimingInputStream - 本次耗时 = 4ms
20:36:31.485 [main] INFO com.xyz.bu.utils.TimingInputStream - 本次耗时 = 0ms
```