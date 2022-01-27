# Communicating with the operating system

**In this chapter I want to dive into:**

- What a System Call is
- Abstraction levels
- Challenges of writing low-level cross-platform code

## Syscall primer

Communication with the operating system is done through `System Calls` or
"syscalls" as we'll call them from now on. This is a public API that the operating system provides and that programs we write in "userland" can use to communicate with the OS.

Most of the time these calls are abstracted away for us as programmers by the language or the runtime we use. A language like Rust makes it
trivial to make a `syscall` though which we'll see below.

Now, `syscalls` is an example of something that is unique to the kernel you're communicating with, but the UNIX family of kernels has many similarities. UNIX systems expose this through **`libc`**.

Windows, on the other hand, uses its own API, often referred to as WinAPI, and that can be radically different from how the UNIX based systems operate.

Most often though there is a way to achieve the same things. In terms of functionality, you might not notice a big difference but as we'll see below and especially when we dig into how `epoll`, `kqueue` and `IOCP` work, they can differ a lot in how this functionality is implemented.

## Syscall example

To get a bit more familiar with `syscalls` we'll implement a very basic one for the three architectures: `BSD(macos)`, `Linux` and `Windows`. We'll also see how this is implemented in three levels of abstractions.

The `syscall` we'll implement is the one used when we write something to `stdout` since that is such a common operation and it's interesting to see how it really works.

### The lowest level of abstraction

