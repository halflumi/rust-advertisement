Memory safety has been a problem for C/C++ the longest time. The ownership design of Rust and the compile-time check it enforces are the saving grace for the memory safety issues.

In this summary, we'll discuss:

- Resource management in C/C++ and Rust 
- How Rust solves the memory safety issues that are often seen in C/C++.

## Resource Management

To illustrate how Rust improves in terms of resource management, let's first take a look at how C/C++ lays down the foundation in that regard.

### The C Primitives

We only have scope-based stack management and we need to keep an eye for everything on the heap in the vanilla C. Once we allocate something on the heap, we have to manually free it later, which leads to classic memory safety issues like *memory leak*, *use after free* and *double free*. For example, when you open a file, you must call `close` at some point:

```C
{
    File file("...");
    // ...
    file.close();
}
```

Apart from those safety issues that originate from human errors, logic errors like failing to free the allocated resources when exceptions arise can lead to memory unsafety as well.

### The RAII Pattern

The RAII pattern introduced in C++ solves a large portion of the aforementioned memory safety issues in C. Destructors of structs will be called when out of scope:

```c++
{
    File file("...");
    // ...
} // destructor of file called, which essentially calls file.close()
```

So now the allocated resource is recycled where it is not before, whether it is due to free forgot to be called or failed to be called.

### The Smart Pointers

The RAII pattern doesn't solve all the problems and can be trivial to apply when dealing with raw pointers. Thus smart pointers are introduced in C++11. They are related to the RAII concept, providing a more explicit and accurate expression of data *ownership*.

Take a snippet in the source code of the OpenArk Compiler for example:

```c++
MapleVector<MIRSymbol*> symbolTable;

MIRSymbol *CreateSymbol(uint8 scopeID) {
    MIRSymbol *st = mAllocator->GetMemPool()->New<MIRSymbol>(symbolTable.size(), scopeID);
    symbolTable.push_back(st);
    return st;
}
```

In this case, a new `MIRSymbol` is allocated. A pointer of the data is stored in the container `symbolTable` and also passed to the return value of the function, which leads to a scenario with more than one reference to the data, a scenario that is open to memory safety issues.

From the snippet, there are a few possibilities of how the ownership of  the allocated `MIRSymbol` operates. Maybe the `symbolTable` and `st` shares the ownership of the data. Perhaps the `symbolTable` owns the data and `st` only has a reference to it or the other way around. It is unclear how they operate without more info from the function callers. If the code were to be written with smart pointers:

```C++
MapleVector<std::unique_ptr<MIRSymbol>> symbolTable;

MIRSymbol *CreateSymbol(uint8 scopeID) {
    auto st = std::make_unique<MIRSymbol>(mAllocator->GetMemPool()->New<MIRSymbol>(symbolTable.size(), scopeID));
    symbolTable.push_back(st);
    return &*st;
}
```

Now we can be sure that `symbolTable` owns the allocated `MIRSymbol` and it manages the lifetime of `st`.

As a side note, this might not comply with the logic of original intention since the source code is not fully published yet. However, `symbolTable` is not freed in the destructor of the parent class in the original code, which leads to a memory leak if the unique pointer mimic does comply.

### Resource Management in Rust

In terms of resource management, C++ is already a big step up from C, so the question for Rust is how much more it brings on the table compared to C++. 

For the record, the RAII pattern is also adopted in Rust and it has smart pointers (which are arguably better than those in C++). Though more importantly, Rust brings the concept of *ownership* to a whole new level.

In Rust, a value is borrowed if something else has a reference to it or into it (a reference to a field of a struct or an element of a collection). 

The borrow check mechanism is summarized in *the Book* as follow:

> At any given time, you can have *either* (but not both of) one mutable reference or any number of immutable references.
>
> References must always be valid.

Here is a direct reflection of it in action:

```rust
{
    let mut x = 5;
    let r1 = &mut x; // first mutable borrow of x
    let r2 = &mut x; // second mutable borrow of x is not allowed
    let mut y = 5;
    let r3 = &mut y; // mutable borrow of y
    let r4 = &y; // immutable borrow of y is not allowed when there is mutable borrow
}
```

Rustc will enforce these checks at compile-time, which ultimately is what memory safety in Rust built upon, as we'll demonstrate in the next section of the summary.

## Memory Safety in Rust

In this section, we'll demonstrate in detail how Rust prevents the memory safety issues that are common in C/C++.

### Use After Free

C/C++ never forbid the *use after free* paradigm:

```c++
{
    auto word = new std::string("hello world");
    delete word;
    std::cout << *word << std::endl;
}
```

*Use after free*, *dereference dangling pointer* are undefined behaviors and they may cause severe vulnerabilities. Rustc will catch them at compile time:

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

Modern C/C++ compilers provide arguments for lifetime checks, but they are not water-proof. For example:

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

The code would compile with no warnings or errors, even if it is an *use after free* scenario. Now if the same code is written in Rust:

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

It will fail the borrow check:

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

*Iterator invalidation* presents in C++:

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

Vector `v` is initialized to have a capacity of 5 and when inserted, the reallocation gets triggered which invalids the iterators. C++ compiler doesn't have complains about this, it will run straight into exceptions. If we replicate the program in Rust:

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

The undelaying cause is the method `push` of `vec` takes a mutable reference of itself:

```rust
pub fn push(&mut self, value: T) {...}
```

It's a violation of the 'only one mutable reference' rule, which effectively prohibits the *iterator invalidation* problem. The reason behind this error is that the memory of vector might get re-allocated after push operation which renders the immutable reference `first` invalid.

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

You tell C to write to `pf->buf[5]` and it will happily do so. C++ deals with buffer overflow issues better than C does:

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

### Data Races

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

Of course it won't compile:

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

## Conclusion

Memory safety is one of Rust's biggest selling points. Not only does it have what C++ has to offer, but it also has:

- Borrow check that prevents various kinds of access errors like *use after free* and *iteration invalidation*
- More streamlined buffer overflow checks
- Guarantees against all data races

We can conclude that Rust is indeed an improvement to C/C++ as far as memory safety concerns.

