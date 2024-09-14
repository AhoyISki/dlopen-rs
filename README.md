[![](https://img.shields.io/crates/v/dlopen-rs.svg)](https://crates.io/crates/dlopen-rs)
[![](https://img.shields.io/crates/d/dlopen-rs.svg)](https://crates.io/crates/dlopen-rs)
[![license](https://img.shields.io/crates/l/dlopen-rs.svg)](https://crates.io/crates/dlopen-rs)
# dlopen-rs

dlopen-rs supports loading dynamic libraries from both memory and files, and is compatible with the `no_std` environment. This gives you greater flexibility in loading and managing dynamic libraries, offering a viable option for using them in no_std contexts. Additionally, it integrates seamlessly with the system’s dynamic linker in std environments.   
Currently, it supports `x86_64`, `x86`, `RV64`, and `AArch64` architectures.

## Feature
| Feature   | Default | Description                                                                                                                                           |
| --------- | ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| ldso      | Yes     | Allows dynamic libraries to be loaded using system dynamic loaders (ldso).                                                                            |
| std       | Yes     | Enable `std`                                                                                                                                          |
| debug     | Yes     | Enable this to use gdb/lldb for debugging loaded dynamic libraries. Note that only dynamic libraries loaded using dlopen-rs can be debugged with gdb. |
| mmap      | Yes     | Enable this on platforms that support `mmap`                                                                                                          |
| tls       | Yes     | Enable this to use thread local storage.                                                                                                              |
| nightly   | No      | Enable this for faster loading, but you’ll need to use the nightly compiler.                                                                          |
| unwinding | No      | Enable this to use the exception handling mechanism provided by dlopen-rs.                                                                            |
| libgcc    | Yes     | Enable this if the program uses libgcc to handle exceptions.                                                                                          |
| libunwind | No      | Enable this if the program uses libunwind to handle exceptions.                                                                                       |


## Example
```rust
use dlopen_rs::ELFLibrary;
use nix::libc::size_t;
use std::{ffi::c_void, path::Path};

extern "C" fn mymalloc(size: size_t) -> *mut c_void {
    println!("malloc:{}bytes", size);
    unsafe { nix::libc::malloc(size) }
}

fn main() {
    let path = Path::new("./target/release/libexample.so");
    let libc = ELFLibrary::sys_load("libc.so.6").unwrap();
    let libgcc = ELFLibrary::sys_load("libgcc_s.so.1").unwrap();
    let libexample = ELFLibrary::from_file(path)
        .unwrap()
        .relocate_with_func(&[libc, libgcc], |name| {
            if name == "malloc" {
                return Some(mymalloc as _);
            } else {
                return None;
            }
        })
        .unwrap();
    let f = unsafe {
        libexample
            .get::<extern "C" fn(i32) -> i32>("c_fun_add_two")
            .unwrap()
    };
    println!("{}", f(2));

    let f = unsafe {
        libexample
            .get::<extern "C" fn()>("c_fun_print_something_else")
            .unwrap()
    };
    f();
}
```
## NOTE
If you encounter any issues while using it or need any new features, feel free to raise an issue on GitHub.