For this to work we need to write some [inline assembly](https://doc.rust-lang.org/1.0.0/book/inline-assembly.html). We'll start by focusing on the instructions we write to the CPU.

> If you want a more thorough introduction to inline assembly I can refer you to the [relevant chapter in my previous book](https://cfsamson.gitbook.io/green-threads-explained-in-200-lines-of-rust/an-example-we-can-build-upon) if you haven't read it already.

Now at this level of abstraction, we'll write different code for all three platforms.

On Linux and macOS the `syscall` we want to invoke is called `write`. Both systems operate based on the concept of `file descriptors` and `stdout` is one of these already present when you start a process.

**On Linux a `write` syscall can look like this** \
(You can run the example by clicking "play" in the right corner)

```rust, edition2021
# // #![feature(force_mdbook_nightly_playground)]
use std::arch::asm;

fn main() {
    let message = String::from("Hello world from interrupt!\n");
    syscall(message);
}

#[cfg(target_os = "linux")]
#[inline(never)]
fn syscall(message: String) {
    let msg_ptr = message.as_ptr();
    let len = message.len();

    unsafe {
        asm!(
            "mov rax, 1",      // system call 0x2000004 is write on macos
            "mov rdi, 1",      // file handle 1 is stdout
            "syscall",         // call kernel, syscall interrupt
            in("rsi") msg_ptr, // address of string to output
            in("rdx") len,     // number of bytes
            out("rax") _, out("rdi") _, lateout("rsi") _, lateout("rdx") _
        );
    }
}
```

The code to initiate the `write` syscall on Linux is `1` so when we write `1` we're writing the literal value 1 to the `rax` register.

Coincidentally, placing the value `1` into the `rdi` register means that we're referring to `stdout` which is the file descriptor we want to write to. This has nothing to do with the fact that the `write` syscall also has the code `1`.

Rusts inline assembly syntax takes a little getting used to but when we write `in("rsi") msg_ptr` and `in("rdx") len` we pass in the address of our string buffer and the length of the buffer in the registers `rsi` and `rdx` respectively.

> We also need to clobber the registers we write to so that we let the compiler know that we're modifying them and it can't rely on storing any values in these. We do this by declaring the registers as outputs but with _ instead of a
> variable name. We use `lateout` for the registers that also serve as inputs since a register marked with `out` can't have an initial value at the start of the assembly block.

Lastly we call the `syscall` instruction.

> The `syscall` instruction is a rather new one. On the earlier 32-bit systems in the `x86` architecture, you invoked a syscall by issuing a software interrupt `int 0x80`. A software interrupt is considered slow at the level we're working at here so later a separate instruction for it called `syscall` was added. The `syscall` instruction uses [VDSO](http://articles.manugarg.com/systemcallinlinux2_6.html), which is a memory page attached to each process' memory, so no context switch is necessary to execute the system call.

**On macOS, the syscall will look something like this:** \
(since the Rust playground is running Linux, we can't run this example here)

```rust, ignore
use std::arch::asm;

fn main() {
    let message = String::from("Hello world from interrupt!\n");
    syscall(message);
}

#[cfg(target_os = "linux")]
#[inline(never)]
fn syscall(message: String) {
    let msg_ptr = message.as_ptr();
    let len = message.len();

    unsafe {
        asm!(
            "mov rax, 0x2000004", // system call 0x2000004 is write on macos
            "mov rdi, 1",         // file handle 1 is stdout
            "syscall",            // call kernel, syscall interrupt
            in("rsi") msg_ptr,    // address of string to output
            in("rdx") len,         // number of bytes
            out("rax") _, out("rdi") _, lateout("rsi") _, lateout("rdx") _
        );
    }
}
```

As you see this is not that different from the one we wrote for Linux, with the exception of the fact that syscall `write` has the code `0x2000004` instead of `1`.

**What about Windows?**

This is a good opportunity to explain why writing code like we do above is a bad idea.

You see, if you want your code to work for a long time you have to worry about what `guarantees` the OS gives you. As far as I know, both Linux and macOS give some guarantees that for example `$$0x2000004` on macOS will always refer to `write` (I'm not sure how strong these guarantees are though). Windows gives absolutely zero guarantees when it comes to low-level internals like this.

Windows has changed it's internals numerous times and provides no official documentation. The only thing we got is reverse engineered tables like [this](https://j00ru.vexillium.org/syscalls/nt/64/). That means that what was `write` can be changed to `delete` the next time you run Windows update.

## The next level of abstraction

The next level of abstraction is to use the API which all three operating systems provide for us.

Already we can see that this abstraction helps us remove some code since fortunately for us, in this specific example, the syscall is the same on Linux and on macOS so we only need to worry if we're on Windows and therefore use the `#[cfg(not(target_os = "windows"))]` conditional compilation flag. For the Windows syscall, we do the opposite.

### Using the OS provided API in Linux and macOS

You can run this code directly here in the window. However, the Rust playground
runs on Linux, you'll need to copy the code over to a Windows machine if you
want to try it out the code for Windows further down.

**Our syscall will now look like this** \
(You can run this code here. It will work for both Linux and macOS)

```rust
use std::io;

fn main() {
    let sys_message = String::from("Hello world from syscall!\n");
    syscall(sys_message).unwrap();
}

// and: http://man7.org/linux/man-pages/man2/write.2.html
#[cfg(not(target_os = "windows"))]
#[link(name = "c")]
extern "C" {
    fn write(fd: u32, buf: *const u8, count: usize) -> i32;
}

#[cfg(not(target_os = "windows"))]
fn syscall(message: String) -> io::Result<()> {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    let res = unsafe { write(1, msg_ptr, len) };

    if res == -1 {
        return Err(io::Error::last_os_error());
    }
    Ok(())
}

```

I'll explain what we just did here. I assume that the `main` method needs no comment.

```rust, ignore
#[link(name = "c")]
```

Every Linux installation comes with a version of `libc` which is a C-library for communicating with the operating system. Having a `libc` with a consistent API means they can change the underlying implementation without breaking everyone's code. This flag tells the compiler to link to the "c" library on the system we're compiling for.

```rust, ignore
extern "C" {
    fn write(fd: u32, buf: *const u8, count: usize);
}
```

`extern "C"` or only `extern` (C is assumed if nothing is specified) means we're linking to specific functions in the "c" library using the "C" calling convention. As you'll see on Windows we'll need to change this since it uses a different calling convention than the UNIX family.

The function we're linking to needs to have the exact same name, in this case, `write`. The parameters don't need to have the same name but they must be in
the right order and it's good practice to name them the same as in the library
you're linking to.

The write function takes a `file descriptor` which in this case is a handle to
`stdout`. In addition, it expects us to provide a pointer to an array of `u8` values and the length of the buffer.

```rust, ignore
#[cfg(not(target_os = "windows"))]
fn syscall_libc(message: String) {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    unsafe { write(1, msg_ptr, len) };
}
```

First, we get a pointer to the underlying buffer of our
string. This will be a pointer of type `*const u8` which matches our `buf`
argument. The `len` of the buffer corresponds to the `count` argument.

You might ask how we know that `1` is the file-handle to `stdout` and where we found that value.

You'll notice this a lot when writing syscalls from Rust. Usually, constants are defined in the C header files which we can't link to, so we need to search them up. 1 is always the file descriptor for `stdout` on UNIX systems.

> Wrapping the `libc` functions and providing these constants is exactly what the
> crate [libc](https://github.com/rust-lang/libc) provides for us and why you'll see that used instead of writing
> the type of code we do here.

A call to an FFI function is always unsafe so we need to use the `unsafe` keyword
here.

### Using the API on Windows

**This syscall will look like this on Windows:** \
(You'll need to copy this code over to a Windows machine to try this out)
```rust, ignore
use std::io;

fn main() {
    let sys_message = String::from("Hello world from syscall!\n");
    syscall(sys_message).unwrap();
}

#[cfg(target_os = "windows")]
#[link(name = "kernel32")]
extern "stdcall" {
    /// https://docs.microsoft.com/en-us/windows/console/getstdhandle
    fn GetStdHandle(nStdHandle: i32) -> i32;
    /// https://docs.microsoft.com/en-us/windows/console/writeconsole
    fn WriteConsoleW(
        hConsoleOutput: i32,
        lpBuffer: *const u16,
        numberOfCharsToWrite: u32,
        lpNumberOfCharsWritten: *mut u32,
        lpReserved: *const std::ffi::c_void,
    ) -> i32;
}

#[cfg(target_os = "windows")]
fn syscall(message: String) -> io::Result<()> {

    // let's convert our utf-8 to a format windows understands
    let msg: Vec<u16> = message.encode_utf16().collect();
    let msg_ptr = msg.as_ptr();
    let len = msg.len() as u32;

    let mut output: u32 = 0;
        let handle = unsafe { GetStdHandle(-11) };
        if handle  == -1 {
            return Err(io::Error::last_os_error())
        }

        let res = unsafe {
            WriteConsoleW(handle, msg_ptr, len, &mut output, std::ptr::null())
            };
        if res  == 0 {
            return Err(io::Error::last_os_error());
        }

    assert_eq!(output as usize, len);
    Ok(())
}
```

Now, just by looking at the code above you see it starts to get a bit more
complex, but let's spend some time to go through line by line what we do here as
well.

```rust, ignore
#[cfg(target_os = "windows")]
#[link(name = "kernel32")]
```

The first line is just telling the compiler to only compile this if the `target_os` is Windows.

The second line is a linker directive, telling the linker we want to link to the library `kernel32` (if you ever see an example that links to `user32` that will also work).

```rust, ignore
extern "stdcall" {
    /// https://docs.microsoft.com/en-us/windows/console/getstdhandle
    fn GetStdHandle(nStdHandle: i32) -> i32;
    /// https://docs.microsoft.com/en-us/windows/console/writeconsole
    fn WriteConsoleW(
        hConsoleOutput: i32,
        lpBuffer: *const u16,
        numberOfCharsToWrite: u32,
        lpNumberOfCharsWritten: *mut u32,
        lpReserved: *const std::ffi::c_void,
    ) -> i32;
}
```

First of all, `extern "stdcall"`, tells the compiler that we won't use the `C`
calling convention but use Windows calling convention called `stdcall`.

The next part is the functions we want to link to. On Windows, we need to link to two functions to get this to work: `GetStdHandle` and `WriteConsoleW`.
`GetStdHandle` retrieves a reference to a standard device like `stdout`.

`WriteConsole` comes in two flavours, `WriteConsoleW` that takes in Unicode text and `WriteConsoleA` that takes ANSI encoded text.

Now, ANSI encoded text works fine if you only write English text, but as soon as you write text in other languages you might need to use special characters that are not possible to represent in `ANSI` but is possible in `utf-8` and our program will break.

That's why we'll convert our `utf-8` encoded text to `utf-16` encoded Unicode codepoints that can represent these characters and use the `WriteConsoleW` function.

```rust, ignore
#[cfg(target_os = "windows")]
fn syscall(message: String) -> io::Result<()> {
    // let's convert our utf-8 to a format windows understands
    let msg: Vec<u16> = message.encode_utf16().collect();
    let msg_ptr = msg.as_ptr();
    let len = msg.len() as u32;

    let mut output: u32 = 0;
    let handle = unsafe { GetStdHandle(-11) };
    if handle == -1 {
        return Err(io::Error::last_os_error());
    }

    let res = unsafe { WriteConsoleW(handle, msg_ptr, len, &mut output, std::ptr::null()) };

    if res == 0 {
        return Err(io::Error::last_os_error());
    }

    assert_eq!(output, len);
    Ok(())
}
```
The first thing we do is to convert the text to utf-16 encoded text which
Windows uses. Fortunately, Rust has a built-in function to convert our `utf-8` encoded text to `utf-16` code points. `encode_utf16` returns an iterator over  `u16` code points that we can collect to a `Vec`.

```rust, ignore
let msg: Vec<u16> = message.encode_utf16().collect();
let msg_ptr = msg.as_ptr();
let len = msg.len() as u32;
```

Next, we get the pointer to the underlying buffer of our `Vec` and get the
length.

```rust, ignore
let handle = unsafe { GetStdHandle(-11) };
if handle  == -1 {
    return Err(io::Error::last_os_error())
}
```

The next is a call to `GetStdHandle`. We pass in the value `-11`. The values we
need to pass in for the different standard devices is actually documented
together with the `GetStdHandle` documentation:

| Handle | Value |
| ------ | ----- |
| Stdin  |   -10 |
| Stdout |   -11 |
| StdErr |   -12 |

Now we're lucky here, it's not that common that we find this information
together with the documentation for the function we call, but it's very convenient when we do.

The return codes to expect is also documented thoroughly for all functions so we handle potential errors here in the same way as we did for the Linux/macOS syscalls.

```rust, ignore
let res = unsafe {
    WriteConsoleW(handle, msg_ptr, len, &mut output, std::ptr::null())
};

if res == 0 {
    return Err(io::Error::last_os_error());
}
```

Next up is the call to the `WriteConsoleW` function. There is nothing too fancy about this.

## The highest level of abstraction

This is simple, most standard libraries provide this abstraction for you. In rust that would simply be:

```rust
println!("Hello world from Stdlib");
```

# Our finished cross-platform syscall

```rust
use std::io;

fn main() {
    let sys_message = String::from("Hello world from syscall!\n");
    syscall(sys_message).unwrap();
}

// and: http://man7.org/linux/man-pages/man2/write.2.html
#[cfg(not(target_os = "windows"))]
#[link(name = "c")]
extern "C" {
    fn write(fd: u32, buf: *const u8, count: usize) -> i32;
}

#[cfg(not(target_os = "windows"))]
fn syscall(message: String) -> io::Result<()> {
    let msg_ptr = message.as_ptr();
    let len = message.len();
    let res = unsafe { write(1, msg_ptr, len) };

    if res == -1 {
        return Err(io::Error::last_os_error());
    }
    Ok(())
}

#[cfg(target_os = "windows")]
#[link(name = "kernel32")]
extern "stdcall" {
    /// https://docs.microsoft.com/en-us/windows/console/getstdhandle
    fn GetStdHandle(nStdHandle: i32) -> i32;
    /// https://docs.microsoft.com/en-us/windows/console/writeconsole
    fn WriteConsoleW(
        hConsoleOutput: i32,
        lpBuffer: *const u16,
        numberOfCharsToWrite: u32,
        lpNumberOfCharsWritten: *mut u32,
        lpReserved: *const std::ffi::c_void,
    ) -> i32;
}

#[cfg(target_os = "windows")]
fn syscall(message: String) -> io::Result<()> {
    // let's convert our utf-8 to a format windows understands
    let msg: Vec<u16> = message.encode_utf16().collect();
    let msg_ptr = msg.as_ptr();
    let len = msg.len() as u32;

    let mut output: u32 = 0;
    let handle = unsafe { GetStdHandle(-11) };
    if handle == -1 {
        return Err(io::Error::last_os_error());
    }

    let res = unsafe { WriteConsoleW(handle, msg_ptr, len, &mut output, std::ptr::null()) };

    if res == 0 {
        return Err(io::Error::last_os_error());
    }

    assert_eq!(output, len);
    Ok(())
}
```
