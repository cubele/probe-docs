# 移植zCore
<https://github.com/cubele/zCore>

为了方便，页表的权限是直接修改`kernel-hal/src/bare/arch/riscv/vm.rs`实现的，没有对分配的页单独操作。

同时还加入了符号表等基础设施，见[zCore基础设施](../zCore%E5%9F%BA%E7%A1%80%E8%AE%BE%E6%96%BD.md)。