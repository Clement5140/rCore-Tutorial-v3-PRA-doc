# 页面置换算法的实现

## 相关数据结构

在 `frame_manager.rs` 中，包含了 `ClockQue`、`LocalFrameManager` 和 `GlobalFrameManager` 等协助完成页面置换算法的类，定义和作用如下：

* `ClockQue`，管理 `Clock` 和改进 `Clock` 页面置换算法中的物理页面队列，在触发缺页异常时按照算法弹出下一个被置换的物理页面。`ppns` 为记录的物理页面循环队列，`ptr` 为算法当前的指针位置，类上定义了 `push` 和 `pop` 方法。

```rust
struct ClockQue {
    ppns: Vec<PhysPageNum>,
    ptr: usize,
}
```

* `LocalFrameManager`，局部页面置换算法的总管理器，保存使用的页面置换算法，`FIFO` 的队列， `Clock` 和改进 `Clock` 的队列，以及协助全局页面置换算法的 `global_ppns` 用来记录每个进程已经分配的物理页面的页号。每个 `MemorySet` 类的实例也就是每个进程的 `memory_set` 中都有一个 `LocalFrameManager` 用来进行局部页面管理，类上定义了 `get_next_frame` 和 `insert_frame` 方法，供 `MemorySet` 进行调用。

```rust
pub struct LocalFrameManager {
    used_pra: PRA,
    fifo_que: Queue<PhysPageNum>,
    clock_que: ClockQue,
    global_ppns: Vec<PhysPageNum>,
}
```

* `GlobalFrameManager`，全局页面置换算法的总管理器，保存使用的页面置换算法，以及两个算法需要用到的变量，在之后相应的算法详细描述中会进行解释。类上定义了 `pff_work`、`check_workingset` 和 `workingset_work` 方法，其中第一个为缺页率置换算法的方法，后两个为工作集置换算法的方法。

```rust
pub struct GlobalFrameManager {
    used_pra: PRA,
    t_last: usize,
    idx: usize,
}
```

## 局部页面置换算法

### FIFO页面置换算法

FIFO页面置换算法实现非常简单，只需要给每一个用户程序记录一个物理页面队列即可，在用户访问申请的内存并触发缺页异常分配物理页面时将对应PPN加入队列的队尾，在需要换出物理页面时将队首的物理页面换出。

```rust
// LocalFrameManager 的 get_next_frame 方法中
match self.used_pra {
    PRA::FIFO => {
        self.fifo_que.pop()
    }
    ...
}

// LocalFrameManager 的 insert_frame 方法中
match self.used_pra {
    PRA::FIFO => {
        self.fifo_que.push(ppn)
    }
    ...
}
```

### Clock页面置换算法

与FIFO页面置换算法相同，在 `LocalFrameManager` 中插入和获取置换的页面，但使用的是 `clock_que` 。

```rust
// LocalFrameManager 的 get_next_frame 方法中
match self.used_pra {
    PRA::Clock => {
        self.clock_que.pop(page_table)
    }
    ...
}

// LocalFrameManager 的 insert_frame 方法中
match self.used_pra {
    PRA::Clock | PRA::ClockImproved => {
        self.clock_que.push(ppn)
    }
    ...
}
```

在 `ClockQue` 中，`push` 和 `pop` 如下：

```rust
pub fn push(&mut self, ppn: PhysPageNum) {
    self.ppns.push(ppn);
}

pub fn pop(&mut self, page_table: &mut PageTable) -> Option<PhysPageNum> {
    loop {
        let ppn = self.ppns[self.ptr];
        let vpn = *(P2V_MAP.exclusive_access().get(&ppn).unwrap());
        let pte = page_table.find_pte(vpn).unwrap();
        if !pte.is_valid() {
            panic!("[kernel] PAGE FAULT: (local) Pte not valid in PRA Clock pop.");
        }
        if !pte.accessed() {
            self.ppns.remove(self.ptr);
            if self.ptr == self.ppns.len() {
                self.ptr = 0;
            }
            return Some(ppn);
        }
        pte.change_access();
        println!("[kernel] PAGE FAULT: (local) Changing pte access, ppn: {}.", ppn.0);
        if pte.accessed() {
            panic!("[kernel] PAGE FAULT: (local) Pte access did not change.");
        }
        self.inc();
    }
}
```

