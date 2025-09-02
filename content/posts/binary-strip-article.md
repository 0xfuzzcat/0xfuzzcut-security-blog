+++
date = '2025-09-02T22:47:47+02:00'
draft = false
title = 'Demystifying Binary Stripping'
+++

This article contains basic info about stripping: 
- what it is
- how it looks in a binary (i.e. executable file)
- what do we need it for?

All the explanations and examples are shown on a Rust app compiled on macOS for ARM64 architecture. Anyway, conclusions will be mostly the same if you are on a different OS like Linux or use a different CPU (Intel or AMD processor).

### What is stripping?
Stripping is a process of removing symbol tables from a binary.

### What is a symbol table?
A symbol table is a part of a binary file. That part contains info about function names that are included in the binary.

### What do we need a symbol table for?
Well, in case your app crashes, the crash log/message will contain much more detailed information about a particular place in code where the crash happened.
Instead of
```shell
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 5', <unknown>
stack backtrace:
   0:        0x102a1c8f0 - <unknown>
   1:        0x102a1c7a4 - <unknown>
   2:        0x102a1b2c8 - <unknown>
   3:        0x102a1c5f4 - <unknown>
   4:        0x102a1c3b8 - <unknown>
   5:        0x102a1c2a0 - <unknown>
   6:        0x102a18f3c - <unknown>
   7:        0x102a18d24 - <unknown>
   8:        0x102a18c98 - <unknown>
note: run with `RUST_BACKTRACE=full` for a verbose backtrace
```
you will get something like this:
```shell
thread 'main' panicked at 'index out of bounds: the len is 3 but the index is 5', src/main.rs:42:18
stack backtrace:
   0: rust_begin_unwind
             at /rustc/d5a82bbd26e1ad8b7401f6a718a9c57c96905483/library/std/src/panicking.rs:575:5
   1: rust_panic
             at /rustc/d5a82bbd26e1ad8b7401f6a718a9c57c96905483/library/std/src/panicking.rs:519:5
   2: std::panicking::rust_panic_with_hook
             at /rustc/d5a82bbd26e1ad8b7401f6a718a9c57c96905483/library/std/src/panicking.rs:267:17
   3: std::panicking::begin_panic_handler
             at /rustc/d5a82bbd26e1ad8b7401f6a718a9c57c96905483/library/std/src/panicking.rs:597:13
   4: core::panicking::panic_fmt
             at /rustc/d5a82bbd26e1ad8b7401f6a718a9c57c96905483/library/core/src/panicking.rs:67:14
   5: core::panicking::panic_bounds_check
             at /rustc/d5a82bbd26e1ad8b7401f6a718a9c57c96905483/library/core/src/panicking.rs:162:5
   6: myapp::process_data
             at ./src/main.rs:42:18
   7: myapp::main
             at ./src/main.rs:15:9
   8: core::ops::function::FnOnce::call_once
             at /rustc/d5a82bbd26e1ad8b7401f6a718a9c57c96905483/library/core/src/ops/function.rs:250:5
note: run with `RUST_BACKTRACE=full` for a verbose backtrace
```
The second sample is much more helpful as it shows the code location (*src/main.rs:42:18*).

### Well, if a symbol table gives us such an advantage for debugging and issue fixing - why does the stripping procedure exist in the first place? 
There are two main reasons why developers may decide to strip a binary:
1. Reducing binary size
2. As an anti-RE (reverse engineering) mechanism.

### How much can a binary be reduced with stripping?
It depends on your app and the language, but usually the difference is **about 20%**.

### What about the anti-RE bit?
The same way the symbol table makes our lives easier, it also makes the life of a reverse engineer easier. 
Reverse engineers / hackers / security engineers may research your software out of curiosity or to use gained information for a cyber attack.
It depends on your business model and the type of software to fully answer the question - should you apply stripping or not.
- If your app is used only to send network requests to your backend where all business logic is running - well, it seems that there is not much to 'research' in your app.
- If your app contains some business logic or implementation that you want to hide - it's better to apply stripping as an anti-RE measure.

### How good is stripping as an anti-RE measure?
Well, stripping makes the life of a reverse engineer a bit worse but it's not the ultimate defense mechanism.
Let me show an example of how the same code looks before stripping and after.
A binary before stripping, disassembled in Ghidra:
![Non-stripped binary in Ghidra](/images/normal_code.png)

The same binary after stripping:
![Stripped binary in Ghidra](/images/stripped_code.png)
Pay attention to the function names - in the listing and in the right part with the functions list.
Instead of normal function names we can see in the stripped example only something like "FUN_100001b8c".
From that picture you also can see why it's not an ultimate defense against reverse engineering. The disassembled code is the same for both cases, so a focused reverse engineer with a lot of time can still reconstruct our code.

### How to apply stripping to a binary file?
Usually it's just 
```shell
strip your-compiled-binary-file
```
Be wary though, that the stripping tool may be different depending on your operating system.
For example, on macOS the strip tool is usually part of Xcode tools (the path might be something like */Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/strip* but it's available just with *strip* in Terminal.app).

### Can we have the best of both worlds - a stripped binary and detailed crash reports?
Actually, yes. When you have a build that you're planning to ship to your customers, you should save debug symbols:
```shell
# on macOS:
dsymutil myapp -o myapp.dSYM
# on Linux you'd probably use objcopy tool: https://ftp.gnu.org/old-gnu/Manuals/binutils-2.12/html_node/binutils_5.html
```
Then, you can convert addresses from a crash report to function names with the following command:
```shell
# let's say, you got an address 0x102a1c8f0 from a crash report that you want to desymbol:
atos -o myapp.dSYM/Contents/Resources/DWARF/myapp 0x102a1c8f0
```