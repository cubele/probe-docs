# zCore基础设施实现
相关代码在`linux-object/src/dbginfo`与`zircon-object/src/symbol`中。

## 符号表
为了方便kprobe的挂载，提供了用内核函数符号挂载kprobe的功能，通过在符号表中查询将符号转换为地址。

原本rCore中是编译时直接在内核elf中预留出空间将符号表写入内核elf。在zCore的实现里我们直接将符号表挂载在文件系统中使用，更加方便。（linux OS的符号表会被放在`/proc/kallsyms`中，我们实现时直接放在了根目录下）

## backtrace
将rCore中利用栈帧backtrace的实现移植到zCore中，实现时有以下几点改动：

1. 编译时强制加入了`cargo.env("RUSTFLAGS", "-C force-frame-pointers=yes -C symbol-mangling-version=v0");`的flag，rCore使用的旧版flag似乎不管用了。

2. 原本的backtrace是通过符号表查找符号名打印，我们在内核中引入了基于gimli实现的addr2line，并将zcore的debuginfo拷贝到内核里读取，从而可以打印出完整的backtrace，包括被inline函数的信息以及对应的文件位置。

# notes
引入gimli库本来是想用debuginfo做async函数相关的trace，不过后面发现这部分工作其实并不需要嵌入到内核里。如果要分析debuginfo做line2addr之类的事情的话完全可以在内核外面做得到地址。详细的backtrace也可以用<https://github.com/rcore-os/rCore/blob/master/tools/addr2line.py>的方法直接在外部得到。所以这部分的确有一些冗余，不过这样backtrace会比较方便。
