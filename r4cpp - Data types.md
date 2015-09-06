原文： https://github.com/nrc/r4cppp/blob/master/data%20types.md
翻译者： Scott Huang 
翻译日期： Sep 6, 2015 于厦门

# Data types 数据类型

在这篇文章中，我将讨论Rust的数据类型。这些大致相当于C++中的类，结构，和枚举。一个区别是Rust的数据和行为比C++(或Java，或其他面向对象语言)更严格地分开。行为是由函数定义的，可以定义在
特质traits和`impl`s（实现），但traits不能包含数据，它们类似于Java的接口。我将在下一篇文章谈到traits和impls，这一篇都是关于数据。

## Structs 结构

一个Rust的结构类似于C或C++中没有方法的结构，它简单地列出字段。最好是举一个例子来看看语法：

```rust
struct S {
    field1: int,
    field2: SomeOtherStruct
}
```

在这里，我们定义了一个带有两个字段的结构称为`S`。字段由逗号分隔，如果你喜欢，你也可以用逗号结束最后一个字段。

结构引入一个类型。例如，我们可以使用`S`作为一种类型。`SomeOtherStruct`是另一个结构(在本例中作为一个类型)，像C++，它包含值，也就是没有指针指到另一个在内存中的结构对象。

结构的字段使用操作符`.`和它们的名字存取。一个使用结构的例子：
```rust
fn foo(s1: S, s2: &S) {
    let f = s1.field1;
    if f == s2.field1 {
        println!("field1 matches!");
    }
}
```

这里`s1`是传值获得的结构对象，`s2`是传引用获得的一个结构对象。至于调用方法，我们使用相同的`.`来访问字段，不需要的`->`。结构初始化使用struct字面量。它们是结构每个字段的名字和值。例如，

```rust
fn foo(sos: SomeOtherStruct) {
    let x = S { field1: 45, field2: sos };  // 用一个结构字面量初始化x
    println!("x.field1 = {}", x.field1);
}
```

结构不能是递归的，即结构的字段名字和类型不能与结构相同而导致循环。这是结构的值语义所导致的。例如，`struct R { r: Option<R>}`是非法的，将导致编译错误(见下面更多关于选项)。如果你需要这样的结构，你应该使用一些指针，指针循环是允许的：

```rust
struct R {
    r: Option<Box<R>>
}
```

如果我们上面的结构没有`Option`，就没有办法实例化struct，并且Rust会发出错误信号。
没有字段的结构不需要使用花括号，但定义确实需要；来结尾，这大概只是为了方便编译器解析。

```rust
struct Empty;

fn foo() {
    let e = Empty;
}
```

## Tuples 元组

元组是匿名的，异构数据序列。作为一种类型，他们是在圆括号中声明为一个序列类型。因为没有名字，他们是通过结构来识别。例如，类型`(int，int)`是一对整数和`(i32，f32，S)`是三元组。元组的值初始化
和元组类型声明采用同样的方式，但以值代替类型组件，例如，`(4，5)`。一个例子：

```rust
// foo利用一个结构并返回一个元组
fn foo(x: SomeOtherStruct) -> (i32, f32, S) {
    (23, 45.82, S { field1: 54, field2: x })
}
```

元组可以通过一个`let`表达式来解构，例如，

```rust
fn bar(x: (int, int)) {
    let (a, b) = x;
    println!("x was ({}, {})", a, b);
}
```

我们会在下次再具体谈谈解构。

## Tuple structs  元组结构

元组结构称为元组，或者，未命名的字段结构。他们使用`struct`关键字声明，一组类型被列在括号中，
加上一个分号。这样的声明将它们的名称作为一种类型。他们的字段必须被解构（如元组）后才能访问，而不是按名字访问。元组结构不是很常见。

```rust
struct IntPoint (int, int);

fn foo(x: IntPoint) {
    let IntPoint(a, b) = x;  // 注意，我们需要元组结构的名字来解构
    println!("x was ({}, {})", a, b);
}
```

## Enums 枚举

枚举类型像C++的枚举或联合，是可以获取多值的类型。最简单的一种枚举类型就像C++枚举：

```rust
enum E1 {
    Var1,
    Var2,
    Var3
}

fn foo() {
    let x: E1 = Var2;
    match x {
        Var2 => println!("var2"),
        _ => {}
    }
}
```

然而，Rust的枚举比这强大的多。每一个变体可以包含数据。像元组，这些是由一列的类型所定义。在这种情况下，他们更像C++的联合而不是C++的枚举。Rust枚举是有标记的联合而不是非标记的联合（在C++），这意味着你不会在运行时误用一个枚举的变量到另外一个枚举的变量。一个例子：

```rust
enum Expr {
    Add(int, int),
    Or(bool, bool),
    Lit(int)
}

fn foo() {
    let x = Or(true, false);   // x 有类型Expr
}
```

许多面向对象多态的简单的案例都可以在Rust中用枚举来处理的更好。使用枚举我们通常使用模式匹配。记住，这些和C++的开关语句类似。我将在下次再深入研究用这以及其他方式来解构数据。下面是一个例子：

```rust
fn bar(e: Expr) {
    match e {
        Add(x, y) => println!("An `Add` variant: {} + {}", x, y),
        Or(..) => println!("An `Or` variant"),
        _ => println!("Something else (in this case, a `Lit`)"),
    }
}
```

模式匹配的每一个分支匹配一个`Expr`变体。所有变体必须被覆盖。最后一种情况下（`_`）涵盖了所有剩余的变体，虽然在本例中只有`Lit`。变体中的任何数据都可以被绑定到一个变量中。
在`Add`分支我们绑定两个整数在到`x`和`y`。如果我们不关心的数据，我们可以使用`..`来匹配任何数据，如我们在`Or`分支所做的那样。

