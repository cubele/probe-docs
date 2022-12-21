# 移植rCore-Tutorial的例子
<https://github.com/cubele/rCore-Tutorial-Code-2022A/tree/main/os8>

1. 把probe模块复制到`os8/src`下，在`main.rs`引入模块并加入仓库里写的dependency：

```toml
[dependencies]
lazy_static = { version = "1.4", features = ["spin_no_std"] }
lock = { git = "https://github.com/DeathWish5/kernel-sync", rev = "8486b8" }
riscv-decode = { git = "https://github.com/latte-c/riscv-decode", rev = "bc8da4e" }
```
2. 更改`probe/osutils.rs`，需要将里面的工具根据Tutorial重写：

PAGE_SIZE：
``` rust
pub const PAGE_SIZE: usize = crate::config::PAGE_SIZE;
```

页分配函数：
```rust
/// Allocate a page of memory, return virtual address
/// The page need to be readable and executable by user, and writable by kernel
pub fn alloc_page() -> usize {
    let pa = raw_frame_alloc().unwrap().into();
    let va = pa; // identity mapping in kernel
    va
}

/// Deallocate a page of memory from virtual address
pub fn dealloc_page(va: usize) {
    let pa = va;
    raw_frame_dealloc(pa.into());
}
```
因为tutorial中使用frame的生命周期实现dealloc，和probe里面手写的Drop无法兼容，所以需要写两个raw函数：
```rust
pub fn raw_frame_alloc() -> Option<PhysAddr> {
    FRAME_ALLOCATOR.exclusive_access().alloc().map(|ppn| ppn.into())
}

pub fn raw_frame_dealloc(pa: PhysAddr) {
    FRAME_ALLOCATOR.exclusive_access().dealloc(pa.into());
}
```
直接返回地址而不映射到FrameTracker，兼容probe里的页管理。当然也可以修改probe里的数据结构使用统一风格管理。

*注意分配的页需要有内核RWX权限，目前为了方便，直接在new_kernel()函数初始化时全局更改了物理内存区域的权限，理论上只需要给分配的页权限*

内核的内存复制函数：
```rust
/// Copy memory from src to dst, uses virtual address in kernel
pub fn byte_copy(dst_addr: usize, src_addr: usize, len: usize) {
    let dst = dst_addr as *mut u8;
    let src = src_addr as *const u8;
    unsafe {
        core::ptr::copy(src, dst, len);
    }
}
```

3. 因为Tutorial没有使用Trapframe库，需要wrap一下Trapframe，更改`probe/arch/riscv/trapframe.rs`：
```rust
pub use crate::trap::TrapContext as TrapFrame;

pub fn get_trapframe_pc(tf: &TrapFrame) -> usize {
    tf.sepc
}

pub fn set_trapframe_pc(tf: &mut TrapFrame, pc: usize) {
    tf.sepc = pc;
}

pub fn get_trapframe_ra(tf: &TrapFrame) -> usize {
    tf.x[1]
}

pub fn set_trapframe_ra(tf: &mut TrapFrame, ra: usize) {
    tf.x[1] = ra;
}

pub fn get_reg(tf: &TrapFrame, reg: u32) -> usize {
    let index = reg as usize;
    if index != 0 {
        tf.x[index]
    } else {
        0
    }
}

pub fn set_reg(tf: &mut TrapFrame, reg: u32, val: usize) {
    let index = reg as usize;
    if index != 0 {
        tf.x[index] = val;
    }
}
```
将Tutorial自己的Trapcontext包装为Trapframe，实现一些简单的操作。

4. 由于probe需要替换指令，.text段的权限也要改为RWX，在new_kernel()函数里修改即可。

5. 在ebreak中S->S的trap函数里调用函数`kprobes_breakpoint_handler`：
```rust
#[no_mangle]
pub fn trap_from_kernel(_trap_cx: &TrapContext) {
    let scause = scause::read();
    let stval = stval::read();
    match scause.cause() {
        Trap::Exception(Exception::Breakpoint) => {
            println!("[kernel] breakpoint at {:#x}", _trap_cx.sepc);
            unsafe {kprobes_breakpoint_handler(_trap_cx);}
        }
        _ => {
            panic!(
                "Unsupported trap from kernel: {:?}, stval = {:#x}!",
                scause.cause(),
                stval
            );
        }
    }
}
```
2022A的Tutorial没有实现S->S的trap，需要从<https://github.com/rcore-os/rCore-Tutorial-v3/tree/main/os/src/trap>里复制过来。

6. 在main函数初始化完成后调用`probe::run_tests();`即可测试。

移植完毕后相关的syscall可以通过`probe`模块里的接口函数实现。可以参考后续eBPF移植时syscall的实现。

如果要实现通过符号注册probe的话需要引入内核的符号表并在`osutils`中提供符号转地址的函数，实现以后就可以直接用`register_kprobe_with_symbol`的接口。