---
title: "tokio-异步基础类型"
date: 2021-12-19T14:55:59+08:00
draft: false
toc: true
---

为了适配 async 模型，tokio 重新实现了标准库中的 fs，net，channel 等模块，提供想对应的 async 方法。

本文以 TcpListener 为例，剖析 tokio 如何实现异步版本的 TcpListener，以及如何与 Reactor 整合。

# 数据结构关系-自顶向下

```rust
pub struct TcpListener {
    io: PollEvented<mio::net::TcpListener>,
}
// 表示一个与 Reactor 关联的 IO 资源
pub(crate) struct PollEvented<E: Source> {
    io: Option<E>,
    registration: Registration,
}
// 表示一个已经注册到了 Reactor Io 资源
// handle 表示注册的 Reactor Driver handle
// shared 表示注册的 Io 相关信息引用，可用了观测 IO 状态
pub(crate) struct Registration {
    /// Handle to the associated driver.
    handle: Handle,

    /// Reference to state stored by the driver.
    shared: slab::Ref<ScheduledIo>,
}
// 注册的 IO source 信息，包括就绪 events等
pub(crate) struct ScheduledIo {
    /// Packs the resource's readiness with the resource's generation.
    readiness: AtomicUsize,

    waiters: Mutex<Waiters>,
}
```

PollEvented 是一个可注册到 Reactor 的 Source 通用实现。实现了 mio Source Trait 的资源都可以借助此类型实现异步版本的方法。

Registration 是与 Reactor 交互的关键，可通过 handle 访问 Reactor(Driver), 可通过 shared 访问自身注册的事件状态。

# accept 实现

```rust
pub async fn accept(&self) -> io::Result<(TcpStream, SocketAddr)> {
    let (mio, addr) = self
    .io
    .registration()
    .async_io(Interest::READABLE, || self.io.accept())
    .await?;

    let stream = TcpStream::new(mio)?;
    Ok((stream, addr))
}

pub(crate) async fn async_io<R>(&self, interest: Interest, mut f: impl FnMut() -> io::Result<R>) -> io::Result<R> {
    loop {
        let event = self.readiness(interest).await?;

        match f() {
            Err(ref e) if e.kind() == io::ErrorKind::WouldBlock => {
                self.clear_readiness(event);
            }
            x => return x,
        }
    }
}

fn readiness_fut(&self, interest: Interest) -> Readiness<'_> {
    Readiness {
        scheduled_io: self,
        state: State::Init,
        waiter: UnsafeCell::new(Waiter {
            pointers: linked_list::Pointers::new(),
            waker: None,
            is_ready: false,
            interest,
            _p: PhantomPinned,
        }),
    }
}
```

Registration 提供的 async_io 抽象了异步 io。通过传入 Interest 和 执行函数，可以搭建需要的异步方法。

ScheduledIo 提供的 readiness_fut 函数，返回实现了 Future 的 Readiness 结构，作为最底层的叶子。

```rust
struct Readiness<'a> {
    scheduled_io: &'a ScheduledIo,

    // 记录状态机的状态
    state: State,

    /// Entry in the waiter `LinkedList`.
    waiter: UnsafeCell<Waiter>,
}
```

那注册的事件 Ready 时，如何唤醒 Readiness Future 呢？

在 poll 无法返回 Ready 时，需要让 Reactor 更新 wake 函数。通过设置 Readiness.waiter 的 waker 函数，并将 waiter 加入ScheduledIo.waiters 中。而 ScheduledIo 是 Reactor 和 PollEvented 共享的，从而实现了 Reactor 中注册的 Resouce 的 waker 函数更新。

在这里， Readiness.waiter 代表了 await 在 readiness 上的上层 Future，即 PollEvented。对 Reactor 来说，即 waite 在相关事件上的某种东西，具体是什么 Reactor 不关心，只在 Ready 时，唤醒所有的 waiter。

```rust
fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
    ...
     // Safety: called while locked
    unsafe {
        (*waiter.get()).waker = Some(cx.waker().clone());
    }
    ...
     // Insert the waiter into the linked list
    // safety: pointers from `UnsafeCell` are never null.
    waiters
    .list
    .push_front(unsafe { NonNull::new_unchecked(waiter.get()) });
    ...
}


```

在 Reator 的实现中, Ready 事件 dispatch 实现，会遍历所有 waiter，并调用 wake 函数。

