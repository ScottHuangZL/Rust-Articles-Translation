* 原文： 
    * http://hermanradtke.com/2015/05/03/string-vs-str-in-rust-functions.html
    * Written by Herman J. Radtke III on 03 May 2015
* 翻译者： Scott Huang
* 翻译日期： Sep 12, 2015 于厦门

#String vs &str in Rust functions

这篇文章写给那些不得不使用`to_string()`而让程序通过编译的沮丧的人。对于那些不太理解为什么Rust有两个字符串类型`String`和`&str`的人，我希望给你一些帮助。

##Functions That Accept A String | 接受一个String的函数

我想讨论如何构建接受字符串的接口。我是一个狂热的超媒体迷，我痴迷于设计使用方便的界面。让我们从接受一个字符串的方法开始吧。我们的调研显示`std::string::String`是一个很好的选择。

```rust
fn print_me(msg: String) {
    println!("the message is {}", msg);
}

fn main() {
    let message = "hello world";
    print_me(message);
}
```
这导致一个编译器错误：
```rust
期望 `collections::string::String`,
    发现 `&str`
```
所以一个字符串字面量是`&str`类型,它不与`String`类型兼容。我们可以改变message类型为`String`， `let message = "hello world".to_string();` 可以编译成功。这可行，但它是类似于使用`clone()`来绕过所有权/借贷错误。相反的，这里有三个理由来改变print_me来允许接受`&str`：

* `&`符号是一个引用类型，意味着我们借用这个变量。当print_me用完那个变量，所有权将返回给原来的拥有者。除非我们有很好的理由将message变量的所有权转移到我们的函数，不然我们应该选择借用。
* 使用引用更有效。使用String的message意味着程序必须复制该值。使用一个引用时，如`&str`，不需要复制。
* 一个String类型可以神奇地变成了一个&str类型当使用Defer特质和类型强制。这将使一个例子更有意义。

##Deref Coercion例子：

这个例子通过四种不同的方式创建strings，并且都可以成功调用print_me函数。使这一切可工作的关键是通过引用传值。我们通过`&String`传递，而不是通过把owned_string作为一个String传递给print_me。当编译器看到一个&String传递给一个采取&Str的函数，它强制`&String`转为`&Str`。这一强制同时也发生在引用计数和自动引用计数的字符串那里。String变量已经是一个引用，所以当调用print_me（string）函数时就没有必要使用&。知道了这一点，我们不再需要调用我们`.to_string()`来制造一些垃圾到我们的代码。

```rust
fn print_me(msg: &str) { println!("msg = {}", msg); }

fn main() {
    let string = "hello world";
    print_me(string);

    let owned_string = "hello world".to_string(); 
    // 或者String::from_str("hello world")
    print_me(&owned_string);

    let counted_string = std::rc::Rc::new("hello world".to_string());
    print_me(&counted_string);

    let atomically_counted_string = std::sync::Arc::new("hello world".to_string());
    print_me(&atomically_counted_string);
}
```

