原文：   https://github.com/nrc/r4cppp/blob/master/arrays.md
翻译者：  Scott Huang   
日期：   August 22,2015 于 厦门

# 数组和向量 Arrays and Vectors

Rust arrays are pretty different from C arrays. For starters they come in
statically and dynamically sized flavours. These are more commonly known as
fixed length arrays and slices. As we'll see, the former is kind of a bad name
since both kinds of array have fixed (as opposed to growable) length. For a
growable 'array', Rust provides the `Vec` collection.

Rust数组和C数组很不同。新人觉得数组有静态的和动态的容量特性而被引入。 这些更经常被认为是固定长度的数组Array和切片Slice。正如我们所见，前者名字起的不好，因为所有的数组Array拥有固定的（和可增长相对）长度。对于可变长的数组`array`，Rust提供向量`Vec`集合。


## 固定长度的数组  Fixed length arrays

The length of a fixed length array is known statically and features in it's
type. E.g., `[i32; 4]` is the type of an array of `i32`s with length four.

固定长度的数组Array有已知的静态长度和类型。例如： `[i32; 4]`是指类型为`i32`的长度为4的数组。

Array literal and array access syntax is the same as C:
数组字面量和存取语法和C语言一致：

```rust
let a: [i32; 4] = [1, 2, 3, 4];     // As usual, the type annotation is optional.  一般来说，类注释是可选的
println!("The second element is {}", a[1]);
```

You'll notice that array indexing is zero-based, just like C.
你会发现数组的索引以0开始，就像C一样。

However, unlike C/C++, array indexing is bounds checked. In fact all access to
arrays is bounds checked, which is another way Rust is a safer language.
然而，和C/C++不一样，Rust数组的索引动作会检查边界。实际上所有关于数组的存取动作都会进行边界检查，这也从另一方面说明Rust是一个安全的语言。

If you try to do `a[4]`, then you will get a runtime panic. Unfortunately, the
Rust compiler is not clever enough to give you a compile time error, even when
it is obvious (as in this example).
如果你尝试`a[4]`,你会得到一个运行时恐慌panic。不幸的是,Rust编译器还没有足够聪明到给你编译时错误，即使有时候错误看起来显而易见（就如本例）。

If you like to live dangerously, or just need to get every last ounce of
performance out of your program, you can still get unchecked access to arrays.
To do this, use the `get_unchecked` method on an array. Unchecked array 
accesses
must be inside an unsafe block. You should only need to do this in the rarest
circumstances.
如果你喜欢不安全，或者必须榨出你程序的最后一点性能，你仍然可以不检查数组的边界。这么做的话，用`get_unchecked`方法在数组Array上。不检查数组边界的语句必须包在unsafe的语句边界内。
你应该只在极少数情况下使用这种方法。

Just like other data structures in Rust, arrays are immutable by default and
mutability is inherited. Mutation is also done via the indexing syntax:
和Rust其他数据结构一样，数组缺省是不可变的，可变可以从继承获取。改变值也是通过索引语法来完成：

```rust
let mut a = [1, 2, 3, 4];
a[3] = 5;
println!("{:?}", a);
```

And just like other data, you can borrow an array by taking a reference to it:
就像其它数据一样，你可以通过获取引用来借用数组的使用权：

```rust
fn foo(a: &[i32; 4]) {
    println!("First: {}; last: {}", a[0], a[3]);
}

fn main() {
    foo(&[1, 2, 3, 4]);
}
```

Notice that indexing still works on a borrowed array.
注意的是，索引动作仍然对借来的数组有效。

