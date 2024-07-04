# Asynchronous Programming in Rust

## 异步程序是如何执行的？

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

## Pin是干嘛用的

> 本质上就是用来处理引用和自引用的有效性问题的。`Pin`内部包裹了另一个指针，只要里头那个指针指向的内容（类型`T`）没有实现`Unpin`，他就会保证`T`不会被移动（编译时报错）。

考虑这样的结构体：

```rust
struct Test {
    a: String,
    b: *const String,
}
```

如果我们让`b`指向`a`的话，通常不会出什么问题，但是假如整个结构体位置变化了（但是其中的值没有改变），那么就很难说`b`指向了哪里，对它所引用对象的生命周期分析从而也就无法进行了。

### Unpin

这个`trait`表明当移动这个东西的时候，并不会影响依赖这个东西内存位置的代码（这一般是说这个东西实现了`Copy`，移动后原处的东西依旧可用）。比方说`u8`是`Unpin`的，这是显而易见的。

书里说：

> Pinning an object to the stack will always be unsafe if our type implements `!Unpin`.

这就是在说，譬如说一个`String`，它在栈上保存的东西是一个指向堆中实际数据的智能指针，当它`move`时原处的智能指针结构体会被废弃，因此指定它是`Pin`的是不安全的。

> If `T: Unpin` (which is the default), then `Pin<'a, T>` is entirely equivalent to `&'a mut T`. In other words: `Unpin` means it's OK for this type to be moved even when pinned, so Pin will have no effect on such a type.

`Pin`是一种保证，如果类型实现了`Unpin`，这说明就算移动了也不影响原处的东西（从代码逻辑上来讲，本来指针或者引用也是依赖着原处的东西）。

但是另一方面：

> Getting a `&mut T` to a pinned `T` requires `unsafe` if `T: !Unpin`.

这也是显而易见的，毕竟默认地这些东西会在移动的时候造成原处内容的不可用。

> For pinned data where `T: !Unpin` you have to maintain the invariant that its memory will not get invalidated or repurposed from the moment it gets pinned until when drop is called. This is an important part of the pin contract.

嗯。

### Pin是怎么做的

简单来讲，就是对被`Pin`包裹住的指针做一个遮蔽`shadow`：如果里头指针指向的类型没有实现`Unpin`，就确保外界不能在`safe`下获得对它的可变借用。

当然为了灵活性，它也允许在`unsafe`下提供一个`get_unchecked_mut()`方法来允许拿到可变借用，只不过在这种情况下安全要你自己来保证。

### Pin和Future的关系

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}
```

这里我们看到`poll()`函数传入的参数必须要包在`Pin`里面，这指出实现了`Future`的这个任务在`poll()`的过程中是不能被移动的。

某种意义上讲，`poll()`的内容可以说是自定义的，因此必须要约束在推进任务执行的过程中不许改变任务结构体的内存位置（尤其是如果有结构体存在自引用的话）。
