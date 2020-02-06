## Ownership in Rust

### General Behavior

Ownership in Rust enforces that every value in Rust has and only has one owner variable, and when the owner goes out of scope, the value will be freed.

This holds true for both the good old value type that lies wholly on the stack and the dynamically allocated pointer type which consists of data on the heap. The value type acts just like how they are in C:

```rust
{
  let i = 1;
} // i is no longer invalid
```

The structs that contain pointers follow the RAII pattern in C++ where the destructor is called when out of scope:

```rust
{
  let s = String::from("hello");
} // drop(s) will be called and the "hello" on the heap will be deallocated
```

What ownership rules in Rust further imply is the move semantic. Variable binding in Rust defaults to move operation instead of copy operation:

```rust
let s1 = String::from("hello");
let s2 = s1; // s1 is invalidated post this point. the ownership is transferred
```

This behavior prohibits the *double free* problem mostly seen in C/C++ (prior to smart pointers in C++).

### Borrow Check

In Rust, a value is borrowed if something else has a reference to it or into it (a reference to a field of a struct or an element of a collection).

The borrow check mechanism in Rust is summarized in *the Book* as follow:

> At any given time, you can have *either* (but not both of) one mutable reference or any number of immutable references.
>
> References must always be valid.

Firstly it implies two or more mutable references cannot exist at the same time:

```rust
{
    let mut x = 5;
    let r1 = &mut x;
    let r2 = &mut x;
    println!("{}", r1);
}
```

```
error[E0499]: cannot borrow `x` as mutable more than once at a time
 --> src\main.rs:4:14
  |
3 |     let r1 = &mut x;
  |              ------ first mutable borrow occurs here
4 |     let r2 = &mut x;
  |              ^^^^^^ second mutable borrow occurs here
5 |     println!("{}", r1);
  |                    -- first borrow later used here
```

Secondly, mutable reference and immutable references cannot exist together:

```rust
{
    let mut x = 5;
    let r1 = &mut x;
    let r2 = &x;
    println!("{}", r1);
}
```

```
error[E0502]: cannot borrow `x` as immutable because it is also borrowed as mutable
 --> src\main.rs:4:14
  |
3 |     let r1 = &mut x;
  |              ------ mutable borrow occurs here
4 |     let r2 = &x;
  |              ^^ immutable borrow occurs here
5 |     println!("{}", r1);
  |                    -- mutable borrow later used here
```

One of the reasons these restrictions are cast is to prevent *data races*. More interestingly, the following code won't compile as well:

```rust
{
    let mut v = vec![1, 2, 3, 4, 5];
    let first = &v[0];
    v.push(6);
    println!("{}", first);
}
```

```
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
 --> src\main.rs:4:5
  |
3 |     let first = &v[0];
  |                  - immutable borrow occurs here
4 |     v.push(6);
  |     ^^^^^^^^^ mutable borrow occurs here
5 |     println!("{}", first);
  |                    ----- immutable borrow later used here
```

The undelaying cause is the method `push` of `vec` takes a mutable reference of it:

```rust
pub fn push(&mut self, value: T) {...}
```

The reason behind this error is that the memory of vector might get re-allocated after push operation which renders the immutable reference `first` invalid.

## Memory Safety in Rust

Due to the freedom of pointers in C/C++, memory safety has been a problem for the longest time. The ownership design and compile-time check of the Rustc are the saving grace for the memory safety issues.

### Uninitialized Variables

Uninitialized variables are not allowed in Rust.

```rust
{
    let x: i32;
    println!("{}", x);
}
```

Code above wouldn't compile:

```
error[E0381]: borrow of possibly uninitialized variable: `x`
 --> src\main.rs:3:20
  |
3 |     println!("{}", x);
  |                    ^ use of possibly uninitialized `x`
```

### Memory Leak

The malloc/free and new/delete are supposed to be always used in pairs. When failed to do so, it'll rise memory leak issues.

