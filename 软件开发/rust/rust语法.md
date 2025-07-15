## rust语法

### 1. panic
- panic是rust中的一种错误处理机制，当程序发生panic时，rust会打印panic信息，并退出程序
- panic信息中包含panic的位置、panic的原因、panic的上下文
- panic可以手动触发，也可以自动触发
- 手动触发panic的方式：`panic!("panic message")`
- 自动触发panic的方式：
  - 访问空指针
  - 数组越界
  - 栈溢出
  - 除以0
  - 等等

### 2. 循环
- rust中的循环有两种：`for`循环和`while`循环
- `for`循环的语法：
  ```rust
  for item in collection {
      // do something with item
  }
  ```
- `while`循环的语法：
  ```rust
  while condition {
      // do something
  }
  ```

### 3. 溢出（overflow）和饱和（saturating）
- 溢出：当进行数值运算时，如果结果超过了数值类型所能表示的范围，就会发生溢出。在rust中，溢出默认是不允许的，会导致panic。如果关闭溢出检查，进行运算时，会wrapping around。（即对数值类型的范围进行取模运算）
- wrapping：wrapping是rust中的一种溢出处理机制，当发生溢出时，rust会将结果对数值类型的范围进行取模运算，得到一个在数值类型范围内的结果。这个方法的行为和关闭溢出检查时的行为是一致的。

    ```rust
    // 示例
    let a = i8::MAX;
    let b = a + 1;
    println!("{}", b); // 溢出，panic
    let a = i8::MAX;
    let b = a.wrapping_add(1);
    println!("{}", b); // -128
    ```

- saturating：saturating是rust中的一种溢出处理机制，当发生溢出时，rust会将结果限制在数值类型的范围内，不会panic。

    ```rust
    // 示例
    let a = i8::MAX;
    let b = a.saturating_add(1);
    println!("{}", b); // 127
    ```

### 4. traits —— rust中的接口
- trait是rust中的一种概念，它定义了一种行为。trait可以被用于定义类型的行为，也可以被用于定义函数的行为。
- trait的定义：
  ```rust
  trait TraitName {
      // 定义方法
      fn method_name(&self) -> return_type;
  }
  ```
- trait的实现：
  ```rust
  impl TraitName for TypeName {
      // 实现方法
      fn method_name(&self) -> return_type {
        // do something
      }
  }
  ```

#### 4.1 操作符重载
- PartialEq接口 用于重载==和!=操作符
- Add接口 用于重载+操作符
- Sub接口 用于重载-操作符
- Mul接口 用于重载*操作符
- Div接口 用于重载/操作符
- Rem接口 用于重载%操作符
- Index接口 用于重载[]操作符

#### 4.2 derive
- derive是rust中的一个宏，它可以自动为结构体派生实现trait。
- 示例：
  ```rust
  #[derive(Debug)]
  struct Person {
      name: String,
      age: u8,
  }
  ```

#### 4.3 接口边界(trait bounds)
- trait bounds是rust中的一个概念，它定义了一个泛型类型的上边界。
- 示例：
  ```rust
  fn print<T: Debug>(t: &T) {
      println!("{:?}", t);
  }
  ```
- 这个示例中，`T`是一个泛型类型参数，`Debug`是一个trait。`T: Debug`表示`T`必须为实现`Debug`trait的类型。

#### 4.4 Deref trait
- deref接口是rust中的一个trait，它定义了一个类型的解引用行为，可以将一个类型的引用解引用为另一个类型的引用。如String类型就实现了deref接口，将String类型的引用解引用为str类型的引用。
- deref接口的定义：
  ```rust
  trait Deref {
      type Target;
      
      fn deref(&self) -> &Self::Target;
  }
  ```
- deref接口的实现：
  ```rust
  impl Deref for String {
      type Target = str;
      
      fn deref(&self) -> &str {
          // [...]
      }
  }
  ```

#### 4.5 Sized trait
- Sized trait是rust中的一个trait，它表示一个类型在编译时能够确定其占用的内存大小。
- Sized trait在rust中是一种marker（标记），它本身是空trait，没有任何方法，仅用于编译器在编译时的一些判断和优化。
- 未实现Sized的类型被称为DST（dynamic sized type, 动态大小类型），如str是一个DST类型。
- 未实现Sized的类型不能用于函数参数的传递，必须使用其引用类型。

#### 4.6 From和Into trait
- From和Into是一对双生的trait，它们用于表示某个类型可以从另一个指定类型转换而来。实现From trait会同时自动实现Into trait。

#### 4.7 关联类型(Associated Types)和泛型(Generics)
- 当一个trait实现需要唯一确定的类型时，使用关联类型
- 当你想要允许同一个类型对同一个trait有多个不同输入类型的实现时，使用泛型参数