`push` 就是简单的将新的物理页面号加入队尾，`pop` 相对复杂一些，循环寻找替换出的物理页面直到找到为止，每次循环中查看该物理页面的页表项的 `access` 位，若为 `0` 则选择该物理页面弹出，否则将其 `access` 位置零后查看循环队列的下一位。

### 改进的Clock页面置换算法

改进的Clock和Clock页面置换算法相似，只有 `pop` 操作不同，改进后的如下：

```rust
pub fn pop_improved(&mut self, page_table: &mut PageTable) -> Option<PhysPageNum> {
    loop {
        let ppn = self.ppns[self.ptr];
        let vpn = *(P2V_MAP.exclusive_access().get(&ppn).unwrap());
        let pte = page_table.find_pte(vpn).unwrap();
        if !pte.is_valid() {
            panic!("[kernel] PAGE FAULT: (local) Pte not valid in PRA Clock pop.");
        }
        if !pte.accessed() && !pte.dirty() {
            self.ppns.remove(self.ptr);
            if self.ptr == self.ppns.len() {
                self.ptr = 0;
            }
            return Some(ppn);
        }
        if pte.accessed() {
            pte.change_access();
            println!("[kernel] PAGE FAULT: (local) Changing pte access, ppn: {}.", ppn.0);
            if pte.accessed() {
                panic!("[kernel] PAGE FAULT: (local) Pte access did not change.");
            }
        }
        else {
            pte.change_dirty();
            println!("[kernel] PAGE FAULT: (local) Changing pte dirty, ppn: {}.", ppn.0);
            if pte.dirty() {
                panic!("[kernel] PAGE FAULT: (local) Pte dirty did not change.");
            }
        }
        self.inc();
    }
}
```

每次循环中先检查该物理页面的页表项 `access` 和 `dirty`，若都为 `0` 则选择该页面弹出，否则若 `access` 为 `1`，则将其置零并循环下一位，若 `access` 为 `0` 而 `dirty` 为 `1`，则将 `dirty` 置零并循环下一位，直到找到为止。

## 全局页面置换算法

### 缺页率置换算法

缺页率算法设置一个常数 `PFF_T` 表示两次缺页异常间隔的阈值。具体实现时，如果两次缺页异常的间隔大于这个阈值，则换出所有在这段时间内不活跃的物理页面，如果小于等于这个阈值，则将所有页面的访问位清除以便下次判断每个页面是否在这次和下次之间的时间内活跃。

此算法的实现在 `GlobalFrameManager` 的 `pff_work` 方法中，首先是 `t_current-t_last > PFF_T` 的情况，需要换出这段时间内不活跃的物理页面：

```rust
for i in 0..task_manager.ready_queue.len() {
    let process = task_manager.ready_queue[i].process.upgrade().unwrap();
    let mut pcb = process.inner_exclusive_access();
    let token = pcb.get_user_token();
    let memory_set = &mut pcb.memory_set;
    for j in (0..memory_set.frame_manager.global_ppns.len()).rev() {
        let ppn = memory_set.frame_manager.global_ppns[j];
        ...
    }
}
for i in (0..memory_set_.frame_manager.global_ppns.len()).rev() {
    let ppn = memory_set_.frame_manager.global_ppns[i];
    ...
}
```

需要换出所有任务的不活跃物理页面，第一个循环是处理在 `task_manager` 中的任务，第二个循环是处理当前触发缺页异常的任务，上面代码省略号中，获取到物理页面( `ppn` )后进行如下处理：

