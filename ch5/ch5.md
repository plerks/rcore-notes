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

## 关于stride调度算法
[stride算法](https://learningos.cn/rCore-Camp-Guide-2025S/chapter5/4exercise.html#stride)的思路为：

1. 为每个进程赋予一个 stride 值，每个进程有自己的优先级 priority。进程 stride 值初始为0，stride 调度要求进程优先级 >= 2。

2. 要选出进程调度时，选择 stride 值最小的那个进程。

3. 对于获得调度的进程，将对应的 stride 加上其对应的步长 pass = BigStride / priority。BigStride 表示一个预先定义的较大常数。

每次选最小的 stride 出来运行，而每次加上的步长为 BigStride / priority，所以进程的 priority 值越大，stride 值增加越慢，越容易得到执行。

关于stride算法对stride值可能发生溢出的比较处理，直接把[报告](https://github.com/LearningOS/2025s-rcore-plerks/blob/ch5/reports/lab3.md)中的笔记贴过来：

---

我们要比较两个stride值的大小，但是stride值随着不断+pass，是有可能溢出的，如何处理？

首先，参考[rcore-camp-guide](https://learningos.cn/rCore-Camp-Guide-2025S/chapter5/4exercise.html)，
stride 调度要求进程优先级 >= 2，初始各进程stride值都为0，每次选最小的stride出来，加上步进值pass = BigStride / priority，所以系统变化过程中会维持：

STRIDE_MAX – STRIDE_MIN <= BigStride / 2

在没有发生溢出的情况下，我们是能正确比较stride值的大小的，但是问题在于进程运行时间久了stride值可能溢出，例如当两个进程stride值`{1111_1110, 1111_1111}` -> `{(1)0000_0000, 1111_1111}`时(第一个stride值加了pass 2)，
无符号数比较会认为0000_0000小从而调度运行它，实际上0000_0000是变大越界的数。

把stride值当成有符号数来比较也是不行的，虽然上面的边界情况能处理，但是对于`{0111_1110, 0111_1111}` -> `{1000_0000, 0111_1111}`，
会继续选择1000_0000出来运行。

那么，怎样的策略才能使得即便考虑溢出，我们也能正确进行stride值的比较？

机器数构成一个环 0000_0000 ... 0111_1111 1000_0000 ... 1111_1111

在环上，stride值都是顺时针在跑（stride值增大），只要不超过一整圈，两个stride值a, b相减就是他们之间的距离（可正可负）。

（一圈是指例如stride是u8，所有u8的数构成一个环）

于是我们考虑 a - b，把 a - b 转成有符号数，判断正负即可知道到底是a在前面还是b在前面。（也可以不转有符号数，直接判断最高位是0/1）

但是这里还有个问题，a - b 不能溢出符号位，否则判断 a - b 的正负会有问题。也就是说a不能在b前面太多，a如果在b前面1000_0000步，我们会误认为这是个负数，会误以为a在b后面。

而由STRIDE_MAX – STRIDE_MIN <= BigStride / 2，BigStride可以设为255，即保证了 a - b 不会溢出符号位，也保证了不会超过一整圈。

为什么BigStride要取上界？

不然的话有效的priority取值少，例如BigStride是3，那么不管priority怎么取，BigStride / priority只会是 1 或者 0

**总结一句话：算 a - b 的正负，且要控制差值不能溢出符号位。**

通过判断 (signed)(a - b) 的正负，本来要无限位长来记stride值，现在不用了。

---

BigStride / priority 得到的结果值域不全，例如 BigStride 为255，那么能得到的商依次是 127, 85, 63...

是不是可以改成直接 stride += priority 而不是 stride += BigStride / priority ?

这样虽然"priority值的大小"与"容易获得调度"相反了，但是能增加步进值的取值种数。

补充：[rcore-camp-guide](https://learningos.cn/rCore-Camp-Guide-2025S/chapter5/4exercise.html)有提：

> 可以证明，如果令 P.pass = BigStride / P.priority，其中 P.priority 表示进程的优先权（大于 1），而 BigStride 表示一个预先定义的大常数，则该调度方案为每个进程分配的时间将与其优先级成正比。

把步长定义为 BigStride / priority 应该是为了达到**各个进程被运行次数与 priority 成正比**的效果(即各进程的运行次数 / priority都大致相等)。

要走距离 L，步长为 BigStride / priority，于是需要的被运行次数就为 priority * (L / BigStride)，步长设计成除以 priority 能达到正比效果。设计成 + priority 会变成反比。