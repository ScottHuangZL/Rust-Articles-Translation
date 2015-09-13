* 原文： 
    * http://hermanradtke.com/2015/05/06/creating-a-rust-function-that-accepts-string-or-str.html
    * Written by Herman J. Radtke III on 06 May 2015
* 翻译者： Scott Huang
* 翻译日期： Sep 13, 2015 于厦门

#Creating a Rust function that accepts String or &str 创建一个接受String或&str的Rust函数

在我的[上一篇博文](https://github.com/ScottHuangZL/Rust-Articles-Translation/blob/master/string-vs-str-in-rust-functions.md)
我们谈了很多关于使用&str作为函数接受一个string参数的优选。在那篇文章快结尾时，我们讨论了一些何时在结构中使用`String`还是`&str`。我认为这个建议是好的，但在有些情况下，使用&str代替string不是最优的，我们需要另一种策略来对付这些使用用例。

#A struct Containing Strings 包含字符串的结构

考虑如下Person结构。为了讨论，让我们假设Person有一个真正需要拥有所有权的name变量。因此我们选择使用String类型而不是&str。
```rust
struct Person {
    name: String,
}
```
现在我们需要实现一个new()函数。根据我的上一篇博客的讨论，我们倾向使用&str：
```rust
impl Person {
    fn new (name: &str) -> Person {
        Person { name: name.to_string() }
    }
}
```

只要我们记得在`new()`函数里调用`.to_string()`，上述代码可以工作。然而，这个函数的人体工程学设计的不太理想。如果我们使用string字面量，那么我们可以像`Person.new("Herman")`那样创建一个新的Person。如果我们已经有一个String，我们需要一个引用来指向这个String：
```rust
let name = "Herman".to_string(); //String
let person = Person::new(name.as_ref());
```
这感觉就像我们在兜圈子。我们有一个`String`，然后我们调用`as_ref()`把它变成一个`&str`，然后把它放回了new()函数，使用`.to_stirng()`再把它变回`String`。我们可以回去，使用一个String像使用`fn new(name: String) -> Person {`，但这意味着我们需要强迫调用者当使用string字面量时要`.to_string()`。

#Into conversions  Into转换

我们可以用[Into trait](http://doc.rust-lang.org/nightly/core/convert/trait.Into.html)来使我们的函数更容易被使用者调用。这一特性将自动转换为一个&str到一个string。如果我们已经有了一个String，则没有转换发生。
```rust
struct Person {
    name: String,
}

impl Person {
    fn new<S: Into<String>>(name: S) -> Person {
        Person { name: name.into() }
    }
}

fn main() {
    let person = Person::new("Herman");  //&str
    let person = Person::new("Herman".to_string()); //String
}
```
 
这个new()语法看起来有点不同。我们使用[泛型](http://doc.rust-lang.org/nightly/book/generics.html)和[特性](http://doc.rust-lang.org/nightly/book/traits.html)来告诉Rust那个S类型必须为Sring类型实现Into特质。String类型实现`Into<String>`为空操作，因为我们已经有一个String。 `&str`类型实现`Into<String>`采用相同的`.to_string()`方法像在我们原本在new()函数做的那样。所以我们不是回避需要`.to_string()`调用，但我们正在为调用者取走调用需求。你可能会问，如果使用`Into<String>`会影响性能吗，答案是否定的。Rust使用[静态调度](http://doc.rust-lang.org/nightly/book/trait-objects.html#static-dispatch)和[单型](http://stackoverflow.com/questions/14189604/what-is-monomorphisation-with-context-to-c/14198060#14198060)的概念在编译期来处理所有的这些。

别担心那些像静态调度和单型的事情会混淆不清。你只需要知道使用上面的语法你可以创建一个可以同时接受String和&str的函数。如果你认为`fn new<S: Into<String>>(name:S) -> Person {`拥有大量的语法，它是。重要的是要指出，`Into<String>`没有什么特别的，它只是一个特性，是Rust标准库的一部分。如果你想的话，你可以自己实现这一特性。你可以实现类似的你发现有用的特质并把他们发布到[crates.io](https://crates.io/)。所有这些用户创建能力使得Rust成为一个令人敬畏的语言。

#Another Way To Write Person::new() 另外一种写Person::new()的方法

下面的另外一种写法在语法上可能更容易阅读，尤其是当函数签名变得更加复杂：

```rust
struct Person {
    name: String,
}

impl Person {
    fn new<S>(name: S) -> Person where S: Into<String> {
        Person { name: name.into() }
    }
}
```
#相关链接

* [String vs &str in Rust functions](https://github.com/ScottHuangZL/Rust-Articles-Translation/blob/master/string-vs-str-in-rust-functions.md)
* [Creating a Rust function that returns a &str or String](http://hermanradtke.com/2015/05/29/creating-a-rust-function-that-returns-string-or-str.html)