```rust
let data_old = ppn.get_bytes_array();
let mut p2v_map = P2V_MAP.exclusive_access();
let vpn = *(p2v_map.get(&ppn).unwrap());
if let Some(pte) = memory_set.page_table.translate(vpn) {
    if pte.is_valid() && !pte.accessed() {
        IDE_MANAGER.exclusive_access().swap_in(token, vpn, data_old);
        for k in 0..memory_set.areas.len() {
            if vpn >= memory_set.areas[k].vpn_range.get_start() && vpn < memory_set.areas[k].vpn_range.get_end() {
                memory_set.areas[k].unmap_one(&mut memory_set.page_table, vpn);
            }
        }
        p2v_map.remove(&ppn);
        memory_set.frame_manager.global_ppns.remove(j);
        println!("[kernel] PAGE FAULT: (global) Swapping out ppn:{} frame.", ppn.0);
    }
}
```

查看该物理页面的页表项，若 `access` 位为 `0` 则需要换出，将其保存到磁盘内，然后取消 `memory_set` 中该物理页面的映射。

然后是 `t_current-t_last <= PFF_T` 的情况，则将所有物理页面的访问位清除：

```rust
for i in 0..task_manager.ready_queue.len() {
    let process = task_manager.ready_queue[i].process.upgrade().unwrap();
    let mut pcb = process.inner_exclusive_access();
    let memory_set = &mut pcb.memory_set;
    for j in 0..memory_set.frame_manager.global_ppns.len() {
        let ppn = memory_set.frame_manager.global_ppns[j];
        ...
    }
}
for i in 0..memory_set_.frame_manager.global_ppns.len() {
    let ppn = memory_set_.frame_manager.global_ppns[i];
    ...
}
```

和上述过程类似，获取到物理页面( `ppn` )后进行如下处理：

```rust
let p2v_map = P2V_MAP.exclusive_access();
let vpn = *(p2v_map.get(&ppn).unwrap());
if let Some(pte) = memory_set.page_table.find_pte(vpn) {
    if pte.is_valid() && pte.accessed() {
        println!("[kernel] PAGE FAULT: (global) Changing pte access, ppn: {} pte.ppn: {}.", ppn.0, pte.ppn().0);
        pte.change_access();
    }
}
```

将该物理页面的页表项的 `access` 位置零。
上一章缺页异常的全局页面置换算法的预处理会调用该 `pff_work` 方法，参数 `PFF_T` 设置合理的情况下，在处理后保证有空余的物理页面。

### 工作集置换算法

工作集置换算法相较于其他算法实现更复杂，因为操作系统无法准确获取用户程序每次访存的位置，也就无法获取准确的工作集，但我们可以通过定期时钟中断和引用位来近似得到工作集模型。

例如，假设工作集大小 $\Delta$ 为最近 20000 个引用，并且每两次时钟中断之间平均会有 5000 个引用，当得到一个定时器中断时，复制并清除所有页面的引用位，发生缺页异常时可以检查当前的引用位和复制的位于内存的引用位n个位共n+1个位，这些位可以确定过去的 5000*(n+1) 次引用中该页面是否被使用过，若被使用过，这些位中至少有一位会被打开，若没有使用过，那么这些位会被关闭。至少有一位被打开的页面被视为在工作集中。

具体实现中，在每次时钟中断时，先调用 `GlobalFrameManager` 的 `check_workingset` 方法，复制并清除所有用户程序的所有页面的引用位：

```rust
Trap::Interrupt(Interrupt::SupervisorTimer) => {
    check_workingset();
    set_next_trigger();
    check_timer();
    suspend_current_and_run_next();
}
```

在 `check_workingset` 的实现中，遍历所有用户程序分配的物理页面，获取其物理页号：

```rust
let mut pte_recorder = PTE_RECORDER.exclusive_access();
pte_recorder[self.idx].clear();
let task_manager = TASK_MANAGER.exclusive_access();
for i in 0..task_manager.ready_queue.len() {
    let process = task_manager.ready_queue[i].process.upgrade().unwrap();
    let mut pcb = process.inner_exclusive_access();
    let token = pcb.get_user_token();
    let memory_set = &mut pcb.memory_set;
    for j in (0..memory_set.frame_manager.global_ppns.len()).rev() {
        let ppn = memory_set.frame_manager.global_ppns[j];
        ...
    }
}

let process = current_process();
let mut pcb = process.inner_exclusive_access();
let token = pcb.get_user_token();
let memory_set = &mut pcb.memory_set;
for j in (0..memory_set.frame_manager.global_ppns.len()).rev() {
    let ppn = memory_set.frame_manager.global_ppns[j];
    ...
}
self.idx = (self.idx + 1) % WORKINGSET_DELTA_NUM;
```

