**为什么Rust可执行程序巨大**

[原文链接](https://lifthrasiir.github.io/rustlog/why-is-a-rust-executable-large.html)

译者：[Scott Huang](https://github.com/ScottHuangZL)   20160628 于厦门

假设你是一个主要用可编译语言工作的程序员。莫名其妙的或者基于一些正当的原因，你已经厌倦这些语言了，并且听说一个时髦的语言叫Rust。浏览了一些网页和官方论坛，觉得还不错，你决定试试看。Rust过去似乎不容易安装，但感谢Rustup，这个问题看起来已经解决了。 Cargo看起来也很好，所以你根据书的第一部分用新语言写了一个小的欢迎程序。
```
fn main() {
    println!("Hello, world!");
}
```

令人惊讶的是Cargo很顺畅的就编译完成。这和你过去不得不配置一个编译脚本或者make文件然后才可以编译来比简直就是某种奇迹。你会注意到在target/debug/hello目录里有编译完成的可执行文件。你本能的用 ls -al 或者 dir列一下文件，但你几乎不相信自己的眼睛：
```
$ ls -al target/debug/hello
-rwxrwxr-x 1 lifthrasiir 650711 May 31 20:00 target/debug/hello*
```
（译者Scott: windows下甚至高达1689K）
650K的文件仅仅是打印一些文字？！你记得Rust几乎是唯一的有机会取代C++的语言，并且C++被大家熟知并诟病它的代码膨胀；难道Rust无法解决这个C++的大问题？出于好奇心，你用C语言写了一个同样的程序并且编译了它。结果令人瞠目：
```
$ cat hello-c.c
#include <stdio.h>
int main() {
    printf("Hello, World!\n");
}
$ make hello-c
$ ls -al hello-c
-rwxrwxr-x 1 lifthrasiir 8551 May 31 20:03 hello-c*
```
你想也许C语言得益于拥有贴近系统内核的裸金属库。这次你用C++也用iostream写了一个同样的程序，iostream比C的原始printf要安全一些。但令人吃惊的是编译出来的程序仍然比Rust的小很多：
```
$ cat hello-cpp.cpp
#include <iostream>
using namespace std;
int main() {
    cout << "Hello, World!" << endl;
}
$ make hello-cpp
$ ls -al hello-cpp
-rwxrwxr-x 1 lifthrasiir 9168 May 31 20:06 hello-cpp*
```
Rust哪里出问题了？
看起来许多人会不爽Rust令人吃惊的编译后的大二进制文件。这个问题不是新的；在StakOverflow上搜索这个知名的虽然有点老的问题“为什么Rust编译后文件巨大”，你会找到更多类似问题。 考虑到如此多的类似疑问，我们居然还没有一篇明确的文章来阐明它确实令人意外。所以，我尝试写这篇文章。

谨慎一点：这是一个值得考虑的问题吗？我们有几百G的存储空间，即使不是几T，而且人们今天应该在用相当好的ISP所提供的网络，所以二进制文件大小不应该是个问题，对吧？答案是，这仍然值得考虑，虽然和以前比不那么重要了:)

阿卡麦互联网调查显示虽然超过80%的发达国家用户享受4Mbps或者更高的速率，但是在发展中国家，则远远低于这个数字。平均连接速度提高了很多（几乎所有国家都能达到平均1Mbps），但是整体的分布仍然停止不前。我有幸在一个每月只需30$就能享受G速率的互联网的国家，但许多其他人也许并不是。

普通的消费者只拥有有关计算的狭窄知识面，并且他们喜欢用他们所知道的来解释任何相关问题。一个共同的观点是可执行文件膨胀导致执行缓慢。不幸的是，这是真的，而你想要避免这种情况出现。

为一些好奇心重的读者：所有例子都用Rust 1.9.0 和 1.11.0 nightly (a967611d8 2016-05-30)测试。除非特别指明，主要的操作系统是Linux 3.13.0 on x86-64. 你的机器也许不一样。

**优化水平**

如果问道上面问题，几乎所有有经验的Rust用户都会回问你： 你打开了release开关吗？

这个开关会关掉调试参数。Cargo的文档精确的解释两者的不同，但通常来讲，release编译会去除一些只有开发过程需要的路由和数据，并且进行了许多优化。这个开关不是缺省打开的，因为，好吧，调试需求比发布需求更多。

请注意Jorge Aparicio已经正确的指出release编译并不会产生最小可能的可执行文件。 这是因为release编译缺省优化水平设置为3 (-C opt-level=3)，这可能为性能而牺牲一些文件大小。文件大小优化水平(-C opt-level=s or -C opt-level=z) 最近发布了，所以你可以在后面用这些参数。现在，让我们仍然坚持缺省参数。

让我们尝试一下release编译！
```
$ cargo build --release
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 646467 May 31 20:10 target/release/hello*
```
这实际上并没有什么区别！这是因为优化仅仅处理了用户代码，而我们几乎没有写什么用户代码。几乎所有的二进制代码来自标准库，而且我们不清楚可以做些啥...

**链接时间优化(LTO)**

…可惜我们可以进入链接时间优化。

因此故事就如下面：我们可以分别优化每一个crate，实际上所有标准库已经用优化形式发布。一旦编译器产生一个优化后的二进制文件，它被"the linker"组装成一个单独的可执行文件。但我不需要整个标准库：举例，一个简单的"Hello, workd"程序当然不需要std::net库。然而，链接器这么笨以至于它不会尝试移除crates中没用到的部分；它仅仅把这些库贴到最后的可执行文件中。

其实传统linker这么做有一个好的理由。链接器通常用在C和C++，而且每个文件单独编译。这和Rust有很大的不同，因为整个crate是一起编译的。除非需要的函数分布在各个文件中，链接器可以相当容易的一次性去除一些没有用到的文件。这不完美，但基本接近我们的需要：移除没有用到的函数。一个不利的地方是编译器没有办法优化函数调用，当它指向其他文件时；简单来说是缺少需要的信息。

C和C++伙计们已经适应这个大约几十年了，但在最近的一二十年，他们觉得受够了，并且开始提供一个选项来启用链接时间优化(link-time optimization LTO)在这个计划里，编译器为每个文件产生优化的二进制代码而没有关注其它方面，然后链接器积极的看所有的东西并且尝试优化最终的二进制文件。这比直接处理（内部简化）源文件要难多了，而且这使编译的时间大大变长了，但又值得一试，如果必须获得一个更小的且/或更快的可执行文件。

到目前为止，我们谈论了C和C++，但LTO对Rust而言更加有益。Cargo.toml有一个参数可以启用LTO:
```
[profile.release]
lto = true
```
这有用吗？嗯，有些。

```
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 615725 May 31 20:17 target/release/hello*
```
（注，在译者的机器上从1689K减少到295K)
这比调整优化水平所获得效果要大的多，但还不够。也许现在是时候瞧瞧可执行程序自身。