Memory leak issues are not hard to find. For example, it exists in the source code of the OpenArkCompiler:

```c++
ErrorCode MapleCombCompiler::Compile(const MplOptions& options, MIRModulePtr& theModule) {
	MemPool* optMp = memPoolCtrler.NewMemPool("maplecomb mempool");
	std::string fileName = GetInputFileName(options);
	theModule = new MIRModule(fileName); // theModule never gets deallocated
	std::unique_ptr<MeOption> meOptions;
	std::unique_ptr<Options> mpl2mplOptions;
	auto it = std::find(options.GetRunningExes().begin(), options.GetRunningExes().end(), kBinNameMe);
	if (it != options.GetRunningExes().end()) {
		meOptions.reset(MakeMeOptions(options, *optMp));
	}
	auto iterMpl2Mpl = std::find(options.GetRunningExes().begin(), options.GetRunningExes().end(), kBinNameMpl2mpl);
	if (iterMpl2Mpl != options.GetRunningExes().end()) {
		mpl2mplOptions.reset(MakeMpl2MplOptions(options, *optMp));
	}

	LogInfo::MapleLogger() << "Starting mpl2mpl&mplme\n";
	PrintCommand(options);
	DriverRunner runner(theModule, options.GetRunningExes(), mpl2mplOptions.get(), fileName, meOptions.get(),
		fileName, fileName, optMp,
		options.HasSetTimePhases(), options.HasSetGenVtableImpl(), options.HasSetGenMeMpl());
	ErrorCode nErr = runner.Run();

	memPoolCtrler.DeleteMemPool(optMp);
	return nErr;
}
```

It won't be a problem in Rust because the new/delete pair is implicitly bundled together.

### Use After Free

C/C++ never forbid the *use after free* paradigm:

```c++
{
    auto word = new std::string("hello world");
    delete word;
    std::cout << *word << std::endl;
}
```

*Use after free*, *dereference dangling pointer* are undefined behaviors. They may cause severe vulnerabilities. Rustc will catch them at compile time:

```rust
{
  let word = String::from("hello world");
  drop(word);
  println!("{}", word);
}
```

```
error[E0382]: borrow of moved value: `word`
 --> src\main.rs:4:20
  |
2 |     let word = String::from("hello world");
  |         ---- move occurs because `word` has type `std::string::String`, which does not implement the `Copy` trait
3 |     drop(word);
  |          ---- value moved here
4 |     println!("{}", word);
  |                    ^^^^ value borrowed here after move
```

Modern C/C++ compilers(GCC and Clang) provide lifetime check compiling options, but they are not water-proof. For example:

```c++
std::string_view first_word(std::string_view s) {
    return s.substr(0, s.find_first_of(' '));
}

int main() {
    std::string s = "hello world";
    auto word = first_word(s);
    s.clear();
    std::cout << word << std::endl; // what word points to is no longer valid
    return 0;
}
```

Code above would compile with no warnings or errors, even if it is an *use after free* scenario. Now if the same code is written in Rust:

```rust
fn first_word(s: &str) -> &str {
    let bytes = s.as_bytes();
    for (i, &item) in bytes.iter().enumerate() {
        if item == b' ' {
            return &s[0..i];
        }
    }
    &s[..]
}

fn main() {
    let mut s = String::from("hello world");
    let word = first_word(&s);
    s.clear();
    println!("{}", word);
}
```

Thanks for the borrow check in Rust, the compiler won't allow it:

```
error[E0502]: cannot borrow `s` as mutable because it is also borrowed as immutable
  --> src\main.rs:14:5
   |
13 |     let word = first_word(&s);
   |                           -- immutable borrow occurs here
14 |     s.clear();
   |     ^^^^^^^^^ mutable borrow occurs here
15 |     println!("{}", word);
   |                    ---- immutable borrow later used here
```

### Iterator Invalidation

*Iterator invalidation* might present like this in C++:

