How is hot-patching supported in Rust?

How is hot-patching in Rust compared to that in C/C++? 

These are the questions we're hopefully able to answer in this report.

## Function Detour

Hot-patching essentially consists of 3 steps: registering the new function, finding the address of the original function and patching the original function with the new function. The action of patching is often referred to as *function detour*. In order to validate the hot-patching in Rust, we must first implement *function detour* in Rust.

### Function Detour in C/C++

Start with a simple example so we're on the same page - *function detour* in C/C++:

```c++
#include <iostream>
#include <Windows.h>

void func1() { std::cout << "func1" << std::endl; }
void func2() { std::cout << "func2" << std::endl; }

void detour(void* sourceFunc, void* targetFunc) {
    DWORD originProtect;
    char* source = (char*)sourceFunc;
    char* target = (char*)targetFunc;
    int offset = target - source - 5;

    VirtualProtect(source, 5, PAGE_EXECUTE_READWRITE, &originProtect);
    *source = 0xE9;
    *(int*)(source + 1) = offset;
    VirtualProtect(source, 5, originProtect, &originProtect);
}

int main() {
    func1();
    detour(func1, func2);
    func1();
    return 0;
}
```

This is a kind of old-fashioned instruction injection on 32-bit Windows. In the example, what happens is that `func1` gets patched to jump to `func2` in substitution to executing its own logics, thus effectively redirecting the call of `func1` to `func2`. Fire up the program in debug mode and it outputs as intended:

```
func1
func2
```

### Function Detour in Rust

So how's *function detour* supposed to be in Rust? Let's try to replicate the above example in Rust. Coming in at first I wasn't sure about how raw pointer manipulation and system API calls are presented in Rust. Hopefully, as a system-level programming language, Rust doesn't come short in terms of low-level operations.

Rust supports function address peeking through `usize` conversions. We can calculate function address offset for the relative jump by converting function pointers to `usize`:

```rust
fn caclPatchOffset(source: fn(), target: fn()) -> usize{
    target as usize - source as usize - 5
}
```

Raw pointer manipulation is supported as well:

```rust
fn detour(source: fn(), target: fn()) {
    unsafe { *(source as *mut u8) = 0xE9; }
}
```

Rust also has bindings to system APIs. For Windows API, `winapi` crate is provided:

```rust
use winapi;

fn detour(source: fn(), target: fn()) {
    winapi::um::memoryapi::VirtualProtect(
        source as winapi::um::winnt::PVOID,
        5 as winapi::shared::basetsd::SIZE_T,
        winapi::um::winnt::PAGE_EXECUTE_READWRITE,
        &mut originalProtect,
    );
}
```

Piece these together - it turns out we can have a direct translation of the example written in C++:

```rust
use winapi;

fn func1() { println!("func1") }
fn func2() { println!("func2") }

fn detour(source: fn(), target: fn()) {
    let mut originalProtect = 0;
    let offset = target as usize - source as usize - 5;
    unsafe {
        winapi::um::memoryapi::VirtualProtect(
            source as winapi::um::winnt::PVOID,
            5,
            winapi::um::winnt::PAGE_EXECUTE_READWRITE,
            &mut originalProtect,
        );
        *(source as *mut u8) = 0xE9;
        *((source as *mut u8).offset(1) as *mut usize) = offset;
        winapi::um::memoryapi::VirtualProtect(
            source as winapi::um::winnt::PVOID,
            5,
            originalProtect,
            &mut originalProtect,
        );
    }
}

fn main() {
    func1();
    detour(func1, func2);
    func1();
}
```

Compiled in debug mode, it outputs as expected:

```
func1
func2
```

The direct translation works but it's not pretty. It turns out Rust actually has a crate named `detour` particularly designed for function detour purposes. Crate `detour` has a nice capsulation which makes the function detour logic less verbose:

```rust
use std::error::Error;
use detour::GenericDetour;

fn func1() { println!("func1") }
fn func2() { println!("func2") }

fn main() -> Result<(), Box<dyn Error>> {
    func1();
    let hook = unsafe { GenericDetour::<fn()>::new(func1, func2)? };
    unsafe { hook.enable()? };
    func1();
    Ok(())
}
```

It has recoverable mechanics and uses mutexes to rule race conditions. Crate `detour` should be taken into consideration when function detour is necessary in Rust.

### A Deeper Dive into The Function Detour in Rust

The function detour in Rust works, but how? There is a catch I only found out after a deeper dive into the assembly later in the experiment.

The `main()` in the Rust example looks like this in assembly:

