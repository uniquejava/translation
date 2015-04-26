# Rust一次编写到处运行

    英文原文:http://blog.rust-lang.org/2015/04/24/Rust-Once-Run-Everywhere.html
    翻译：Cyper(345343747@qq.com)

Rust从不准备在一夜之间统治全球, 它先要和现有的世界交互, 且容易程度就像在自言自语. 基于这一考量， **Rust可以和现有的C API无痛交流, 同时还能利用自身的ownership体系为这些API提供更强的安全保障**.

为了调用其它语言, Rus提供了FFI(foreign function interface,外来函数接口). 遵循Rust的设计原则, FFI提供了**零开销的抽象层**(zero-cost abstraction), 使得Rust调用C和在C中调用C一样快. FFI绑定利用Rust中的ownership和borrowing大法提供了一种**安全接口**(safe interface),该安全接口能够使得指针和其它资源在一种协议框架下存在.这些协议或约定最多出现于C语言的API文档, 但是Rust让这种协议更加直白(explicit).

在这篇帖子里我将探究怎么以安全零开销的方式去封装不安全的FFI C调用.　我们仅仅拿C语言做为一个例子, 你还将看到在Rust中使用其它语言比如Python和Ruby就像在Rust中使用C一样简单.

##Rust调用C

先来看个简单的Rust调用C的例子. 以下是段C代码 - 把输入简单的乘以2并返回.
```c
int double_input(int input) {
    return input * 2;
}
```
为了在Rust中调用这段C代码,你大概会这样写:

```rust
extern crate libc;

extern {
    fn double_input(input: libc::c_int) -> libc::c_int;
}

fn main() {
    let input = 4;
    let output = unsafe { double_input(input) };
    println!("{} * 2 = {}", input, output);
}
```
就这么简单.你可以[从github中检出这段代码][1],然后在检出的目录中执行cargo run命令. **从源代码来看, 除了函数声明复杂点以外使用起来毫无痛感. 一会你将看到即便是生成的汇编代码也非常干净**. 不过Rust还是有些微妙的东西待我一一说来.

首先我们看到了`extern crate libc`, [libc crate]提供了大量给FFI使用的类型定义, 使得C语言和Rust语言在类型定义上能达成共识.这让我们能愉快地书写如下代码:
```c
extern {
    fn double_input(input: libc::c_int) -> libc::c_int;
}
```
这是对外部函数的**声明**.你可以把它想像成C中的头文件.编译器在这里知晓函数的输入和输出类型, 你可以看到这和在C中的定义是一致的. 最后我们进入程序的主体.

```rust
fn main() {
    let input = 4;
    let output = unsafe { double_input(input) };
    println!("{} * 2 = {}", input, output);
}
```

这里有Rust FFI最为关键的地方之一 -- `unsafe`块. 编译器对`double_input`的实现一无所知, 它总是假定外来函数会出现内存不安全的情况. 总之程序员要对`unsafe`块负责,保证调用的时候不会出现内存错误以维持Rust最基本的安全保障.这听起来似乎有点不靠谱, 但是Rust有它自己的一套工具允许使用者不必担心`unsafe`带来的问题(一会还会接着讨论)

既然我们已经知道了如何在Rust中调用C函数, 接下来我们看看生成的汇编代码是否干净.几乎所有的编程语言都能以某种方式去调用c语言, 但通常会有运行时类型转换问题或在语言层面做文章带来的额外开销. 为了搞清楚Rust到底做了什么, 让我们直接看下Rust在`main`中调用`double_input`产生的汇编代码:
```asm
mov    $0x4,%edi
callq  3bc30 <double_input>
```
一如既往(的优秀!) 仅此而已.	从这里可以看到在参数就位后, 仅使用了一条指令调用C函数, 这和直接在C中调用C函数没有差别.

##安全的抽象(Safe Abstractions)
大多数Rust的feature都和它的核心概念ownership有关, FFI也不例外.在你绑定一个C库到Rust的时候不仅没有带来额外的性能开销,相反还带来了安全方面的好处. **C是在头文件中以注释方式告知调用者如何去使用API, 而Rust Bindings直接借助ownership和borrowing大法**.

举个例子, 考虑使用某个C库去解析一个tar文件.这个库文件暴露了如何去读取某个tar文件包含的子文件的方法, 大致如下:
```c
// 读取tar文件中的第index个子文件, 如果该子文件不存在返回NULL
// 如果成功, 则`size`指针包含了文件的大小信息.
const char *tarball_file_data(tarball_t *tarball, unsigned index, size_t *size);
```
这个函数隐式的规定了它的调用方法, 但是, 它假定返回的`char*`指针不会晚于输入参数tarball销毁. 当进行Rust绑定的时候, API会变成如下这个样子:

