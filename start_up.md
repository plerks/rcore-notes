# rcore的上手准备
记录一下最开始的一些文档链接以及环境配置。主要目的在于记录如何把 rcore-camp-guide 的运行环境搭建起来。

[rcore camp第二阶段](https://opencamp.cn/os2edu/camp/2025spring/stage/2)

rcore camp第二阶段就是刷[rcore-camp-guide](https://learningos.cn/rCore-Camp-Guide-2025S/0setup-devel-env.html)或者[rcore-tutorial](https://rcore-os.cn/rCore-Tutorial-Book-v3/)的阶段。rcore-camp-guide相比rcore-tutorial要简略一些，作业难度也更低。最终是要完成rcore-camp-guide的5个实验作业。

[第二阶段说明](https://github.com/LearningOS/rust-based-os-comp2025/blob/main/2025-spring-scheduling-2.md)

## 实验仓库
从[rcore camp第二阶段](https://opencamp.cn/os2edu/camp/2025spring/stage/2)的网页上可以创建实验仓库，我的仓库地址为<https://github.com/LearningOS/2025s-rcore-plerks>，在此基础上看代码并完成章节的编程作业。

## 环境配置
我的机器环境是 Ubuntu 24

环境配置文档：[rcore-camp-guide 第0章](https://learningos.cn/rCore-Camp-Guide-2025S/0setup-devel-env.html)

### 1. 安装rust
[ArceOS Tutorial Book](https://rcore-os.cn/arceos-tutorial-book/ch01-02.html)中说了要用nightly版本的rust。不过实验仓库的/rust-toolchain.toml中指定了rust的版本，运行时rustup会去下。

安装rustup，rustup是帮忙管理rust版本的：
```shell
curl https://sh.rustup.rs -sSf | sh
```

安装 rustc 的 nightly 版本，并把该版本设置为 rustc 的默认版本：

```shell
rustup install nightly
rustup default nightly
```

`rustc show`查看已安装和正在用的rust版本，我记得实验仓的运行指令会触发下特定rust版本，在/rust-toolchain.toml中有指定rust版本，`rustup install nightly`这条下载命令没有指定特定版本也不必担心。

cargo包管理器换源，`~/.cargo/config.toml`：
```toml
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"
replace-with = 'ustc'
[source.ustc]
registry = "git://mirrors.ustc.edu.cn/crates.io-index"
```

### 安装qemu
直接把[指导书第0章](https://learningos.cn/rCore-Camp-Guide-2025S/0setup-devel-env.html)中内容贴过来，注意指导书推荐用Qemu 7.0.0

```shell
# 安装编译所需的依赖包
sudo apt install autoconf automake autotools-dev curl libmpc-dev libmpfr-dev libgmp-dev \
              gawk build-essential bison flex texinfo gperf libtool patchutils bc \
              zlib1g-dev libexpat-dev pkg-config  libglib2.0-dev libpixman-1-dev git tmux python3
# 下载源码包
# 如果下载速度过慢可以使用我们提供的网盘链接：https://pan.baidu.com/s/1i3M-DjtlfBtUy0urGvsl4g
# 提取码 lnpw
wget https://download.qemu.org/qemu-7.0.0.tar.xz
# 解压
tar xvJf qemu-7.0.0.tar.xz
# 编译安装并配置 RISC-V 支持
cd qemu-7.0.0
./configure --target-list=riscv64-softmmu,riscv64-linux-user
make -j$(nproc)
```
编译过程有问题问chatgpt，我记得我是遇到了一个我系统python版本过高的问题，ubuntu又不许我改系统的python版本。然后问chatgpt，要用python的venv。

```shell
export PATH="$HOME/<Your qemu path>/qemu-7.0.0/build/:$PATH"
export PATH="$HOME/<Your qemu path>/qemu-7.0.0/build/riscv64-softmmu:$PATH"
export PATH="$HOME/<Your qemu path>/qemu-7.0.0/build/riscv64-linux-user:$PATH"
```

确认 Qemu 的版本：

```shell
qemu-system-riscv64 --version
qemu-riscv64 --version
```

### 2. 拉自己的实验仓库
```shell
git clone https://github.com/LearningOS/2025S-rcore-plerks
cd 2025S-rcore-plerks
```
实验仓库有ch1-8，8个分支，对应[指导书](https://learningos.cn/rCore-Camp-Guide-2025S/chapter1/index.html)的章节，其中3、4、5、6、8有编程作业。

我们先运行不需要处理用户代码的 ch1 分支：
```shell
git checkout ch1
cd os
LOG=DEBUG make run
```
有打印 rustsbi 的 logo 和一些日志就算成功了。

### 3. 拉user/用户程序仓库
rcore-camp-guide从[第二章](https://learningos.cn/rCore-Camp-Guide-2025S/chapter2/0intro.html#id3)开始，我们的os/需要加载用户程序运行。

拉用户程序的仓库：
```shell
$ git clone https://github.com/LearningOS/rCore-Camp-Code-2025S.git
$ cd rCore-Camp-Code-2025S
$ git checkout ch2
$ git clone https://github.com/LearningOS/rCore-Tutorial-Test-2025S.git user
```
上面的指令会将测例仓库克隆到代码仓库下并命名为 user ，注意 /user 在代码仓库的 .gitignore 文件中，因此不会出现 .git 文件夹嵌套的问题，并且你在代码仓库进行 checkout 操作时也不会影响测例仓库的内容。

在os/文件夹下，`LOG=DEBUG make run BASE=2`运行当前分支章节的所有基础用例和实验用例。os/的Makefile规则会去调用user/的Makefile规则，总之二者会联系上。

### 4. 拉ci-user/测试仓库
还有个测试方式见实验仓库/README.md的[Grading](https://github.com/LearningOS/2025s-rcore-plerks?tab=readme-ov-file#grading)那段：

```shell
# Replace <YourName> with your github ID 
$ git clone git@github.com:LearningOS/2025s-rcore-<YourName>
$ cd 2025s-rcore-<YourName>
$ rm -rf ci-user
$ git clone git@github.com:LearningOS/rCore-Tutorial-Checker-2025S ci-user
$ git clone git@github.com:LearningOS/rCore-Tutorial-Test-2025S ci-user/user
$ git checkout ch<Number>
$ cd ci-user
$ make test CHAPTER=<Number>
```
注意是要在ci-user/目录下`LOG=DEBUG make test CHAPTER=<章节号>`，这会自动把章节号往前的所有用户测例都运行了。

这个ci-user/路径也在实验仓库的.gitignore规则里。

user/或ci-user/都可以用来测试，更多说明见ch3.md，"实验的本地运行测试"那段。

---

这样就搭建好了rcore的运行环境，然后就是读[rcore-camp-guide](https://learningos.cn/rCore-Camp-Guide-2025S/chapter1/index.html)并完成编程作业。[rcore-tutorial](https://rcore-os.cn/rCore-Tutorial-Book-v3/)讲得要详细些，如果某个章节看不懂可以去看rcore-tutorial（我印象里虚拟内存那章有必要去读 rcore-tutorial）。

我的实验仓库各个章节编程代码都已经写完，所以拉下来运行是能通过所有用例的。

我仓库ch8的[ci结果](https://github.com/LearningOS/2025s-rcore-plerks/actions), basic-test成功，但是Deploy to pages阶段失败，是因为原本只有ch3、4、5、6、8有编程作业，每个100分，共500分。ch7没有编程作业，但是我ch7把前面章节代码合并了并提了个ci通过的commit，ci没写忽略ch7的逻辑，所以在ch7就有500分了。ch8 ci basic-test阶段通过之后Deploy to pages阶段会报错：
```shell
* Connection #0 to host api.opencamp.cn left intact
Response: {"result":400,"msg":"score is error"}
Error: The result field does not contain 1.
Error: Process completed with exit code 1.
```
应该是分数会算出来600分，所以拒绝了。basic-test阶段通过说明代码没问题。

补充: 把ch7上的commit force push消掉，就能让ch8的ci正常跑，现在ch8能正常通过所有ci阶段，所以ch8代码没有问题。但是我把ch7上的commit重新推回去，结果ch7 ci竟然也通过了。现在训练营网页上统计成600分了，难蹦。

## 更多

更多内容见本仓库各个章节的笔记。

[实验仓库链接](https://github.com/LearningOS/2025s-rcore-plerks)，8个章节分支，已完成所有编程作业。

[实验报告链接](https://github.com/LearningOS/2025s-rcore-plerks/tree/ch8/reports)，实验报告中也有一些笔记性的内容。