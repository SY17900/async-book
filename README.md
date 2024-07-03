# Asynchronous Programming in Rust

## 执行详解

### new_executor_and_spawner

这个函数创建一个管道，然后返回`Executor`还有`Spawner`实例。

### Spawner::spawn()

`spawn()`函数传入一个实现了`Future`的任务，然后用`Spawner`实例里的`task_sender`把任务给送到管道里，这是第一次把任务注册到执行器里。注意这里实现了`Future`的是一个`async`块，并不是我们自己代码里的`TimerFuture`。

### drop(spawner)

把`spawner`释放掉之后，管道的写端没有了，所以`Executor`把管道读空就退出`run()`了。

### executor::run()

之后进入`run()`的循环，不停地从管道里读东西出来。核心是调用`poll()`。经书里说这个函数是用来推动`Future`执行的，如果调用结果是`Pending`的话，就表明还没有执行完，这是要给未完成的任务设置一个`waker`，用来在某种方式上通知执行器所以自己的执行已经有进度了，可以再次`poll()`来推进其执行。

第一次从管道里读东西之后，这个`poll()`应该是实现在`tokio`库里的，这里先略过

然后函数会在管道接收的位置阻塞住，此时子线程应该已经开始执行并等待`sleep()`完成。当子线程完成之后，它会通过调用`waker`来把自己发到主线程里`Executor`的任务队列里：

```rust
impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        let cloned = arc_self.clone();
        arc_self.task_sender.send(cloned).expect("too many tasks queued");
    }
}
```

然后主线程`Executor`从管道里读出`Task`，并且调用`TimerFuture`里的`poll()`去取得任务执行完毕后的结果。

### .await

当对一个实现了`Future`的实例调用`.await`时，执行器会暂停对这个任务的执行，直到该任务完成；完成时任务会自行调用`wake()`来告知执行器可以继续`poll()`我。在示例里这个任务就是子线程：

```rust
thread::spawn(move || {
    thread::sleep(duration);
    let mut shared_state = thread_shared_state.lock().unwrap();
    shared_state.completed = true;
    if let Some(waker) = shared_state.waker.take() {
        waker.wake();
    }
});
```
在我们的例子里，当`async`块里第一次遇到`.await`时，这一行`TimerFuture`的构造函数被执行完了，然后又调用了`poll()`去判断计时器有没有记完时。事实上调用`.await`的是一个实现了`Future`的类实例（这里是`TimerFuture`），执行器是通过`poll()`函数的返回结果去判断任务有没有完成的，而不是一般意义上的这一行有没有执行完。

当下一行语句开始执行时，`.await`这一行的东西必然已经被完成了。
