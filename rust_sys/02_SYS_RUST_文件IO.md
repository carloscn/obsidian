声明：本文知识树和Linux的知识点来源于《Beginning Linux Programming 4th Edition, Neil Matthew/Richard Stones》。只是替换了其中C语言部分，对应到RUST上。RUST的知识点来源于RUST官方文档手册、《Programming Rust: Fast, Safe Systems Development》等。本文的目的是让Linux的C语言选手快速转向Linux的Rust语言。

* Linux文件结构：
	* 目录
	* 文件和设备
* 系统调用和驱动
* 库函数
* 底层文件访问
	* write系统调用
	* read系统调用
	* open系统调用
	* 访问权限和初始值
	* 其他与文件管理有关的系统调用
* 格式化输入输出
* 文件和目录维护
* 错误处理
* fcntl和mmap

# 底层文件访问

Rust标准库针对输入输出的特性是通过3个特型，即Read、BufRead和Write。

## Write系统调用

在Linux C中，使用系统调用write把缓冲buf的前nbytes字节写入文件描述符所关联的文件中。write调用成功之后返回实际写入的字节数。如果文件描述符在写入过程中有错误或者设备驱动对于写入长度敏感，那么返回值可能小于请求写入的长度nbytes。如果函数返回0则没有写入任何数据；如果长度返回是-1，就表示调用write的时候出现了错误。

在Rust中，是通过Write特型进行写入。实现Write的值既支持字节输出也支持UTF-8的文本输出，这些值叫做Wrtiter（写入器）。

Write在`std::fs::Write`中进行了实现。可以参考 https://doc.rust-lang.org/std/fs/struct.File.html 。

Linux C中提供了标准输入输出的`fwrite`函数及系统调用`write`函数。在标准输入输出中，Linux会自动分配缓存。Rust中的Write方法更和Linux C的`fwrite`标准IO函数对标。

根据fwrite的官方文档， https://man7.org/linux/man-pages/man3/fwrite.3p.html 。fwrite的接口表示为：

```C
#include <stdio.h>

size_t fwrite(const void *restrict _ptr_, 
			  size_t _size_,
			  size_t _nitems_,
              FILE *restrict _stream_);
```

Rust中的文件处理远比Linux C的丰富，可以参考  https://doc.rust-lang.org/std/io/trait.Write.html#provided-methods 。我这里只整理几个常用的例子：

在开发的时候经常用到（UTF-8）ASCII的写入。Rust提供 `write` 方法可以实现ASCII的写入。

```Rust
use std::io::prelude::*;
use std::fs::File;
pub fn write_file_ascii(file_name:&str, content:String) -> Result<(), io::Error> {
	let mut f = File::create(str);
    
    let rc = match f.write(content.as_bytes()) {
        Ok(_) => Ok(()),
        Err(error) => return Err(error),
    };
        
    return rc;
}
```

Rust也提供了`write!`的宏，也可以达到和上面一样的效果。以下例子展示多种write写入字符串的方法，可以根据自己的实际需求使用。

```Rust
use std::io::prelude::*;
use std::fs::File;

pub fn write_file_ascii() -> Result<(), io::Error> {
	let mut buffer = File::create(str)?;
	write!(buffer, "hello")?;
	write!(buffer, "{:.*}", 2, 1.234567)?;
	buffer.write_fmt(format_args!("{:.*}", 2, 1.234567))?;
    return rc;
}
```

以上是访问ASCII的例子，下面展示一些写入二进制的方法：
```Rust
pub fn write_file_bin(file_name:&str, buf:&Vec<u8>) -> Result<usize, io::Error> {
	let mut ctx = File::create(str)?;
    let buf_slice:&[u8] = &buf;
    ctx.write_all(buf_slice)?;
	ctx.write_all(b"some bytes")?;
	
    return Ok(());
}
```

也可以写入多个buffer：

```Rust
#![feature(write_all_vectored)]
use std::io::IoSlice;
use std::io::prelude::*;
use std::fs::File;
use std::io::{Write, IoSlice};

let mut writer = Vec::new();
let bufs = &mut [
    IoSlice::new(&[1]),
    IoSlice::new(&[2, 3]),
    IoSlice::new(&[4, 5, 6]),
];

writer.write_all_vectored(bufs)?;
// Note: the contents of `bufs` is now undefined, see the Notes section.

assert_eq!(writer, &[1, 2, 3, 4, 5, 6]);
```