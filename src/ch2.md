# 页面置换机制

如果要实现页面置换机制，只考虑页替换算法的设计与实现是远远不够的，还需考虑其他问题：

* 哪些页可以被换出？
* 一个虚拟的页如何与硬盘上的扇区建立对应关系？
* 何时进行换入和换出操作？
* 如何设计数据结构以支持页替换算法？
* 如何完成页的换入换出操作？

## 可以被换出的页

在操作系统的设计中，一个基本的原则是：并非所有的物理页都可以交换出去的，只有映射到用户空间且被用户程序直接访问的页面才能被交换，而被内核直接使用的内核空间的页面不能被换出。

这里面的原因是什么呢？操作系统是执行的关键代码，需要保证运行的高效性和实时性，如果在操作系统执行过程中，发生了缺页现象，则操作系统不得不等很长时间（硬盘的访问速度比内存的访问速度慢 2~3 个数量级），这将导致整个系统运行低效。而且，不难想象，处理缺页过程所用到的内核代码或者数据如果被换出，整个内核都面临崩溃的危险。并且操作系统内核运行在S态，在rCore-Tutorial中将S态下的中断异常禁止了。

我的设计中将用户程序动态申请的内存对应的物理页面视为可交换出去的，这在之后的算法具体实现中可以看出来。

## 虚拟磁盘

在QEMU里实际上并没有真正模拟“硬盘”，所以为了实现“页面置换”的效果，我采取的措施是，从内核的静态存储(static)区里面分出一块内存， 声称这块存储区域是“硬盘”，然后包裹一下给出“硬盘IO”的接口。思考一下，内存和硬盘，除了一个掉电后数据易失一个不易失，一个访问快一个访问慢，其实并没有本质的区别。对于我们的页面置换算法来说，也不要求硬盘上存多余页面的交换空间能够“不易失”，反正这些页面存在内存里的时候就是易失的。

理论上，我们完全可以把一块机械硬盘加以改造，写好驱动之后，插到主板的内存插槽上作为内存条使用，当然性能就别想了。那么我们就把QEMU模拟出来的一块ram叫做“硬盘”，用作页面置换时的交换区，完全没有问题。你可能会觉得，这样折腾一通，我们总共能使用的页面数并没有增加，原先能直接在内存里使用的一些页面变成了“硬盘”，只是在自娱自乐。确实，我们在这里只关心页面置换的原理以及各个算法的不同，并不关心实际性能，所以无关算法的地方使用了适当的简化。

我们使用`IDE`来表示磁盘，ide在这里不是integrated development environment的意思，而是Integrated Drive Electronics的意思，表示的是一种标准的硬盘接口。我们这里写的东西和Integrated Drive Electronics并不相关，这个命名是ucore的历史遗留。

```rust
pub const MAX_PAGES: usize = 512;
pub const IDE_SIZE: usize = MAX_PAGES * PAGE_SIZE;

#[repr(align(4096))]
struct IDE {
    pub data: [u8; IDE_SIZE],
}
```

这里使用`4096`对齐，为了方便我将虚拟磁盘的一个磁盘扇区设置成了刚好一个物理页面的大小，这样将物理页面写入和读出磁盘就更方便了：

```rust
pub fn ide_read(idx: usize, dst: &mut [u8]) -> usize {
    if !ide_valid(idx) {
        return 1;
    }
    let base = idx * PAGE_SIZE;
    unsafe {
        let ide_ptr = &IDE.data[base..(base+PAGE_SIZE)];
        dst.copy_from_slice(ide_ptr);
    }
    0
}

pub fn ide_write(idx: usize, src: &[u8]) -> usize {
    if !ide_valid(idx) {
        return 1;
    }
    let base = idx * PAGE_SIZE;
    unsafe {
        let ide_ptr = &mut IDE.data[base..(base+PAGE_SIZE)];
        ide_ptr.copy_from_slice(src);
    }
    0
}
```

## 虚存中的页与硬盘上的扇区之间的映射关系

如果一个页被置换到了硬盘上，那操作系统如何能简捷来表示这种情况呢？在 rcore 的设计上，充分利用了页表中的 PTE 来表示这种情况：页表中的页表项记录了物理页号和对应页的各种属性信息，处理器根据虚拟地址中的虚页号（Virtial Page Number， VPN）为页表索引，可最终查找到虚拟地址所在的物理页位置。这是页表项的基本功能。当我们需要提供可远大于物理地址空间的虚拟地址空间时，页表项中的内容能发挥新的作用。

我们重新梳理一下某任务让处理器访问被换出到存储设备上的数据所经历的过程。在处理器访问某数据之前，操作系统在、已把包含该数据的物理内存换出到了存储设备上，并需要提供关键的关联信息，便于操作系统后续的换入工作：

* 该数据的虚拟地址是属于某任务的地址空间：可在任务控制块中包含任务的合法空间范围的记录
* 该数据的页虚拟地址所对应的存储设备的扇区地址：可在页虚拟地址对应的页表项中包含存储设备的扇区地址的记录
* 该数据的虚拟页没有对应的物理页：在页虚拟地址对应的页表项中的存在位（Present Bit）置“0”，表示物理页不存在

