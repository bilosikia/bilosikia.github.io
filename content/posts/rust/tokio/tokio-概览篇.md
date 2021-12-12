---
title: "tokio-概览篇"
date: 2021-12-12T14:55:59+08:00
draft: false

---

# runtime 构成

- Task: 

    runtime 每 spawn 一个异步函数就会产生一个 Task。Task 是 runtime 的调度单元，每个 Task 包含需要执行的 Future 和状态信息。

- Excutor：

    用来执行，管理，调度 Task，包含线程池，每个线程被抽象为一个 Worker。

- Driver:

    Task 的执行，需要 await 的地方通常是 IO，Timer等，需要依赖外部状态的改变。当外部状态改变时，Task 需要被重新调度，Driver 用于驱动 Task 的状态变更。

    Driver 对应传统的 Reator 模型，只是不仅仅包含 IO, 还包含 Timer，Signal 等。

- Waker：

    属于 Task 的一部分，用于关联 Driver 和 Exuctor。当 Driver 任务 Task 就绪可以被重新调度时，通过 Waker 唤醒任务。

    Waker 将 Driver 和 Excutor 解耦。

rust 只定义了异步运行时的接口，即 Future 和 Waker。runtime 按照定义的规范实现个组件之间的协作。





除了 runtime 自身，凡是可阻塞操作都需要被实现为一个 Future，如 IO read/write，sleep，mutex 等，这些基础组件的实现依赖于运行时的实现，和 runtime 强绑定。

tokio 提供了实现了 Future 的 IO 对象，Sockets，mutext，Channel等。

# Excutor , Waker,  Driver 的协作

Excutor 由 thread-pool 构成，每一个执行线程对应一个 Worker。

```
if worker 有 Notified 的 Task {
	处理 Notified Task
} else if 是否可以从其他 worker 中 steal Task {
	steal Task 并处理
} else {
	调用 Driver 的 `park`，`park_timeout`，park 将调用 Reator poll。
    if 如果有就绪的 IO 事件 {
       	调用 Task 的 waker， 将 Task 推入执行队列，excutor 从 park 中返回，并执行 Task
    } else {
       	阻塞等待 Ready 事件
    }
}
```

不同于传统的 Reactor 模型，由一个单独的线程监听 Ready 事件，并分发到线程池。tokio 的所有 worker 职责是一样的，都会 poll reactor。

Reactor 只有在被 poll 时，才会有执行 Driver 相关代码的机会，如果 Worker 的 run queue 有足够多的 Task 待执行，即便 Driver 注册的事件 Ready，也不会立马被推入可执行队列。  

​     

当 Driver 被 poll 时，通过调用注册事件关联的 Waker 的 waker 函数，再通过 waker 关联的 Task，将 Task 放回执行者队列。

对 Driver 来说，并不需要关心如何将 Task 加入执行者队列，具体实现由 Waker 定义的虚函数实现。

Waker 是 Task 流转的关键点。  

​      

Excutor 和 Driver 之间的协作，则通过定义了 Park Trait，Park Trait 抽象了 poll 模型。Driver 实现 Park，Excutor 调用相关函数，驱动 Driver 的事件被消费。在消费的过程中，使用 Waker 封装消费细节。

# Driver 的层级关系

在 tokio 的实现中，IO，Timer，Signal 等都有自己的 Driver，不同的类型的 Driver 是否启用是可配置的。

从 runtime 的视角，应该只有一个 Driver，且不需要关心哪些 Driver 被启用，这导致了 tokio 的 Driver 的层级关系，最顶层的 Driver 会驱动下层的 Driver。

```rust
// runtime Driver
pub(crate) struct Driver {
    inner: TimeDriver,
}

type TimeDriver = crate::park::either::Either<crate::time::driver::Driver<IoStack>, IoStack>;

type IoStack = crate::park::either::Either<ProcessDriver, ParkThread>;

type ProcessDriver = crate::process::unix::driver::Driver;

// ProcessDriver
pub(crate) struct Driver {
    park: SignalDriver,
    signal_handle: SignalHandle,
}

// SignalDriver
pub(crate) struct Driver {
    /// Thread parker. The `Driver` park implementation delegates to this.
    park: IoDriver
  	...
}
```

当启用所有 featurs，建立了如下的层级关系：

```
- RuntimeDriver
  - TimerDriver
    - ProcessDriver
      - SignalDriver
        - IoDriver
          - Mio
```

当不启动 IO Driver 时，建立如下的层级关系：

```
// 通过条件变量实现 Driver 的 park
- RuntimeDriver
  - TimerDriver
    - ParkThread
      - Condvar
```

# 驱动子 Driver

worker 在没有 Task 可执行时，调用 Driver 的 park 函数（类似于 epoll_wait 的作用），当没有任何 Read Task 时，Worker 线程将被阻塞，否则将 Ready 的 Task 放入可执行队列，并开始执行 Task。

Driver 判断是否有 Ready Task，不仅需要考虑自身的，还需要考虑下层级的 Driver。以 Timer Driver 实现为例：

```rust
// Timer Driver
pub(crate) struct Driver<P: Park + 'static> {
    /// Timing backend in use.
    time_source: ClockTime,

    /// Shared state.
    handle: Handle,

  	/// 子层级 Driver，所有 Driver 都需要实现 Park Trait
    /// Parker to delegate to.
    park: P,
  	...
}

fn park_internal(&mut self, limit: Option<Duration>) -> Result<(), P::Error> {
    ...
    match next_wake {
      Some(when) => {
        ...
        if duration > Duration::from_millis(0) {
					// 下一个 timer 距当前还有一定时间，对 Timer Driver 来说，是需要 park 的
          // 但子层级的 Driver 可能有 Ready。
          // 只要子层级的 Driver 有 Ready 事件，Woker将不会被阻塞。
          self.park_timeout(duration)?;
        } else {
          // 判断是否子层级的 Driver 也有 Read 的 Task，如果有的话，加入可执行队列。
          // 如果没有的话，也不阻塞，因为 Timer 有 Ready 的。
          self.park.park_timeout(Duration::from_secs(0))?;
        }
      }
      ...
    }

  	// 调用 Ready Timer 的 wake 函数，加入可执行队列
    // Process pending timers after waking up
    self.handle.process();

    Ok(())
}

fn park_timeout(&mut self, duration: Duration) -> Result<(), P::Error> {
    let clock = &self.time_source.clock;

    if clock.is_paused() {
     ...
    } else {
      // 调用子层级 Driver 的 park，如果子 Driver 无 Ready，将 park duration 时间
      // 到时最近的 Timer 将会到期
      self.park.park_timeout(duration)?;
    }

    Ok(())
}

```

所有的 Driver 实现都遵循一样的模式，IO Driver 的底层是 mio, 且属于最底层的 Driver，当没有任务 Ready 事件时，将阻塞在 mio 的 poll 上

```rust
fn turn(&mut self, max_wait: Option<Duration>) -> io::Result<()> {
    ...
    // Block waiting for an event to happen, peeling out how many events
    // happened.
    match self.poll.poll(&mut events, max_wait) {
      Ok(_) => {}
      Err(ref e) if e.kind() == io::ErrorKind::Interrupted => {}
      Err(e) => return Err(e),
    }
    ...
    Ok(())
}
```