```rust
pub struct Tarball { raw: *mut tarball_t }

impl Tarball {
    pub fn file(&self, index: u32) -> Option<&[u8]> {
        unsafe {
            let mut size = 0;
            let data = tarball_file_data(self.raw, index as libc::c_uint,
                                         &mut size);
            if data.is_null() {
                None
            } else {
                Some(slice::from_raw_parts(data as *const u8, size as usize))
            }
        }
    }
}

```
这里`*mut tarball_t`指针属于`Tarball`, 而Tarball负责析构和清理工作, 这样我们可以足够清楚tarball的内存生命期. 除此以外，file方法返回`borrowed slice`，　这个返回值的生命期和tarball自己的生命期一致(通过&self参数传递). Rust通过这种方式提示返回的slice只能在tarball的生命期中存在. 这可以防止经常在C中出现的空悬指针的问题(如果你对borrowing还不是很熟，看看[Yehuda Katz关于ownership的博客][2].

这是个安全的函数(Rust绑定最重要的方面之一), 意味着使用者不必使用unsafe块来调用它! 尽管Rust有`unsafe`*实现*(因为要去调用FFI函数),但是Rust的*接口*使用了borrowing以确保在Rust代码中不存在内存不安全的问题. 得益于Rust的静态检查, 在从Rust中调用API的时候不太可能出现段错误(segfault). 不要忘了, 所有的这些都是零开销: 在C中的所有原生类型在Rust中都有对等物, 无需额外去分配空间也没有其它开销.

了不起的Rust社区已经安全绑定了不少现有的C库, 包括[OpenSSL], [libgit2], [libdispatch], [libcurl], [sdl2], [Unix APIs], 和 [libsodium], 这一列表在[crates.io]上正快速增长，所以你最喜爱的C库可能已经被绑或正在绑定中.

##C调用Rust

**除了内存安全,Rust既没有gc也无需运行时(runtime), 无需准备工作即可在c中直接调用rust**.这意味着零开销的FFI既适用于Rust调用C, 也适用于C调用Rust. 我们依然使用上面的例子, 但这次反过来使用. 和前面一样,所有的代码都[可以在github上找到][3]. 让我们先从Rust代码开始:

```rust
#[no_mangle]
pub extern fn double_input(input: i32) -> i32 {
	input * 2
}
```
和之前一样, 没有过多的Rust代码但还是有些细微的地方值得一提. 首先我们给function帖上了`#[no_mangle]`标签, 提示编译器不要破坏`double_input`这个符号名, Rust和C++一样使用了命名粉碎规则(name mangling), 以防止库之间的名称冲突. 有了这个属性(attribute), 在C代码中就不必使用像`double_input::h485dee7f568bebafeaa`这样的名字.

下一步是定义我们的函数,最有趣的地方莫过于关键字`extern`了, 它是声明[ABI for a function]的一种特别形式, 这一声明使该函数兼容C函数调用.

最后如果你[看一眼Cargo.toml文件][4], 就会发现这个库不是编译成了普通的Rust库文件(rlib)而是编译成了被Rust称为'staticlib'的静态归档文件,这使得所有与Rust相关的代码静态链接到我们即将编写的C代码.

现在我们的Rust代码已经完成了, 接下来我们编写调用Rust函数的C代码.
```c
#include <stdint.h>
#include <stdio.h>

extern int32_t double_input(int32_t input);

int main() {
    int input = 4;
    int output = double_input(input);
    printf("%d * 2 = %d\n", input, output);
    return 0;
}

```
我们看到C调用Rust和Rust调用C一样, 在C中首先声明先前在Rust中定义的`double_input`函数. 除此以外, 万事俱备. 如果你从[github检出的目录下][3]运行`make`命令你会看到这段示例代码被编译-链接-直至生成可执行文件, 并打印输出`4 * 2 = 8`.

Rust缺少gc和运行时使得从C无缝切换到Rust成为可能. 外部的C代码无需为Rust做任何准备工作, 这让从C过渡到Rust变得容易.


##不仅是C
到目前为止我们已经看到了Rust中的FFI是如何实现零开销以及怎么利用Rust中的ownership安全绑定C库．如果你不用C也没关系. Rust的这些特征让它能被[Python], [Ruby], [JavaScript]等众多语言调用.

当用这些语言编写代码的时候，有时需要优化对性能要求极为苛刻的模块，过去我们只能用C语言来实现，但C语言不像这类语言那样内存安全,有较高层次的抽象和那么高效.

Rust易于与C语言交互的事实意味着它能代替C所做的这类工作. 在产品中使用Rust的用户之一, [Skylight]仅通过Rust语言就在它们的数据收集器(data collection agent)产品中取得性能提升以及内存优化, 效果立竿见影, 而且所有的Rust代码同时被发布成Ruby的gem模块.

为提高性能把Python和Ruby代码转换成C代码相当有难度, 以一种难以debug的方式写的程序，很难保证程序不崩溃.  Rust不仅带来了零开销的FFI而且保留了与直接使用源语言一致的安全性. 从长远来看，使用其它语言的程序员在需要的时候能够很方便的使用Rust做一些系统级别的编程以提高性能.

FFI是Rust工具箱中众多工具的一种. 它是Rust所采纳的最关键组件之一, 因为它让Rust可以无缝地集成到现有的代码库．一想到Rust能使越来越多的项目受益我就感到无比激动内牛满面.

[1]:https://github.com/alexcrichton/rust-ffi-examples/tree/master/rust-to-c
[2]:http://blog.skylight.io/rust-means-never-having-to-close-a-socket/
[3]:https://github.com/alexcrichton/rust-ffi-examples/tree/master/c-to-rust
[4]:https://github.com/alexcrichton/rust-ffi-examples/blob/master/c-to-rust/Cargo.toml#L8
[libc crate]:https://crates.io/crates/libc
[ABI for a function]:http://doc.rust-lang.org/reference.html#extern-functions
[Skylight]:https://www.skylight.io/
[OpenSSL]:https://crates.io/crates/openssl
[libgit2]:https://crates.io/crates/git2
[libdispatch]:https://crates.io/crates/dispatch
[libcurl]:https://crates.io/crates/curl
[sdl2]:https://crates.io/crates/sdl2
[Unix APIs]:https://crates.io/crates/nix
[libsodium]:https://crates.io/crates/sodiumoxide
[crates.io]:https://crates.io/
[Python]:https://github.com/alexcrichton/rust-ffi-examples/tree/master/python-to-rust
[Ruby]:https://github.com/alexcrichton/rust-ffi-examples/tree/master/ruby-to-rust
[JavaScript]:https://github.com/alexcrichton/rust-ffi-examples/tree/master/node-to-rust