```asm
00841450 | sub esp,8
00841453 | call rust-winapi.8412A0
00841458 | lea eax,dword ptr ds:[8412A0]
0084145E | mov dword ptr ss:[esp],eax
00841461 | lea eax,dword ptr ds:[8412F0]
00841467 | mov dword ptr ss:[esp+4],eax
0084146B | call rust-winapi.841340
00841470 | call rust-winapi.8412A0
00841475 | add esp,8
00841478 | ret
```

`call rust-winapi.8412A0` at line 3 and 8 is the call to `func1`.

`call rust-winapi.841340` at line 7 is the call to `detour`.

`func2` sets in address `rust-winapi.8412F0`.

`func1` before `detour` is called:

```assembly
008412A0 | push esi
008412A1 | sub esp,30
008412A4 | xor eax,eax
008412A6 | mov ecx,dword ptr ds:[85C1B8]
008412AC | mov edx,dword ptr ds:[85C1BC]
........ | ...
008412EA | ret
```

`func1` after `detour` is called:

```assembly
008412A0 | jmp rust-winapi.8412F0
008412A5 | ror byte ptr ds:[ebx-7A3E47F3],0
008412AC | mov edx,dword ptr ds:[85C1BC]
........ | ...
008412EA | ret
```

As we can see, `detour` changed the 5 bytes from `008412A0` to `008412A4` into a `jmp` instruction, which is a direct substitution to the function body of `func1`. This is the one of the oldest tricks in the book of function detours.

However, this particular type of function detour has multi-threaded issues since it offers no guarantee for the atomicity of the three patched instructions from `008412A0` to `008412A4`.

Adding preserved NOP instructions is the most frequently used solution to problems `detour` exposes, commonly referred to as compiling functions to be patchable. Additionally, using a single NOP instruction with the length of 5 bytes is one of the methods to execute such compilation. If `func1` is compiled to be patchable using this method, it could be compiled to:

```assembly
008412A0 | nopl 8(%rax,%rax)
008412A5 | push esi
008412A6 | sub esp,30
008412A9 | xor eax,eax
008412AB | mov ecx,dword ptr ds:[85C1B8]
008412B1 | mov edx,dword ptr ds:[85C1BC]
........ | ...
008412EA | ret
```

Now the 5 bytes to be replaced by the `jmp` instruction from `008412A0` to `008412A4` is a single instruction, which is guaranteed to be atomic, ruling out the multi-thread issues.

I carried out this investigation because I couldn't find any Rustc arguments for hot-patchable functions. With that in the mix, there is indeed a missing piece of the puzzle in terms of the hot-patching support in Rust. Until Rustc supports compiling of hot-patchable functions, we may need to insert the NOP instructions manually or try to dig into the compiler to add a function attribute for it, though that's not in the range of this report.

## Hot-Patching

Now that we know Rust is capable of *function detour*,  we can continue to examine how hot-patching can be done in Rust.

### Hot-Patching in C/C++

In the context of a hot-patch, there are three components: the target process, the loader, and the patch itself. The most standard way to carry out the hot-patching task would be seen as following:

- Make the patch that will be used for hot-patching. Most commonly this will be a modified(fixed) version of the function to patch with identical signature. Moreover, on Windows, the patch will be in the form of a *.dll* file.
- Use a loader to inject the hot-patch into the target process. The loader is a kind of general-purpose program. Its task is to locate the target process and load the patch into the memory of the process. How it functions largely depends on the OS.
- Find the address of the function to patch in the target process. The address can be obtainable in a variety of ways depending on the OS. On the extreme case, it can also be manually hard-coded.
- Detour the original function in replacement of the new function.

Let's look at a minimal example of the whole process on Windows.

For the record, we have a target program called *target.exe* which includes the function `EchoOncePerSecond`, that we want to patch:

```c++
void EchoOncePerSecond() {
    using namespace std::chrono_literals;
    std::cout << "Echo" << std::endl;
    std::this_thread::sleep_for(1s);
}

int main() {
    while (1) {
        EchoOncePerSecond();
    }
    return 0;
}
```

To start, we write the hot-patch and call it as *patching-lib.dll*. It contains the new implementation of the original `EchoOncePerSecond`:

```c++
#include "detour.h"

void PatchedPrintOncePerSecond()
{
	using namespace std::chrono_literals;
	std::cout << "PrintOncePerSecond has been patched!" << std::endl;
	std::this_thread::sleep_for(1s);
}

BOOL WINAPI DllMain(HMODULE hModule, DWORD dwReason, LPVOID lpReserved)
{
	if (dwReason == DLL_PROCESS_ATTACH)
	{
		DWORD targetFuncAddress = (DWORD)GetModuleHandle(NULL) + 0x31C0;
		detour((void*)targetFuncAddress, PatchedPrintOncePerSecond);
	}
	return TRUE;
}
```

