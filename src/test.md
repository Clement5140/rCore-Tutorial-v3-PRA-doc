# 用户态程序测试

主要使用`mmap`和`munmap`两个系统调用分配内存后进行不同的测试。

## 初步测试

初步测试使用`ch4`中的测例进行测试，测试算法为 `FIFO` ，目的为检验页面置换机制的正确性，结果如下：

首先可以看到操作系统启动后可以使用的物理页面数量为454：

<div align=center>

![plic_cpu](assets/test/1.png)

</div>

对于`test2_mmap0`，将其中len设置得较大，使得其分配的内存大于操作系统可以使用得总的物理页面内存，就可以触发物理页面的换入和换出：

```rust
#[no_mangle]
fn main() -> i32 {
    let start: usize = 0x10000000;
    let len: usize = 4096 * 100;
    let prot: usize = 3;
    assert_eq!(0, mmap(start, len, prot));
    for i in start..(start + len) {
        let addr: *mut u8 = i as *mut u8;
        unsafe {
            *addr = i as u8;
        }
    }
    for i in start..(start + len) {
        let addr: *mut u8 = i as *mut u8;
        unsafe {
            assert_eq!(*addr, i as u8);
        }
    }
    println!("Test 04_1 OK!");
    0
}
```

将其设置为`4096 * 100`，即需要100个物理页面，此时操作系统可用的物理页面足够，只触发`StorePageFault`并通过测试：

<div align=center>

![plic_cpu](assets/test/2.png)

</div>

将其设置为`4096 * 800`，即需要800个物理页面，此时操作系统可用的物理页面不足，需要换入和换出物理页面，会触发`StorePageFault`和`LoadPageFault`，并通过测试：

<div align=center>

![plic_cpu](assets/test/3.png)

</div>

对于`test2_mmap1`，程序分配了只读的一个物理页面的内存后，尝试修改内存：

```rust
#[no_mangle]
fn main() -> i32 {
    let start: usize = 0x10000000;
    let len: usize = 4096;
    let prot: usize = 1;
    assert_eq!(0, mmap(start, len, prot));
    let addr: *mut u8 = start as *mut u8;
    unsafe {
        *addr = start as u8;
    }
    println!("Should cause error, Test 04_2 fail!");
    0
}
```

结果如下，先触发缺页异常分配一个物理页面，然后同一条指令再次触发缺页异常，被判断为修改不可写的物理页面结束进程:

<div align=center>

![plic_cpu](assets/test/4.png)

</div>

对于`test2_mmap2`，程序分配了只写的一个物理页面的内存后，尝试访问内存：

```rust
#[no_mangle]
fn main() -> i32 {
    let start: usize = 0x10000000;
    let len: usize = 4096;
    let prot: usize = 2;
    assert_eq!(0, mmap(start, len, prot));
    let addr: *mut u8 = start as *mut u8;
    unsafe {
        // *addr = start as u8; // can't write, R == 0 && W == 1 is illegal in riscv
        assert!(*addr != 0);
    }
    println!("Should cause error, Test 04_2 fail!");
    0
}
```

结果如下，先触发缺页异常分配一个物理页面，然后同一条指令再次触发缺页异常，被判断为读取不可读的物理页面结束进程:

<div align=center>

![plic_cpu](assets/test/5.png)

</div>

对于`test2_unmap`，其用来测试`mmap`和`munmap`的正确性，也可以测试缺页异常处理的正确性：

```rust
#[no_mangle]
fn main() -> i32 {
    let start: usize = 0x10000000;
    let len: usize = 4096;
    let prot: usize = 3;
    assert_eq!(0, mmap(start, len, prot));
    assert_eq!(mmap(start + len, len * 2, prot), 0);
    assert_eq!(munmap(start, len), 0);
    assert_eq!(mmap(start - len, len + 1, prot), 0);
    // println!("~~~~~~");
    for i in (start - len)..(start + len * 3) {
        let addr: *mut u8 = i as *mut u8;
        unsafe {
            *addr = i as u8;
        }
    }
    for i in (start - len)..(start + len * 3) {
        let addr: *mut u8 = i as *mut u8;
        unsafe {
            assert_eq!(*addr, i as u8);
        }
    }
    println!("Test 04_5 ummap OK!");
    0
}
```