This is a good time to talk about the most interesting aspect of Rust arrays for
C++ programmers - their representation. Rust arrays are value types: they are
allocated on the stack like other values and an array object is a sequence of
values, not a pointer to those values (as in C). So from our examples above, 
这是一个恰当的时机来谈谈Rust数组中一些最让C++程序员感兴趣的方面 - 它们的表现。Rust数组是值类型的：它们在栈Stack分配内存，和其他值一样，数组对象是一系列的值，
并不是一个指针指向这些值（C是这么干的）。所以从我们前面的例子，
`let
a = [1_i32, 2, 3, 4];` will allocate 16 bytes on the stack and executing 将会从栈分配16字节并执行`
let b= a;` will copy 16 bytes. If you want a C-like array, you have to explicitly
make a pointer to the array, this will give you a pointer to the first element.
将会拷贝16字节。  如果你喜欢一个C-类似的数组，你不得不显式为数组制造一个指针来指向第一个元素。

A final point of difference between arrays in Rust and C++ is that Rust arrays
can implement traits, and thus have methods. To find the length of an array, for
example, you use `a.len()`.
Rust和C++关于数组最后一点不一样的地方是可以实现特性Traits，然后拥有方法。比如你可以用`a.len()`来查出数组的长度。

## 切片 Slices

A slice in Rust is just an array whose length is not known at compile time. The
syntax of the type is just like a fixed length array, except there is no length:
一个切片在Rust看来仅仅是一个在编译期时长度未知的数组Array。类型的语法就像是长度固定的数组，除了没有长度：
e.g., `[i32]` is a slice of 32 bit integers (with no statically known length).
比如，`[i32]`是一个32位整数的切片（在静态编译时还未知长度）

There is a catch with slices: since the compiler must know the size of all
objects in Rust, and it can't know the size of a slice, then we can never have a value with slice type. If you try and write `fn foo(x: [i32])`, for example, the compiler will give you an error.
slices要注意的一点：由于Rust编译器必须知道所有对象的长度，而slice的长度未知，因此我们从来不能指定slice作为一个值的类型。 如果你尝试写`fn foo(x: [i32])`，编译器就会报错。

So, you must always have pointers to slices (there are some very technical
exceptions to this rule so that you can implement your own smart pointers, but
you can safely ignore them for now). You must write `fn foo(x: &[i32])` (a
borrowed reference to a slice) or `fn foo(x: *mut [i32])` (a mutable raw pointer
to a slice), etc.
因此，你必须一直有指针指向切片slices（这条规定有一些非常技术性的期望，所以你可以实现你自己的智能指针，不过，你现在可以暂时安全的忽略它）。你必须这样写`fn foo(x: &[i32])` (一个借用来的引用指向切片slice) 或者 `fn foo(x: *mut [i32])` (一个可写的原始指针来指向切片slice), 等等。


The simplest way to create a slice is by coercion. There are far fewer implicit
coercions in Rust than there are in C++. One of them is the coercion from fixed
length arrays to slices. Since slices must be pointer values, this is
effectively a coercion between pointers. 
创建切片slice的最简单的方法是强制coercion转换。Rust比C++更少采用隐式转换。其中一种强制转换可以把固定长度的数组转为切片slices。由于切片必须指向值，这是一种有效率的值和指针之间的强制转换。For example, we can coerce举例，我们可以转换 `&[i32; 4]`
到 `&[i32]`, 例如,

```rust
let a: &[i32] = &[1, 2, 3, 4];
```

Here the right hand side is a fixed length array of length four, allocated on
the stack. We then take a reference to it (type `&[i32; 4]`). That reference is
coerced to type `&[i32]` and given the name `a` by the let statement.
这里，右边是一个从栈分配的固定长度为4的数组，然后取一个引用指向它(type `&[i32; 4]`)。那个引用强制转换类型为`&[i32]`，并且用let声明取名为`a`。

Again, access is just like C (using `[...]`), and access is bounds checked. You
can also check the length yourself by using `len()`. So clearly the length of
the array is known somewhere. In fact all arrays of any kind in Rust have known
length, since this is essential for bounds checking, which is an integral part
of memory safety. The size is known dynamically (as opposed to statically in the
case of fixed length arrays), and we say that slice types are dynamically sized
types (DSTs, there are other kinds of dynamically sized types too, they'll be
covered elsewhere).
再次强调，存取就像C(用`[...]`)，并且存取会进行边界检查。你还可以用`len()`来自己检查长度。所以，很明显，数组的长度是在某处已知的。实际上Rust的所有类型的数组都有已知长度，因为这是边界检查的必要条件，而这是保证内存安全的必须部分。大小已知是动态变化的（和静态的长度固定的数组相对立），并且我们说切片slice类型是动态大小类型（DSTs，还有其他种动态大小类型，它们会在其它地方提到）。