`0x31C0` is the offset of the function `EchoOncePerSecond` in *target.exe*. `detour` is exactly the same as demonstrated in the beginning.

Then, the loader:

```c++
int main() {
    LPCSTR dllFullPath = "C:\\injection\\bin\\patching-lib\\patching-lib.dll";
    HANDLE hProcess = OpenProcess(PROCESS_ALL_ACCESS, FALSE, 12844);
    LPVOID pDllPath = VirtualAllocEx(hProcess, 0, strlen(dllFullPath) + 1, MEM_COMMIT, PAGE_READWRITE);
    WriteProcessMemory(hProcess, pDllPath, (LPVOID)dllFullPath, strlen(dllFullPath) + 1, 0);
    HANDLE hLoadThread = CreateRemoteThread(hProcess, 0, 0,
        (LPTHREAD_START_ROUTINE)GetProcAddress(GetModuleHandleA("Kernel32.dll"), "LoadLibraryA"), pDllPath, 0, 0);
    WaitForSingleObject(hLoadThread, INFINITE);
    VirtualFreeEx(hProcess, pDllPath, strlen(dllFullPath) + 1, MEM_RELEASE);
    return 0;
}
```

For the sake of simplicity, the full path of the patch and the pid of the target process are hard-coded into the loader. All the loader does is to allocate memory for the patch, copy the patch to the memory of the target process and make the process load the patch. Once the patch is loaded, `EchoOncePerSecond` gets patched to jump to `PatchedPrintOncePerSecond`. Output in the console would be something like this:

```
...
Echo
Echo
PrintOncePerSecond has been patched!
PrintOncePerSecond has been patched!
...
```

### Hot-Patching in Rust

Now we've seen how hot-patching is carried out in C++ on Windows, it's not hard to observe that the hot-patching process isn't entirely bound to the programming language. In fact, the hot-patching that is shown above consists of a large portion of calls to system APIs, which Rust is perfectly capable of as we've discussed in the previous sections.

Let's quickly replicate the minimal example in Rust. First the target process:

```Rust
fn EchoOncePerSecond() {
    println!("Echo");
    std::thread::sleep(time::Duration::from_secs(1));
}

fn main() {
    loop {
        EchoOncePerSecond();
    }
}
```

Then the hot-patch:

```Rust
use std::{mem, ptr, time};
use winapi::{ctypes::c_void, um::libloaderapi::GetModuleHandleA};

fn PatchedPrintOncePerSecond() {
    println!("PrintOncePerSecond has been patched and it's in Rust!");
    std::thread::sleep(time::Duration::from_secs(1));
}

#[no_mangle]
pub extern "stdcall" fn DllMain(module: u32, reason: u32, reserved: *mut c_void) {
    match reason {
        1 => patching(),
        _ => (),
    };
}

fn patching() {
    let targetFunc = unsafe {
        let targetFuncAddress = GetModuleHandleA(ptr::null()) as usize + 0x1120;
        mem::transmute::<*const (), fn()>(targetFuncAddress as *const ())
    };
    detour(targetFunc, PatchedPrintOncePerSecond);
}
```

Now we can re-use the loader in the C++ example, the whole hot-patch actually works like a charm:

```
...
Echo
PrintOncePerSecond has been patched and it's in Rust!
PrintOncePerSecond has been patched and it's in Rust!
...
```

As we can see, hot-patching in Rust doesn't have much of a difference from that in C++.

Well, if we look closely we can spot that the function offset of the patch target `EchoOncePerSecond` has changed from `0x31C0` to `0x1120`. Though you would get the offset by probing the PDB file in both cases, that's not really different either.

In the best-case scenario, it won't be necessary for us to lean so hard on probing the address of un-exported functions. If a function is designed to be patchable we would want to export it and also stops the function mangling. We have syntaxes like this in C++:

```c++
extern "C" void __declspec(dllexport) EchoOncePerSecond() {}
```

It is also entirely possible for Rust to declare such export:

```Rust
#[no_mangle]
pub extern "C" fn EchoOncePerSecond() {}
```

## Conclusion

How is hot-patching supported in Rust?

It is fully supported other than the lack of patchable function compiler arguments.

How is hot-patching in Rust compared to that in C/C++? 

Almost identical. There are minor differences in terms of interacting with the system APIs but that's as far as the difference goes. The hot-patching procedure itself is the same.