结果如下，触发四次缺页异常并分配物理页面通过测试：

<div align=center>

![plic_cpu](assets/test/6.png)

</div>

## 综合测试结果

## 综合测例

综合测试共七个测例，分别对每个算法进行测试，目的为检验算法正确性以及测试各个算法的效率。

* 测例一

  ```rust
  fn main() -> i32 {
    let start: usize = 0x10000000;
    let len: usize = 4096 * 100;
    let prot: usize = 3;
    assert_eq!(0, mmap(start, len, prot));
    for i in start..(start + len) {
        let addr: *mut u8 = i as *mut u8;
        unsafe {
            *addr = i as u8;
        }
    }
    for i in start..(start + len) {
        let addr: *mut u8 = i as *mut u8;
        unsafe {
            assert_eq!(*addr, i as u8);
        }
    }
    println!("Test1 OK!");
    0
  }
  ```
  
  此测例为基础测例，依顺序访问申请的内存地址，其申请的页面不超过用户可用的物理页面，也即所有页面可同时存放在内存中。
* 测例二

  ```rust
  fn main() -> i32 {
    let start: usize = 0x10000000;
    let len: usize = 4096 * 400;
    let prot: usize = 3;
    assert_eq!(0, mmap(start, len, prot));
    for i in start..(start + len) {
        let addr: *mut u8 = i as *mut u8;
        unsafe {
            *addr = i as u8;
        }
    }
    for i in start..(start + len) {
        let addr: *mut u8 = i as *mut u8;
        unsafe {
            assert_eq!(*addr, i as u8);
        }
    }
    println!("Test2 OK!");
    0
  }
  ```

  此测例和测例一基本相同，但申请的页面若再算上申请页表用去的页面超过了用户可用的物理页面，测试算法是否能正常运行并且通过测试。
* 测例三

  ```rust
  fn main() -> i32 {
    let start: usize = 0x10000000;
    let len: usize = 4096 * 100;
    let prot: usize = 3;
    let seed: isize = get_time();
    let mut v: isize = seed;
    assert_eq!(0, mmap(start, len, prot));
    for i in 0..len {
        let mut k = i as isize + random(4096, v);
        while k<0 || k>=len as isize {
            k = i as isize + random(4096, k);
        }
        let addr: *mut u8 = (start + k as usize) as *mut u8;
        unsafe {
            *addr = (k as usize) as u8;
        }
        v = k;
    }

    let mut v: isize = seed;
    for i in 0..len {
        let mut k = i as isize + random(4096, v);
        while k<0 || k>=len as isize {
            k = i as isize + random(4096, k);
        }
        let addr: *mut u8 = (start + k as usize) as *mut u8;
        unsafe {
            assert_eq!(*addr, (k as usize) as u8);
        }
        v = k;
    }
    println!("Test3 OK!");
    0
  }
  ```

  此测例用了我自己写的伪随机函数 `pub fn random(m: isize, last: isize)`，生成 `[-m, m)` 区间内的整数，`last` 为上一个随机数。
  此测例与测例一相似，大致依顺序访存，但每次访存时加上一个 `[-4096, 4096)` 的随机数。
* 测例四
  
  ```rust
  fn main() -> i32 {
    let start: usize = 0x10000000;
    let len: usize = 4096 * 400;
    let prot: usize = 3;
    let seed: usize = get_time() as usize;
    let mut v: usize = seed;
    assert_eq!(0, mmap(start, len, prot));
    for _ in 0..4096 {
        let k = random(len, v);
        let addr: *mut u8 = (start + k) as *mut u8;
        unsafe {
            *addr = (k) as u8;
        }
        v = k;
    }

    let mut v: usize = seed;
    for _ in 0..4096 {
        let k = random(len, v);
        let addr: *mut u8 = (start + k as usize) as *mut u8;
        unsafe {
            assert_eq!(*addr, (k as usize) as u8);
        }
        v = k;
    }
    println!("Test4 OK!");
    0
  }
  ```

  此测例用的伪随机函数 `pub fn random(m: isize, last: isize)` 稍有不同，生成的是 `[0, m)` 区间的整数，测例进行了 4096 次整个申请内存上的随机访存并检验正确性。