你也可以使用Deref来强制其他类型，如Vector。毕竟，String只是一个拥有8字节字符的vector。你可以参考[Deref coercions](http://doc.rust-lang.org/nightly/book/deref-coercions.html)来阅读更多内容。

##Introducing struct

在这一点上我们应该没有外来to_string()调用我们的功能。然而，当我们试图引入一个结构时我们会遇到了一些问题。用我们刚学过的，我们可以做一个象这样的结构：

```rust
struct Person {
    name: &str,
}

fn main() {
    let _person = Person { name: "Herman" };
}
```
我们得到错误:
```rust
<anon>:2:11: 2:15 error: missing lifetime specifier [E0106] 漏掉声明使用期
<anon>:2     name: &str,
```

Rust试图保证Person不能有活的更久的引用指向name。如果Person确实有这种指针的话，那么我们就面临程序崩溃的风险。Rust做的所有一切就是防止这种意外出现。所以让我们开始试着编译这个代码。我们需要指定一个使用期，或范围，然后Rust可以保证我们安全。传统的使用期说明符是`'a`。我不知道为什么人们选它，但让我们先这么着吧。
```rust
struct Person {
    name: &'a str,
}

fn main() {
    let _person = Person { name: "Herman" };
}
```

再次编译器，然后我们得到另外一个错误：
```rust
<anon>:2:12: 2:14 error: use of undeclared lifetime name `'a` [E0261]
<anon>:2     name: &'a str,
```
让我们想想这个。我们知道我们想暗示Rust编译器，我们的Person结构不应该outlive name。所以，我们需要为Person结构声明使用期。一些搜索会告诉我们`<'a>`是用来声明使用期的语法。

```rust
struct Person<'a> {
    name: &'a str,
}

fn main() {
    let _person = Person { name: "Herman" };
}
```
现在编译通过！我们通常为我们的结构实现方法。让我们给我们的Person类加上一个greet函数。

```rust
struct Person<'a> {
    name: &'a str,
}

impl Person {
    fn greet(&self) {
        println!("Hello, my name is {}", self.name);
    }
}

fn main() {
    let person = Person { name: "Herman" };
    person.greet();
}
```

我们现在得到错误：
```rust
<anon>:5:6: 5:12 error: wrong number of lifetime parameters: expected 1, found 0 [E0107]
<anon>:5 impl Person {
```

我们Person结构有一使用期参数，所以我们的实现也应该有。我们声明`'a`使用期到Person的实现代码，就像`impl Person<'a>`。不幸的是，编译器给了一个让我们疑惑的错误提示：

```rust
<anon>:5:13: 5:15 error: use of undeclared lifetime name `'a` [E0261]
<anon>:5 impl Person<'a> {
```
为了让我们声明使用期，我们需要在实施`impl <'a> Person {`后马上指定使用期。再次编译，我们得到错误：
```rust
<anon>:5:10: 5:16 error: wrong number of lifetime parameters: expected 1, found 0 [E0107]
<anon>:5 impl<'a> Person {
```
现在我们回到正轨。让我们加回我们的使用期参数到Person的实现，就像 `impl <'a> Person<'a> {`。现在我们的程序通过编译。这里是完整的工作代码：

```rust
struct Person<'a> {
    name: &'a str,
}

impl<'a> Person<'a> {
    fn greet(&self) {
        println!("Hello, my name is {}", self.name);
    }
}

fn main() {
    let person = Person { name: "Herman" };
    person.greet();
}
```
##String or &str In struct

现在的问题是是否使用一个String或一个&str在你的结构中。换句话说，什么时候我们要使用一个引用到结构中的另一个类型？我们应该使用一个引用如果我们的结构不需要拥有变量的所有权。这个概念可能有点模糊，但让我用一些规则来回答。

* 我是否需要使用变量在我的结构以外？
这里是一个做作的例子：

```rust
struct Person {
    name: String,
}

impl Person {
    fn greet(&self) {
        println!("Hello, my name is {}", self.name);
    }
}

fn main() {
    let name = String::from_str("Herman");
    let person = Person { name: name };
    person.greet();
    println!("My name is {}", name); // 所有权移动错误
}
```
我应该在这里使用一个引用，因为我以后还需要使用这个变量。这是一个真实的例子[rustc_serialize](https://github.com/rust-lang/rustc-serialize/blob/master/src/json.rs#L552)。`Encoder`结构不需要拥有`std::fmt::Write`实现中的writer变量，只是借用它一会儿。事实上，String实现Write。在这个例子中，使用[encode](https://github.com/rust-lang/rustc-serialize/blob/master/src/json.rs#L372)函数，`String`类型的变量传递给编码器，然后返回`encode`到编码的调用方。

* 我的类型是大的吗？如果类型是大的，那么通过引用传递将节省不必要的内存开支。请记住，通过引用并不会引起变量拷贝。考虑一个包含大量数据的String缓冲区。复制那些将会导致程序变慢得多。

我们现在应该能够创建函数接受字符串是否为&str, String 或者 event reference counted。我们也可以创建有某些变量拥有引用的结构。这个结构的使用期被联系到这些被引用的变量以确保结构不比引用活的短，而避免造成对程序不好的影响。我们也有一个初步的了解，我们的结构的变量是否应该是类型或引用类型。

##What about 'static

值得一提的是，我们可以用一个`'static` 使用期来让我们最初的例子通过编译，但我得警告你：

```rust
struct Person {
    name: &'static str,
}

impl Person {
    fn greet(&self) {
        println!("Hello, my name is {}", self.name);
    }
}

fn main() {
    let person = Person { name: "Herman" };
    person.greet();
}
```
`'static`使用期对整个程序的有效。你可能不需要Person或name存在这么久。

##Related

* [Creating a Rust function that accepts String or &str](http://hermanradtke.com/2015/05/06/creating-a-rust-function-that-accepts-string-or-str.html)
* [中文](https://github.com/ScottHuangZL/Rust-Articles-Translation/blob/master/creating-a-rust-function-that-accepts-string-or-str.md)
