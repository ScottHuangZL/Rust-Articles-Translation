* 原文： 
    * http://hermanradtke.com/2015/05/29/creating-a-rust-function-that-returns-string-or-str.html
    * Written by Herman J. Radtke III on 29 May 2015
* 翻译者： Scott Huang
* 翻译日期： Sep 15, 2015 于厦门

#Creating a Rust function that returns a &str or String 创建一个返回&str或String的Rust函数

我们已经学了如何[创建一个接受String或&str的Rust函数](https://github.com/ScottHuangZL/Rust-Articles-Translation/blob/master/creating-a-rust-function-that-accepts-string-or-str.md). 现在我想向您展示如何创建一个函数，返回`String`或`&str`。我也要讨论为什么我们要这样做。让我们从写一个删除一个给定字符串的所有空格的函数开始，我们的函数可能看起来像这样：
```rust
fn remove_spaces(input: &str) -> String {
   let mut buf = String::with_capacity(input.len());

   for c in input.chars() {
      if c != ' ' {
         buf.push(c);
      }
   }

   buf
}
```
这个函数为字符串缓冲区分配内存，循环检查每个输入的字符，并把所有非空白字符追加到字符串缓冲区。现在我问：假如我的输入不包含空格的话，会怎样？输入的值将和缓冲区的值相同。在这种情况下，不先创建缓冲区将是更有效的。相反，我们想把给定的输入返回给调用方。输入的类型是一个&str但我们的函数返回一个String，虽然,我们可以将输入的类型更改为String：

```rust
fn remove_spaces(input: String) -> String { ... }
```
但这会导致2个问题。首先，输入是String类型的话，我们强迫调用方将输入的所有权转移到我们的函数中。这会阻止调用者在未来继续使用该值。我需要获取所有权仅当我们真的需要它。第二，输入可能已经是&str类型，我们现在迫调用方把它转换成String，这将导致我们创建缓冲区而不分配新内存的尝试失败。

#Clone-on-write

我们真正想要的是当没有空格时可以直接返回我们的输入字符串的能力（`&str`）和当需要删除空白时返回一个新字符串（`String`）。这是一种可以使用`clone-on-write`或者`Cow`类型的地方。Cow类型可以让我们抽象无论是自有或者是借贷来的东西。在我们的例子中，&str是一个已有字符串的引用，将借贷数据。如果有空格，那么我们需要为一个新的字符串分配内存。这个新的字符串的缓冲区由buf变量拥有。通常，我们会将缓冲区的所有权返回给调用者。用Cow的时候，我们要把buf所有权转移到Cow并且返回。

```rust
use std::borrow::Cow;

fn remove_spaces<'a>(input: &'a str) -> Cow<'a, str> {
    if input.contains(' ') {
        let mut buf = String::with_capacity(input.len());

        for c in input.chars() {
            if c != ' ' {
                buf.push(c);
            }
        }

        return Cow::Owned(buf);
    }

    return Cow::Borrowed(input);
}
```
我们函数现在检查给定的输入是否包含一个空格，然后才将内存分配给一个新的缓冲区。如果输入不包含空格，则简单地返回输入。我们增加了许多[运行时复杂度]来优化我们如何分配内存。注意，我们的Cow类型拥有和&str类型相同的使用期。正如我们前面所讨论的，编译器需要跟踪`&str`参考以便知道什么时候可以安全的释放（或终止）内存。

Cow的美在于它实现了`Deref`特质，所以你调用immutable函数而不用知道是否该结果是一个新的string缓冲区或者不是。例子

```rust
let s = remove_spaces("Herman Radtke");
println!("Length of string is {}", s.len());
```
如果我真的需要改变s，那么我可以使用`into_owned()`函数来将它转换成一个拥有所有权的变量。如果Cow的变体已经拥有所有权，那么我们就只需移动所有权。如果Cow的变体是借来的，那么我们就要分配内存。这让我们只有当想写（或改变）变量时才懒克隆（分配内存）。

当Cow::Borrowed是可改变的例子： 
```rust
let s = remove_spaces("Herman"); // s 是一个 Cow::Borrowed 变量 
let len = s.len(); // immutable function call using Deref 
let owned: String = s.into_owned(); // memory is allocated for a new string
```

当Cow::Owned是可改变的例子：
```rust
let s = remove_spaces("Herman Radtke"); // s 是一个 Cow::Owned 变量 
let len = s.len(); // immutable function call using Deref 
let owned: String = s.into_owned(); // no new memory allocated as we already had a String
```
Cow的想法是双重的：

* 尽可能的延迟分配内存。在最好的情况下，我们永远不需要分配任何新的内存。
* 允许remove_spaces函数调用者不用关心是否需要分配内存。在任何情况下，使用的Cow类型是相同的。

#Leveraging the Into Trait 利用Into特质

我们之前讨论的使用`Into`特征的转换&str为String。我们还可以使用`Into`特征的转换&str或String到一个合适的Cow变体。通过调用`.into()`，编译器将自动完成转换。使用.into()不会加快或减慢代码。这是一个简单的选择，以避免显式地指定`Cow::Owned`或者`Cow::Borrowed`

```rust
fn remove_spaces<'a>(input: &'a str) -> Cow<'a, str> {
    if input.contains(' ') {
        let mut buf = String::with_capacity(input.len());
        let v: Vec<char> = input.chars().collect();

        for c in v {
            if c != ' ' {
                buf.push(c);
            }
        }

        return buf.into();
    }
    return word.into();
}
```
#Real World Uses of Cow

我去空格的例子看起来有点做作，但有一些杰出的实际应用也采取这一策略。在Rust的核心有一个函数[有损地转换字节到UTF-8](https://github.com/rust-lang/rust/blob/720735b9430f7ff61761f54587b82dab45317938/src/libcollections/string.rs#L153)和一个函数将[转换CRLF到LF](https://github.com/rust-lang/rust/blob/c23a9d42ea082830593a73d25821842baf9ccf33/src/libsyntax/parse/lexer/mod.rs#L271)。这些函数都有这样一个用例：在最佳的情况下,一个&str可以返回，另一种情况下，一个String必须分配。其他的例子我想有正确编码的XML / HTML字符串或一个SQL查询正确转义。在许多情况下，输入已正确编码或是转义。在这种情况下，最好是将输入字符串返回给调用方。当输入需要修改时，我们不得不用字符串缓冲区的形式来分配新的内存，并返回给调用方。

#Why use String::with_capacity() ?

当我们谈到有效管理内存的话题，请注意我用`Stirng::with_capaticy()`代替`String::new`创建字符串缓冲区的时候。你可以使用String：：with_capacity()而不是String::new()，因为它是更有效地一次性分配内存缓冲而不是随着我们压入更多的字符到缓冲区时不断重新分配内存。让我们简单过一遍当我们使用String::new()，然后把字符压到到字符串的案例。

一个String实际上是一个UTF-8代码点向量。当String：：new()被调用，Rust创建一个零字节容量的向量。如果我们将字符压到字符串缓冲区。就像`input.push('a')`，Rust不得不增加vector的容量。在这种情况下，它将分配2个字节的内存。当我们压入更多的字符并且超过容量，Rust将通过每一次扩容一倍的方法重新分配内存。内存分配顺序为0，2，4，8，16，32…2^n，n是Rust发现容量被超过的次数。重新分配内存很慢（注：kmc_v3 [解释](https://www.reddit.com/r/rust/comments/37q8sr/creating_a_rust_function_that_returns_a_str_or/croylbu)，它可能不是我想的那么慢）。不仅要向内核要新的内存，它还必须从旧的内存空间复制内容到新的内存空间。查看[Vec::push](https://github.com/rust-lang/rust/blob/720735b9430f7ff61761f54587b82dab45317938/src/libcollections/vec.rs#L628)源码来看第一手的resizing逻辑。

一般来说，只有当我们需要它时我们才要分配新内存，且只要分配我们所需要的大小。对于小的字符串，像`remove_spaces("Herman Radtke")`，重新分配内存不是一件大事。但如果我想移除我们网站上每个JavaScript文件的空白呢？重新分配内存的压力将很大。当压入数据到一个向量时，我们最好先指定一个容量。最好的情况是，你已经知道长度，然后可以精确设置容量。Vec的[代码评论](https://github.com/rust-lang/rust/blob/720735b9430f7ff61761f54587b82dab45317938/src/libcollections/vec.rs#L147-152)给出类似的警告。

#相关链接

* [String vs &str in Rust functions](https://github.com/ScottHuangZL/Rust-Articles-Translation/blob/master/string-vs-str-in-rust-functions.md)
* [Creating a Rust function that accepts String or &str](https://github.com/ScottHuangZL/Rust-Articles-Translation/blob/master/creating-a-rust-function-that-accepts-string-or-str.md)
