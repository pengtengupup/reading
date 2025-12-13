## 多线程同步

==mutex==

使用互斥量需要注意：

1. 死锁问题。
2. Lock Convoy ：高频锁竞争 + 不均衡持有时间。
3. False Sharing    多核修改同一 cache line    缓存行频繁无效化

==condition variable==

条件变量常用来等待某个条件发生，期望加锁时能够立即获得锁（条件成立），来提高效率。

使用方式一般较为固定：

```c_cpp
std::mutex mutex;
std::conditon_variable cond;
std::queue<int> queue;
void DeQueue() {
  std::unique<std::mutex> lock(mutex);
  // 防止虚假唤醒
  cond.wait(lock, []{ return !queue.empty(); });
  queue.pop();
}
void EnQueue(int x) {
  {
    std::unique<std::mutex> lock(mutex);
    queue.push_back(x);
  }
  cond.notify_one(); // signal不一定要上锁
}
```

虚假唤醒：即在唤醒线程与执行表达式检查之间表达式再次被改变，使得检查无法通过而没有执行，C11中可以在wait中传入条件检查来解决。

==CountDownLatch==

常用于主线程等待多个子线程完成某条件，或者多个子线程等待主线程。使用join无法与线程池结合。

## 使用muduo网络库

#### 线程模型

muduo的线程模型为one loop per thread+thread pool模型。每个线程最多有一个EventLoop，每个TcpConnection必须归某个EventLoop管理，所有的IO会转移到这个线程。换句话说，一个file descriptor只能由一个线程读写。

TcpConnection和EventLoop是线程安全的，可以跨线程调用。

TcpServer直接支持多线程，它有两种模式：· 单线程，accept(2)与TcpConnection用同一个线程做IO。· 多线程，accept(2)与EventLoop在同一个线程，另外创建一个EventLoop-ThreadPool，新到的连接会按round-robin方式分配到线程池中。

#### TCP网络编程本质

基于事件的非阻塞网络编程的思想区别于阻塞式的“主动调用recv等待数据，主动调用accept等待新连接，主动调用send发送数据”的思想最大的不同：**“注册多个回调函数，网络库会在接收新连接/数据到达/需要发送数据等时候执行我们的回调函数，等到调用回调的时候，数据的准备已经做好了”**

chenshuo认为TCP编程本质是处理三个半事件

1、TCP连接的建立。如connect->accpet

2、连接的断开。close/shutdown。

3、消息到达，文件描述符可读。这是最为重要的一个事件，对它的处理方式决定了网络编程的风格（阻塞还是非阻塞，如何处理分包，应用层的缓冲如何设计，等等）​。

3.5、发送消息完成。

#### 简单Echo服务

```c_cpp
#include <muduo/base/Logging.h>
#include <muduo/base/Timestamp.h>
#include <muduo/net/EventLoop.h>
#include <muduo/net/TcpServer.h>

#include <functional>

using namespace std::placeholders;

class EchoServer {
public:
    EchoServer(muduo::net::EventLoop* loop, const muduo::net::InetAddress& addr);
    void start();  // call server_.start()

private:
    void onConnection(const muduo::net::TcpConnectionPtr& conn);
    void onMessage(const muduo::net::TcpConnectionPtr& conn, muduo::net::Buffer* buf,
                   muduo::Timestamp time);
    muduo::net::EventLoop* loop_;
    muduo::net::TcpServer server_;
};

EchoServer::EchoServer(muduo::net::EventLoop* loop, const muduo::net::InetAddress& addr)
    : loop_(loop), server_(loop, addr, "EchoServer") {
    server_.setConnectionCallback(std::bind(&EchoServer::onConnection, this, _1));
    server_.setMessageCallback(std::bind(&EchoServer::onMessage, this, _1, _2, _3));
}

void EchoServer::start() { server_.start(); }

void EchoServer::onConnection(const muduo::net::TcpConnectionPtr& conn) {
    LOG_INFO << conn->peerAddress().toIpPort() << " -> " << conn->localAddress().toIpPort()
             << " is " << (conn->connected() ? "UP" : "DOWN");
}

void EchoServer::onMessage(const muduo::net::TcpConnectionPtr& conn, muduo::net::Buffer* buf,
                           muduo::Timestamp time) {
    muduo::string msg(buf->retrieveAllAsString());
    LOG_INFO << conn->name() << " echo " << msg.size() << " bytes, "
             << "data received at " << time.toString();
    conn->send(msg);
}

int main() {
    LOG_INFO << "pid = " << getpid();
    muduo::net::EventLoop loop;
    muduo::net::InetAddress listenAddr(2025);
    EchoServer server(&loop, listenAddr);
    server.start();
    loop.loop();
}
```