在后续某时刻，该任务让处理器访问该数据时，首先处理器根据虚拟地址获得虚页号，然后检查MMU中的TLB中是否由匹配的项目，如果TLB未命中，则会进一步根据页表基址寄存器信息，查找内存中的页表，并根据VPN找到虚拟页对应的页表项。硬件会进一步查找该页表项的存在位，由于已经被操作系统设置为“0”，表示该页不在物理内存中，处理器会产生“Page Fault”异常，并把控制权交给操作系统的“Page Fault”异常处理例程进行进一步处理。

目前在我的实现中，操作系统使用`IdeManager`来管理磁盘的读写以及用户程序申请内存的物理页面是否被换出到磁盘上：

```rust
pub struct IdeManager {
    current: usize,
    end: usize,
    recycled: Vec<usize>,
    map: BTreeMap<(usize, VirtPageNum), usize>,
}
```

其中`current`、`end`和`recycled`类似于`frame_allocator`的实现，用来分配磁盘的扇区，而`map`用来记录`(token, vpn)`到磁盘扇区编号的映射，`token`为用户程序的标识。

## 缺页异常处理

实现虚存管理的一个关键是 page fault 异常处理，其过程中主要涉及到函数—— `do_pgfault` 的具体实现。比如，在程序的执行过程中由于某种原因（页框不存在/写只读页等）而使 CPU 无法最终访问到相应的物理内存单元，即无法完成从虚拟地址到物理地址映射时，CPU 会产生一次页访问异常，从而需要进行相应的页访问异常的中断服务例程。这个页访问异常处理的时机被操作系统充分利用来完成虚存管理，即实现“按需调页”/“页换入换出”处理的执行时机。当相关处理完成后，页访问异常服务例程会返回到产生异常的指令处重新执行，使得应用软件可以继续正常运行下去。

具体而言，当启动分页机制以后，如果一条指令或数据的虚拟地址所对应的物理页框不在内存中或者访问的类型有错误（比如写一个只读页或用户态程序访问内核态的数据等），就会发生页访问异常。产生页访问异常的原因主要有：

* 目标页帧不存在(页表项全为 0，即该线性地址与物理地址尚未建立映射或者已经撤销)；
* 相应的物理页帧不在内存中(页表项非空，但存在标志位=0，比如在 swap 分区或磁盘文件上)；
* 不满足访问权限(此时页表项 V 标志=1，但低权限的程序试图访问高权限的地址空间，或者有程序试图写只读页面).

### 页异常处理函数的实现

接下来介绍缺页异常处理函数 `do_pgfault` 的实现：

```rust
pub fn do_pgfault(addr: usize, flag: usize) -> bool
```

其中`addr`为当前触发缺页异常的指令访问的虚拟地址，`flag`为0、1、2记录该缺页异常是`LoadPageFault`、`StorePageFault`还是`InstructionPageFault`。

首先判断该虚拟地址对应的物理页面是否存在且有效，此时一定是不满足访问权限导致的缺页异常：

```rust
if let Some(pte) = memory_set.page_table.translate(vpn) {
    if pte.is_valid() {
        if !pte.readable() && flag==0 {
            println!("[kernel] PAGE FAULT: Frame not readable.");
            return false;
        }
        if !pte.writable() && flag==1 {
            println!("[kernel] PAGE FAULT: Frame not writable.");
            return false;
        }
        if !pte.executable() && flag==2 {
            println!("[kernel] PAGE FAULT: Frame not executable.");
            return false;
        }
    }
}
```

然后分为两个部分，局部页面置换算法和全局页面置换算法：

```rust
if PRA_IS_LOCAL {
    local_pra(memory_set, vpn, token)
}
else {
    global_pra(memory_set, vpn, token)
}
```

### 局部页面置换部分的处理

在局部页面置换算法的处理中，接下来在当前用户程序的 `memory_set` 的 `areas` 中查看是否包含触发异常的 `vpn` ，若不包含则说明该用户程序访问了一个无效的虚拟地址，直接返回错误即可，若包含则说明该用户程序申请了包含该虚拟地址的内存，但内核还未为其分配物理页面，或分配后已经被交换到磁盘中暂时失效了。

```rust
for i in 0..memory_set.areas.len() {
    if vpn >= memory_set.areas[i].vpn_range.get_start() && vpn < memory_set.areas[i].vpn_range.get_end() {
        ...
    }
}
```

此前，当用户在 `memory_set` 中申请一段内存时，若可以先不分配则不分配，只将其记录到 `areas` 当中：

```rust
// memory_set的成员函数
fn push(&mut self, mut map_area: MapArea, data: Option<&[u8]>) {
    if let Some(data) = data {
        map_area.map(&mut self.page_table);
        map_area.copy_data(&mut self.page_table, data);
    }
    self.areas.push(map_area);
}
```

回到缺页异常的处理，若触发异常的 `vpn` 包含在 `memory_set` 的 `areas` 中，则说明内核还未为其分配物理页面，或分配后已经被交换到磁盘中暂时失效了，不管怎样我们都需要分配一个物理页面给他使用，此时需要查看物理页面是否足够：

