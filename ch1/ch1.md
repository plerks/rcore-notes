# ch1代码逻辑分析

[仓库地址](https://github.com/LearningOS/2025s-rcore-plerks)

ch1不需要添加内容。

## 最基本的运行
把/os/src/先删掉，其它保持不动，其实只需要4个文件就能运行起来，只是没有任何输出。

注意：

1. /rust-toolchain.toml里的`channel = "nightly-2024-05-02" # 这行需要，必须用这个版本，不然要改写法`

2. /os/cargo.toml里的`edition = "2021"`都是需要的

不然会因为编译器版本和使用的feature问题有报错。

ch1最基本的4个文件为：main.rs, lang_items.rs, linker.ld, entry.asm

* main.rs是主逻辑，lang_items.rs里定义了必须定义的rust panic时的处理函数

* lingker.ld指导链接器如何链接，其中`ENTRY(_start)`向链接器指明了入口是`_start`这个函数

* entry.asm是最开始的入口，里面定义了`_start`，并且`_start`直接调用了`rust_main`这个函数（在main.rs中定义）

这4个文件详细注释如下：

main.rs:
```Rust
#![no_std]
#![no_main]

use core::arch::global_asm;

mod lang_items;

// 用汇编写的整个函数的入口，global_asm会把汇编文件的内容展开在这里，里面定义了linker.ld中规定的入口_start。Rust编译器不会自动处理.s/.S/.asm等汇编文件，所以用global_asm!在.rs中引入，这样汇编代码就会生成在.o文件中。
global_asm!(include_str!("entry.asm"));

#[no_mangle]
extern "C" fn rust_main() {
    loop {};
}
```

lang_items.rs:
```Rust
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

linker.ld:
```ld
OUTPUT_ARCH(riscv)
ENTRY(_start) /* 指明程序入口是_start这个函数 */
BASE_ADDRESS = 0x80200000; /* 程序加载进内存的基地址 */

SECTIONS
{
    . = BASE_ADDRESS; /* .ld里的".", 代表当前地址指针 */
    skernel = .;

    stext = .; /* 接着马上要放.text段，把.text段的开始地址放到stext这个变量中，这个变量外部可见，汇编或者rust文件里可以使用 */
    .text : { /* .section_name : { ... }的写法代表，在当前位置创建一个段并放入内容 */
        *(.text.entry) /* "*"代表通配符，代表所有符合条件的文件部分，比方说这条规则相当于把所有.o .s文件中定义的.text.entry段放到这里。段命名可以带.，段的名字就是.text.entry */
        *(.text .text.*)
    }

    . = ALIGN(4K); /* 将当前位置对齐到 4KB 的整数倍 */
    etext = .;
    srodata = .;
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }

    . = ALIGN(4K);
    erodata = .;
    sdata = .;
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }

    . = ALIGN(4K);
    edata = .;
    .bss : {
        *(.bss.stack)
        sbss = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
    }

    . = ALIGN(4K);
    ebss = .;
    ekernel = .;

    /DISCARD/ : { /* 丢弃.eh_frame段，.eh_frame段存储与异常处理和堆栈展开相关的信息，通常用于调试和异常处理框架 */
        *(.eh_frame)
    }
}
```

entry.asm:
```asm
    .section .text.entry # 表示接下来的代码放置在 .text.entry 段中
    .globl _start # 将 _start 符号声明为全局符号，使得它可以被链接器和其他代码访问
_start:
    la sp, boot_stack_top # 栈顶放到sp中，RISC-V中sp是x2的别名
    call rust_main # 调用写的rust_main函数

    .section .bss.stack
    .globl boot_stack_lower_bound
boot_stack_lower_bound:
    .space 4096 * 16 # 栈大小
    .globl boot_stack_top
boot_stack_top: # boot_stack_top代表栈顶，boot_stack_top的定义在la指令后面，这样也行，汇编程序会先解析所有符号
```

在os/文件夹下，`cargo build`之后`LOG=DEBUG make run`能运行，但是由于main.rs中rust_main()没有打印任何东西，只是单纯死循环了，所以不会有任何输出。只会风扇狂转，无法退出。

如何退出这个死循环？

ctrl c没用，得先ctrl A，然后按x，这样qemu才会退出。

原因见chatgpt的回答：
```
在运行 QEMU 时，它要模拟一个操作系统（比如 Linux），而这些系统也会使用自己的快捷键，比如 Ctrl + C、Ctrl + Alt + Del 等。

为了防止你的 主机的快捷键干扰虚拟机，QEMU 设置了一个“前缀键”，让你可以先告诉 QEMU：“我要对你下命令，而不是对虚拟机操作”。

Ctrl + A 是 QEMU 的“命令前缀键”
当你按下 Ctrl + A，QEMU 并不直接执行什么动作，而是等待你再按一个键，来决定执行哪条命令。