* 测例五
  
  ```rust
  fn main() -> i32 {
    let start: usize = 0x10000000;
    let len: usize = 4096 * 400;
    let prot: usize = 3;
    let seed: usize = get_time() as usize;
    let mut v: usize = seed;
    assert_eq!(0, mmap(start, len, prot));
    for i in 0..64 {
        for _j in 0..64 {
            let k = random(len / 64, v) + len / 64 * i;
            let addr: *mut u8 = (start + k) as *mut u8;
            unsafe {
                *addr = (k) as u8;
            }
            v = k;
        }
    }

    let mut v: usize = seed;
    for i in 0..64 {
        for _j in 0..64 {
            let k = random(len / 64, v) + len / 64 * i;
            let addr: *mut u8 = (start + k as usize) as *mut u8;
            unsafe {
                assert_eq!(*addr, (k as usize) as u8);
            }
            v = k;
        }
    }
    println!("Test5 OK!");
    0
  }
  ```

  此测例与测例四相似，但将 4096 次访存分成 64 轮，将申请的内存也平均分成 64 块，每轮访问的内存为申请内存的其中一块，在这一块中进行随机访存，以模拟计算机程序的局部性，在测试结果中可以看见缺页异常的次数相较于测例四有显著下降。
* 测例六

  ```rust
  fn main() -> i32 {
    for i in 0..3 {
        let pid = fork();
        if pid == 0 {
            println!("I am child {}", i);
            let start: usize = 0x10000000;
            let len: usize = 4096 * 400;
            let prot: usize = 3;
            let seed: usize = get_time() as usize;
            let mut v: usize = seed;
            assert_eq!(0, mmap(start, len, prot));
            for i in 0..64 {
                for _j in 0..64 {
                    let k = random(len / 64, v) + len / 64 * i;
                    let addr: *mut u8 = (start + k) as *mut u8;
                    unsafe {
                        *addr = (k) as u8;
                    }
                    v = k;
                }
            }
        
            let mut v: usize = seed;
            for i in 0..64 {
                for _j in 0..64 {
                    let k = random(len / 64, v) + len / 64 * i;
                    let addr: *mut u8 = (start + k as usize) as *mut u8;
                    unsafe {
                        assert_eq!(*addr, (k as usize) as u8);
                    }
                    v = k;
                }
            }
            exit(0);
        } else {
            println!("forked child pid = {}", pid);
        }
        assert!(pid > 0);
    }
    let mut exit_code: i32 = 0;
    for _ in 0..3 {
        if wait(&mut exit_code) <= 0 {
            panic!("wait stopped early");
        }
    }
    if wait(&mut exit_code) > 0 {
        panic!("wait got too many");
    }
    println!("Test6 OK!");
    0
  }
  ```

  此测例 `fork` 了三个子进程，每个子进程都进行测例五的测试。
