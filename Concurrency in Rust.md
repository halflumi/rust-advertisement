## Concurrency in Rust

This summary mainly focuses on the differences in concurrent programming between Rust and C++ .

The differences will be demonstrated in a variety of aspects of concurrent programming with a focus on the improvements Rust provides in terms of Concurrency facilities compared to C++.

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

 As we've already discussed in the summary about ownership, Rust offers compile time guarantees against all data races. The same code in Rust won't pass the borrow checks of the Rustc:

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

