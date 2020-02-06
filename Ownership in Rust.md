## Ownership in Rust

### General Behavior

Ownership in Rust enforces that every value in Rust has and only has one owner variable, and when the owner goes out of scope, the value will be freed.

This holds true for both the good old value type that lies wholly on the stack and the dynamically allocated pointer which consists of data on the heap. The value type acts just like how they are in C:

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

This behavior prohibits the *double free* problem mostly seen prior to smart pointers in C++.

### Borrow Check

In Rust, a value is borrowed if something else has a reference to it or into it (a reference to a field of a struct or to an element of a collection).

The borrow check mechanism in Rust is summarized in *the Book* as follow:

> At any given time, you can have *either* (but not both of) one mutable reference or any number of immutable references.
>
> References must always be valid.

Firstly it implies two mutable references cannot exist at the same time:

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

These restrictions are casted to prevent *data races*. More interestingly, following code won't compile as well:

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

The undelaying cause is `push` method of `vec` takes a mutable reference of it:

```rust
pub fn push(&mut self, value: T) {...}
```

Reason behind this error is that the memory of vector might get re-allocated after push operation which renders the immutable reference `first` invalid.

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

An even number eraser program might be implemented like this in C++:

```c++
{
	std::vector<int> v = { 1,2,3,4,5 };
	for (auto it = v.begin(); it != v.end(); it++) {
		if (*it % 2 == 0) {
			v.erase(it); // it is invalid after this gets executed
		}
	}
}
```

C++ compiler doesn't have complains about this, the code will run straight into exceptions. If we replicate the program in Rust:

```rust
{
    let mut v = vec![1, 2, 3, 4, 5];
    for n in v.iter_mut() {
        if *n % 2 == 0 {
            v.remove_item(n);
        }
    }
}
```

However, it won't compile:

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
5 |             v.remove_item(n);
  |             ^ second mutable borrow occurs here
```

It's a violation to the 'only one mutable reference' rule, which effectively prohibits the *iterator invalidation* problem.

### Buffer Overflow

C doesn't really have the concept of an array. Everything is pointer at runtime.

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





------



## Rust中的所有权模型

Rust的所有权，简单来说，就是在Rust中的每一份数据都为一个变量唯一持有，且当变量生命周期结束时，数据会被释放。

这意味着不论是全都在栈上的值类型还是

（中文翻译太难了，中文版推迟）