#### 4.8 Clone trait
- 用于对一个对象进行深拷贝，会在内存中单独开辟一块空间用于存储拷贝后的对象。
- 示例：
  ```rust
  struct Person {
      name: String,
  }
  impl Clone for Person {
      fn clone(&self) -> Self {
          Self {
              name: self.name.clone(),
          }
      }
  }
  ```

- 当一个对象作为函数参数传递时，其所有权会交给函数，函数调用后再无法使用该对象。这时我们就可以使用clone方法对原对象进行拷贝，然后将拷贝后的对象作为参数传递给函数。

#### 4.9 Copy trait
- Copy trait是Clone trait的子trait，它可以隐式地拷贝对象，使得对象作为函数参数等传递时，并不会直接交出所有权，而是使用拷贝的新对象进行操作。
- 实现Copy trait时一定要先实现Clone trait，可以用#[derive(Clone, Copy)]的方式自动实现。
- 注意并不是所有的类型都可以实现Copy trait，需要至少满足以下条件：
  - 首先，由于Copy是Clone的子trait，所以必须实现Clone trait。这是很合理的：如果Rust可以隐式地创建一个类型的新实例，那么它也应该能够通过调用.clone()显式地创建新实例。
  - 该类型不能在其在内存中占用的std::mem::size_of字节之外管理任何额外的资源（例如堆内存、文件句柄等）
  - 该类型不能是可变引用（&mut T）。
- String类型不能实现Copy，因为它管理了额外的资源（字符串在堆内存中的实际数据）；u32等类型实现了Copy接口，因为它们本身只有一个简单的整数值，没有任何额外的资源；&mut u32等类型不能实现Copy，这是因为如果&mut u32实现了Copy，你就可以创建多个指向同一个值的可变引用，并在多个地方同时修改它——这将违反Rust的借用规则！因此，无论T是什么类型，&mut T都不会实现Copy。

#### 4.10 Drop trait
- Drop trait用于清理某个对象额外管理的资源。当对象离开其作用域时，Rust会自动调用它的drop方法。
- 由于实现Drop trait的类型一定持有了额外的资源，因此实现Drop trait的类型不能实现Copy trait。

#### 4.11 使用trait的一些原则
- 如果一个函数总是使用同一种类型，就不要把它写成泛型函数。泛型会在代码库中引入间接性，使代码更难理解和维护。
- 如果一个trait只有一个实现，就不要创建这个trait。这表明这个trait是不必要的。
- 在合适的情况下，为你的类型实现标准trait（如Debug、PartialEq等）。这会使你的类型更符合rust惯用法，更易于使用，并能解锁标准库和生态系统crate提供的大量功能。
- 如果你需要第三方crate生态系统中的功能，就为你的类型实现这些crate中的trait。
- 要谨慎使用泛型仅仅是为了在测试中使用mock。这种方法的维护成本可能很高，通常使用其他测试策略会更好。

### 5. 枚举
#### 5.1 enum
- 枚举是一种可以将多个相关值组合到一起的类型。

  ```rust
  /// 枚举定义
  enum IpAddrKind {
      V4,
      V6,
  }
  /// 枚举的使用
  let four = IpAddrKind::V4;
  let six = IpAddrKind::V6;
  ```
- 与Java等语言所不同的是，rust中的枚举值可以包含数据，如：
  ```rust
   /// 枚举定义
  enum IpAddr {
      V4(u8, u8, u8, u8),
      V6(String),
  }
  /// 枚举的使用
  let home = IpAddr::V4(127, 0, 0, 1);
  let loopback = IpAddr::V6(String::from("::1"));
  ```

#### 5.2 match
- match表达式可以用于枚举类型，用于匹配不同的枚举值。
- match表达式的语法：
  ```rust
  enum IpAddr {
      V4,
      V6,
  }
  /// 枚举的使用
  let four = IpAddr::V4;
  let six = IpAddr::V6;
  match four {
      IpAddr::V4 => println!("ipv4"),
      IpAddr::V6 => println!("ipv6"),
  }
  ```
- 注意在rust中，match必须匹配所有的枚举值，否则会编译错误。

#### 5.3 if let和let else
- if let和let else用于只匹配一个枚举值，而match表达式需要匹配所有的枚举值。
- if let和let else的语法：
  ```rust
  enum IpAddr {
      V4,
      V6,
  }
  /// 枚举的使用
  let four = IpAddr::V4;
  let six = IpAddr::V6;
  if let IpAddr::V4 = four {
      println!("ipv4");
  } else {
      println!("ipv6");
  }
  ```
- 如果else的分支是需要尽早返回的错误场景，可以用let else语法：

  ```rust
      let IpAddr::V4 = four else {
        panic!("ipv6");
      };
  ```