## Option 选项

在一个特别常见的Rust枚举是`Option`。它有2个变体-`Some`和`None`。`None`没有数据和`Some`有一个单一的类型为`T`的字段（`Option`是一个通用的枚举，我们将在后面谈，但希望和C++分的很清楚）。Options用来表示一个值可能有或可能没。在C++中，任何使用一个空指针来表示一个值在某种程度上是不确定的，未初始化，或假的地方，你都应该使用Rust的Option。使用Option是安全的，因为您必须总是在使用前检查它；没有办法解引用一个空指针。他们也更通用，你可以就像使用指针那样使用它们的值。一个例子
```rust
use std::rc::Rc;

struct Node {
    parent: Option<Rc<Node>>,
    value: int
}

fn is_root(node: Node) -> bool {
    match node.parent {
        Some(_) => false,
        None => true
    }
}
```


在这里，父字段可以是一个`None`或者一个包含一个`Rc<Node>`的`Some`。在例子中，我们从来没有真正使用该有效载荷，但在现实生活中你通常会。

还有一些方便的方法，所以你可以在主体中像`node.is_none()`或`!node.is_some()`那样写`is_root`。

## Inherited mutabilty and Cell/RefCell  继承的可变性和Cell/RefCell

Rust的局部变量是默认不可变的，可以使用标记`mut`来使其可变。我们在结构或枚举里并没有标记字段为可变的，他们的可变性是继承得来的。这意味着，在一个结构对象的字段是可变或不可变的取决于对象本身是可变的和不变的。例子：

```rust
struct S1 {
    field1: int,
    field2: S2
}
struct S2 {
    field: int
}

fn main() {
    let s = S1 { field1: 45, field2: S2 { field: 23 } };
    // s 是深度不可变的，下面的改变动作是被禁止的
    // s.field1 = 46;
    // s.field2.field = 24;

    let mut s = S1 { field1: 45, field2: S2 { field: 23 } };
    // s 是可变的，下面是允许的
    s.field1 = 46;
    s.field2.field = 24;
}
```

Rust的继承可变性在使用引用时停止。这是类似于C++,你可以通过const对象的指针来修改非const对象。如果你想要要改变一个引用字段，你必须在字段类型上使用`&mut`：

```rust
struct S1 {
    f: int
}
struct S2<'a> {
    f: &'a mut S1   // 可变引用字段
}
struct S3<'a> {
    f: &'a S1       // 不可变引用字段
}

fn main() {
    let mut s1 = S1{f:56};
    let s2 = S2 { f: &mut s1};
    s2.f.f = 45;   // 合法，即使s2自己是不可变的
    // s2.f = &mut s1; // 不合法，s2是不可变的
    let s1 = S1{f:56};
    let mut s3 = S3 { f: &s1};
    s3.f = &s1;     // 合法，s3是可变的
    // s3.f.f = 45; // 不合法，s3.f 是不可变的
}
```

（在`S2`和`S3`上的`'a`是使用期参数，我们会很快涉及到他们）

有时，当对象在逻辑上是不可变的，它有部分需要内部是可变的。考虑各种缓存或引用计数（这不会带来真正的逻辑上的不可变性，因为改变引用计数的影响可以通过析构观察到）。在C++中，即使对象是const的，你也可以使用`mutable`关键词允许这样的改变。在Rust里，我们有Cell和RefCell。这些允许不可变对象的一些部分是可变的。虽然这是有用的，但这意味着你需要知道，当你看到Rust里一个不可改变的对象，它的有些部分可能实际上是可变的。

RefCell和Cell让你绕过Rust的可变性和别名性的严格规定。他们可以被安全的使用，因为他们确保Rust的不变量都是受注意的动态，即使编译器不能保证这些不变保持静态。Cell和RefCell都是单线程的对象。

使用带有复制语义的Cell类型（相当多的只是原始类型）。Cell有`get`和`set`的方法，用于改变存储的值，和一个`new`方法来初始化单元值。Cell是一个非常简单的对象，不需要任何智能，因为带有复制语义的对象不能保持引用在Rust的其他地方，他们不能共享线程，所以没有太大的犯错可能。

使用具有移动语义类型RefCell，这意味着Rust里几乎所有的东西，结构对象是一个常见的例子。RefCell也用`new`来创建，并且有`set`方法。要取得RefCell里的值，你必须用借的方法借它（`borrow`，`borrow_mut`，`try_borrow`，`try_borrow_mut`）这会给你一个借来的RefCell参考对象。这些方法遵循相同的规则，作为静态的借用-你只能有一个可变的借贷，不能同时有一个可变的和不可变的借贷。不然，你将得到一个运行时失败，而不是编译错误。`try_`变体返回一个选项-如果值可以借，你得到`Some(val)`，反之，得到一个`None`。如果一个值是借来的，调用`set`将失败。这里有一个例子使用引用计数的指针指向一个RefCell（常见的情况）：

```rust
use std::rc::Rc;
use std::cell::RefCell;

Struct S {
    field: int
}

fn foo(x: Rc<RefCell<S>>) {
    {
        let s = x.borrow();
        println!("the field, twice {} {}", s.f, x.borrow().field);
        // let s = x.borrow_mut(); // 错误，我们已经借用x的内容 
    }

    let s = x.borrow_mut(); // 哦，早期的借贷已经超出范围了。
    s.f = 45;
    // println!("The field {}", x.borrow().field); 
    // 错误，不能同时拥有mut和immut借贷
    println!("The field {}", s.f);
}
```

如果你使用Cell/RefCell,，你应该尽量把它们放在最小的对象上。就是，倾向于把它们放在结构的几个字段上，而不是
整个结构。它们像单线程锁，细粒度锁定是更好的，因为你更可能避免锁冲突。