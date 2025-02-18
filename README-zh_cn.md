[![](https://img.shields.io/crates/v/dlopen-rs.svg)](https://crates.io/crates/dlopen-rs)
[![](https://img.shields.io/crates/d/dlopen-rs.svg)](https://crates.io/crates/dlopen-rs)
[![license](https://img.shields.io/crates/l/dlopen-rs.svg)](https://crates.io/crates/dlopen-rs)
# dlopen-rs

[文档](https://docs.rs/dlopen-rs/)

一个 `Rust` 库，实现了与libc行为一致的`dlopen`，`dlsym`等一系列接口，为动态库加载和符号解析提供了支持。

这个库有四个目的：
1. 提供一个纯`Rust`编写的动态链接器。
2. 为 #![no_std] 目标提供加载 `ELF` 动态库的支持。
3. 能够轻松地在运行时用自己的自定义符号替换共享库中的符号。
4. 大多数情况下有比`ld.so`更快的速度（加载动态库和获取符号）

## 特性

| 特性      | 是否默认开启 | 描述                                                                                               |
| --------- | ------------ | -------------------------------------------------------------------------------------------------- |
| std       | 是           | 启用Rust标准库                                                                                     |
| debug     | 否           | 启用后可以使用 gdb/lldb 调试已加载的动态库。注意，只有使用 dlopen-rs 加载的动态库才能用 gdb 调试。 |
| mmap      | 是           | 启用在有mmap的平台上的默认实现                                                                     |  |
| version   | 否           | 在寻找符号时使用符号的版本号                                                                       |
| tls       | 是           | 启用后动态库中可以使用线程本地存储。                                                               |  |
| unwinding | 否           | 启用后可以使用 dlopen-rs 提供的异常处理机制。                                                      |
| libgcc    | 是           | 如果程序使用 libgcc 处理异常，启用此特性。                                                         |
| libunwind | 否           | 如果程序使用 libunwind 处理异常，启用此特性。                                                      |
## 示例

### 示例1
使用`dlopen`接口加载动态库，`dlopen-rs`中的`dlopen`与`libc`中的`dlopen`行为一致。此外本库使用了`log`库，你可以使用自己喜欢的库输出log，来查看dlopen-rs的工作流程，本库的例子中使用的是`env_logger`库。
```rust
use dlopen_rs::{ElfLibrary, OpenFlags};
use std::path::Path;

fn main() {
    std::env::set_var("RUST_LOG", "trace");
    env_logger::init();
    dlopen_rs::init();
    let path = Path::new("./target/release/libexample.so");
    let libexample =
        ElfLibrary::dlopen(path, OpenFlags::RTLD_LOCAL | OpenFlags::RTLD_LAZY).unwrap();
    let add = unsafe { libexample.get::<fn(i32, i32) -> i32>("add").unwrap() };
    println!("{}", add(1, 1));

    let print = unsafe { libexample.get::<fn(&str)>("print").unwrap() };
    print("dlopen-rs: hello world");

	let dl_info = ElfLibrary::dladdr(print.into_raw() as usize).unwrap();
    println!("{:?}", dl_info);

    ElfLibrary::dl_iterate_phdr(|info| {
        println!(
            "iterate dynamic library: {}",
            unsafe { CStr::from_ptr(info.dlpi_name).to_str().unwrap() }
        );
        Ok(())
    })
    .unwrap();
}
```
### 示例2
利用`LD_PRELOAD`将libc中dlopen等函数替换为本库中的实现。
```shell
# 将本库编译成动态库形式
cargo build -r -p cdylib
# 编译测试用例
cargo build -r -p dlopen-rs --example preload
# 使用本库中的实现替换libc中的实现
RUST_LOG=trace LD_PRELOAD=./target/release/libdlopen.so ./target/release/examples/preload
```

### 示例3
细粒度地控制动态库的加载流程,可以将动态库中需要重定位的某些函数换成自己实现的函数。下面这个例子中就是把动态库中的`malloc`替换为了`mymalloc`。
```rust
use dlopen_rs::{ElfLibrary, OpenFlags};
use libc::size_t;
use std::{ffi::c_void, path::Path};

extern "C" fn mymalloc(size: size_t) -> *mut c_void {
    println!("malloc:{}bytes", size);
    unsafe { libc::malloc(size) }
}

fn main() {
    std::env::set_var("RUST_LOG", "debug");
    env_logger::init();
    dlopen_rs::init();
    let path = Path::new("./target/release/libexample.so");
    let libc = ElfLibrary::load_existing("libc.so.6").unwrap();
    let libgcc = ElfLibrary::load_existing("libgcc_s.so.1").unwrap();

    let libexample = ElfLibrary::from_file(path, OpenFlags::CUSTOM_NOT_REGISTER)
        .unwrap()
        .relocate_with(&[libc, libgcc], &|name: &str| {
            if name == "malloc" {
                return Some(mymalloc as _);
            } else {
                return None;
            }
        })
        .unwrap();

    let add = unsafe { libexample.get::<fn(i32, i32) -> i32>("add").unwrap() };
    println!("{}", add(1, 1));

    let print = unsafe { libexample.get::<fn(&str)>("print").unwrap() };
    print("dlopen-rs: hello world");
}
```
## 未完成
* dlinfo还未实现。dlerror目前只会返回NULL。
* dlsym的RTLD_NEXT还未实现。
* 在调用dlopen失败时，新加载的动态库虽然会被销毁但没有调用.fini中的函数。
* 是否有方法能够支持更多的重定位类型。
* 缺少在多线程高并发情况下的正确性与性能测试。
* 更多的测试。
## 补充
如果在使用过程中遇到问题可以在 GitHub 上提出问题，十分欢迎大家为本库提交代码一起完善dlopen-rs的功能。😊
