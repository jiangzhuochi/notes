## 常用库

### 命令行库

- `clap`
- `colored`
- `jsonxf` `json` 美化
- 语法高亮 `syntect`


### 命令行工具

- 文件查找 [fd](https://github.com/sharkdp/fd) 

```
cargo install fd-find
```

- 代码统计 [tokei](https://github.com/XAMPPRocky/tokei)

```
cargo install tokei
```


- 终端美化 [starship](https://github.com/starship/starship)
```
见官网
```

- 基准测试 [hyperfine](https://github.com/sharkdp/hyperfine)

```
cargo install hyperfine
```

## `match` 更新 binding 模式

```rust
// 模仿 Option

pub struct Somee<T>(T);

#[derive(Debug)]
pub enum Myop<T> {
    Nonee,
    Somee(T),
}

impl<T> Myop<T> {
    // pub fn as_reff(&self) -> Myop<&T> {
    //     match self {
    //         Myop::Somee(x) => Myop::Somee(x),
    //         Myop::Nonee => Myop::Nonee,
    //     }
    // }

    // be equivalent to
    pub fn as_reff(&self) -> Myop<&T> {
        match *self {
            Myop::Somee(ref x) => Myop::Somee(x),
            Myop::Nonee => Myop::Nonee,
        }
    }
    // 参见 https://doc.rust-lang.org/reference/patterns.html#binding-modes
    // 请注意这句话 Each time a reference is matched using a non-reference pattern, it will automatically dereference the value and update the default binding mode.

    pub fn map<U, F: FnOnce(T) -> U>(self, f: F) -> Myop<U> {
        match self {
            Myop::Somee(x) => Myop::Somee(f(x)),
            Myop::Nonee => Myop::Nonee,
        }
    }
}

fn main() {
    let a = Myop::Somee(6);
    let b = a.as_reff().map(|x| x * x);
    println!("{:?}", a);
    println!("{:?}", b)
}

```

## macOS 交叉编译树莓派

下载 [toolchains | Pre-built ARM/Linux C cross-compilers for MacOS](https://thinkski.github.io/osx-arm-linux-toolchains/)

项目根目录创建 `.cargo/config.toml`，填入

```toml
[target.armv7-unknown-linux-gnueabihf]
linker = "/Users/jrjzc/Downloads/arm-unknown-linux-gnueabihf/bin/arm-unknown-linux-gnueabihf-gcc"
```

在 `Cargo.toml` 中添加依赖

```
openssl = { version = "0.10", features = ["vendored"] }
```

终端执行

```
rustup target add armv7-unknown-linux-gnueabihf
cargo build --target armv7-unknown-linux-gnueabihf --release
```

出现 `/bin/sh: arm-linux-gnueabihf-gcc: command not found` 时，终端输入

```
export CC=/Users/jrjzc/Downloads/arm-unknown-linux-gnueabihf/bin/arm-unknown-linux-gnueabihf-gcc
```

然后再

```
cargo build --target armv7-unknown-linux-gnueabihf --release
```

中间 Mac 会提示不信任的开发者，在安全性与隐私信任即可

也可以用 musl 静态链接，但似乎 CC 仍然需要 arm-unknown-linux-gnueabihf-gcc

如果使用 armv7l-linux-musleabihf-gcc 或 arm-linux-musleabihf-gcc 则报错 cc1: error: '-mfloat-abi=hard': selected processor lacks an FPU

## 方法调用过程

### 起因

中文论坛上有人提问，Self 到底是什么类型

https://rustcc.cn/article?id=7cca66e4-1bee-480c-a119-ee10ae3147f9

### 资料

https://doc.rust-lang.org/nomicon/dot-operator.html?highlight=auto#the-dot-operator

https://stackoverflow.com/questions/28519997/what-are-rusts-exact-auto-dereferencing-rules/28552082#28552082

### 分析实例

```rust
use std::any::type_name;

trait Ext {
    fn ext_with_at_self(&self) {
        //
        println!(
            "Ext for T call with &self, type_name of Self is {}",
            type_name::<Self>()
        )
    }
    fn ext_with_self(self)
    where
        Self: Sized,
    {
        println!(
            "Ext for T call with  self, type_name of Self is {}",
            type_name::<Self>()
        )
    }
}

impl<T> Ext for T {}

trait ExtAt {
    fn ext_at_with_at_self(&self) {
        println!(
            "Ext for &T call with &self, type_name of Self is {}",
            type_name::<Self>()
        )
    }
    fn ext_at_with_self(self)
    where
        Self: Sized,
    {
        println!(
            "Ext for &T call with  self, type_name of Self is {}",
            type_name::<Self>()
        )
    }
}

impl<T> ExtAt for &T {}

fn main() {
    let a = String::new();
    println!("self is String");
    a.ext_with_at_self();
    // 第一步，由于特征为泛型T实现，接受的是&Self，即&T，明确函数接受类型：&T
    // 第二步，a的类型是String
    // 第三步，进入匹配循环，令U=String，即a的类型
    // 第四步，U是接受类型&T吗？String不是某个类型的引用
    // 第五步，&U是接受类型&T吗？是，匹配完成，且T=String
    // 第六步，传入String::ext_with_at_self(&a)
  
  	// 2022.3.7，nomicon的解释有问题
    // &String = &Self，于是Self = String
    // T::foo(value)即String::ext_with_at_self(&String)
  	// <&T>::foo(value)即<&String>::ext_with_at_self(&&String)
  	// 怎么样都无法匹配，不用这种解释
  
  	// 2022.3.7，reference的解释较好
  	// receiver是a，类型String
   	// 候选人序列：a, &a, &mut a
    // 找可见方法：针对 a 的，没有可见方法 
  	// 针对 &a 的，有可见方法 T = String 时的 String::ext_with_at_self(&String)
    // 匹配完成，结束
    a.ext_with_self();
    // 第一步，明确函数接受类型：T
    // 第四步，检查U是接受类型T吗？是，匹配完成，且T=String
    // 第六步，传入String::ext_with_self(a)
    // String = Self，于是Self = String
  
  	// reference的解释较好
  	// receiver是a，类型String
   	// 候选人序列：a, &a, &mut a
    // 找可见方法：针对 a 的，有可见方法 T = String 时的 String::ext_with_at_self(String)
    // 匹配完成，结束
    let a = &String::new();
    println!("self is &String");
    a.ext_with_at_self();
    // 第二步，a的类型是&String
    // 第三步，进入匹配循环，令U=&String，即a的类型
    // 第四步，U是接受类型&T吗？是，匹配完成，&String是类型String的引用，且T=String
    // 第六步，传入String::ext_with_at_self(a)
    // &String = &Self，于是Self = String
  	
  	// 2022.3.7
    // nomicon的dot operator一节更新了
    // a的类型是T = &String
    // 有方法<&String>::ext_with_at_self(&&String)与a不相匹配
  	// <&T>::foo(value)即<&&String>::ext_with_at_self(&&&String)
  	// <&mut T>::foo(value)即<&mut &String>::ext_with_at_self(&&mut &String)
    // 也不对
    // T: Deref<Target = U> 即 &String: Deref<Target = String>
    // 于是 String::ext_with_at_self(&String) 匹配，a是&String
    // 所以调用String::ext_with_at_self(a)，泛型T = Self = String
  	// 感觉这样是有问题的，见第一个例子
  
  	// reference的解释较好
  	// receiver是a，类型&String
   	// 候选人序列：a, &a, &mut a, *a, &*a, &mut *a,
    // 找可见方法：针对 a 的，有可见方法 T = String 时的 String::ext_with_at_self(&String)
    // 匹配完成，结束
  
    a.ext_with_self();
    // 第二步，a的类型是&String
    // 第三步，进入匹配循环，令U=&String，即a的类型
    // 第四步，U是接受类型T吗？是，匹配完成，且T=&String
    // 第六步，传入(&String)::ext_with_self(a)
    // &String = Self，于是Self = &String
  
  	// reference的解释较好
  	// receiver是a，类型&String
   	// 候选人序列：a, &a, &mut a, *a, &*a, &mut *a,
    // 找可见方法：针对 a 的，有可见方法 T = &String 时的 <&String>::ext_with_self(&String)
    // 匹配完成，结束
    // 也有针对 *a 的可见方法 T = String 时的 <String>::ext_with_self(String) 
    // 次序在后面，得不到匹配
    println!();

    let a = String::new();
    println!("self is String");
    // a.ext_at_with_at_self(); //this line refused by compiler, method not found in `String`
    // 第一步，由于特征为泛型&T实现，接受的是&Self，即&&T，明确函数接受类型：&&T
    // 第二步，a的类型是String
    // 第三步，进入匹配循环，令U=String，即a的类型
    // 第四步，U是接受类型&&T吗？String不是&&T
    // 第五步，&U是接受类型&&T吗？&String也不是&&T
    // 回到循环头，令U=*U，错误，String不能解引用
    // 退出循环，匹配失败
    // 在上面考虑泛型之前，已经考虑了a的类型String的固有方法，和特征实现方法
    // 再加上泛型方法也找不到，因此method not found in `String`
    // 上面这句话的String就是a本身的类型
  
    // reference的解释较好
  	// receiver是a，类型String
   	// 候选人序列：a, &a, &mut a
    // 找可见方法：没有可见方法
    // 最少有两个&&的可见方法<&String>::ext_at_with_at_self(&&String)
    // a无法满足，至多自动引用一次
    println!("Ext for &T call with &self, method not found in `String`");

    a.ext_at_with_self();
    // 第一步，由于特征为泛型&T实现，接受的是Self，即&T，明确函数接受类型：&T
    // 第二步，a的类型是String
    // 第三步，进入匹配循环，令U=String，即a的类型
    // 第四步，U是接受类型&T吗？String不是&T
    // 第五步，&U是接受类型&T吗？是，且T=U=String
    // 第六步，传入(&String)::ext_at_with_self(&a)
    // &String = Self，于是Self = &String

  	// reference的解释较好
  	// receiver是a，类型String
   	// 候选人序列：a, &a, &mut a
    // 找可见方法：&a 匹配 <&String>::ext_at_with_self(&String)
    let a = &String::new();
    println!("self is &String");
    a.ext_at_with_at_self();
    // 第一步，由于特征为泛型&T实现，接受的是&Self，即&&T，明确函数接受类型：&&T
    // 第二步，a的类型是&String
    // 第三步，进入匹配循环，令U=&String，即a的类型
    // 第四步，U是接受类型&&T吗？&String不是&&T
    // 第五步，&U是接受类型&&T吗？是，&&String = &&T，于是T=String
    // 第六步，传入(&String)::ext_at_with_at_self(&a)
    // &&String = &Self，于是Self = &String
  
    // reference的解释较好
  	// receiver是a，类型&String
   	// 候选人序列：a, &a, &mut a, *a, &*a, &mut *a,
    // 找可见方法：&a 匹配 <&String>::ext_at_with_at_self(&&String)
    a.ext_at_with_self();
    // 第一步，由于特征为泛型&T实现，接受的是Self，即&T，明确函数接受类型：&T
    // 第二步，a的类型是&String
    // 第三步，进入匹配循环，令U=&String，即a的类型
    // 第四步，U是接受类型&T吗？&String是&T，于是T=String
    // 第六步，传入(&String)::ext_at_with_self(a)
    // &String = Self，于是Self = &String
  
    // reference的解释较好
  	// receiver是a，类型&String
   	// 候选人序列：a, &a, &mut a, *a, &*a, &mut *a,
    // 找可见方法：a 匹配 <&String>::ext_at_with_self(&String)
}

// self is String
// Ext for T call with &self, type_name of Self is alloc::string::String
// Ext for T call with  self, type_name of Self is alloc::string::String
// self is &String
// Ext for T call with &self, type_name of Self is alloc::string::String
// Ext for T call with  self, type_name of Self is &alloc::string::String

// self is String
// Ext for &T call with &self, method not found in `String`
// Ext for &T call with  self, type_name of Self is &alloc::string::String
// self is &String
// Ext for &T call with &self, type_name of Self is &alloc::string::String
// Ext for &T call with  self, type_name of Self is &alloc::string::String

```

### 换一种视角

从上例中 `a` 本身的类型入手，见官方文档。

https://doc.rust-lang.org/reference/expressions/method-call-expr.html

```rust
//Example 1
//////////////////////////////////////////////////////////////////
struct Foo {}

trait Bar {
  fn bar(&self);
}

impl Foo {
  // Foo::bar(&mut Foo)
  fn bar(&mut self) {
    println!("In struct impl!")
  }
}
impl Bar for Foo {
  // <Foo as Bar>::bar(&Foo)
  fn bar(&self) {
    println!("In trait impl!")
  }
}

fn main() {
  let mut f = Foo{};
  f.bar();
}

// receiver是f，类型Foo
// 候选人序列: f, &f, &mut f
// 类型: Foo, &Foo, &mut Foo
// 找可见方法：
// 接受f: Foo的bar函数没有。【先看固有方法impl Foo，再看特征方法impl ... for Foo】
// 接受&f: &Foo的bar函数有吗？【先看固有方法impl Foo，再看特征方法impl ... for Foo】
// 有，存在特征方法，即<Foo as Bar>::bar(&Foo)
// 传入<Foo as Bar>::bar(&f)


//Example 2
//////////////////////////////////////////////////////////////////
trait Foo {
    fn foo(self);
}
trait Bar {
    fn bar(&self);
}
impl Foo for &String {
    fn foo(self) {
        println!("foo")
    }
}
impl Bar for String {
    fn bar(&self) {
        println!("bar")
    }
}
fn main() {
    let x = String::from("yes");
    // 候选人列表: x, &x, &mut x
    // 类型: String, &String, &mut String
    x.foo();
    // 有接受x的foo函数吗？没有，不存在by-value
    // 有接受&x的foo函数吗？有，autoref，<&String>::foo(&String)
    <&String>::foo(&x);
    x.bar();
    // 有接受x的bar函数吗？没有，不存在by-value
    // 有接受&x的bar函数吗？有，autoref，<String>::bar(&String)
    <String>::bar(&x);
}


//Example 3
//////////////////////////////////////////////////////////////////
trait Foo {
    fn foo(self);
}
trait Foo2 {
    fn foo(&self);
}
impl Foo for &String {
  	// Foo::foo(&String)
    fn foo(self) {
        println!("foo")
    }
}
impl Foo2 for String {
  	// Foo2::foo(&String)
    fn foo(&self) {
        println!("foo2")
    }
}

fn main() {
    let x = String::from("yes");
    // 候选人列表: x, &x, &mut x
    // 类型: String, &String, &mut String
    // x.foo();
    // 有接受x的foo函数吗？没有，不存在by-value
    // 有接受&x的foo函数吗？有两个，autoref
    // <&String as Foo>::foo(&String)
    // <String as Foo2>::foo(&String)
    // 此时 x.foo(); 需要用完全限定名称调用
    <&String as Foo>::foo(&x);
    <String as Foo2>::foo(&x);
}


//Example 4
//////////////////////////////////////////////////////////////////
struct MyStr {}

trait Foo {
    fn foo(self);
}
trait Foo2 {
    fn foo(&self);
}
impl Foo for &MyStr {
  	// Foo::foo(&MyStr)
    fn foo(self) {
        println!("trait foo")
    }
}
impl Foo2 for MyStr {
  	// Foo2::foo(&MyStr)
    fn foo(&self) {
        println!("trait foo2")
    }
}

impl MyStr {
  	// MyStr::foo(&MyStr)
    fn foo(&self) {
        println!("inherent foo")
    }
}

fn main() {
    let x = MyStr {};
    // 候选人列表: x, &x, &mut x
    // 类型: MyStr, &MyStr, &mut MyStr
    // 有接受x的foo函数吗？没有，不存在by-value
    // 有接受&x的foo函数吗？有三个，autoref
    // MyStr::foo(&MyStr)
    // <&MyStr as Foo>::foo(&MyStr)
    // <MyStr as Foo2>::foo(&MyStr)
    // 此时 x.foo(); 先调用inherent method
    x.foo();
    <&MyStr as Foo>::foo(&x);
    <MyStr as Foo2>::foo(&x);
}

//Example 5
//////////////////////////////////////////////////////////////////
use std::ops::Deref;

struct MyStr {}
struct MyPath {
    mystr: MyStr,
}
impl Deref for MyPath {
    type Target = MyStr;

    fn deref(&self) -> &Self::Target {
        &self.mystr
    }
}
trait Foo {
    fn foo(self);
}
trait Foo2 {
    fn foo(&self);
}
impl Foo for &MyStr {
    fn foo(self) {
        println!("trait foo")
    }
}
impl Foo2 for MyStr {
    fn foo(&self) {
        println!("trait foo2")
    }
}

impl MyStr {
    fn foo(&self) {
        println!("inherent foo")
    }
}

fn main() {
    let x = MyPath { mystr: MyStr {} };
    // 候选人列表: x, &x, &mut x, *x, &*x, &mut *x
    // 类型: MyPath, &MyPath, &mut MyPath, MyStr, &MyStr, &mut MyStr
    // 有接受x的foo函数吗？没有，不存在by-value
    // 有接受&x, &mut x的foo函数吗？没有，不存在autoref
    // autoderef *x, &*x, &mut *x
    // 有接受*x的foo函数吗？没有，不存在by-value
    // 有接受&*x的foo函数吗？有三个
    // 此时 x.foo(); 先调用inherent method
    x.foo();
    <&MyStr as Foo>::foo(&x);
    // 这里实际上还隐式的进行了https://doc.rust-lang.org/reference/type-coercions.html#coercion-types
    // &T or &mut T to &U if T implements Deref<Target = U>
    // 实际上是下面这个
    <&MyStr as Foo>::foo(&*x);

    <MyStr as Foo2>::foo(&x);
    <MyStr as Foo2>::foo(&*x);
}

```

还有个问题没搞清楚，就是确定候选者的类型之后，是怎么找到它的固有方法和特征方法的？

以下为猜想：

查找函数foo签名为String，则只能在impl String找固有方法【fn foo(self)】，在impl x for String找特征方法【fn foo(self)】。

查找函数foo签名为&String，则只能在impl String找固有方法【fn foo(&self)】，在impl x for String【fn foo(&self)】和impl x for &String【fn foo(self)】找特征方法。

查找函数foo签名为&&String，则不存在固有方法【因为[E0118](https://doc.rust-lang.org/error-index.html#E0118)不允许impl &String】，在impl x for &String【fn foo(&self)】和impl x for &&String【fn foo(self)】找特征方法。

下面这段话是否可以作证？

https://stackoverflow.com/questions/28519997/what-are-rusts-exact-auto-dereferencing-rules/28552082#28552082

>Notably, everything considers the "receiver type" of the method, *not* the `Self` type of the trait, i.e. `impl ... for Foo { fn method(&self) {} }` thinks about `&Foo` when matching the method, and `fn method2(&mut self)` would think about `&mut Foo` when matching.

因此impl x1 for String【fn foo(&self)】和impl x2 for &String【fn foo(self)】地位相同，同时被匹配，可以用x1::foo(&String)和x2::foo(&String)表示，见例子3。

```rust
x: MyStr
x.foo();
函数名 foo 候选类型列表 MyStr &MyStr &mut MyStr 
匹配 foo(MyStr) 找不到
匹配 foo(&MyStr) 找到以下三个方法，用完全限定名称表示
MyStr::foo(&MyStr)
Foo::foo(&MyStr)
Foo2::foo(&MyStr)
第一个是固有方法，因此调用 MyStr::foo(&x)
```

所以回答是，确定候选类型列表之后，对每个候选类型，按该函数名【foo】和首个参数的类型是候选类型【&String】这两个条件，确定所匹配的方法，先在固有方法中找。

## 一些符号的英文

```rust
';' => Semi,
',' => Comma,
'.' => Dot,
'(' => OpenParen,
')' => CloseParen,
'{' => OpenBrace,
'}' => CloseBrace,
'[' => OpenBracket,
']' => CloseBracket,
'@' => At,
'#' => Pound,
'~' => Tilde,
'?' => Question,
':' => Colon,
'$' => Dollar,
'=' => Eq,
'!' => Bang,
'<' => Lt,
'>' => Gt,
'-' => Minus,
'&' => And,
'|' => Or,
'+' => Plus,
'*' => Star,
'^' => Caret,
'%' => Percent,
```

## rustup 更新冲突的解决办法

先删掉出错的工具链，再安装即可

```
rustup toolchain uninstall stable
rustup toolchain install stable
```

## Rust W2L 交叉编译

https://github.com/messense/rust-musl-cross

```powershell
docker pull messense/rust-musl-cross:armv7-musleabihf

docker run --rm -it -v ${pwd}:/home/rust/src messense/rust-musl-cross:armv7-musleabihf 

cargo build --release

exit
```

或者设置别名快速操作

```powershell
Function armv7-musleabihf-release {docker run --rm -it -v ${pwd}:/home/rust/src messense/rust-musl-cross:armv7-musleabihf cargo build --release}

Set-Alias -Name armv7-release -Value armv7-musleabihf-release


Function armv7-musleabihf {docker run --rm -it -v ${pwd}:/home/rust/src messense/rust-musl-cross:armv7-musleabihf $args}

Set-Alias -Name armv7 -Value armv7-musleabihf
```

Powershell 通过函数扩展别名

https://docs.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_functions?view=powershell-5.1#positional-parameters

https://www.pstips.net/powershell-extend-alias-by-function.html

## Subtyping and Variance

https://doc.rust-lang.org/nomicon/subtyping.html

我发现函数类型的大小和偏序还有关系。t1是t2的子类，u1是u2的子类，那么f(t2)->u1是f(t1)->u1和f(t2)->u2的子类，这两之间无类的关系，但都是f(t1)->u2的子类，所以f(t2)->u1也是f(t1)->u2的子类。

宽进严出的函数是严进宽出函数的子类，也就是说一个高阶函数接受一个严进宽出函数作为参数，那么也能接受其对应的宽进严出的函数。这是因为宽进严出必然比严进宽出多做了一些什么才能保证正确性，即子类具有父类的特点之外还有更多。用rust那一章的话说就是Subtype is a Supertype and more。

也可以理解成严进宽出函数的输入范围包括在宽进严出的函数的输入范围之内，而输出范围则包括了后者的输出范围，所以可以接受前者必然可以接受后者。

rust的&mut都被认为是不变类型的，所以必然接受相等的类型，保证了接受&'_ mut S的地方不会接受&'_ mut A，即便S是A的超类，但&'_ mut S并不是&'_ mut A的超类，它们无类型关系。

```rust
&'a T = 'a + T
&'a mut T = 'a + |T|
Box<T> = T
UnsafeCell<T> = |T|
fn(T) -> U = U - T

小于号表示前者是子类
若第一个参数为参照
&mut T = &mut &'static str 
故
T = &'static str = 'static + str 
&'spike_str str = 'spike_str + str
因为
'static outlives 'spike_str
有
'static < 'spike_str
'static + str < 'spike_str + str
故
T = &'static str < &'spike_str str
第二个参数 T = &'static str 不能接受超类 &'spike_str str，拒绝

若第二个参数为参照
T = &'spike_str str
则第一个参数需要接受
&mut T = &mut &'spike_str str = |&'spike_str str| = |'spike_str + str|
实际传入
&mut &'static str = |&'static str| = |'static + str|
'static outlives 'spike_str
'static < 'spike_str
|'static + str| 与 |'spike_str + str| 无法判断大小
意味着 &mut &'static str 与 &mut &'spike_str str 无类型关系，不相匹配，拒绝
```

## Py str.translate 简单实现

```rust
use std::collections::HashMap;
use unicode_segmentation::UnicodeSegmentation;

pub trait Translate {
    fn maketrans<T, U, V>(intab: T, outtab: U) -> HashMap<Self, V>
    where
        T: IntoIterator<Item = Self>,
        U: IntoIterator<Item = V>,
        V: ToString,
        Self: Sized;

    fn translate<T>(&self, table: &HashMap<Self, T>) -> Self
    where
        T: ToString,
        Self: Sized;
}

impl Translate for String {
    fn maketrans<T, U, V>(intab: T, outtab: U) -> HashMap<Self, V>
    where
        T: IntoIterator<Item = Self>,
        U: IntoIterator<Item = V>,
        V: ToString,
    {
        intab.into_iter().zip(outtab.into_iter()).collect()
    }

    fn translate<T>(&self, table: &HashMap<Self, T>) -> Self
    where
        T: ToString,
        Self: Sized,
    {
        self.graphemes(true)
            .map(|c| table[c].to_string())
            .collect::<Vec<String>>()
            .join("")
    }
}

fn main() {
    let s = String::from("asdf我y̆");
    let v: Vec<_> = s.graphemes(true).map(|s| s.to_owned()).collect();
    let table1 = String::maketrans(v.clone(), 0..6);
    println!("{table1:?}");
    let table2 = String::maketrans(v.clone(), vec!["r", "u", "s", "t", "u", "p"]);
    println!("{table2:?}");

    let s1 = String::from("asdf我y̆我fdsa");
    let trans1 = s1.translate(&table1);
    let trans2 = s1.translate(&table2);

    println!("{s1:?}");
    println!("{trans1:?}");
    println!("{trans2:?}");
}

// {"d": 2, "a": 0, "s": 1, "我": 4, "y\u{306}": 5, "f": 3}
// {"a": "r", "s": "u", "d": "s", "f": "t", "我": "u", "y\u{306}": "p"}
// "asdf我y\u{306}我fdsa"
// "01234543210"
// "rustuputsur"
/////////////////////////////////第二版//////////////////////////////////////
use std::collections::HashMap;
use unicode_segmentation::UnicodeSegmentation;

pub trait Translate<V> {
    fn maketrans<T, U>(intab: T, outtab: U) -> HashMap<Self, V>
    where
        T: IntoIterator<Item = Self>,
        U: IntoIterator<Item = V>,
        Self: Sized;

    fn translate(&self, table: &HashMap<Self, V>) -> Self
    where
        Self: Sized;
}

impl<V: ToString> Translate<V> for String {
    fn maketrans<T, U>(intab: T, outtab: U) -> HashMap<Self, V>
    where
        T: IntoIterator<Item = Self>,
        U: IntoIterator<Item = V>,
        V: ToString,
    {
        intab.into_iter().zip(outtab.into_iter()).collect()
    }

    fn translate(&self, table: &HashMap<Self, V>) -> Self
    where
        Self: Sized,
    {
        self.graphemes(true)
            .map(|c| table[c].to_string())
            .collect::<Vec<String>>()
            .join("")
    }
}

fn main() {
    let s = String::from("asdf我y̆");
    let v: Vec<_> = s.graphemes(true).map(|s| s.to_owned()).collect();
    let table1 = String::maketrans(v.clone(), 0..6);
    println!("{table1:?}");
    let table2 = String::maketrans(v.clone(), vec!["r", "u", "s", "t", "u", "p"]);
    println!("{table2:?}");

    let s1 = String::from("asdf我y̆我fdsa");
    let trans1 = s1.translate(&table1);
    let trans2 = s1.translate(&table2);

    println!("{s1:?}");
    println!("{trans1:?}");
    println!("{trans2:?}");
}

// {"d": 2, "a": 0, "s": 1, "我": 4, "y\u{306}": 5, "f": 3}
// {"a": "r", "s": "u", "d": "s", "f": "t", "我": "u", "y\u{306}": "p"}
// "asdf我y\u{306}我fdsa"
// "01234543210"
// "rustuputsur"
```

## 生命周期问题

```rust
use std::iter::Iterator;

struct Parser {}

pub struct ParseIter<'parser> {
    parser: &'parser mut Parser,
    buf: &'parser [u8],
}

impl Parser {
    pub fn parse<'a, 'b>(&'b mut self, buf: &'a [u8]) -> Option<&'a u8> {
        Some(&buf[0])
    }

    pub fn iter<'a>(&'a mut self, buf: &'a [u8]) -> ParseIter<'a> {
        ParseIter { parser: self, buf }
    }
}

impl<'parser> Iterator for ParseIter<'parser> {
    type Item = &'parser u8;
    fn next<'next>(&'next mut self) -> Option<Self::Item> {
        // pub fn parse<'a>(&'a mut self, buf: &'a [u8]) -> Option<&'a u8> {
        // self 类型为 &'next mut ParseIter<'parser>
        // self.parser 和 self.buf 同生命周期
        // let buf = self.buf;
        // 和下面的等价, copy角度理解
        let buf: &'parser [u8] = self.buf;

        // let parser = self.parser; // 报错不能移动可变引用
        // self 类型为 &'next mut ParseIter<'parser>
        // autoderef 得到 *self 为 ParseIter<'parser>
        // self.parser 于是为 &'parser mut Parser
        // 和下面的等价? 好像不是, 报错信息不一样, 主动写类型涉及到 reborrow?
        // let parser: &'parser mut Parser = self.parser;
        // 报错 parser 是借来的 不能超过被借的生命周期 'next
        // 改成 'next 这句不报错
        // reborrow 似乎是
        let parser: &'next mut Parser = &mut *((*self).parser);
        // cargo rustc -- --emit mir=zz.mir
        // mir 验证了猜想

        // 改成下面的正常
        // parse<'a, 'b>(&'b mut self, buf: &'a [u8]) -> Option<&'a u8> {
        return parser.parse(buf);
    }
}

fn main() {
    let mut p = Parser {};
    let v = vec![1, 2, 3];
    let mut i = p.iter(v.as_ref());
    let first = i.next();
    let second = i.next();
    println!("{first:?}, {second:?}!");
}

// #[stable(feature = "rust1", since = "1.0.0")]
// impl<'a, T: ?Sized> DerefMut for &'a mut T {
//     fn deref_mut<'dm>(&'dm mut self) -> &'a mut T {
//         // 'a : 'dm
//         // 'a < 'dm
//         // self 类型是 &'dm mut &'a mut T
//         // 不会递归吗? deref_mut 里面的*不再重复调用deref_mut?
//         *self
//         // *self 类型是 &'a mut T
//     }
// }

```
## reborrow? coercion?

```rust
use std::fmt::Debug;

#[derive(Debug)]
struct Parser {}

fn main() {
    let mut p = Parser {};
    let mut ref_p = &mut p;
    let ref2_p = &mut ref_p;
    // let p1 = *ref2_p; // error
    let p1: &mut Parser = *ref2_p; // reborrow, 相当于
    // let p1: &mut Parser = &mut (**ref2_p);

    println!("{p1:?}!");
}


// use std::fmt::Debug;

// #[derive(Debug)]
// struct Parser {}

// fn main() {
//     'a{
//         let mut p = Parser {};
//         'b{
//             let mut ref_p = &'b mut p;
//             'c{
//                 let ref2_p = &'c mut ref_p;
//                 'd{
//                     // let p1 = *ref2_p; // error
//                     // let p1: &mut Parser = ref2_p; // reborrow, 相当于
//                     let p1: &mut Parser = &'d mut (**ref2_p);

//                     println!("{p1:?}!");
//                     }
//             }
//         }
//     }
// }

let p1 : &'d mut Parser = ref2_p
ref2_p 类型是 &'c mut &'b mut Parser
T实现了 DerefMut 到 U，则有coercion， &mut T -> &mut U 
这里T是&'b mut Parser，U是Parser，则有coercion到&'c mut Parser
T是U的子类型，则有coercion，则有T -> U
这里T是&'c mut Parser，U是&'d mut Parser 
于是就用 &'d mut **ref2_p 来创建所需类型的值，赋给p1
reborrow是coercion的特例？
```

## 查看 MIR

```
cargo rustc -- --emit mir=zz.mir
```

## Kmp 算法

```rust
// Rust Program for KMP Algorithm
// Reference https://oi-wiki.org/string/kmp/

use std::vec;
use unicode_segmentation::UnicodeSegmentation;

#[derive(Debug)]
struct Kmp<'a> {
    pat: Vec<&'a str>,
    pi: Vec<usize>,
}

impl<'a> Kmp<'a> {
    fn new(pat: &'a str) -> Self {
        let pat: Vec<&str> = pat.graphemes(true).collect();
        let n = pat.len();
        let mut pi = vec![0; n];
        let mut j;
        for i in 1..n {
            j = pi[i - 1];
            while j > 0 && pat[i] != pat[j] {
                j = pi[j - 1]
            }
            if pat[i] == pat[j] {
                j += 1
            }
            pi[i] = j
        }
        Self { pat, pi }
    }

    fn search(&self, txt: &str) -> Option<usize> {
        let txt: Vec<&str> = txt.graphemes(true).collect();
        let pat = &self.pat;
        let pi = &self.pi;
        let mut state = 0;
        for (i, &char) in txt.iter().enumerate() {
            while state > 0 && char != pat[state] {
                state = pi[state - 1];
            }
            if char == pat[state] {
                state += 1;
            }
            if state == pat.len() {
                return Some(i + 1 - state);
            }
        }
        None
    }
}

fn main() {
    let kmp = Kmp::new("ababc");
    println!("{kmp:?}");
    let r = kmp.search("ababa");
    println!("{r:?}");
    let r = kmp.search("ababaababc");
    println!("{r:?}");
    // Kmp { pat: ["a", "b", "a", "b", "c"], pi: [0, 0, 1, 2, 0] }
    // None
    // Some(5)
    let kmp = Kmp::new("野蛮");
    println!("{kmp:?}");
    let r = kmp.search("非常的谔谔，非常的野蛮");
    println!("{r:?}");
    // Kmp { pat: ["野", "蛮"], pi: [0, 0] }
    // Some(9)
    let kmp = Kmp::new("🗻🌏🗻🗻🌕🗻🗻🌏🗻🌞");
    println!("{kmp:?}");
    let r = kmp.search("🗻🌏🗻🗻🌕🗻🗻🌏🗻🗻🌕🗻🗻🌏🗻🌞");
    println!("{r:?}");
    // Kmp { pat: ["🗻", "🌏", "🗻", "🗻", "🌕", "🗻", "🗻", "🌏", "🗻", "🌞"], pi: [0, 0, 1, 1, 0, 1, 1, 2, 3, 0] }
    // Some(6)
}

```

## trait 对象

```rust
fn print_hash<T: Hash>(t: &T) {
    println!("The hash is {}", t.hash())
}

// The compiled code:
__print_hash_bool(&true);  // invoke specialized bool version directly
__print_hash_i64(&12_i64);   // invoke specialized i64 version directly

fn __print_hash_bool(t: &bool) {
    println!("The hash is {}", bool::hash(t))
}



fn print_hash(t: &dyn Hash) {
    println!("The hash is {}", t.hash())
}

print_hash(&12_i64); 


let t = &12_i64 as &dyn Hash;
猜想编译器做了如下工作：
首先确定是指针指向的是 i64 转到 Hash 对象
于是TraitObject.data填充数据指针
TraitObject.vtable找impl Hash for i64的方法，做成虚表
let t = TraitObject {
    data: &12,
    vtable: &Hash_for_i64_vtable {
        drop: fn xxx,
        size: num,
        align: num,
        hash: fn i64::hash() -> xxx,
    }
} 
print_hash(t)

fn print_hash(t) {
    // 编译时不知道t的具体类型
    // t' concrete type is ???
    println!("The hash is {}", t.hash())
因为有 impl<'a>  Hash for dyn Hash + 'a {}
https://huonw.github.io/blog/2015/01/object-safety/
t.hash() 展开 {

    (t.vtable.hash)(t.data)
    &Hash_for_i64_vtable.hash(t.data)
    i64::hash(t)
}
}

https://articles.bchlr.de/traits-dynamic-dispatch-upcasting
https://huonw.github.io/blog/2015/01/object-safety/
https://blog.rust-lang.org/2015/05/11/traits.html
```

##　指针

```rust
fn main() {
    let slice: Box<[u8]> = Box::new([9, 8, 7]);
    let slice_ptr = Box::into_raw(slice);
    let ptr = slice_ptr as *const u8;
    // ptr 指向了 9.
    let b: u8 = unsafe { *ptr.add(1) };
    // 加 1 个字节 (u8 的大小).
    println!("{b}"); // b 指向了 8.
    let slice_ptr_addr = (&slice_ptr) as *const _ as *const usize;
    // 为了查看胖指针的两个字段, 构造胖指针 slice_ptr 的引用以的获得其地址.
    // 注意 as 不具有传递性, A as B as C 不能推出 A as C.
    let ptr_to_u8 = unsafe { *slice_ptr_addr } as *const u8;
    println!("{ptr_to_u8:?}");
    assert_eq!(ptr, ptr_to_u8);
    let len = unsafe { *slice_ptr_addr.add(1) };
    // 加 8 个字节 (usize 的大小), 指向第二个 field, 即 &[u8] 胖指针的 len.
    println!("{len}"); // len 为 3.

    // 从胖指针再变成 Box
    let x = unsafe { Box::from_raw(slice_ptr) };
    println!("{x:?}");

    println!(
        "{}, {}, {}, {}",
        std::mem::size_of::<Box<i8>>(),
        std::mem::size_of::<Box<&[i8]>>(),
        std::mem::size_of::<Box<[i8]>>(),
        std::mem::size_of::<Box<()>>(),
    );
}

```

## Drop

https://doc.rust-lang.org/nomicon/exception-safety.html#binaryheapsift_up

用 safe rust 的 move 语义无法拿出来，而用 `ptr::read` 可以拿出来，是按位 copy，于是被拿出来的位置成为一个洞。如果不标记为洞的话，对于二叉堆里的元素是 `Box<T>` 的情况，按位 copy 拿出来的元素析构时，由于是一个所有权类型，Drop 时会释放指向的内存。这样还在数组里的同样的 `Box<T>` 析构时就会遇到已经释放了的内存，就出错。


```rust
struct A;

fn main() {
    let v = vec![A, A, A];
    let b = v[0];
}
```

是不能 move 出来的，container 这种东西，safe 下不能出现未初始化的内存。

```rust
#[derive(Debug)]
struct A(i32);
use std::ptr::read;
fn main() {
    let v = vec![Box::new(A(1)), Box::new(A(1)), Box::new(A(1))];
    let b = unsafe { read(&v[0]) };
    drop(b);
    println!("{v:?}")
}

// [A(788512848), A(1), A(1)]
// practice(31764,0x105eb7600) malloc: *** error for object 0x600000f7c050: pointer being freed was not allocated
// practice(31764,0x105eb7600) malloc: *** set a breakpoint in malloc_error_break to debug
```

`drop(b)` 时候已经把 `v` 里的 `0` 号元素指向的内存给标记为未初始化了，后续 `v` 自动 `drop` 时就炸了。

启示：`ptr::read`一定要小心，如果是所有权类型且底层是原始指针，例如 `String`、`Vec`、`Box` 这些。要注意一方面是 alias 了，另一方面是 double drop。

### impl Drop for Vec

https://doc.rust-lang.org/nomicon/vec/vec-dealloc.html

```rust
impl<T> Drop for Vec<T> {
    fn drop(&mut self) {
        if self.cap != 0 {
            // Note that calling pop is unneeded if T: !Drop.
            while let Some(_) = self.pop() {}
            // 这一句是必要的？
            // 有必要, 试想 Vec<Box<T>>.
            // self.pop() 利用 ptr::read 创建了按位拷贝的 Box<T>, 
            // 而后被 drop, 其内部指向的地址的值被释放.
            // 此时 Vec 里虽然还仍有刚刚被拷贝的 Box<T>,
            // 但是内部的指针指向的值已经未初始化了, 再去访问就出错.
            // 如果去掉上面一句, 则 Vec<Box<T>> 直接被整个释放, 但是
            // 遗留了内部大量 Box<T> 指向的数据未被 drop, 也就是泄露.
            // 所以对于 T: Drop 而言, 上面一句是有必要的.
            let layout = Layout::array::<T>(self.cap).unwrap();
            unsafe {
                alloc::dealloc(self.ptr.as_ptr() as *mut u8, layout);
            }
        }
    }
}
```

## 系列色十六进制颜色码

```
63b2ee - 76da91 - f8cb7f - f89588 - 7cd6cf
9192ab - 7898e1 - efa666 - eddd86 - 9987ce
```

## 更新所有二进制包

https://stackoverflow.com/questions/34484361/does-cargo-install-have-an-equivalent-update-command

https://stackoverflow.com/questions/4068629/how-to-match-hyphens-with-regular-expression/4068725#4068725

https://docs.microsoft.com/zh-cn/powershell/scripting/learn/ps101/06-flow-control?view=powershell-7.2#foreach-object

### Windows

版本 5.0

```powershell
cargo install --list | Select-String -Pattern '^[a-zA-Z0-9_-]+ v[0-9.]+:$' | ForEach-Object { $PSItem.ToString().split(" ")[0] } | ForEach-Object { cargo install $_ }

# 或者

cargo install (cargo install --list | Select-String -Pattern '^[a-zA-Z0-9_-]+ v[0-9.]+:$' | ForEach-Object { $PSItem.ToString().split(" ")[0] })
```

在前面的示例中，`$_` 为当前对象。 从 PowerShell 版本 3.0 开始，可以使用 `$PSItem` 而不是 `$_`。 但我发现，大多数经验丰富的 PowerShell 用户仍更喜欢使用 `$_`，因为它向后兼容，需要输入的内容更少。

### macOS

```bash
cargo install --list | grep '^[a-zA-Z0-9_-]* v[0-9.]*:$' | cut -d ' ' -f1 | xargs cargo install
```