* 测例七

  ```rust
  fn main() -> i32 {
    for i in 0..3 {
        let pid = fork();
        if pid == 0 {
            println!("I am child {}", i);
            if i == 0 {
                let start: usize = 0x10000000;
                let len: usize = 4096 * 400;
                let prot: usize = 3;
                let seed: usize = get_time() as usize;
                let mut v: usize = seed;
                assert_eq!(0, mmap(start, len, prot));
                for i in 0..64 {
                    for _j in 0..64 {
                        let k = random(len / 64, v) + len / 64 * i;
                        let addr: *mut u8 = (start + k) as *mut u8;
                        unsafe {
                            *addr = (k) as u8;
                        }
                        v = k;
                    }
                }
            
                let mut v: usize = seed;
                for i in 0..64 {
                    for _j in 0..64 {
                        let k = random(len / 64, v) + len / 64 * i;
                        let addr: *mut u8 = (start + k as usize) as *mut u8;
                        unsafe {
                            assert_eq!(*addr, (k as usize) as u8);
                        }
                        v = k;
                    }
                }
            }
            if i == 1 {
                let start: usize = 0x10000000;
                let len: usize = 4096 * 400;
                let prot: usize = 3;
                let seed: usize = get_time() as usize;
                let mut v: usize = seed;
                assert_eq!(0, mmap(start, len, prot));
                for _ in 0..4096 {
                    let k = random(len, v);
                    let addr: *mut u8 = (start + k) as *mut u8;
                    unsafe {
                        *addr = (k) as u8;
                    }
                    v = k;
                }
            
                let mut v: usize = seed;
                for _ in 0..4096 {
                    let k = random(len, v);
                    let addr: *mut u8 = (start + k as usize) as *mut u8;
                    unsafe {
                        assert_eq!(*addr, (k as usize) as u8);
                    }
                    v = k;
                }
            }
            if i == 2 {
                let start: usize = 0x10000000;
                let len: usize = 4096 * 400;
                let prot: usize = 3;
                assert_eq!(0, mmap(start, len, prot));
                for i in start..(start + len) {
                    let addr: *mut u8 = i as *mut u8;
                    unsafe {
                        *addr = i as u8;
                    }
                }
                for i in start..(start + len) {
                    let addr: *mut u8 = i as *mut u8;
                    unsafe {
                        assert_eq!(*addr, i as u8);
                    }
                }
            }
            exit(0);
        } else {
            println!("forked child pid = {}", pid);
        }
        assert!(pid > 0);
    }
    let mut exit_code: i32 = 0;
    for _ in 0..3 {
        if wait(&mut exit_code) <= 0 {
            panic!("wait stopped early");
        }
    }
    if wait(&mut exit_code) > 0 {
        panic!("wait got too many");
    }
    println!("Test7 OK!");
    0
  }
  ```

  此测例 `fork` 了三个子进程，分别进行测例二、四、五的测试。

### 测试结果

得到的测试结果如下表所示(缺页次数)：

|算法|测例一|测例二|测例三|测例四|测例五|测例六|测例七|
|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
|FIFO|100|800|100|1884|2684|5084|3272|
|Clock|100|800|100|638|800|2400|3214|
|改进Clock|100|785|100|633|781|2404|3261|
|缺页率(PFF_T=100000)|200|800|200|7614|975|7301|10418|
|缺页率(PFF_T=200000)|100|800|200|5999|818|3019|失败|
|工作集(5)|200|800|200|7816|800|5562|11358|
|工作集(20)|100|800|100|6934|800|3589|9789|

可以从非常多的角度去分析这个表格：

* 首先，测例一二三由于访存非常规律所以得到的缺页次数基本固定且可人为计算
* 对于全随机的测例四，FIFO和全局算法的表现都并不好，而两个Clock算法表现较好，甚至比顺序访存表现更好
* 比较全随机测例四和有一定局部性的随机测例五，局部算法缺页次数增加，而全局算法缺页次数减少，说明全局算法对程序局部性更敏感也更需求
* 对于缺页率算法而言，缺页时间间隔T设置越大则运行时工作集越大，缺页会更少，但有失败的风险，由于设置间隔过大页面释放不及时，当然可以进行补救，在算法中缺页时物理页面不足时释放上次缺页异常和这次之间不活跃的页面即可，这里只是单纯测试算法所以没有进行补救
* 对于工作集算法而言，后面的数字代表工作集大小，工作集越大缺页越少
* 对比测例六和测例七：FIFO缺页次数降低，因为其在测例五上表现不好，而测例七中有两个子进程使用了二和四，整体表现提升；Clock缺页次数增加，因为局部Clock算法没有考虑进程间的访存差异，访存规律一致时表现非常好，而访存不一致时表现下降；全局算法表现都有下降，因为测例七中将有局部性的随机测例五换成了全随机的测例四，导致整体缺页次数增加