Since a slice is just a sequence of values, the size cannot be stored as part of
the slice. Instead it is stored as part of the pointer (remember that slices
must always exist as pointer types). A pointer to a slice (like all pointers to
DSTs) is a fat pointer - it is two words wide, rather than one, and contains the
pointer to the data plus a payload. In the case of slices, the payload is the
length of the slice.
So in the example above, the pointer `a` will be 128 bits wide (on a 64 bit
system). The first 64 bits will store the address of the `1` in the sequence
`[1, 2, 3, 4]`, and the second 64 bits will contain `4`. Usually, as a Rust
programmer, these fat pointers can just be treated as regular pointers. But it
is good to know about (in can affect the things you can do with casts, for
example).
由于切片slice仅仅是一系列值，大小不能存为切片的一部分。相反，它被存为指针的一部分（记得切片必须总是以指针类型存在）。一个指向切片slice的指针（像所有DSTs指针）是一个胖指针 - 有两个字words宽，而不仅一个字宽,并且指向数据加一个有效负荷。这里，负荷指的是切片的长度。
所以，上述例子，指针`a`将有128 bits宽（在64位系统中）。第一个64 bits将存序列`[1,2,3,4]`中`1`的地址。通常，作为Rust程序员，这些胖指针可以仅看待为一个普通指针。但应该知道它的原理（比如它可以影响casts转换等等）

### 切片符号和范围   slicing notation and ranges

A slice can be thought of as a (borrowed) view of an array. So far we have only
seen a slice of the whole array, but we can also take a slice of part of an
array. There is a special notation for this which is like the indexing
syntax, but takes a range instead of a single integer. E.g., `a[0..4]`, which
takes a slice of the first four elements of `a`. Note that the range is
exclusive at the top and inclusive at the bottom. Examples:
一个切片可以看作是数组的视图（借来的）。到现在为止，我们仅仅看到整个数组的切片，但我们也可以取数组的部分作为切片。这里有一个特殊的符号，就像索引语法，但获取一个范围而不是一个简单的整数。比如，`a[0..4]`，就是取`a`的前面4个元素。注意，范围是在起始位置是包含的，而在结尾部分是排除的（译者:原文有错，这边直接改了）。比如：

```rust
let a: [i32; 4] = [1, 2, 3, 4];
let b: &[i32] = &a;   // Slice of the whole array. 整个数组的切片
let c = &a[0..4];     // Another slice of the whole array, also has type &[i32]. 另外一种整个数组的切片，并且带有类型 &[i32]
let c = &a[1..3];     // The middle two elements, &[i32]. 中间两个元素
let c = &a[1..];      // The last three elements. 最后三个
let c = &a[..3];      // The first three element. 前面三个
let c = &a[..];       // The whole array, again. 所有
let c = &b[1..3];     // We can also slice a slice. 可以取切片的切片
```

Note that in the last example, we still need to borrow the result of slicing.
The slicing syntax produces an unborrowed slice (type: `[i32]`) which we must
then borrow (to give a `&[i32]`), even if we are slicing a borrowed slice.
注意最后一个例子，我们需要借用切片动作的结果。这个切片动作的语法产生一个没有借用的切片（类型：`[i32]`),我们必须接着借用（得到 a `&[i32]`），以至于我们在一个切片基础上继续切片。

Range syntax can also be used outside of slicing syntax. `a..b` produces an
iterator which runs from `a` to `b-1`. This can be combined with other iterators
in the usual way, or can be used in `for` loops:
范围语法也可以用在切片语法的外面。 `a..b`产生一个迭代器从`a`到`b-1`。这可以和其他迭代器结合在一起，通常，可以用在`for`循环中：

