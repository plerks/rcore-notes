# ch5实现shell并以shell为入口等待接收用户操作
ch5实现了fork, exec等系统调用，不再像之前一样内核启动后直接执行确定的程序，而是可以通过shell接收用户操作并执行。

## ch5的执行流程是怎样的？
os/src/main.rs中，会去调`task::add_initproc();`，这会将INITPROC加到任务队列中，这就是init进程。这个init进程对应的文件在user/src/bin/ch5b_initproc.rs，其会fork自己，子进程会exec ch5b_user_shell.rs变成shell，而原本的自己则用来回收僵尸进程。user_shell则接收用户输入并通过fork, exec等系统调用执行用户程序。

## 再来分析下__switch和__restore
os/src/task/processor.rs run_tasks()里`__switch(idle_task_cx_ptr, next_task_cx_ptr)`，取出的task是通过add_task()往TASK_MANAGER中添加的。

add_task()的一个被调用处在os/src/syscall/process.rs sys_fork()，对应的task是通过`current_task.fork()`构造出来的。

那么，这个构造出来的task，其TaskContext是怎样的？

在os/src/task/task.rs fork()中，TaskControlBlockInner的task_cx是通过`TaskContext::goto_trap_return(kernel_stack_top)`构造的，构造出的taskContext的ra会是trap_return。那么，当第一次__switch运行起来任务时，就会__switch -> trap_return -> __restore，这样就通过__restore把进程运行起来了，其中__switch跳到trap_return是通过**ret + ra**的机制（ret会将pc设置为ra寄存器的值），而ra该赋的值从taskContext中读出为trap_return。

__restore如何跳到用户进程入口？

通过**sret + sepc**，sret会将pc设置为sepc寄存器的值，spec需要变成的值会从TrapContext中读出，这样就实现了跳到U态开始执行用户程序。

TrapContext怎么知道的用户程序入口？

os/src/mm/memory_set.rs from_elf()会通过`elf.header.pt2.entry_point() as usize`解析出用户程序的入口并返回，在例如TaskControlBlock::new这样的函数中会调from_elf，得到的入口点会赋值给变量entry_point，然后有代码：
```Rust
// os/src/task/task.rs TaskControlBlock::new()

let trap_cx = task_control_block.inner_exclusive_access().get_trap_cx();
*trap_cx = TrapContext::app_init_context(
    entry_point,
    user_sp,
    KERNEL_SPACE.exclusive_access().token(),
    kernel_stack_top,
    trap_handler as usize,
);
```
这样头一次__restore运行用户进程时就能从trapContext中找出入口点赋值给sepc。

当一个用户进程时间片用完换出后，再次通过__switch切换重新得到时间片运行的情况和ch3相似，taskContext里有ra，触发返回退栈后会顺着执行到__restore。

整个过程和ch3类似，具体见ch3的笔记。

## sys_fork()是如何实现在父子进程中返回值不同的？
[rcore-camp-guide](https://learningos.cn/rCore-Camp-Guide-2025S/chapter5/3implement-process-mechanism.html#id5)

os/src/syscall/process.rs sys_fork():
```Rust
pub fn sys_fork() -> isize {
    trace!("kernel:pid[{}] sys_fork", current_task().unwrap().pid.0);
    let current_task = current_task().unwrap();
    let new_task = current_task.fork();
    let new_pid = new_task.pid.0;
    // modify trap context of new_task, because it returns immediately after switching
    let trap_cx = new_task.inner_exclusive_access().get_trap_cx();
    // we do not have to move to next instruction since we have done it before
    // for child process, fork returns 0
    trap_cx.x[10] = 0;
    // add new task to scheduler
    add_task(new_task);
    new_pid as isize
}
```
内核接收到想要fork的请求后，`trap_cx.x[10] = 0`将TrapContext中的a0寄存器的值修改为了0，并通过`add_task(new_task)`将子进程加到了scheduler中，这样内核完成了子进程的创建。sys_fork()会将new_pid返回到 os/src/syscall/mod.rs syscall() -> os/src/trap/mod.rs trap_handler() 中：

```Rust
// os/src/trap/mod.rs

match scause.cause() {
    Trap::Exception(Exception::UserEnvCall) => {
        // jump to next instruction anyway
        let mut cx = current_trap_cx();
        cx.sepc += 4;
        // get system call return value
        let result = syscall(cx.x[17], [cx.x[10], cx.x[11], cx.x[12]]);
        // cx is changed during sys_exec, so we have to call it again
        cx = current_trap_cx();
        cx.x[10] = result as usize;
    }
```
变量result接收到new_pid后，会把当前进程的trapContext的a0寄存器的值改成new_pid。也就是说，当前进程调用fork()进入内核态后，内核会往scheduler中加一个新的task，其trapContext中的a0为0，而当前的进程的trapContext中的a0为new_pid。这样，当内核用trapContext恢复执行用户程序后，父子进程fork()的返回值一个是new_pid，一个是0。这样就实现了fork()在父子进程中的返回值不同。

## idle控制流
### 理解下[rcore tutorial](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter5/2core-data-structures.html#idle)中说的idle控制流。

ch4中，任务个数是提前确定的，保存在了TaskManagerInner的`tasks: Vec<TaskControlBlock>`中，运行完了整个qemu就会退出。

而在ch5中，我们实现了shell并以shell为入口等待用户的操作，要运行的进程个数不是提前确定好的，而是在run_tasks()中通过fetch_task()拿到一个任务来运行。这就还需要一个执行流，即run_tasks()里的无限loop取任务来运行的执行流，**这个内核的无限loop执行流应该就是指导书中说的idle执行流**。当没有任务ready时，会执行这个loop执行流等待shell输入要执行的任务。这个loop执行流是shell的那个不需要结束的取用户程序的任务（和用户程序流不同的是其在内核）

Processor.idle_task_cx存的是什么？

第一次 run_tasks() -> loop -> fetch_task()成功 -> __switch(idle_task_cx_ptr, next_task_cx_ptr) 后，loop执行流就被保存在了idle_task_cx中，所以Processor.idle_task_cx保存了loop执行流。

如果当前进程时间片用完，会调suspend_current_and_run_next()，suspend_current_and_run_next()会把当前在运行的任务取出并add到队列末尾，然后调用schedule()。

如果当前进程调exit()结束，会调exit_current_and_run_next()，exit_current_and_run_next()做一些结束工作，此外并不会把当前任务add回去，最后也会调用schedule()。

而在 schedule() 中，__switch的调用方式为 __switch(switched_task_cx_ptr, idle_task_cx_ptr)，又切回了loop控制流，于是又回到了尝试fetch用户task运行的状态。

---

### ch4中这个idle执行流怎么体现的？

ch4不需要无限loop，ch4中的流程是 run_first_task() -> (处理Trap) -> run_next_task() -> 执行完所有用户程序后结束。而在run_first_task()中，__switch是这样调用的：

```Rust
// os/src/task/mod.rs run_first_task()

let mut _unused = TaskContext::zero_init();
// before this, we should drop local variables that must be dropped manually
unsafe {
    __switch(&mut _unused as *mut _, next_task_cx_ptr);
}
```
直接丢弃了第一次运行用户程序时的内核状态，因为不需要。
