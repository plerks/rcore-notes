ch1的代码在`os/`下，是实现操作系统的代码。

ch2说明了要在原仓库下clone一个子仓库，这个子仓库为`user/`，在原仓库的.gitignore里，所以不会影响原仓库。
```Shell
$ git clone https://github.com/LearningOS/rCore-Camp-Code-2025S.git
$ cd rCore-Camp-Code-2025S
$ git checkout ch2
$ git clone https://github.com/LearningOS/rCore-Tutorial-Test-2025S.git user
```

clone之后新增了一个user/文件夹，是用户级的程序，对于chapter 2，user/是示范的用户级程序，将由os/以多道批处理系统的方式运行这些用户级程序。

## 启动执行用户程序

相比ch1，main.rs只是最后多了3行：

```Rust
trap::init();
batch::init();
batch::run_next_app();
```

`trap::init();`初始化了trap的响应处理逻辑，`batch::init();`初始化了用户批程序。`batch::run_next_app();`真正开启了第一个用户程序的执行，如果发生trap(例如用户程序非法或者执行完了)，根据我们已经准备好的trap处理逻辑，切换到下一个用户程序继续执行。

---

实现批处理的功能，具体的分析如下：

## user/
参考: 

* [Rust - cargo项目里多个二进制binary crate的编译运行](https://blog.csdn.net/weixin_44539199/article/details/134570516)
* [rust 模块组织结构](https://www.cnblogs.com/li-peng/p/13587910.html)

user/这个项目，src/下是lib.rs，但是有src/bin，里面是多个用户代码文件，例如：
```Rust
hello_world：在屏幕上打印一行 Hello, world!

bad_address：访问一个非法的物理地址，测试批处理系统是否会被该错误影响

power：不断在计算操作和打印字符串操作之间切换
```
这是cargo的多二进制crate的声明方式，把各个二进制crate源码放在src/bin目录下，可以引用src/lib.rs的内容。

src/bin/*.rs的每个文件都会被自动识别为一个独立的可执行程序（binary crate）

这时候：

src/lib.rs 是这个crate的“主库”

src/bin/hello.rs是一个独立的程序，它可以用extern crate user_lib;来引用它同项目的库部分。

[rcore tutorial](https://learningos.cn/rCore-Camp-Guide-2025S/chapter2/2application.html#id2)有说明：

user_lib这个外部库其实就是user目录下的lib.rs以及它引用的若干子模块。在user/Cargo.toml中我们对于库的名字进行了设置： name =  "user_lib" 。它作为bin目录下的源程序所依赖的用户库，等价于其他编程语言提供的标准库。

在user目录下make build：

1. 对于src/bin下的每个应用程序，在target/riscv64gc-unknown-none-elf/release目录下生成一个同名的 ELF可执行文件；

2. 使用objcopy二进制工具删除所有ELF header和符号，得到.bin后缀的纯二进制镜像文件。它们将被链接进内核，并由内核在合适的时机加载到内存。

## os/
os/src/link_app.S里写明了将user/src/bin中的文件生成的.bin放入.data段中的逻辑，最后各自有一对全局符号app_*_start, app_*_end指示各个用户程序的开始和结束位置。

最终得到的布局为：
| 符号  | 含义 |
| ---   | --- |
| _num_app  | 数量 7 |
| app_0_start  | 第 0 个 app 的起始地址 |
| app_1_start  | 第 1 个 app 的起始地址 |
| ... | ... |
| app_6_start | 第 6 个 app 的起始地址 |
| app_6_end | 第 6 个 app 的结束地址 |

然后紧接着就是二进制内容了。

os/src/batch.rs中定义了AppManager，记录了user各个程序的基本信息，初始化逻辑为：
```Rust
lazy_static! {
    static ref APP_MANAGER: UPSafeCell<AppManager> = unsafe {
        UPSafeCell::new({
            extern "C" {
                fn _num_app();
            }
            let num_app_ptr = _num_app as usize as *const usize; // 这里读出了link_app.S中定义的_num_app的地址
            let num_app = num_app_ptr.read_volatile();
            let mut app_start: [usize; MAX_APP_NUM + 1] = [0; MAX_APP_NUM + 1];
            let app_start_raw: &[usize] =
                core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1);
            app_start[..=num_app].copy_from_slice(app_start_raw);
            AppManager {
                num_app,
                current_app: 0,
                app_start,
            }
        })
    };
}
```

这样AppManager就记录了各个用户程序的基本信息：
```Rust
struct AppManager {
    num_app: usize,
    current_app: usize,
    app_start: [usize; MAX_APP_NUM + 1],
}
```
AppManager还实现了个load_app()，load指定id的用户程序。

## 内核栈与用户栈
为什么需要两个栈？

现在os/要跑user/下定义的用户程序，用户程序运行时需要一个用户栈，而os/这个内核调度用户程序的运行也需要一个栈，所以各自要有个栈。

os/src/batch.rs中以全局数组的方式定义了内核栈与用户栈：
```Rust
static KERNEL_STACK: KernelStack = KernelStack {
    data: [0; KERNEL_STACK_SIZE],
};
static USER_STACK: UserStack = UserStack {
    data: [0; USER_STACK_SIZE],
};
```

## Trap与换栈
当用户程序执行发生trap后，我们的os/需要处理trap，本章的处理方式为识别trap类型，并load下一个用户程序运行。

用户程序为什么会发生trap？

user/src/bin/ch2b_bad_address.rs:
```Rust
pub fn main() -> isize {
    unsafe {
        #[allow(clippy::zero_ptr)]
        (0x0 as *mut u8).write_volatile(0);
    }
    panic!("FAIL: T.T\n");
}
```
这里在写0地址，会触发trap。

user/src/bin/ch2b_bad_instructions.rs:
```Rust
pub fn main() -> ! {
    unsafe {
        core::arch::asm!("sret");
    }
    panic!("FAIL: T.T\n");
}
```
这里执行sret指令，但是sret是S特权级的指令，所以会触发trap。

user/src/bin/ch2b_power_3.rs:
```Rust

fn main() -> i32 {
    let p = 3u64;
    let m = 998244353u64;
    let iter: usize = 200000;
    let mut s = [0u64; LEN];
    let mut cur = 0usize;
    s[cur] = 1;
    for i in 1..=iter {
        let next = if cur + 1 == LEN { 0 } else { cur + 1 };
        s[next] = s[cur] * p % m;
        cur = next;
        if i % 10000 == 0 {
            println!("power_3 [{}/{}]", i, iter);
        }
    }
    println!("{}^{} = {}(MOD {})", p, iter, s[cur], m);
    println!("Test power_3 OK!");
    0
}
```
这个运行结束后，又是怎么触发trap，让os/运行下一个任务的？

user/src/lib.rs中的_start函数末尾是这样的：`exit(main(argc, v.as_slice()));`。执行结束后就调用exit()系统调用，把退出码送给内核，触发ecall → trap → 内核执行 run_next_app()切换下一个程序。注意user/src/bin下是cargo的多二进制目标编译，最后每个.bin (用户程序)都会有_start函数作为入口。

具体的换栈与恢复逻辑在os/src/trap/trap.S，主要有两个函数：

1. __alltraps 切换到内核栈并调用trap_handler函数处理

2. __restore 恢复到用户栈

添加os/src/trap/trap.S注释如下：
```asm
.altmacro
.macro SAVE_GP n
    sd x\n, \n*8(sp)
.endm
.macro LOAD_GP n
    ld x\n, \n*8(sp)
.endm
# SAVE_GP 和 LOAD_GP为保存和恢复寄存器的宏，SAVE_GP 5等价于 sd x5, 5*8(sp)，也就是把寄存器 x5 的值存入 sp + 5*8 的地址上（一个寄存器8字节）
# LOAD_GP 5等价于 ld x5, 5*8(sp)
    .section .text
    .globl __alltraps
    .globl __restore
    .align 2
__alltraps:
    csrrw sp, sscratch, sp # csrrw的作用为 sp <- sscratch, sscratch <- sp，sscratch保存了内核栈顶，这行作用为用sscratch暂存用户栈顶sp，并将sp切换为内核栈顶sscratch。即交换sscratch和sp的值，从而实现用户栈 -> 内核栈
    # now sp->kernel stack, sscratch->user stack
    # allocate a TrapContext on kernel stack
    addi sp, sp, -34*8 # 一共 34 * 8 = 272 字节，从栈中分配一块区域，用来保存寄存器上下文，一共34个寄存器
    # save general-purpose registers
    sd x1, 1*8(sp) # 保存ra
    # skip sp(x2), we will save it later
    sd x3, 3*8(sp) # 保存gp
    # skip tp(x4), application does not use it
    # save x5~x31
    .set n, 5 # 设置一个汇编器的临时变量n = 5(不会出现在.o文件中)
    .rept 27 # 重复这段代码27次，批量保存 x5 到 x31 的通用寄存器
        SAVE_GP %n
        .set n, n+1
    .endr
    # we can use t0/t1/t2 freely, because they were saved on kernel stack
    csrr t0, sstatus
    csrr t1, sepc
    sd t0, 32*8(sp)
    sd t1, 33*8(sp) # 保存sstatus和sepc。sd 指令只能存储通用寄存器（x寄存器，x0–x31），不能直接操作 CSR（控制和状态寄存器），所以要经过t0和t1
    # read user stack from sscratch and save it on the kernel stack
    csrr t2, sscratch
    sd t2, 2*8(sp)
    # set input argument of trap_handler(cx: &mut TrapContext)
    mv a0, sp # a0指向TrapContext，保存了寄存器的值供trap_handler()使用，不能让trap_handler()当场读，因为发生调用trap_handler()，寄存器的值会变，trap_handler()也会用寄存器
    call trap_handler # 调用mod.s中的trap_handler()

# __restore是__alltraps的反过程

__restore:
    # case1: start running app by __restore
    # case2: back to U after handling trap
    mv sp, a0
    # now sp->kernel stack(after allocated), sscratch->user stack
    # restore sstatus/sepc
    ld t0, 32*8(sp)
    ld t1, 33*8(sp)
    ld t2, 2*8(sp)
    csrw sstatus, t0
    csrw sepc, t1
    csrw sscratch, t2
    # restore general-purpuse registers except sp/tp
    ld x1, 1*8(sp)
    ld x3, 3*8(sp)
    .set n, 5
    .rept 27
        LOAD_GP %n
        .set n, n+1
    .endr
    # release TrapContext on kernel stack
    addi sp, sp, 34*8
    # now sp->kernel stack, sscratch->user stack
    csrrw sp, sscratch, sp
    sret

```
[指导书](https://learningos.cn/rCore-Camp-Guide-2025S/chapter2/4trap-handling.html#trap)也有详细说明trap.S。

具体的trap响应由os/src/trap/mod.rs中的trap_handler()执行。

注意：__alltraps和__restore一个处理用户程序发生的trap，一个去恢复执行用户程序，这两个函数都是由os/执行的。

这里还有个问题，也是[指导书ch2](https://learningos.cn/rCore-Camp-Guide-2025S/chapter2/4trap-handling.html)末尾的思考题：
```asm
__alltraps:
    csrrw sp, sscratch, sp
```
这里交换sscratch和sp实现换栈，那么sscratch的初值是何时被赋予的？为什么这个时候sscratch已经是内核栈栈顶了？

main.rs的rust_main()末尾通过`batch::run_next_app();`开启了以多道批的方式运行用户程序。而run_next_app里是会调用__restore的：
```Rust
unsafe {
    __restore(KERNEL_STACK.push_context(TrapContext::app_init_context(
        APP_BASE_ADDRESS,
        USER_STACK.get_sp(),
    )) as *const _ as usize);
}
```
push_context()往内核栈里推了个TrapContext进去供__restore消费，且push_context的返回值为新内核栈栈顶。
```Rust
impl KernelStack {
    fn get_sp(&self) -> usize {
        self.data.as_ptr() as usize + KERNEL_STACK_SIZE
    }
    pub fn push_context(&self, trap_cx: TrapContext) -> usize {
        let trap_cx_ptr = (self.get_sp() - core::mem::size_of::<TrapContext>()) as *mut TrapContext;
        unsafe {
            *trap_cx_ptr = trap_cx;
        }
        trap_cx_ptr as usize
    }
}
```

push_context之后内核栈顶往下移动了sizeof(TrapContext)，trap_cx_ptr(最新的内核栈顶位置)作为参数通过a0传递给__restore函数，而__restore函数：
```asm
__restore:
    # case1: start running app by __restore
    # case2: back to U after handling trap
    mv sp, a0 # 把最新的内核栈顶赋给sp，现在sp指向内核栈顶，现在为用户程序的执行恢复上下文
    # now sp->kernel stack(after allocated), sscratch->user stack
    # restore sstatus/sepc
    ld t0, 32*8(sp)
    ld t1, 33*8(sp)
    ld t2, 2*8(sp)
    csrw sstatus, t0
    csrw sepc, t1 # sepc现在是用户程序要执行的下一条指令的地址
    csrw sscratch, t2
    # restore general-purpuse registers except sp/tp
    ld x1, 1*8(sp)
    ld x3, 3*8(sp)
    .set n, 5
    .rept 27
        LOAD_GP %n
        .set n, n+1
    .endr
    # release TrapContext on kernel stack
    addi sp, sp, 34*8 # 把内核栈顶的TrapContext释放掉，TrapContext的大小就是34 * 8
    # now sp->kernel stack, sscratch->user stack
    csrrw sp, sscratch, sp # 交换之后sp指向用户栈，而sscratch指向内核栈
    sret # 执行 sret（Supervisor Return）指令时，机器会自动跳转到 sepc 中保存的地址，这样就切换到执行用户程序了
```

__restore末尾执行了`csrrw sp, sscratch, sp`，这时候的sp是内核栈顶！于是，交换完成后sscratch就是内核栈顶了。

## 发生trap时要做什么解决了，如何告诉机器发生trap时要进入__alltraps函数呢？
[指导书](https://learningos.cn/rCore-Camp-Guide-2025S/chapter2/4trap-handling.html#trap-hw-mechanism)：

CPU会跳转到stvec所设置的Trap处理入口地址，并将当前特权级设置为S，然后从Trap处理入口地址处开始执行。

而在os/src/trap/mod.rs中：
```Rust
global_asm!(include_str!("trap.S")); // 展开trap.S

/// initialize CSR `stvec` as the entry of `__alltraps`
pub fn init() {
    extern "C" {
        fn __alltraps();
    }
    unsafe {
        stvec::write(__alltraps as usize, TrapMode::Direct);
    }
}
```
这里stvec::write就将stvec寄存器写为了__alltraps的地址。

## os程序和用户程序的具体地址在哪？
os/src/linker.ld开头：`BASE_ADDRESS = 0x80200000;`，所以os的代码放在了0x80200000的位置。

用户程序在哪里则有些复杂：

首先，os/src/main.rs中包含进了link_app.S: `global_asm!(include_str!("link_app.S"));`。link_app.S上面已经分析过。link_app.S中通过`app_5_start: .incbin "../user/build/bin/ch2b_power_5.bin"`这样的代码把用户程序的纯二进制链接进来了，地址会被app_5_start这个符号记录。而在AppManager(os/src/batch.rs/batch.rs)中，`UPSafeCell::new({ ...`那段通过汇编定义的符号找到了各个用户程序段的位置。然后：
```Rust
let app_src = core::slice::from_raw_parts(
    self.app_start[app_id] as *const u8,
    self.app_start[app_id + 1] - self.app_start[app_id],
);
let app_dst = core::slice::from_raw_parts_mut(APP_BASE_ADDRESS as *mut u8, app_src.len());
app_dst.copy_from_slice(app_src);
```
app_src是用户程序原本在的位置，通过app_dst.copy_from_slice，把用户程序加载到了APP_BASE_ADDRESS (0x80400000) 处进行运行。这就是批处理！

实际上，AppManager.print_app_info()已经写了打印用户程序各段位置的代码，`LOG=DEBUG make run`，前面的输出里就有：
```Shell
...
[kernel] num_app = 7
[kernel] app_0 [0x8020a048, 0x8020e0f0)
[kernel] app_1 [0x8020e0f0, 0x80212198)
[kernel] app_2 [0x80212198, 0x80216240)
[kernel] app_3 [0x80216240, 0x8021a2e8)
[kernel] app_4 [0x8021a2e8, 0x8021e390)
[kernel] app_5 [0x8021e390, 0x80222438)
[kernel] app_6 [0x80222438, 0x802264e0)
...
```