```rust
// Print all numbers from 1 to 10. 打印所有的数据，从1到10
for i in 1..11 {
    println!("{}", i);
}
```

## 向量 Vecs

A vector is heap allocated and is an owning reference. Therefore (and like
`Box<_>`), it has move semantics. We can think of a fixed length array
analogously to a value, a slice to a borrowed reference. Similarly, a vector in
Rust is analogous to a `Box<_>` pointer.
It helps to think of `Vec<_>` as a kind of smart pointer, just like `Box<_>`,
rather than as a value itself. Similarly to a slice, the length is stored in the
'pointer', in this case the 'pointer' is the Vec value.
一个向量vector从堆heap中分配内存并且有自己的引用。因此（就像`Box<_>`）,它带有移动语义。我们可以想象一个固定长度的数组类似一个值，一个切片来借用引用。 同样的，想象Rust中一个向量vector类似一个`Box<_>`指针。 把`Vec<_>`想象为某种智能指针而不是一个值本身，就像`Box<_>`。和切片slice类似，长度是存在指针`pointer`里，这种情况下`pointer`就是向量Vec的值。


A vector of `i32`s has type `Vec<i32>`. There are no vector literals, but we can
get the same effect by using the `vec!` macro. We can also create an empty
vector using `Vec::new()`:
向量`i32`s拥有类型`Vec<i32>`。没有向量字变量，但我没可以用`vec!`宏取得同样效果。我们也可以用`Vec::new()`来创建空的向量。


```rust
let v = vec![1, 2, 3, 4];      // A Vec<i32> with length 4.  长度为4的Vec<i32>
let v: Vec<i32> = Vec::new();  // An empty vector of i32s. i32s的空向量
```

In the second case above, the type annotation is necessary so the compiler can
know what the vector is a vector of. If we were to use the vector, the type
annotation would probably not be necessary.
在上述第二个例子，类型注释是必不可少的，这样编译器就知道向量用来装啥了。如果向量有内容，那么类型注解则不是必须的。

Just like arrays and slices, we can use indexing notation to get a value from
the vector (e.g., `v[2]`). Again, these are bounds checked. We can also use
slicing notation to take a slice of a vector (e.g., `&v[1..3]`).
就像数组arrays和切片slices，我们可以用索引符号来从向量vector中取得一个值（比如`v[2]`)。 再次，这些都会做边界检查。我们也可以用切片符号来从向量获取切片（比如，`&v[1..3]`)。