```rust
if let Some(frame) = frame_alloc() { // enough frame
    ppn = frame.ppn;
    memory_set.areas[i].data_frames.insert(vpn, frame);
    println!("[kernel] PAGE FAULT: Frame enough, allocating ppn:{} frame.", ppn.0);
}
else {  // frame not enough, need to swap out a frame
    ppn = memory_set.get_next_frame().unwrap();
    let data_old = ppn.get_bytes_array();
    let mut p2v_map = P2V_MAP.exclusive_access();
    let vpn_old = *(p2v_map.get(&ppn).unwrap());
    ide_manager.swap_in(token, vpn_old, data_old);
    for j in 0..memory_set.areas.len() {
        if vpn_old >= memory_set.areas[j].vpn_range.get_start() && vpn_old < memory_set.areas[j].vpn_range.get_end() {
            memory_set.areas[j].unmap_one(&mut memory_set.page_table, vpn_old);
        }
    }
    p2v_map.remove(&ppn);
    println!("[kernel] PAGE FAULT: (local) Frame not enough, swapping out ppn:{} frame.", ppn.0);

    let frame = frame_alloc().unwrap();
    memory_set.areas[i].data_frames.insert(vpn, frame);
}
```

物理页面足够时直接分配，主要考虑物理页面不足需要换出的情况，`ppn = memory_set.get_next_frame().unwrap()` 获得要换出的物理页面对应的PPN，根据不同算法这里获得的页面不同，然后将该页面的数据复制到磁盘中，并且将其从页表中删除，最后再将该物理页面分配给触发异常的虚拟地址。

如果该虚拟地址之前被分配过物理页面并被换出，我们需要将原来的数据拿出来放到新分配的物理页面中：

```rust
if ide_manager.check(token, vpn) {
    let data = ppn.get_bytes_array();
    ide_manager.swap_out(token, vpn, data);
    println!("[kernel] PAGE FAULT: (local) Swapping in vpn:{} ppn:{} frame.", vpn.0, ppn.0);
}
```

最后建立新的映射：

```rust
if !frame_check() {
    let ppn = memory_set.get_next_frame().unwrap();
    let data_old = ppn.get_bytes_array();
    let mut p2v_map = P2V_MAP.exclusive_access();
    let vpn_old = *(p2v_map.get(&ppn).unwrap());
    ide_manager.swap_in(token, vpn_old, data_old);
    for j in 0..memory_set.areas.len() {
        if vpn_old >= memory_set.areas[j].vpn_range.get_start() && vpn_old < memory_set.areas[j].vpn_range.get_end() {
            memory_set.areas[j].unmap_one(&mut memory_set.page_table, vpn_old);
        }
    }
    p2v_map.remove(&ppn);
    println!("[kernel] PAGE FAULT: (local) Swapping out ppn:{} frame.", ppn.0);
}
println!("[kernel] PAGE FAULT: (local) Mapping vpn:{} to ppn:{}.", vpn.0, ppn.0);
memory_set.page_table.map(vpn, ppn, memory_set.areas[i].get_flag_bits());
P2V_MAP.exclusive_access().insert(ppn, vpn);
memory_set.insert_frame(ppn);
```

由于建立映射时页表可能需要新的物理页面，所以我们先检查一下物理页面是否还有剩余，若没有则提前换出一个，保证页表可以正常工作。

### 全局页面置换部分的处理

一开始和局部页面置换部分的处理相同，先在当前用户程序的 `memory_set` 的 `areas` 中查看是否包含触发异常的 `vpn` ，若包含则进行进一步处理。

```rust
for i in 0..memory_set.areas.len() {
    if vpn >= memory_set.areas[i].vpn_range.get_start() && vpn < memory_set.areas[i].vpn_range.get_end() {
        ...
    }
}
```

首先进行全局页面置换算法的预处理，两个页面置换算法的处理是不同的，在下一章中会详细讲解，如下：

```rust
GFM.exclusive_access().work(memory_set, token);
```

由于是全局页面置换算法，参数设置合理的情况下，这时保证一定有空余的物理页面，直接分配即可。同样地，如果该虚拟地址之前被分配过物理页面并被换出，我们需要将原来的数据拿出来放到新分配的物理页面中：

```rust
let frame = frame_alloc().unwrap();
ppn = frame.ppn;
memory_set.areas[i].data_frames.insert(vpn, frame);

if IDE_MANAGER.exclusive_access().check(token, vpn) {
    let data = ppn.get_bytes_array();
    IDE_MANAGER.exclusive_access().swap_out(token, vpn, data);
    println!("[kernel] PAGE FAULT: (global) Swapping in vpn:{} ppn:{} frame.", vpn.0, ppn.0);
}
```

同样地，最后建立新的映射：

```rust
memory_set.page_table.map(vpn, ppn, memory_set.areas[i].get_flag_bits());
P2V_MAP.exclusive_access().insert(ppn, vpn);
memory_set.insert_frame(ppn);
```

到这里缺页异常就处理完毕了，中断返回后重新执行该指令时可以正常执行。