```rust
fn dispatch(&mut self, token: mio::Token, ready: Ready) {
    ...
    let io = match resources.get(addr) {
        Some(io) => io,
        None => return,
    };
    ...
    io.wake(ready);
}

fn ScheduledIo::wake0(&self, ready: Ready, shutdown: bool) {
    ...
    wakers.wake_all();
    ...
}
```

在 Reactor dispatch 时，会通过 set_readiness 函数设置 ScheduledIo 的 readiness。在 PollEvented 第一次被唤醒时，会先检查

ScheduledIo.readiness, 然后才检查 Waiter 自己的 is_ready 状态。因为第一次 poll 时，ScheduledIo.readiness 就是最新的状态。



# 其他异步 io 实现

## fs

并非所有的异步版本 io 资源都需要 Reactor，比如文件系统。

```rust
pub async fn open(path: impl AsRef<Path>) -> io::Result<File> {
    let path = path.as_ref().to_owned();
    let std = asyncify(|| StdFile::open(path)).await?;

    Ok(File::from_std(std))
}

pub(crate) async fn asyncify<F, T>(f: F) -> io::Result<T>
where
    F: FnOnce() -> io::Result<T> + Send + 'static,
    T: Send + 'static,
{
    match spawn_blocking(f).await {
        Ok(res) => res,
        Err(_) => Err(io::Error::new(
            io::ErrorKind::Other,
            "background task failed",
        )),
    }
}
```

通过 spawn_blocking 返回一个 JoinHandle，即返回了一个可以 await 的 future，实现了将 open 操作转移到了 block 线程池，但自身确实非阻塞的。

## Channel

```rust
pub struct UnboundedReceiver<T> {
    /// The channel receiver
    chan: chan::Rx<T, Semaphore>,
}

pub async fn UnboundedReceiver::recv(&mut self) -> Option<T> {
    use crate::future::poll_fn;

    poll_fn(|cx| self.poll_recv(cx)).await
}

/// Future for the [`poll_fn`] function.
pub struct PollFn<F> {
    f: F,
}

impl<F> Unpin for PollFn<F> {}

/// Creates a new future wrapping around a function returning [`Poll`].
pub fn poll_fn<T, F>(f: F) -> PollFn<F>
where
    F: FnMut(&mut Context<'_>) -> Poll<T>,
{
    PollFn { f }
}
```

UnboundedReceiver 本身没有实现 Future，但是通过 poll_fn 函数，返回的 PollFn 实现了 Future。

UnboundedReceiver 的成员函数作为了 poll 函数的一部分。即 UnboundedReceiver 自身保存了future 的状态。



在 ready 时，如何通知 await 的 future 呢？

在往 channel push，调用 rx_waker.wake()。同时，在 recv 无法 ready 时，也会更新 waker 函数。

```rust
impl<T, S> Chan<T, S> {
    fn send(&self, value: T) {
        // Push the value
        self.tx.push(value);

        // Notify the rx task
        self.rx_waker.wake();
    }
    
    pub(crate) fn recv(&mut self, cx: &mut Context<'_>) -> Poll<Option<T>> {
        ...
        self.inner.rx_waker.register_by_ref(cx.waker());
        ...
    }
}

```

## time

time 是最上层的 driver，本身采用时间轮的实现方式。park 时，会得到最近的一次唤醒时间，然后调用 park_timeout 在下层 driver。
最下层的 driver 是 io driver，io driver park_timeout 时调用 mio 的 poll 方法，并指定 timeout，从而实现 timeout 后唤醒 driver 树。

## mio 如何唤醒 driver 树

如果 runtime 添加了新的 task，而此时 driver 还可能处于 park 状态。spawn task 时，会先将任务加入 run_queue, 然后调用 driver 的 unpark。unpark 会一直调用到 io driver 的 unpark，然后调用 mio 的 waker，从 poll 中返回。

## 底层的 task，mio 事件是何时添加的？

在创建的时候注册到 mio，通过 PollEvented 完成注册。poll 时，只会去检查事件是否 ready，不会再 poll 时注册。

```rust
pub(crate) fn new(listener: mio::net::TcpListener) -> io::Result<TcpListener> {
   let io = PollEvented::new(listener)?;
   Ok(TcpListener { io })
}
```