The extra feature of vectors is that their size can change - they can get longer
or shorter as needed. For example, `v.push(5)` would add the element `5` to the
end of the vector (this would require that `v` is mutable). Note that growing a
vector can cause reallocation, which for large vectors can mean a lot of
copying. To guard against this you can pre-allocate space in a vector using
`with_capacity`, see the [Vec docs](https://doc.rust-lang.org/std/vec/struct.Vec.html)
for more details.
向量vectors的额外特色是它们的容量大小可以改变 - 它们可以按需变长或者变短。 例如，`v.push(5)`将把元素`5`加入向量vector的末尾（这将要求`v`是可变的）。注意，改变向量会导致重新分配内存，对于大的向量来说这意味着一堆拷贝动作。为了避免这个动作，你可以用`with_capacity`预先为向量分配空间，请参考[Vec docs](https://doc.rust-lang.org/std/vec/struct.Vec.html)

## `索引`特质 The `Index` traits

Note for readers: there is a lot of material in this section that I haven't
covered properly yet. If you're following the tutorial, you can skip this
section, it is a somewhat advanced topic in any case.
读者请注意：　在这一节，我还有很多资料没有很好的谈到过，如果你是按照培训资料顺序阅读的话，你可以跳过这一节，底下是一些高级话题。

The same indexing syntax used for arrays and vectors is also used for other
collections, such as `HashMap`s. And you can use it yourself for your own
collections. You opt-in to using the indexing (and slicing) syntax by
implementing the `Index` trait. This is a good example of how Rust makes
available nice syntax to user types, as well as built-ins (`Deref` for
dereferencing smart pointers, as well as `Add` and various other traits, work in a similar way).
相同的索引indexing语法被同时用给数组和向量，以及其他一些集合collections，比如`HashMap`s。并且你可以用在你自己的集合类。你可以选择性的用索引（和切片slicing)语法来实现`Index`特质。这是一个好例证来说明Rust可以如何用好的语法来同时服务内置的和用户的类型（`Deref`用来为智能指针解引用，就像`Add`以及其他各种各样的特质都用相似的方法起作用）


The `Index` trait looks like
`Index`特质看起来像：

```rust
pub trait Index<Idx: ?Sized> {
    type Output: ?Sized;

    fn index(&self, index: Idx) -> &Self::Output;
}
```

`Idx` is the type used for indexing. For most uses of indexing this is `usize`.
For slicing this is one of the `std::ops::Range` types. `Output` is the type
returned by indexing, this will be different for each collection. For slicing it
will be a slice, rather than the type of a single element. `index` is a method
which does the work of getting the element(s) out of the collection. Note that
the collection is taken by reference and the method returns a reference to the
element with the same lifetime.
`Idx`是用来索引的类型。对于大多数的索引indexing，这是`usize`类型。对于切片来说，这是一个`std::ops：：Range`类型。`Output`是一种用来返回索引的类型，因集合不同而不同。 对于切片动作slicing来说，它将返回切片，而不是一个简单类型元素。注意，集合将通过引用获得，并且这个方法返回一个具有一样的生命周期lifetime的引用给那个元素。



Let's look at the implementation for `Vec` to see how what an implementation
looks like:
让我们研究`Vec`的实现，来看一看一个实现长啥样：

```rust
impl<T> Index<usize> for Vec<T> {
    type Output = T;

    fn index(&self, index: usize) -> &T {
        &(**self)[index]
    }
}
```

As we said above, indexing is done using `usize`. For a `Vec<T>`, indexing will
return a single element of type `T`, thus the value of `Output`. The
implementation of `index` is a bit weird - `(**self)` gets a view of the whole
vec as a slice, then we use indexing on slices to get the element, and finally
take a reference to it.
正如我们上面说的，索引indexing采用`usize`。对于`Vec<T>`,索引动作将返回一个类型为`T`的单一元素，就是`Output`的值。`index`的实现看起来有点怪异 - `(**self)`获取整个向量的切片视图，然后我们用切片索引动作来获取元素，最后取得指向它的一个引用。

If you have your own collections, you can implement `Index` in a similar way to
get indexing and slicing syntax for your collection.
如果你有你自己的集合，你可以用类似的方法实现`Index`。

## 初始化语法  Initialiser syntax

As with all data in Rust, arrays and vectors must be properly initialised. Often
you just want an array full of zeros to start with and using the array literal
syntax is a pain. So Rust gives you a little syntactic sugar to initialise an
array full of a given value: `[value; len]`. So for example to create an array
with length 100 full of zeros, we'd use `[0; 100]`.
和Rust里面的所有数据一样，数组和向量必须恰当的初始化。你经常想要一个初始值都是0的数组，如果用数组字面量语法的将写的很痛苦。所以Rust给你一点语法糖来初始化给定值的数组：`[value; len]`。所以，比如要创建一个含100个0，你可以用`[0; 100]`


Similarly for vectors, `vec![42; 100]` would give you a vector with 100
elements, each with the value 42.
类似的，`vec![42; 100]`将创建一个含100个元素的向量，并且初始值都是42。


The initial value is not limited to integers, it can be any expression. For
array initialisers, the length must be an integer constant expression. For
`vec!`, it can be any expression with type `usize`.
初始值并没有限制为整数，它可以是任何表达式。对于数组的初始化，长度必须是整数常量表达式。对于`vec!`，它可以使任何返回`usize`的表达式。
