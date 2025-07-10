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