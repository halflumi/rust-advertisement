## Concurrency in Rust

This summary mainly focuses on the differences in concurrent programming between Rust and C++.

The differences will be demonstrated with a focus on the improvements Rust provides in terms of Concurrency facilities compared to C++.

It is also worth noting that safe Rust is what's assumed by 'Rust' in the context of this summary unless otherwise stated.

### Data Races

An accidentally left out `atomic` declaration leads to undefined behaviors in C++:

```c++
{
	int count = 0; // count should be wrapped in a std::atomic
	std::thread thread1([&count]() {count++;});
	std::thread thread2([&count]() {std::cout << count << std::endl; });
	thread1.join();
	thread2.join();
}
```

 As we've already discussed in the summary about ownership, Rust offers compile-time guarantees against all data races. The same code in Rust won't pass the borrow checks of the Rustc:

```rust
{
    let mut count = 0;
    let thread1 = thread::spawn(|| {
        count = count + 1;
    });
    let thread2 = thread::spawn(|| { // immutable borrow fails at thread2
        println!("{}", count);
    });
    thread1.join().unwrap();
    thread2.join().unwrap();
}
```

Though the difference is not dramatic, it is a decent quality of life improvement over C++ since static checks for atomicity aren't offered in C++.

### Mutexes

In C++, a mutex doesn't *own* the data the program tries to access when locking it. Just like the case in the data races, this exposes unwanted vulnerabilities when the data is accessed with the corresponding mutex forgotten to be locked:

```c++
std::mutex mutex;  // protects count

void counter(int& count) {
    // lock can be potentially left out
    const std::lock_guard<std::mutex> lock(mutex);
    count++;
}

int main() {
    int count = 0;
    std::thread t1(counter, std::ref(count));
    std::thread t2(counter, std::ref(count));
	t1.join();
    t2.join();
}
```

However, this isn't the end of the world in C++ since we can always write a wrapper for the mutex to bind the data to the mutex, which is exactly how mutexes in Rust work (`Arc` is used in the Rust example because threads in Rust are static by default):

```rust
fn counter(count: &Arc<Mutex<i32>>) {
    let mut num = count.lock().unwrap(); // can't access the data without the lock
    *num += 1;
}

fn main() {
    let count = Arc::new(Mutex::new(0));
    let mut threads = vec![];

    for _ in 0..2 {
        let count = Arc::clone(&count);
        let thread = thread::spawn(move || counter(&count));
        threads.push(thread);
    }

    for handle in threads {
        handle.join().unwrap();
    }
}
```

In Rust, a mutex *owns* the data it protects. Instead of first locking then accessing the shared memory, the data is not available if you don't hold the lock. As a matter of fact, `count.lock().unwrap()` returns a `MutexGuard` in this case, which is a RAII lock guard similar to that in C++. 

This may seem subtle, but the kind of enforcement Rust has on mutexes makes it more streamlined to use compared to the mutexes in C++.

### Thread-safety Tracking

Rust's type system employs some notions to track thread safety, more specifically, the marker traits `Send` and `Sync`. The `Send` trait marks the type that can be moved from one thread to another, and the `Sync` trait implies the type is safe to be referenced from multiple threads at the same time.

These marker traits enable thread-safety tracking in Rust.  A type is only `Send` and/or `Sync` if all their fields are, so the concord of thread safety for a particular type will be kept.

If we were to change `Arc` to `Rc` in the previous rust mutex example:

```rust
fn counter(count: &Rc<Mutex<i32>>) {
    let mut num = count.lock().unwrap();
    *num += 1;
}

fn main() {
    let count = Rc::new(Mutex::new(0));
    let mut threads = vec![];

    for _ in 0..2 {
        let count = Rc::clone(&count); // Rc cannot be moved to another thread
        let thread = thread::spawn(move || counter(&count));
        threads.push(thread);
    }

    for handle in threads {
        handle.join().unwrap();
    }
}
```

Rustc wouldn't let it compile:

```
error[E0277]: `std::rc::Rc<std::sync::Mutex<i32>>` cannot be sent between threads safely
   --> src\main.rs:12:22
    |
12  |         let thread = thread::spawn(move || counter(&count));
    |                      ^^^^^^^^^^^^^ ----------------------- within this `[closure@src\main.rs:12:36: 12:59 count:std::rc::Rc<std::sync::Mutex<i32>>]`
    |                      |
    |                      `std::rc::Rc<std::sync::Mutex<i32>>` cannot be sent between threads safely
```

On the surface, the error is reported because `Rc` doesn't implement the `Send` trait. Yet behind the scenes, it is due to the implementation of  `Rc` not being thread-safe. Unlike `Arc` that uses atomic operations to update reference count, the manipulation of reference count in `Rc` is not atomic. If the `Rc` example above were to be compiled, it might result in a data race. At the end of the two spawned threads, both of them are trying to drop their own `Rc`, thus effectively creating a simultaneous write scenario. With the `Send` trait in the loop, this kind of threat is prohibited upfront.

On the side of C++, this kind of tracking paradigm is absent. `std::shared_pointer` is either atomic or not atomic, depending on the implementation provided by the compilers. In MSVC, `std::shared_pointer` is always atomic, which causes inefficiency when atomicity is unnecessary. Though one may argue the cost of atomic operations is negligible.

## Conclusion

Rust and C++ both have a "1:1" design of concurrency with focus on minimum runtime overhead, but concurrency in Rust is safer than C++ as it offers:

- Guarantees against data races
- Data owning mutexes in replacement of the error-prone lock and unlock
- Thread safety tracking explicitly marking the concurrency properties of the data

Besides, it is not mentioned that async support is still lacking in the current state of the C++ standard. Hopefully with C++ 20 introduced, the concurrency discussion between Rust and C++ can proceed at a larger scale.