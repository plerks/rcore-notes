# ch8实现线程，锁，信号量，条件变量
ch8实现线程之后，ProcessControlBlock为进程控制块，TaskControlBlock为线程控制块。

[实验报告](https://github.com/LearningOS/2025s-rcore-plerks/blob/ch8/reports/lab5.md)，里面有ch8实现的银行家算法的笔记以及另外一些描述。

ch8实现的mutex, semaphore和condition都是操作系统级别的资源。

## Mutex
os/src/sync/mutex.rs中实现了两种方式的mutex。

MutexSpin，当线程lock()获取锁失败后，调用`suspend_current_and_run_next()`

MutexBlocking，当线程lock()获取锁失败后，调用`block_current_and_run_next()`

先说`block_current_and_run_next()`与`suspend_current_and_run_next()`，`block_current_and_run_next()`会将当前线程从调度队列ready_queue中拿出，放入相应mutex的阻塞队列，因此线程不会得到调度，得等待mutex唤醒它。而`suspend_current_and_run_next()`只是会进行任务切换，任务仍在ready_queue中，能被再次调度到(也就是说获取锁失败后，仍然可能再被调度，线程可能再次醒来得到执行权)。

对于MutexSpin，lock()失败后会被切换，而不会被阻塞。所以MutexSpin.lock()要在一个死循环`loop`中尝试获取锁，线程得到调度后要重新读取锁的状态并尝试获取锁。

对于MutexBlocking，lock()失败后会被阻塞，当前线程会被从调度队列ready_queue中取出，放入mutex的阻塞队列中。线程不会得到调度，而是等待MutexBlocking.unlock()，唤醒mutex阻塞队列中的第一个线程。MutexBlocking.unlock()，如果unlock时发现阻塞队列中有线程，那么其不会把锁标记为释放，而是保持记录为锁住状态，然后唤醒线程，MutexBlocking.lock()中**没有**一个死循环loop，被唤醒后线程从`block_current_and_run_next()`中返回后，lock()也直接返回，相当于认为自己直接获取了锁(事实也正如此)，或者说锁直接转移给了被唤醒的那个线程。

## Semaphore
os/src/sync/semaphore.rs

实现的semaphore同样有一个阻塞队列，线程semaphore.down()失败后会被从调度队列ready_queue中取出，放入对应semaphore的阻塞队列，然后由semaphore.up()唤醒阻塞队列中的线程。避免线程无意义地重试。

## Condition
os/src/sync/condvar.rs

这里只记录下，条件变量wait()时会释放锁，是怎么体现的，Condvar.wait()的代码:
```Rust
/// blocking current task, let it wait on the condition variable
pub fn wait(&self, mutex: Arc<dyn Mutex>) {
    trace!("kernel: Condvar::wait_with_mutex");
    mutex.unlock(); // 条件变量的特性，wait()时释放锁
    let mut inner = self.inner.exclusive_access();
    inner.wait_queue.push_back(current_task().unwrap());
    drop(inner);
    block_current_and_run_next();
    mutex.lock();
}
```

Condvar.signal()的唤醒:
```Rust
/// Signal a task waiting on the condition variable
pub fn signal(&self) {
    let mut inner = self.inner.exclusive_access();
    if let Some(task) = inner.wait_queue.pop_front() {
        wakeup_task(task);
    }
}
```

## exit与孤儿进程
参考[指导书](https://learningos.cn/rCore-Camp-Guide-2025S/chapter8/1thread-kernel.html#id5)，rcore ch8的`exit`系统调用设计为用来退出线程，如果主线程调用了`exit`，则直接退出整个进程，系统调用`waittid`也变成了：

```Rust
/// 参数：tid表示线程id
/// 返回值：如果线程不存在，返回-1；如果线程还没退出，返回-2；其他情况下，返回结束线程的退出码
pub fn sys_waittid(tid: usize) -> i32
```
（不过我记得一般的exit()，无论哪个线程调用都是直接退出整个进程）

在`sys_exit` -> `exit_current_and_run_next`中，会判断tid是否为0，如果为0，则意味着退出进程，这会把子进程变为孤儿进程，然后由init进程收养，这个过程的代码如下：

```Rust
// os/src/task/mod.rs exit_current_and_run_next()

{
    // move all child processes under init process
    let mut initproc_inner = INITPROC.inner_exclusive_access();
    for child in process_inner.children.iter() {
        child.inner_exclusive_access().parent = Some(Arc::downgrade(&INITPROC));
        initproc_inner.children.push(child.clone());
    }
}
```

# rcore的单核性
整个rcore的代码，都是建立在单核，且内核代码不会被打断的假设前提下(见ch3说的[嵌套中断](https://learningos.cn/rCore-Camp-Guide-2025S/chapter3/4time-sharing-system.html#risc-v)问题)。

注意rcore代码里常见的`inner_exclusive_access()`，只是返回UPSafeCell的一个可变借用。rcore的UPSafeCell是对rust的[RefCell](https://course.rs/advance/smart-pointer/cell-refcell.html)的简单封装(UP应该是uniprocessor的缩写)，RefCell保证了同一时间只存在一个可变借用(运行时也会有借用检查，里面有个借用计数字段)，但是RefCell本身不是线程安全的。

所以`inner_exclusive_access()`只是能在单核环境下获得一个可变借用，在多核环境下不是安全的。如果进程接下来要被`__switch`切换，需要先手动`drop(inner)`。不然另外一个进程在内核态再次获得inner时，会被rust查出来同时存在两个可变借用panic。