**那么，什么东西在可执行程序文件中？**

有些工具可以直接和可执行程序一起工作，但最有用的一个也许是GNU binutils。所有的unix类型系统和windows(举例MinGW有一个单独安装）

Binutils有许多工具，但strings也许是最简单的一个。它简单的爬行二进制文件来发现用0字节结尾的可打印字符序列，一个典型的C表达方式的string。这样它尝试从二进制文件中抽取可读的strings，这对我们很有帮助。所以，让我们试一下，注意准备滚屏:
```
$ strings target/release/hello | head -n 10
/lib64/ld-linux-x86-64.so.2
bki^
 B ,
libpthread.so.0
_ITM_deregisterTMCloneTable
_Jv_RegisterClasses
_ITM_registerTMCloneTable
write
pthread_mutex_destroy
pthread_self
哇哦，确实有一些东西是我们没有预料到的： pthreads. (然而，让我们稍后再谈) 确实有一大堆字符串在我们的可执行文件中：

$ strings target/release/hello | wc -c
   94339
```
哈，我们程序中的六分之一是我并没有用到的strings! 在进一步的检查中，这个观察并不准确因为strings提供了许多假的事实，但也有一些有意义的strings:

那些以jemalloc_和je_开头的字符串， 这些名字来自jemalloc,一个高效的内存分配器。这是Rust替代经典malloc/free，用来做内存管理用的。 这不是一个小库，而且毕竟我们没有自动做任何动态分配动作。

那些以backtrace_ 和 DW_开头的字符串，它们是从库 libbacktrace获得的。一个用于产生栈跟踪的库。 Rust用它来打印一些有用的panic跟踪信息(当 RUST_BACKTRACE=1 环境下有效)。然而，我们没有pacnic。

那些以_ZN开头的字符串,这些是“mangled”的名字，从Rust标准库中来。

为什么我们刚开始会有这些strings？因为它们是调试符号，相对于机器处理的二进制文件，这些合适的名字大概人类还可以阅读一下。你还记得上面提到的libbacktrace？它需要这些调试符号来打印出任何有用的信息。然而，因为我们确实想要发布一个release版，我们可以选择不包含调试信息。Rust本身没有这个选项，因为他们一般是利用一个外部工具叫做strip。因此，让我们来瞧一下用它如何工作。

**调试符号，从我的草坪中去除！**

所以，我们有三个目标：没有jemalloc, 没有libbacktrace, 并且没有调试符号。我已经提到用strip来去除调试符号，所有，让我们先做这个。注意到，strip包含在binutils，所以我们可以执行这个：
```
$ strip target/release/hello
$ target/release/hello
Hello, world!
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 347648 May 31 20:23 target/release/hello*
```
现在，文件更小了！大概一半的可执行文件是调试符号。请注意，当去除了我们的符号后，当panic时我们没有办法获取一个良好的调试信息：
```
$ sed -i.bak s/println/panic/ src/main.rs
$ cat src/main.rs
fn main() {
    panic!("Hello, world!");
}

$ cargo build --release && strip target/release/hello
$ RUST_BACKTRACE=1 target/release/hello
thread '<main>' panicked at 'Hello, world!', src/main.rs:2
stack backtrace:
   1:     0x7fde451c1e41 - <unknown>
Illegal instruction
$ mv src/main.rs.bak src/main.rs     # tidy up
…and it somehow aborted. Probably a libbacktrace issue, I don’t know, but that doesn’t harm much anyway.
```

**搞定jemalloc**

我们已经搞定调试符号了，现在让我们去除剩下的库。一些坏消息：从这个点开始，你进入了nightly rust领域。这个领域并没有你想的那么可怕，因为它并没有打你脸，但它会在一些小地方打击你（这也是为什么我们叫他nightlies!）。这也是为啥我们没有nightly特性在stable版本里的主要理由--它们也许会改变。幸运的是，我们下面在用的那些特性已经相当稳定了，而且你也许可以跟踪这篇文章来得知更多的nightlies。但之后，我会坚持一个特定的nightly版本。

一些好消息是：用rustup安装 nightlies (不管是最新版本的还是某个特定版本）都非常简单。
```
$ rustup override set nightly-2016-05-31
$ cargo build --release
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 620490 May 31 20:35 target/release/hello*
$ strip target/release/hello
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 351520 May 31 20:35 target/release/hello*
好的，文件大小没有多少变化，因为我们刚用过strip。让我们打倒jemalloc先— 书里面有很好的描述，但要旨是这下面这两行来改变allocator:

#![feature(alloc_system)]
extern crate alloc_system;

fn main() {
    println!("Hello, world!");
}
并且，这确实有很大的变化：

$ cargo build --release
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 210364 May 31 20:39 target/release/hello*
$ strip target/release/hello
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 121792 May 31 20:39 target/release/hello*
```
好的！我们从600K减少到120KB。（译者机器显示为1689K->290K->170K）。 Jemalloc 确实是一个大的库; 你可以很好的调试它的性能，但就跑题了。

没有付出没有收获！

我们只剩下libbacktrace没有处理。我们的代码不会panic，所以我们不需要打印backtrace,对吧？好的，我们遇到限制了：libbacktrace深度集成到标准库，唯一避免它的方法是不用 libstd. 简直就是一条死路。

但故事还没有结束。 Panicking 给我们一个backtrace, 但也给我们一个能力来unwind任何东西。且 unwinding 也被另一些代码，叫做 libunwind所支持。这显示我们可以禁用unwinding来去除这些。在Cargo.toml中加入下面这些内容：
```
[profile.release]
lto = true
panic = 'abort'
我们看到一些效果：

$ cargo build --release
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 200131 May 31 20:44 target/release/hello*
$ strip target/release/hello
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 113472 May 31 20:44 target/release/hello*
```
就是这样了，如果你不打算改变代码。

**中场休息,后文待译...**
--------------20160629继续-----------

在探索更模糊的地带前，这是一个合适的时间点来承认我欺骗了大家关于C和C++二进制文件的大小。更公平的比较是下面：
```
$ touch *.c *.cpp
$ make hello-c CFLAGS='-Os -flto -Wl,--gc-sections -static -s'
cc -Os -flto -Wl,--gc-sections -static    hello-c.c   -o hello-c
$ make hello-cpp CXXFLAGS='-Os -flto -Wl,--gc-sections -static -static-libstdc++ -s'
g++ -Os -flto -Wl,--gc-sections -static -static-libstdc++    hello-cpp.cpp   -o hello-cpp
$ ls -al hello-c hello-cpp
-rwxrwxr-x 1 lifthrasiir  758704 May 31 20:50 hello-c*
-rwxrwxr-x 1 lifthrasiir 1127784 May 31 20:50 hello-cpp*
```
（注意：所有这些编译选项是必须，因为需要符合Rust的等价编译选项。 
-Wl,--gc-sections 也许是唯一漏掉的选项；它是一个简化版的类似LTO的选项，它没有进行优化，而仅仅是移掉没有用到的代码—非常感谢Alexis Beingessner and Corey Richardson 指出这个疏忽。Rust缺省加上这个选项，而且可以独立于LTO而应用，所以一个公平的对比也需要它
）
而且，Rust的二进制必须重新编译：
```
$ cargo rustc --release -- -C link-args=-static
$ strip target/release/hello
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 105216 May 31 20:51 target/release/hello*
```
(译者现在windows 7， x64下编译出来的程序是147K，比先前的170K有减少了一些)
好的，看起来Rust实际上远比C和C++的编译效果好。但是...为啥它是“公平”的？不管任何语言，难道一个这么简单的程序需要这么多空间吗？

一个二进制可执行文件不是一个简单的数据格式。它一般是由一个OS routine，也叫"dynamic linker"来处理（不要和之前说的"linker"混淆）。动态链接允许程序动态的调用其他系统自带的（通常是通用的）库，而直到现在，我们这里隐式链接到C和C++的标准库—glibc and libstdc++！然而，Rust不需要（好的，几乎不需要）它们，所以整个比较对Rust不公平。

这种类型的动态链接是一把双刃剑。它使得更新多个程序共享的库变得不那么难，而且理论上，总的二进制文件大小应该减少。然而，有一个很强的少数派观点反对动态链接库，因为这也使得破坏库变得不那么难（可惜的是dynamic linker不像Cardo），且它减少二进制文件大小的效果有点被夸大了。为了在后面详述，这篇文章先前说LTO最终解决了问题，而动态链接库忍受同样的问题，却没有解决方案。在动态链接库之上进行LTO会破坏整个好处。

但，不管这些问题，动态链接库仍然是许多系统上一个通行的选择，而特别的，C和C++标准库通常只提供动态链接库。是的，我们指导linker来静态链接所有东西，但glibc撒谎了，一些函数仍然要求动态库：

```
$ # restore debug symbols
$ touch *.c
$ make hello-c CFLAGS='-Os -flto -static'
cc -Os -flto -static    hello-c.c   -o hello-c
$ strings hello-c | grep dlopen
shared object cannot be dlopen()ed
dlopen
invalid mode for dlopen()
do_dlopen
sdlopen.o
dlopen_doit
__dlopen
__libc_dlopen_mode
```
这实际上是一个聪明的办法来获取一个几乎静态的库，只带有一些动态特性（比如iconv)。当然，如果你依赖这些动态特性，你就完了。这么说吧，你仍然需要启用linker动态链接来避免偏差；这个 **-static** linker 选项因此是必须的。非常有趣的是，我们看到Rust二进制文件在静态链接后实际上比它们小。它确实依赖C标准库，但只依赖一点(举例，标准I/O完全绕过stdio），所以它确实在某种程度上成功减少glibc的重量。

情况已经改变了许多，但这些比较看起来仍然不公。毕竟这都是glibc的错！我们有一些libc的替代品，musl看起来是不错的一个替代品。即使静态链接，它也产生非常紧凑的二进制文件:
```
$ touch *.c
$ make hello-c CFLAGS='-Os -flto -static -s' CC=musl-gcc
musl-gcc -Os -flto -static -s    hello-c.c   -o hello-c
$ ls -al hello-c
-rwxrwxr-x 1 lifthrasiir 5328 May 31 20:59 hello-c*
```
我们可以在Rust里面用musl吗？当然。你可以通过rustup为不同目标平台安装标准库。这一次选项 -C link-args=-static 不是必须的了，因为musl缺省是静态链接的。
```
$ rustup target install x86_64-unknown-linux-musl
$ cargo build --release --target=x86_64-unknown-linux-musl
$ ls -al target/x86_64-unknown-linux-musl/release/hello
-rwxrwxr-x 1 lifthrasiir 263743 May 31 21:07 target/x86_64-unknown-linux-musl/release/hello*
$ strip target/x86_64-unknown-linux-musl/release/hello
$ ls -al target/x86_64-unknown-linux-musl/release/hello
-rwxrwxr-x 1 lifthrasiir 165448 May 31 21:07 target/x86_64-unknown-linux-musl/release/hello*
```
好的，现在终于看起来公平了。这160K的可执行程序可以恰当的归功于Rust的“膨胀”；它包含 libbacktrace and libunwind (总共大概50K), 而且libstd仍然很难完全优化掉（大概40K的纯Rust代码，引用了libc的不同的bit）。这是Rust标准库的目前状态，必须解决掉。好吧，至少这是每一个可执行程序的一次性的代价，所以实际上他们也没啥重要。

我们还有一个尝试可以是比较更加公平。假如Rust可以用动态链接会怎么样？这基本不可能，我们知道所有的操作系统还没自带Rust标准库，但是，好吧，值得尝试。注意，LTO, 定制化内存分配，不同的panic策略无法和动态链接兼容，所以，你不得不让这些选项缺省不变。

```
$ cargo rustc --release -- -C prefer-dynamic
$ ls -al target/release/hello
-rwxrwxr-x 1 lifthrasiir 8831 May 31 21:10 target/release/hello*
```
所以动态链接可以和通常的C/C++程序对比。然而，这里大部分是因为好奇。（译者：因为编译出来的程序缺少标准库而不能运行）

**让我们和libstd说再见吧**

**我们到现在为止尝试在几乎不改变代码的情况下试图大量减少可执行文件的大小。但如果你愿意付出对价，你可以获得一个小的多的执行文件。注意：通常不推荐这样，这一部分会进入到一个非常严格的成本-效益分析。**

我们都知道libstd是非常友好和便利的，但有时它提供太多东西了（而且它是我们没有用到的libbacktrace和libunwind的源泉）。开始避免libstd。 RUST书上有整整一节来讲述如何不用stdlib代码，所以，让我们依葫芦画瓢。新的源码如下：

```
#![feature(lang_items, start)]
#![no_std]

extern crate libc;

#[start]
fn start(_argc: isize, _argv: *const *const u8) -> isize {
    // since we are passing a C string,
    // the final null character is mandatory
    const HELLO: &'static str = "Hello, world!\n\0";
    unsafe { libc::printf(HELLO.as_ptr() as *const _); }
    0
}

#[lang = "eh_personality"] extern fn eh_personality() {}
#[lang = "panic_fmt"] extern fn panic_fmt() -> ! { loop {} }
```
而且你需要添加一个独立的libc crate在Cargo.toml里：
```
[dependencies]
libc = { version = "0.2", default-features = false }
```
就是这样了！这非常接近你从C程序所获得的：strip前8503字节，strip后6288字节。(I’m tried of faking the console output (note the modified time), so I’ll omit them from now on.) musl更进一步，但你需要额外的选项来恰当的链接libc crate 到 musl:
```
$ export RUSTFLAGS='-L native=/usr/lib/x86_64-linux-musl'
$ cargo build --release --target=x86_64-unknown-linux-musl
$ strip target/x86_64-unknown-linux-musl/release/hello
```
现在，我们把字节减少到5360.仅比最好的C程序结果多32字节！我么可以做的更好吗？不用stdio，你可以直接调用系统功能(当然，仅在unix下）：
```
#[start]
fn start(_argc: isize, _argv: *const *const u8) -> isize {
    const HELLO: &'static str = "Hello, world!\n";
    unsafe {
        libc::write(libc::STDOUT_FILENO,
                    HELLO.as_ptr() as *const _, HELLO.len());
    }
    0
}
```
这去除了更多的字节，最终strip的大小是5072字节。

有些人会争议说以上不是真实的Rust代码，而是一个Rust形式的依赖于unix系统调用的代码。好的。没有真实的Rust代码看起来像这样。相反的，我们可以创造我们自己的Stdout类型，直接连线到系统调用，而且用普通的宏格式：

```
use core::fmt::{self, Write};

struct Stdout;
impl Write for Stdout {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        let ret = unsafe {
            libc::write(libc::STDOUT_FILENO,
                        s.as_ptr() as *const _, s.len())
        };
        if ret == s.len() as isize {
            Ok(())
        } else {
            Err(fmt::Error)
        }
    }
}

#[start]
fn start(_argc: isize, _argv: *const *const u8) -> isize {
    let _ = writeln!(&mut Stdout, "Hello, world!");
    0
}
```
这实际上是更自然的Rust代码。Strip后是9248字节，暗示一下，最小的代码成本大约是4KB。然而其他类型强加更多在头上；举例，打印3.14f64大概会产生25K额外的二进制文件大小，因为浮点数到小数转换不容易。而且注意到这没有用buffers(除了为pioes的内核buffer).

还有一件事情...

我们还可以在这份上更进一步，因为我们的可执行文件还包含一些没有用到的大块字节。欢迎来到真正底层的编程（和我们通常用C/C++编程相比）。其实，这个领域有一些先锋已经彻底的探险过，所有我就直接提供链接以便找到他们：

Keegan McAllister made a [151-byte x86-64 Linux program](http://mainisusuallyafunction.blogspot.kr/2015/01/151-byte-static-linux-binary-in-rust.html) printing “Hello!”. Well, that’s not what we wanted, and the proper “Hello, world!” program will probably cost 165 bytes; the original program (ab)used the ELF header to put the string constant there but there isn't much space to put a longer “Hello, world!”.

Peter Atashian made a [1536-byte Windows program](https://github.com/retep998/hello-rs/blob/master/windows/src/main.rs) printing “Hello world!” (note no comma). I’m less sure about the size of the proper program, but yeah, you have an idea. This is notable because Windows essentially forces you to use dynamic linkage (Windows system calls are not stable across versions), and the dynamic symbol table costs bytes.

For the comparison and your geeky pleasure, the shortest known version of a x86-64 Linux program printing “Hello, world” (note no exclamation mark) costs only [62 bytes](http://www.muppetlabs.com/~breadbox/software/tiny/hello.asm.txt). The general technique is better explained in [this classical article.](http://www.muppetlabs.com/~breadbox/software/tiny/teensy.html)

Hey, I’ve learned the hard way that we have tons of variations over the “Hello, world!” programs, and none of those programs print the proper and longest version of greeting!

这篇文章有意探索多种技术（并不是所有这些都可以派上实际用场）来减少程序大小，但如果你需要一些结论，可操作的实用结论是：

**1.用 --release编译**

**2.启用LTO，并strip二进制文件**

**3.如果你的程序没有密集用到内存管理，请用 system allocator (假定你用 nightly版本).**

**4.你也许可以在未来使用编译参数开关 s/z**

我没有提到这点是因为它对于这么一个小程序没有起到改进作用。假如你的可执行文件巨大，你也可以使用UPX或其他可执行程序压缩软件来压缩。

伙计们，这些就是全部了！
