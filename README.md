# Asynchronous Programming in Rust

## 执行详解

### *new_executor_and_spawner*

这个函数创建一个管道，然后返回`Executor`还有`Spawner`实例。

### *Spawner::spawn()*

`spawn()`函数传入一个实现了`Future`的任务，然后用`Spawner`实例里的`task_sender`把任务给送到管道里，这是第一次把任务注册到执行器里。

### *drop(spawner)*

把`spawner`释放掉之后，管道的写端没有了，所以`Executor`把管道读空就退出`run()`了。

### *executor::run()*

之后进入`run()`的循环，不停地从管道里读东西出来。核心是调用`TimerFuture`里实现的`poll()`。经书里说这个函数是用来推动`Future`执行的，如果调用结果是`Pending`的话，就表明还没有执行完，所以把`future`给还回去。看这里`poll()`传入的参数`context`，它归根结底是由`Task`的`waker`弄过来的：

```rust
impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        let cloned = arc_self.clone();
        arc_self.task_sender.send(cloned).expect("too many tasks queued");
    }
}
```

大意就是，当任务准备好被进一步推进时，可以调用这个函数来通知执行器自己可以被进一步`poll`了（这里是把自己送到执行队列里）。现在的问题就是`poll()`是怎么推进执行的。

### *TimerFuture::poll()*

直接贴代码过来：

```rust
impl Future for TimerFuture {
    type Output = ();
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let mut shared_state = self.shared_state.lock().unwrap();
        if shared_state.completed {
            Poll::Ready(())
        } else {
            shared_state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}
```

这里就是判断`shared_state.completed`到底有没有被设为`True`，据此来判断应该返回`Ready`还是`Pending`。

### *.await*

当对一个实现了`Future`的实例调用`.await`时，执行器会暂停对这个任务的执行，直到该任务完成；这个过程中执行完成的任务会调用`wake()`来告知执行器可以继续`poll()`我。在示例里这个任务就是子线程：

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

当子线程执行完`sleep()`的任务之后，它取得锁并更新完成状态，然后调用`wake()`函数来通知执行器。`waker`是在`poll()`函数里注册的：当任务没有被执行完全时，对`poll()`的调用会注册一个`waker`并且返回一个`Pending`。

在我们的例子里这个`waker`归根结底长这样：

```rust
impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        let cloned = arc_self.clone();
        arc_self.task_sender.send(cloned).expect("too many tasks queued");
    }
}
```

很显然地，`waker`通过把任务发到通道里来通知执行器自己可以被再次调度了。



跳一点中间没太看懂的......



最后执行完这里的东西后，`TimerFuture`的创建已经完成，然后计时器状态也被设置为已完成。

```rust
spawner.spawn(async {
    println!("FUCK!");
    TimerFuture::new(Duration::new(20, 0)).await;
    println!("YOU!");
});
```

在随后（这里是关键，此时到底是谁在推动着这个`async`块的执行）`run()`的执行中，很显然地`poll()`返回了`Ready`，然后所有都结束了。