其中，`WORKINGSET_DELTA_NUM` 为近似工作集算法时设置的时钟中断数n，在最近n次时钟中断时都要复制记录下所有页面的引用位，`PTE_RECORDER` 为记录引用位的一个映射循环队列，`idx` 为当前循环指标。上面代码省略号中，获取到物理页面的页号后，进行以下处理：

```rust
let p2v_map = P2V_MAP.exclusive_access();
let vpn = *(p2v_map.get(&ppn).unwrap());
if let Some(pte) = memory_set.page_table.find_pte(vpn) {
    if pte.is_valid() {
        pte_recorder[self.idx].insert((token, vpn), pte.accessed());
        if pte.accessed() {
            println!("[kernel] Supervisor Timer: Changing pte access, ppn: {} pte.ppn: {}.", ppn.0, pte.ppn().0);
            pte.change_access();
        }
    }
}
```

先查询对应的虚拟页号，若页表项有效，则建立用户程序token和虚拟页号到当前页表项 `access` 位的映射，相当于将其记录下来，然后清除该引用位。

然后是 `workingset_work` 方法，同样地，先遍历所有用户程序分配的物理页面，获取其物理页号：

```rust
for i in 0..task_manager.ready_queue.len() {
    let process = task_manager.ready_queue[i].process.upgrade().unwrap();
    let mut pcb = process.inner_exclusive_access();
    let token = pcb.get_user_token();
    let memory_set = &mut pcb.memory_set;
    for j in (0..memory_set.frame_manager.global_ppns.len()).rev() {
        let ppn = memory_set.frame_manager.global_ppns[j];
        ...
    }
}
for i in (0..memory_set_.frame_manager.global_ppns.len()).rev() {
    let ppn = memory_set_.frame_manager.global_ppns[i];
    ...
}
```

然后省略号中进行同样的处理：

```rust
let mut p2v_map = P2V_MAP.exclusive_access();
let vpn = *(p2v_map.get(&ppn).unwrap());
let mut flag = false;
for k in 0..WORKINGSET_DELTA_NUM {
    if let Some(result) = pte_recorder[k].get(&(token, vpn)) {
        if *result == true {
            flag = true;
            break;
        }
    }
}
if let Some(pte) = memory_set.page_table.find_pte(vpn) {
    if pte.is_valid() && pte.accessed() {
        flag = true;
    }
}
if !flag {
    let data_old = ppn.get_bytes_array();
    IDE_MANAGER.exclusive_access().swap_in(token, vpn, data_old);
    for k in 0..memory_set.areas.len() {
        if vpn >= memory_set.areas[k].vpn_range.get_start() && vpn < memory_set.areas[k].vpn_range.get_end() {
            memory_set.areas[k].unmap_one(&mut memory_set.page_table, vpn);
        }
    }
    p2v_map.remove(&ppn);
    memory_set.frame_manager.global_ppns.remove(j);
    println!("[kernel] PAGE FAULT: Swapping out ppn:{} frame.", ppn.0);
}
```

先查询对应的虚拟页号，然后在 `PTE_RECORDER` 记录中查询之前 `WORKINGSET_DELTA_NUM` 个时钟中断中以及这一次时钟中断中是否有访问位为1，也就是 `flag` 的含义，若这些时钟中断之中都没有访问位为1，则该物理页面需要被换出，通过 `IDE_MANAGER` 将其数据保存至磁盘中，然后取消其在 `memeory_set` 中的映射即可。

同缺页率算法一样，上一章缺页异常的全局页面置换算法的预处理会调用该方法，在参数 `WORKINGSET_DELTA_NUM` 设置合理的情况下，在处理后保证有空余的物理页面。