### 6. 空值处理(nullability)和错误处理(Fallibility)

#### 6.1 Option<T>
- Option<T>是rust中用于处理空值的枚举类型。
- Option<T>是一个枚举，它有两个值：Some(T)和None。
- Some(T)表示有一个值，None表示没有值。
- Option<T>的使用场景：
  - 当一个值可能不存在时，使用Option<T>来表示。
  - 当一个函数可能返回错误时，使用Option<T>来表示。
- Option<T>的示例：
  ```rust
  let some_number = Some(5);
  let some_string = Some("a string");
  let absent_number: Option<i32> = None;
  ```
- Option<T>的方法：
  - unwrap：用于获取Some(T)中的值，如果是None，则会panic。
  - expect：与unwrap类似，但是可以自定义panic信息。
  - is_some和is_none：用于判断是否是Some或None。
  - map：用于对Some(T)中的值进行映射。
  - and_then：用于对Some(T)中的值进行映射，如果是None，则返回None。
  - or：用于返回Option<T>中的值，如果是Some(T)，则返回Some(T)，如果是None，则返回另一个Option<T>。
  - or_else：与or类似，但是可以自定义返回值。

#### 6.2 Result<T>
- Result<T>是rust中用于处理错误的枚举类型。
- Result<T>是一个枚举，它有两个值：Ok(T)和Err(E)。
- Ok(T)表示操作成功，Err(E)表示操作失败。
- Result<T>的使用场景：
  - 当一个函数可能返回错误时，使用Result<T>来表示。
- Result<T>的示例：
  ```rust
  let two = string::parse::<i32>("2");
  let two = two.unwrap();
  ```
- Result<T>的方法：
  - unwrap：用于获取Ok(T)中的值，如果是Err(E)，则会panic。
  - expect：与unwrap类似，但是可以自定义panic信息。
  - is_ok和is_err：用于判断是否是Ok或Err。
  - map：用于对Ok(T)中的值进行映射。
  - map_err：用于对Err(E)中的值进行映射。

#### 6.3 错误枚举
- 错误枚举是一种自定义错误类型，用于表示函数可能返回的错误。
- 错误枚举的定义：
  ```rust
  enum ErrorKind {
      InvalidInput,
      OutOfMemory,
      UnexpectedError,
  }
  ```
- 错误枚举的使用：
  ```rust
  fn some_function() -> Result<(), ErrorKind> {
      Err(ErrorKind::InvalidInput)
  }
  ```
- 错误枚举的匹配：
  ```rust
  match some_function() {
      Ok(_) => println!("ok"),
      Err(ErrorKind::InvalidInput) => println!("invalid input"),
      Err(ErrorKind::OutOfMemory) => println!("out of memory"),
      Err(ErrorKind::UnexpectedError) => println!("unexpected error"),
  }
  ```

#### 6.4 错误接口(Error trait)
- Error trait是Rust错误处理的基石，其定义如下：
  ```rust
  pub trait Error: Debug + Display {}
  ```
- 让每个Rust开发者都自己设计错误报告策略是不切实际的：这会浪费时间，而且不同项目之间的错误处理方式也难以协调。这就是为什么Rust提供了std::error::Error trait。

#### 6.5 thiserror——第三方的错误接口
- 我们已经看到了如何为自定义错误类型"手动"实现Error trait。
想象一下，如果你必须为代码库中的大多数错误类型都这样做。这会产生大量的样板代码，不是吗？

- 我们可以通过使用thiserror来减少这些样板代码，这是一个Rust crate，它提供了一个过程宏来简化自定义错误类型的创建。

``` rust
#[derive(thiserror::Error, Debug)]
enum TicketNewError {
    #[error("{0}")]
    TitleError(String),
    #[error("{0}")]
    DescriptionError(String),
}
```

#### 6.6 TryFrom和TryInto trait
- 前面我们提到了From和Into接口，它们是一对伴生接口，用于转换类型，但是它们是不可以失败的（如果失败，程序会panic）。
- TryFrom和TryInto trait是一对用于转换类型的trait，它们可以失败，这是通过返回Result<T, E>来实现的。
- TryFrom trait定义如下：
  ```rust
  pub trait TryFrom<T>: Sized {
      type Error;
      fn try_from(value: T) -> Result<Self, Self::Error>;
  }
  ```
- TryInto trait定义如下：
  ```rust
  pub trait TryInto<T>: Sized {
      type Error;
      fn try_into(self) -> Result<T, Self::Error>;
  }
  ```
- TryFrom和TryInto trait的使用场景：
  - 当你需要将一个类型转换为另一个类型时，但是转换可能失败，并且你希望在失败时返回一个自定义的错误类型。