```c++
{
	std::vector<int> v = { 1,2,3,4,5 };
	for (auto it = v.begin(); it != v.end(); it++) {
		if (*it % 2 == 0) {
			v.push_back(*it); // 'it' is invalid after reallocation
		}
	}
}
```

Vector v is initialized to have a capacity of 5 and when inserted, the reallocation gets triggered which invalids the iterators. C++ compiler doesn't have complains about this, it will run straight into exceptions. If we replicate the program in Rust:

```rust
{
    let mut v = vec![1, 2, 3, 4, 5];
    for n in v.iter_mut() {
        if *n % 2 == 0 {
            v.push(*n);
        }
    }
}
```

However, the snippet won't compile:

```
error[E0499]: cannot borrow `v` as mutable more than once at a time
 --> src\main.rs:5:13
  |
3 |     for n in v.iter_mut() {
  |              ------------
  |              |
  |              first mutable borrow occurs here
  |              first borrow later used here
4 |         if *n % 2 == 0 {
5 |             v.push(*n);
  |             ^ second mutable borrow occurs here
```

It's a violation of the 'only one mutable reference' rule, which effectively prohibits the *iterator invalidation* problem.

### Buffer Overflow

C doesn't really have the concept of an array. Everything is treated as pointers at runtime.

```C
struct foo {
  int buf[4];
  int x;
};
struct foo *pf = malloc(sizeof(struct foo));
pf->buf[5] = 3; // x is now magically 3!
```

You tell C to write to `pf->buf[5]` and it will happily do so. Buffer overflow problems are alleviated in C++, to a degree:

```c++
{
	std::vector v = { 1,2,3,4,5 };
	try {
		std::cout << v.at(5);
	}
	catch (...) {
		std::cout << "buffer overflow caught";
	}
	return 0;
}
```

Though for `span` in C++, `at` is not provided, it often results in the necessity of a wrapper. Bound check is more streamlined in Rust:

```rust
{
    let v = vec![1, 2, 3, 4, 5];
    if let Some(num) = v.get(5) {
        println!("{}", num);
    } else {
        println!("out of bound");
    }
}
```

`vec.get` always returns `Option<&T>` which contains a null value when out of bound.

### Data Race

*Data races* happen when two or more threads concurrently access the same memory with at least one of them being a write operation. *Data races* may cause memory corruptions, which C++ doesn't particularly forbid:

```c++
{
	int count = 0;
	std::thread thread1([&count]() {count++;});
	std::thread thread2([&count]() {std::cout << count << std::endl; });
	thread1.join();
	thread2.join();
}
```

In this super simplified example, it is possible for `count` to be written and read at the same time. For the record, it is not hard to fix the *data race* issue in C++. Simply wraps shared data with atomic:

```c++
{
	std::atomic<int> count = 0;
	std::thread thread1([&count]() {count++;});
	std::thread thread2([&count]() {std::cout << count << std::endl; });
	thread1.join();
	thread2.join();
}
```

Still, we can take a look at the same scenario in Rust:

```rust
{
    let mut count = 0;
    let thread1 = thread::spawn(|| {
        count = count + 1;
    });
    let thread2 = thread::spawn(|| {
        println!("{}", count);
    });
    thread1.join().unwrap();
    thread2.join().unwrap();
}
```

Remember earlier I said borrow check can prevent *data races*? Turns out it's true:

```
error[E0502]: cannot borrow `count` as immutable because it is also borrowed as mutable
 --> src\main.rs:6:33
  |
3 |       let thread1 = thread::spawn(|| {
  |                     -             -- mutable borrow occurs here
  |  ___________________|
  | |
4 | |         count = count + 1;
  | |                 ----- first borrow occurs due to use of `count` in closure
5 | |     });
  | |______- argument requires that `count` is borrowed for `'static`