然后按 X：退出 QEMU
在按下 Ctrl + A 之后再按 X，QEMU 识别为“退出模拟器”的命令
```

## 添加print功能
需要自己实现打印的功能吗？

否，Makefile里build时有个`-bios $(BOOTLOADER)`参数，变量展开后是`-bios ../bootloader/rustsbi-qemu.bin`，意味着硬件加载了一个BootLoader程序，即RustSBI。这个.bin文件是运行在 M-mode的固件，项目里已经提供了，是写好的rustsbi。SBI（Supervisor Binary Interface）是RISC-V架构规范中定义的一套标准接口，RustSBI是使用Rust编写的一个SBI的实现。SBI是硬件和操作系统的中间层，例如[rCore-Camp-Guide-2025S 文档
](https://learningos.cn/rCore-Camp-Guide-2025S/chapter1/4mini-rt-baremetal.html)上的这个图：

![img](img/chap1-intro.png)

所以，通过ecall调sbi就能打印！

[关于特权级](https://learningos.cn/rCore-Camp-Guide-2025S/chapter1/4mini-rt-baremetal.html#id3)，指导书有说明：

应用程序访问操作系统提供的系统调用的指令是ecall，操作系统访问 RustSBI提供的SBI调用的指令也是ecall， 虽然指令一样，但它们所在的特权级是不一样的。简单地说，应用程序位于最弱的用户特权级（User Mode）， 操作系统位于内核特权级（Supervisor Mode），RustSBI位于机器特权级（Machine Mode）。

sbi.rs:
```Rust
/// general sbi call
#[inline(always)]
fn sbi_call(which: usize, arg0: usize, arg1: usize, arg2: usize) -> usize {
    let mut ret;
    unsafe {
        asm!(
            "li x16, 0", // legacy：指定 SBI extension（新版可能不需要这行）
            "ecall", // 触发进入 M-mode 的 RustSBI 处理器
            inlateout("x10") arg0 => ret, // a0：第一个参数，执行后把x10的值读出来，赋值给ret。也可以写成 in("x10") arg0, out("x10") ret
            in("x11") arg1, // a1：第二个参数
            in("x12") arg2, // a2：第三个参数
            in("x17") which, // a7：系统调用号（哪种 SBI 调用）
        );
    }
    ret
}

/// use sbi call to putchar in console (qemu uart handler)
pub fn console_putchar(c: usize) {
    sbi_call(SBI_CONSOLE_PUTCHAR, c, 0, 0);
}
```
console_putchar()通过sbi_call()指定调用号为常量SBI_CONSOLE_PUTCHAR，实现了打印单个字符c的功能。

[risc-v寄存器别名](https://blog.csdn.net/qq_43194080/article/details/132506802)，X10 ~ X17寄存器别名a0 ~ a7，用于函数调用，传递参数和返回值。

## 回到rust_main()
```Rust
extern "C" {
    fn stext(); // begin addr of text segment
    fn etext(); // end addr of text segment
    fn srodata(); // start addr of Read-Only data segment
    fn erodata(); // end addr of Read-Only data ssegment
    fn sdata(); // start addr of data segment
    fn edata(); // end addr of data segment
    fn sbss(); // start addr of BSS segment
    fn ebss(); // end addr of BSS segment
    fn boot_stack_lower_bound(); // stack lower bound
    fn boot_stack_top(); // stack top
}
```
这里写成函数似乎是rust的要求，stext表示在linker.ld中定义的.text段的开始地址。ai的说法是：
```
在 Rust/C 代码中，这些符号被声明为“函数”是因为：

语法兼容性：链接器符号本质上是地址值，但 Rust/C 无法直接声明“裸地址”。通过声明为无参数的函数（fn()），可以方便地获取它们的地址（即函数指针）。

也可以写成：extern "C" { static stext: u8; }

// 方式 1
extern "C" {
    fn stext();
}
let addr = stext as usize;

// 方式 2
extern "C" {
    static stext: u8;
}
let addr = &stext as *const u8 as usize;
```

```Rust
debug!(
    "[kernel] .rodata [{:#x}, {:#x})",
    srodata as usize, erodata as usize
);
```
用debug!宏函数输出内容，用的是Rust的log crate，这里`:#x`代表带0x前缀的十六进制格式。具体的自定义逻辑在logging.rs中，可以像平常一样使用info!, debug!, error! 等宏，但输出行为由自定义的SimpleLogger控制。

logging.rs里是如何控制颜色的？
```Rust
println!(
    "\u{1B}[{}m[{:>5}] {}\u{1B}[0m",
    color,
    record.level(),
    record.args(),
);
```
这个println!不是std的，是console.rs中通过console_putchar()实现的。具体的颜色规则为[ANSI转义序列](https://zh.wikipedia.org/wiki/ANSI%E8%BD%AC%E4%B9%89%E5%BA%8F%E5%88%97)，更详细的[rCore-Tutorial-Book-v3 3.6.0-alpha.1 文档](https://rcore-os.cn/rCore-Tutorial-Book-v3/chapter1/7exercise.html)中有提，[rCore-Camp-Guide-2025S 文档
](https://learningos.cn/rCore-Camp-Guide-2025S/chapter1/4mini-rt-baremetal.html)没有。

注意模式字符串中有3个占位符 {} {:>5} {}

1. 第一个{}填颜色，`\u{1B}[31m`这种形式是ANSI的颜色控制码，这里31代表红色。

2. {:>5}填日志等级，例如DEBUG、INFO等，{:>5}表示右对齐，宽度是 5个字符。

3. 第二个{}填具体内容

4. 最后`\u{1B}[0m`取消染色，不影响后面内容的颜色

rust_main()最后的部分：
```Rust
use crate::board::QEMUExit;
    crate::board::QEMU_EXIT_HANDLE.exit_success(); // CI autotest success
    //crate::board::QEMU_EXIT_HANDLE.exit_failure(); // CI autoest failed
```
这个crate::board::QEMUExit是os/boards/qemu.rs，main.rs上面写了：
```Rust
#[path = "boards/qemu.rs"]
mod board;
```
功能是：调用exit_success()让qemu退出并报告运行成功。