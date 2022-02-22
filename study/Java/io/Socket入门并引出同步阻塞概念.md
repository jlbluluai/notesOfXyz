# Table of Contents

* [Socket入门并引出同步阻塞概念](#socket入门并引出同步阻塞概念)
    * [概述](#概述)
    * [层层递进](#层层递进)
        * [同步阻塞的串行化](#同步阻塞的串行化)
        * [同步阻塞的并发](#同步阻塞的并发)
    * [总结](#总结)


# Socket入门并引出同步阻塞概念

## 概述

说到IO，初学者的第一反应就是文件的处理，其实目前企业级开发中文件IO其实可能用到的不是很多，涉及IO更多也更关键的网络传输，这就要说到Socket了。


## 层层递进

通过创建`ServerSocket`模拟一个最简单的HTTP服务器，并层层优化

### 同步阻塞的串行化

**步骤**

1. 创建一个`ServerSocket`并绑定到端口`8801`
2. 通过`accept`阻塞等待客户端请求并获取客户端的`Socket`
3. 模拟输出HTTP报文头和一串文字
4. 关闭客户端的`Socket`


**代码如下**

```java
public class HttpServer01 {

    public static void main(String[] args) throws IOException {
        ServerSocket serverSocket = new ServerSocket(8801);
        while (true) {
            try {
                Socket socket = serverSocket.accept();
                System.out.println(socket.getInetAddress());
                service(socket);
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private static void service(Socket socket) {
        try (PrintWriter printWriter = new PrintWriter(socket.getOutputStream(), true)) {
            printWriter.println("HTTP/1.1 200 OK");
            printWriter.println("Content-Type:text/html;charset=utf-8");
            String body = "Hello, xyz1";
            printWriter.println("Content-Length:" + body.getBytes().length);
            printWriter.println();
            printWriter.write(body);
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                socket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

}

```

**测验**

启动后，浏览器访问如下地址：

[http://localhost:8801](http://localhost:8801)

就能看到浏览器页面显示了`Hello, xyz1`

启动参数添加`-Xms512m -Xmx512m`，用如下指令压测：

```
wrk -c 40 -d30s http://localhost:8801
```

压测了3次数据（平均每秒能处理的请求也就在400左右）：

```
Running 30s test @ http://localhost:8801
  2 threads and 40 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     4.75ms   11.63ms 440.48ms   98.46%
    Req/Sec   475.46    284.18     1.47k    79.03%
  12851 requests in 30.82s, 1.95MB read
  Socket errors: connect 0, read 68111, write 12, timeout 6
Requests/sec:    416.99
Transfer/sec:     64.77KB

Running 30s test @ http://localhost:8801
  2 threads and 40 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    14.35ms   74.33ms   1.89s    99.05%
    Req/Sec   264.77    126.78     0.97k    76.32%
  10653 requests in 30.03s, 1.46MB read
  Socket errors: connect 40, read 48357, write 2, timeout 0
Requests/sec:    354.70
Transfer/sec:     49.64KB

Running 30s test @ http://localhost:8801
  2 threads and 40 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     6.82ms   26.92ms 745.04ms   99.28%
    Req/Sec   395.68    179.57     1.04k    75.17%
  11635 requests in 30.07s, 1.69MB read
  Socket errors: connect 0, read 58136, write 1, timeout 5
Requests/sec:    386.97
Transfer/sec:     57.65KB
```


### 同步阻塞的并发

引入线程池，多开处理，主要优化点在于等待到了客户端连接后异步去处理与客户端的交互

**代码如下**

```java
public class HttpServer03 {

    public static void main(String[] args) throws IOException {
        ExecutorService executorService = Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors() * 4);
        ServerSocket serverSocket = new ServerSocket(8803);

        while (true) {
            try {
                Socket socket = serverSocket.accept();
                executorService.execute(() -> service(socket));
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    private static void service(Socket socket) {
        try (PrintWriter printWriter = new PrintWriter(socket.getOutputStream(), true)) {
            printWriter.println("HTTP/1.1 200 OK");
            printWriter.println("Content-Type:text/html;charset=utf-8");
            String body = "Hello, xyz3";
            printWriter.println("Content-Length:" + body.getBytes().length);
            printWriter.println();
            printWriter.write(body);
        } catch (IOException e) {
            e.printStackTrace();
        } finally {
            try {
                socket.close();
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

}
```

**测验**

启动后，浏览器访问如下地址：

[http://localhost:8803](http://localhost:8803)

就能看到浏览器页面显示了`Hello, xyz3`

启动参数添加`-Xms512m -Xmx512m`，用如下指令压测：

```
wrk -c 40 -d30s http://localhost:8803
```

压测了3次数据（平均每秒能处理的请求也就在500多）：

```
Running 30s test @ http://localhost:8803
  2 threads and 40 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency    17.73ms   82.88ms   1.36s    97.82%
    Req/Sec   440.59    235.10     1.84k    78.40%
  20998 requests in 30.03s, 2.58MB read
  Socket errors: connect 0, read 77917, write 3, timeout 1
Requests/sec:    699.34
Transfer/sec:     87.87KB

Running 30s test @ http://localhost:8803
  2 threads and 40 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     2.78ms    5.54ms  75.47ms   84.51%
    Req/Sec   438.39    380.47     2.30k    82.17%
  12822 requests in 30.01s, 2.06MB read
  Socket errors: connect 39, read 40516, write 7, timeout 0
Requests/sec:    427.19
Transfer/sec:     70.38KB

Running 30s test @ http://localhost:8803
  2 threads and 40 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency     6.68ms   28.37ms 841.01ms   99.61%
    Req/Sec   521.89    261.39     1.71k    72.03%
  16683 requests in 30.06s, 2.36MB read
  Socket errors: connect 0, read 81773, write 1, timeout 10
Requests/sec:    554.90
Transfer/sec:     80.42KB
```

## 总结

事实证明了在同步阻塞下，由于服务端是阻塞的等待客户端连接的，若是获取到客户端连接后还继续串行处理将会效率非常低下，这时候引入线程池则能有效的提高效率。