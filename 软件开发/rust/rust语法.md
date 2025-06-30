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

