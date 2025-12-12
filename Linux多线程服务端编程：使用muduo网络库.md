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
