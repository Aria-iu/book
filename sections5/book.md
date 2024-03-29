## 锁——互斥锁、条件变量、屏障和读写锁
在本章中，我们会深入研究Ring的升级版——Hopper。

本章结束，我们会学到：
- [x]  讨论Mutex、Condvar、Barriers、RWLock的作用
- [x]  研究一种称为hopper的磁盘备份版本的标准库MPSC
- [x]  了解如何在生产环境中应用QuickCheck、AFL和全面的基准测试

## 读写锁——RWLock
考虑这种情况：你有一个资源，在同一时间必须只有一个线程操作这个资源，但是可以同时被多个线程不可变的访问。也就是说，可以有多个reader，但只有一个writer。尽管可以使用Mutex来保护这个资源，但是有一个不足，Mutex对它的lockers是不加以识别的;无论哪个线程持有等待这个锁，都要被迫等待，无论这个线程想要获取这个锁是否要修改这个资源。RwLock\<T>是一个更佳的选择，它允许两种锁，read锁和write锁。由于Rust语言的特性，同一时刻只能有一个write锁存在，但是可以存在多个read锁。下面是一个示例：
```Rust
use std::thread;
use std::sync::{Arc, RwLock};

fn main() {
    let resource: Arc<RwLock<u16>> = Arc::new(RwLock::new(0));

    let total_readers = 5;

    let mut reader_jhs = Vec::with_capacity(total_readers);
    for _ in 0..total_readers {
        let resource = Arc::clone(&resource);
        reader_jhs.push(thread::spawn(move || {
            let mut total_lock_success = 0;
            let mut total_lock_failure = 0;
            let mut total_zeros = 0;
            while total_zeros < 100 {
                match resource.try_read() {
                    Ok(guard) => {
                        total_lock_success += 1;
                        if *guard == 0 {
                            total_zeros += 1;
                        }
                    }
                    Err(_) => {
                        total_lock_failure += 1;
                    }
                }
            }
            (total_lock_failure, total_lock_success)
        }));
    }

    {
        let mut loops = 0;
        while loops < 100 {
            let mut guard = resource.write().unwrap();
            *guard = guard.wrapping_add(1);
            if *guard == 0 {
                loops += 1;
            }
        }
    }

    for jh in reader_jhs {
        println!("{:?}", jh.join().unwrap());
    }
}
```
这个实例的原理是我们有一个writer线程，有一个共享资源——u16。一旦这个资源被访问，加了100次u之后，writer线程就会退出。此外，有total_readers个reader线程，它们会尝试锁定这个共享资源，即u16，直到它达到100。

我们对这个示例的预期是输出以下内容：
```
(0, 100)
(0, 100)
(0, 100)
(0, 100)
(0, 100)
```
这种情况表明reader线程在获取read锁时从来不会失败，意味着此时没有write锁的存在，reader线程都被调度到writer线程之前执行。

但实际上，
```
rustc rwlock_example00.rs && ./rwlock_example00
(0, 100)
(53687191, 6999697)
(59243322, 6222832)
(52323227, 6790879)
(56255055, 6810877)

rustc rwlock_example00.rs && ./rwlock_example00
(0, 100)
(0, 100)
(0, 100)
(0, 100)
(28343800, 1518172)
```
在实例中，reader线程可能会被调度到writer线程刚刚执行完之后，在它执行时，可能会遇到guard不为0的状况，这样，即使reader线程获取到read锁，也会因为共享资源不为0而重新执行。共享资源是u16类型，想要再次为0，只有溢出后继续加。从reader线程的返回值看出，进行了大量的无用循环。

那么如何避免不必要的大量循环呢。如果我们需要每个reader线程获得每个write的值，那么MPSC是个合理的选择。
```Rust
use std::thread;
use std::sync::mpsc;

fn main() {
    let total_readers = 5;
    let mut sends = Vec::with_capacity(total_readers);

    let mut reader_jhs = Vec::with_capacity(total_readers);
    for _ in 0..total_readers {
        let (snd, rcv) = mpsc::sync_channel(64);
        sends.push(snd);
        reader_jhs.push(thread::spawn(move || {
            let mut total_zeros = 0;
            let mut seen_values = 0;
            for v in rcv {
                seen_values += 1;
                if v == 0 {
                    total_zeros += 1;
                }
                if total_zeros >= 100 {
                    break;
                }
            }
            seen_values
        }));
    }

    {
        let mut loops = 0;
        let mut cur: u16 = 0;
        while loops < 100 {
            cur = cur.wrapping_add(1);
            for snd in &sends {
                snd.send(cur).expect("failed to send");
            }
            if cur == 0 {
                loops += 1;
            }
        }
    }

    for jh in reader_jhs {
        println!("{:?}", jh.join().unwrap());
    }
}
```
结果如下：
```
./writer_example01
6553600
6553600
6553600
6553600
6553600
```
由于n主线程向通道中发送了65536×100次，所以每次reader线程都会响应，data_seen为65536×100，没有多余的信息被获取，没有多余的循环执行。

但是如果reader线程不需要看到每一次的数据更改，我们在之后会有更佳的选择。