6 |       let thread2 = thread::spawn(|| {
  |                                   ^^ immutable borrow occurs here
7 |           println!("{}", count);
  |                          ----- second borrow occurs due to use of `count` in closure
```

The closure of `thread1` borrows a mutable reference of `count` because it alters the value. Second closure, that of `thread2`, automatically borrows an immutable reference of `count` because it reads the value, which essentially violates the borrow check rules by having mutable references and immutable references at the same time and thus is rejected by Rustc.

Although *data races* are not hard to solve, I'd wage life in Rust is easier for letting compiler take care of it for you. One less potential for mistake is most of the time one less mistake for real.



------



## Rust中的所有权

### 一般行为

Rust的所有权，就是在Rust中的每一份数据都为一个变量唯一持有，且当变量生命周期结束时，数据会被释放。

这对全在栈上的值类型和包含堆数据的指针类型来说都是成立的。值类型的行为就和C语言一样：

```rust
{
  let i = 1;
} // i在这出栈
```

包含指针的结构体遵循C++中的“资源获取就是初始化”（RAII, Resource Acquisition Is Initialization）模式，在变量生命周期结束时会析构：

```rust
{
  let s = String::from("hello");
} // drop(s)被调用，堆上的"hello"被释放
```

Rust的所有权规则还指明了Rust的移动语义。赋值语句在Rust中默认为移动操作而非拷贝操作：

```rust
let s1 = String::from("hello");
let s2 = s1; // s1在赋值语句执行后失效，所有权转移给s2
```

这种行为防止了在C/C++（在C++引入智能指针之前）中多见的”多次释放“的内存安全问题。

### 借用检查

在Rust中，如果一个变量或变量的一部分（比如结构体的一个字段、集合的一个元素）被引用了，那么就称其被借用了。

在Rust的官方介绍书籍*The Rust Programming Language*中，借用检查机制总结如下：

> 对于一个变量，在任一时刻，只可拥有一个可变引用或多个不可变引用，且两者不可共存。
>
> 所有引用都必须保持有效。

首先,这意味着两个以上的可变引用不可以同时存在：

```rust
{
    let mut x = 5;
    let r1 = &mut x;
    let r2 = &mut x;
    println!("{}", r1);
}
```

```
error[E0499]: cannot borrow `x` as mutable more than once at a time
 --> src\main.rs:4:14
  |
3 |     let r1 = &mut x;
  |              ------ first mutable borrow occurs here
4 |     let r2 = &mut x;
  |              ^^^^^^ second mutable borrow occurs here
5 |     println!("{}", r1);
  |                    -- first borrow later used here
```

其次，可变引用和不可变引用也不可以同时存在：

```rust
{
    let mut x = 5;
    let r1 = &mut x;
    let r2 = &x;
    println!("{}", r1);
}
```

```
error[E0502]: cannot borrow `x` as immutable because it is also borrowed as mutable
 --> src\main.rs:4:14
  |
3 |     let r1 = &mut x;
  |              ------ mutable borrow occurs here
4 |     let r2 = &x;
  |              ^^ immutable borrow occurs here
5 |     println!("{}", r1);
  |                    -- mutable borrow later used here
```

这么规定的原因之一是为了防止”数据争用“问题。而且有意思的是，下面这样的代码也不会通过编译：

```rust
{
    let mut v = vec![1, 2, 3, 4, 5];
    let first = &v[0];
    v.push(6);
    println!("{}", first);
}
```

```
error[E0502]: cannot borrow `v` as mutable because it is also borrowed as immutable
 --> src\main.rs:4:5
  |
3 |     let first = &v[0];
  |                  - immutable borrow occurs here
4 |     v.push(6);
  |     ^^^^^^^^^ mutable borrow occurs here
5 |     println!("{}", first);
  |                    ----- immutable borrow later used here
```

这是因为`vec`的`push`方法借用了一个它本身的可变引用：

```rust
pub fn push(&mut self, value: T) {...}
```

矢量的内存在增加元素后可能重新分配，从而导致不可变引用`first`失效。这一编译错误背后的道理，就是为了防止这种情况。