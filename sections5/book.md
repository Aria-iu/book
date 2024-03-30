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


## condvar——阻塞线程直到条件改变
上一节的选择，一个是condvar，即CONDition VARiable。条件变量是阻塞线程的一个上好选择。

条件变量的一个工作原理是：在一个mutex上lock后，将MutexGuard传递给Condvar::wait，这会阻塞这个线程。别的线程也可能经过相同的过程，在相同的条件上阻塞。一些别的线程可能会获取同一个的互斥锁，最终在这个condvar上调用notify_one或者notify_all。前者会唤醒一个线程，后者会唤醒等待这个condvar的所有线程。所以condvar可能会收到虚假唤醒，这意味着线程可能会在没有收到正确的唤醒的情况下结束阻塞。因为这个原因，条件变量要在一个loop循环中进行条件变量的检查。

下面是一个示例：
```Rust
use std::thread;
use std::sync::{Arc, Condvar, Mutex};
fn main() {
    let total_readers = 5;
    let mutcond: Arc<(Mutex<(bool, u16)>, Condvar)> =Arc::new((Mutex::new((false, 0)),Condvar::new()));
}
```
我们在mutcond上同步线程，mutcond是Arc<(Mutex<(bool, u16)>, Condvar)>。Mutex<(bool,u16)>的第二个元素就是我们的资源。第一个元素是一个Boolean Flag，我们将它看作是一个信号，表示是否可以写这个资源的权限。下面是我们的reader线程：
```Rust
    for _ in 0..total_readers {
        let mutcond = Arc::clone(&mutcond);
        reader_jhs.push(thread::spawn(move || {
            let mut total_zeros = 0;
            let mut total_wakes = 0;
            let &(ref mtx, ref cnd) = &*mutcond;

            while total_zeros < 100 {
                let mut guard = mtx.lock().unwrap();
                while !guard.0 {
                    guard = cnd.wait(guard).unwrap();
                }
                guard.0 = false;

                total_wakes += 1;
                if guard.1 == 0 {
                    total_zeros += 1;
                }
            }
            total_wakes
        }));
    }
```
reader线程在mutex上上锁，如果不能写资源，就持续等待condvar，这是会释放前面在mutex上加的锁。这个reader线程会被阻塞直到notify_all()被调用。每个reader线程都会竞争，抢占成为第一个获取锁的线程。我们的reader线程之间是不合作的，因为它会立即阻止任何其他reader线程找到可用资源的机会。然而，reader线程有些还是会被虚假的唤醒，然后再次被迫等待。也许，reader线程还与writer线程相互竞争这个互斥锁。

下面是writer线程：
```Rust
    let _ = thread::spawn(move || {
        let &(ref mtx, ref cnd) = &*mutcond;
        loop {
            let mut guard = mtx.lock().unwrap();
            guard.1 = guard.1.wrapping_add(1);
            guard.0 = true;
            cnd.notify_all();
        }
    });

    for jh in reader_jhs {
        println!("{:?}", jh.join().unwrap());
    }
```
writer线程是无限循环。writer线程会获取锁，增加资源，即u16 + 1,唤醒等待的线程，释放锁。

下面是运行结果：
```
rustc condvar_example01.rs && ./condvar_example01
5868295
6075531
7898754
6912950
6847538
```
可见，这里比较之前使用MPSC的实例多了许多循环。但这不是为了说明condvar难以使用，它是很容易使用的，只是它需要与其他类型一起使用。

## 屏障——barrier

