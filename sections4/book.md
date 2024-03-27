# Sync 和 Send , Rust并发的基础
Rust旨在成为一个无俱并发的编程语言。这是什么意思？它如何工作？在第二章，我们讨论了顺序执行的程序的性能，故意将并行程序的讨论放在一边。在第三章，我们大致了解了Rust掌管内存的方式，特别是关于如何组合成为高性能的数据结构。在本章，我们会继续之前所学的，深入挖掘Rust并发的编程方式。

在本章，我们会：
- [x]  讨论Sync和Send这两个trait
- [x]  使用Helgrind工具在ring数据结构中检查多线程竞争 
- [x]  使用锁Mutex来解决竞争
- [x]  使用标准库的MPSC
- [x]  建立一个使用MPSC工作的项目

## Sync 和 Send
有两个Rust并行编程人员必须要了解的trait，Send和Sync。Send这个trait可以被那些能够在线程之间传送的数据结构实现。这意味着，任何类型T: Send，都是可以在线程之间安全移动的。Rc\<T>: !Send意味着它被显式标记为不能安全的在线程之间传送。为什么呢？假设它可以在线程之间安全传送，会发生什么。我们知道Rc\<T>是使用两个计数器来计算弱引用和强引用的计数，并和数据结构T一起封装起来的数据结构。这些计数器就是程序可以耍把戏的地方。假设我们将一个Rc\<T>在线程之间进行传送，分别是A线程和B线程。当A和B同时drop指向Rc\<T>最后一个引用的时候，就会产生竞争，一个会先析构T，另一个会因为重复释放而出错。

更糟糕是，假设增加强引用计数器的指令横跨三条机器指令：
```asm
    LOAD counter            (tmp)
    ADD counter 1 INTO tmp  (tmp = tmp+1)
    STORE tmp INTO counter  (counter = tmp)
```
像这样，减少强引用计数器的指令横跨三条机器指令：
```asm
    LOAD counter            (tmp)
    SUB counter 1 INTO tmp  (tmp = tmp-1)
    STORE tmp INTO counter  (counter = tmp)
```
在单线程上下文中，这工作的很好，但是考虑在多线程中的结果。下面我们假设counter是10：
```asm
    [A] LOAD counter            (tmp_a == 10)
    [B] LOAD counter            (tmp_b == 10)
    [B] SUB counter 1 INTO tmp  (tmp_b = tmp_b -1)
    [A] ADD counter 1 INTO tmp  (tmp_a = tmp_a +1)
    [A] STORE tmp INTO tmp      (counter = tmp_a)  == 11
    [A] ADD counter 1 INTO tmp  (tmp_a = tmp_a +1)
    [A] STORE tmp INTO counter  (counter = tmp_a)  == 12
    [A] ADD counter 1 INTO tmp  (tmp_a = tmp_a +1)
    [A] STORE tmp INTO counter  (counter = tmp_a)  == 13
    [B] STORE tmp INTO counter  (counter = tmp_b) == 9
```
如上，结果，我们丢失三个引用计数，这会导致二次释放或者悬空指针的使用，引起恐慌。

Sync这个特性是从Send派生出来的，如果满足&T: Send,那么必须有T: Sync。这意味着，T只有线程之间共享&T，表现的像是将T传送到各个线程时，才是Sync。如上，我们知道了Rc\<T>: !Send,则T: !Sync,进一步可以得知Rc\<T>: !Sync。Rust的类型继承它们构成部分的Sync和Send的状态。

结论是，任何实现了Send和Sync的类型都是称为线程安全的。任何基于Rc\<T>实现的类型都不是线程安全的。在第三章讨论的UnsafeCell，不是线程安全的。因为源码中：
```Rust
impl<T: ?Sized> !Sync for UnsafeCell<T> {}
```
原始指针也不是线程安全的，因为它们缺乏其他的安全保障。当你在Rust源码中四处搜索时，你会发现原本可以线程安全的trait被标记为不安全。

## 线程竞争
你可以将类型标记为线程安全的，有效地向编译器保证，你已经将所有数据竞争安排好了，它们是线程安全的。下面，我们来看一下，怎么将一个类型变成线程安全的。
首先，看以下代码：
```Rust
    use std::{mem, thread};
    use std::ops::{Deref, DerefMut};

    unsafe impl Send for Ring {}
    unsafe impl Sync for Ring {}
    struct InnerRing {
        capacity: isize,
        size: usize,
        data: *mut Option<u32>,
    }
    #[derive(Clone)]
    struct Ring {
        inner: *mut InnerRing,
    }
```
我们有一个Ring数据结构，或者可以叫做环形缓冲区，InnerRing有一个raw mut pointer原始可变指针，所以它不是线程安全的。但是，我们向Rust保证我们知道正在干什么，并且在Ring上实现了Sync和Send。为什么不在InnerRing上实现呢？当我们在多线程下操作内存中的一个对象时，这个对象的地址必须是固定的。InnerRing，和它所含的数据，必须占据内存中一个稳定的区域。Ring会被四处传送，当然，它可以这样，至少会从创建的线程传递给工作线程。现在，InnerRing中的data呢？它是一个指向一段连续内存块的指针，这块内存被我们开辟出来存放我们的环形缓冲区。在这本书写作的阶段，Rust没有稳定的分配器接口，为了得到连续的内存分配，我们不得不这样做，将一个Vec\<u32>脱去外壳，转变为它的指针。
```Rust
    impl Ring {
        fn with_capacity(capacity: usize) -> Ring {
            let mut data: Vec<Option<u32>> = Vec::with_capacity(capacity);
            for _ in 0..capacity {
                data.push(None);
            }
            let raw_data = (&mut data).as_mut_ptr();
            mem::forget(data);
            let inner_ring = Box::new(InnerRing {
                capacity: capacity as isize,
                size: 0,
                data: raw_data,
            });

            Ring {
                inner: Box::into_raw(inner_ring),
            }
        }
    }
```
Ring::with_capacity函数与Rust生态的其他类型的with_capacity函数十分相像，足够的内存被分配来满足容量的需求。在我们的示例中，我们借助Vec::with_capacity，确保为Option\<u32>分配足够大的空间，初始化为None。如果你回想起来第三章的内容，这样做是因为Vec是懒分配的，只有当我们使用分配地址时，它才会实际分配出来空间。Vec::as_mut_ptr返回一个原始指针指向一个切片，但是不会消耗掉原来的对象（不会是转移语义）。当数据从当前作用域出来时，被分配的块（内存）必须仍然存在，保证不会被回收。标准库的std::mem::forget函数是用来应对这种状况的理想方法。现在分配的内存安全了，一个InnerRing被封装来存储它，然后这个Box就因为Box::into_raw而被消耗掉，传递给Ring，Ring被返回，完成分配。

与一个有着内部原始指针的类型交互是很冗长、麻烦的，因为这需要很多零星分布的unsafe块。为达到这个目的，Ring实现了Deref和DrefMut，它们整理了与Ring的交互方式：
```Rust
impl Deref for Ring {
    type Target = InnerRing;

    fn deref(&self) -> &InnerRing {
        unsafe { &*self.inner }
    }
}

impl DerefMut for Ring {
    fn deref_mut(&mut self) -> &mut InnerRing {
        unsafe { &mut *self.inner }
    }
}
```
这样unsafe块表面上数量就会减少很多，并且这两个方法可以不标记为unsafe。
既然我们已经定义好了Ring，现在我们可以切入关于程序的正题了。我们会定义两个操作，writer和reader，这两个操作会并发进行。writer会增加一个u32值到Ring中，只要当前有空间可以存储。reader会在writer之后读取写入的数。
如下：
```Rust
    fn writer(mut ring: Ring) -> () {
        let mut offset: isize = 0;
        let mut cur: u32 = 0;
        loop {
            unsafe {
                if (*ring).size != ((*ring).capacity as usize) {
                    *(*ring).data.offset(offset) = Some(cur);
                    (*ring).size += 1;
                    cur = cur.wrapping_add(1);
                    offset += 1;
                    offset %= (*ring).capacity;
                } else {
                    thread::yield_now();
                }
            }
        }
    }
    
    fn reader(mut ring: Ring) -> () {
        let mut offset: isize = 0;
        let mut cur: u32 = 0;
        while cur < 1_000 {
            unsafe {
                if let Some(num) = mem::replace(&mut *(*ring).data.offset(offset), None) {
                    assert_eq!(num, cur);
                    (*ring).size -= 1;
                    cur = cur.wrapping_add(1);
                    offset += 1;
                    offset %= (*ring).capacity;
                } else {
                    thread::yield_now();
                }
            }
        }
    }
```
witer是一个快速的循环，它只检查一个简单的条件，并不断循环，这是非常消耗CPU资源和电能的thread::yield_now暗示操作系统，这个线程现在没有工作，应该降低优先级。

reader和writer很相似，除了它不是无限循环外（这样，主线程可以只等待reader线程结束，不必关心writer线程）。

最后，是主线程：
```Rust
    fn main() {
        let capacity = 10;
        let ring = Ring::with_capacity(capacity);

        let reader_ring = ring.clone();
        let reader_jh = thread::spawn(move || {
            reader(reader_ring);
        });
        let _writer_jh = thread::spawn(move || {
            writer(ring);
        });

        reader_jh.join().unwrap();
    }
```
由于使用了move闭包，我们clone了ring，将克隆后的值给闭包捕获到。这个程序运行会如何？
```
rustc -C opt-level=3 data_race00.rs && ./data_race00
thread '<unnamed>' panicked at data_race00.rs:85:17:
assertion `left == right` failed
  left: 254
 right: 244
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
thread 'main' panicked at data_race00.rs:111:22:
called `Result::unwrap()` on an `Err` value: Any { .. }
```
## Ring的瑕疵
为什么上个程序运行出错？我们的ring在内存中为一段连续的内存块，并且我们有一些控制变量在外部。下面是在任何读或写发生之前的系统流程图：
```
size: 0
capacity: 5

rdr
 |
|0: None|1:None|2:None|3：None|4：None
 |
wrt
```
下面是writer写下它的第一个值时的流程图：
```
size: 0
capacity: 5

rdr
 |
|0: Some(0)|1:None|2:None|3：None|4：None
            |
            wrt
```
为了做到这点，writer需要加载size和capacity，进行比较，写入对应偏移块的位置，将size相应增大。这些操作都不能保证有顺序，这是因为特定的执行和编译器的重排。
考虑这种情况：
```
size: 5
capacity: 5

rdr
 |
|0: Some(5)|1:Some(6)|2:Some(7)|3:Some(8)|4:Some(9)
 |
wrt
```
writer扫描了两次，准备写入10,reader预期读到的是5。可能出现下面的情况：
```
size: 5
capacity: 5

rdr
 |
|0: Some(10)|1:Some(6)|2:Some(7)|3:Some(8)|4:Some(9)
             |
            wrt
```
这就是上面实例运行错误的原因。

我们可以使用helgrind工具来验证一下：
```
> valgrind --tool=helgrind --history-level=full --log-file="results.txt" ./data_race00

==15547== ----------------------------------------------------------------
==15547== 
==15547== Possible data race during write of size 4 at 0x4AC5160 by thread #3
==15547== Locks held: none
==15547==    at 0x110710: _ZN3std10sys_common9backtrace28__rust_begin_short_backtrace17h2ae8a33cae57ca28E.llvm.6225174379729753874 (in /home/zyc/rust_projects/Hands-On-Concurrency-with-Rust-master/Chapter04/data_races/data_race00)
==15547==    by 0x1117E7: core::ops::function::FnOnce::call_once{{vtable.shim}} (in /home/zyc/rust_projects/Hands-On-Concurrency-with-Rust-master/Chapter04/data_races/data_race00)
==15547==    by 0x1302C4: std::sys::pal::unix::thread::Thread::new::thread_start (boxed.rs:2016)
==15547==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==15547==    by 0x492EAC2: start_thread (pthread_create.c:442)
==15547==    by 0x49BFA03: clone (clone.S:100)
==15547== 
==15547== This conflicts with a previous write of size 4 by thread #2
==15547== Locks held: none
==15547==    at 0x1107EC: _ZN3std10sys_common9backtrace28__rust_begin_short_backtrace17h93320721e04667faE.llvm.6225174379729753874 (in /home/zyc/rust_projects/Hands-On-Concurrency-with-Rust-master/Chapter04/data_races/data_race00)
==15547==    by 0x111997: core::ops::function::FnOnce::call_once{{vtable.shim}} (in /home/zyc/rust_projects/Hands-On-Concurrency-with-Rust-master/Chapter04/data_races/data_race00)
==15547==    by 0x1302C4: std::sys::pal::unix::thread::Thread::new::thread_start (boxed.rs:2016)
==15547==    by 0x485396A: ??? (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==15547==    by 0x492EAC2: start_thread (pthread_create.c:442)
==15547==    by 0x49BFA03: clone (clone.S:100)
==15547==  Address 0x4ac5160 is 0 bytes inside a block of size 80 alloc'd
==15547==    at 0x484A919: malloc (in /usr/libexec/valgrind/vgpreload_helgrind-amd64-linux.so)
==15547==    by 0x110E41: data_race00::main (in /home/zyc/rust_projects/Hands-On-Concurrency-with-Rust-master/Chapter04/data_races/data_race00)
==15547==    by 0x110792: std::sys_common::backtrace::__rust_begin_short_backtrace (in /home/zyc/rust_projects/Hands-On-Concurrency-with-Rust-master/Chapter04/data_races/data_race00)
==15547==    by 0x110A28: std::rt::lang_start::{{closure}} (in /home/zyc/rust_projects/Hands-On-Concurrency-with-Rust-master/Chapter04/data_races/data_race00)
==15547==    by 0x128850: std::rt::lang_start_internal (function.rs:284)
==15547==    by 0x110FC4: main (in /home/zyc/rust_projects/Hands-On-Concurrency-with-Rust-master/Chapter04/data_races/data_race00)
==15547==  Block was alloc'd by thread #1
==15547== 
==15547== ----------------------------------------------------------------

```
可以看到，helgrind警告我们存在数据竞争和data stomping问题。

让我们来改善这一情况。显然，我们在reader和writer的竞争上存在问题，但是我们在writer的行为上也存在问题。writer完全没有意识到自己已经写过头了。经过简单的调整，我们可以阻止这种行为（验证需要写入的offset上必须是None）：
```Rust
    fn writer(mut ring: Ring) -> () {
        let mut offset: isize = 0;
        let mut cur: u32 = 0;
        loop {
            unsafe {
                if (*ring).size != ((*ring).capacity as usize) {
                    assert!(mem::replace(&mut *(*ring).data.offset(offset), Some(cur)).is_none());
                    (*ring).size += 1;
                    cur = cur.wrapping_add(1);
                    offset += 1;
                    offset %= (*ring).capacity;
                } else {
                    thread::yield_now();
                }
            }
        }
    }
```
运行后得到：
```
rustc data_race01.rs && ./data_race01
thread 'thread '<unnamed>' panicked at <unnamed>' panicked at data_race01.rsdata_race01.rs::65:1786:
:assertion failed: mem::replace(&mut *(*ring).data.offset(offset), Some(cur)).is_none()17
:
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
attempt to subtract with overflow
thread 'main' panicked at data_race01.rs:111:22:
called `Result::unwrap()` on an `Err` value: Any { .. }
```
显然，writer写入的位置上不是None，而经过判断时，size!=capacity。

## 回到安全的代码
之前讨论的例子，是底层且不安全的。我们如何使用Rust提供的类型来建造相同的实例？
```Rust
    use std::{mem, thread};
    use std::sync::{Arc, Mutex};

    struct Ring {
        size: usize,
        data: Vec<Option<u32>>,
    }

    impl Ring {
        fn with_capacity(capacity: usize) -> Ring {
            let mut data: Vec<Option<u32>> = Vec::with_capacity(capacity);
            for _ in 0..capacity {
                data.push(None);
            }
            Ring {
                size: 0,
                data: data,
            }
        }

        fn capacity(&self) -> usize {
            self.data.capacity()
        }

        fn is_full(&self) -> bool {
            self.size == self.data.capacity()
        }

        fn emplace(&mut self, offset: usize, val: u32) -> Option<u32> {
            self.size += 1;
            let res = mem::replace(&mut self.data[offset], Some(val));
            res
        }

        fn displace(&mut self, offset: usize) -> Option<u32> {
            let res = mem::replace(&mut self.data[offset], None);
            if res.is_some() {
                self.size -= 1;
            }
            res
        }
    }

    fn writer(ring_lk: Arc<Mutex<Ring>>) -> () {
        let mut offset: usize = 0;
        let mut cur: u32 = 0;
        loop {
            let mut ring = ring_lk.lock().unwrap();
            if !ring.is_full() {
                assert!(ring.emplace(offset, cur).is_none());
                cur = cur.wrapping_add(1);
                offset += 1;
                offset %= ring.capacity();
            } else {
                thread::yield_now();
            }
        }
    }

    fn reader(read_limit: usize, ring_lk: Arc<Mutex<Ring>>) -> () {
        let mut offset: usize = 0;
        let mut cur: u32 = 0;
        while (cur as usize) < read_limit {
            let mut ring = ring_lk.lock().unwrap();
            if let Some(num) = ring.displace(offset) {
                assert_eq!(num, cur);
                cur = cur.wrapping_add(1);
                offset += 1;
                offset %= ring.capacity();
            } else {
                drop(ring);
                thread::yield_now();
            }
        }
    }

    fn main() {
        let capacity = 10;
        let read_limit = 1_000_000;
        let ring = Arc::new(Mutex::new(Ring::with_capacity(capacity)));

        let reader_ring = Arc::clone(&ring);
        let reader_jh = thread::spawn(move || {
            reader(read_limit, reader_ring);
        });
        let _writer_jh = thread::spawn(move || {
            writer(ring);
        });

        reader_jh.join().unwrap();
    }
```
这段代码和之前的不安全代码是大体上一致的。之前，可以看到，Ring:!Send，Ring在线程之间传送是不安全的。这次，reader和writer的操作是基于Arc\<Mutex\<Ring>>。Arc\<Mutex\<T>>是多线程之间保证安全的经典组合。所以以上代码会轻松通过编译，正常运行。

## 独占实现安全访问
当我们谈论Mutex时，Mutexes provide MUTual EXclusion among threads。任何线程在一个Mutex调用lock会获取锁，或者被阻塞直到其他持有这个锁的线程进行解锁。lock的返回类型是 lock(&self) -> LockResult\<MutexGuard\<T>>，

pub type LockResult\<Guard> = Result\<Guard, PoisonError\<Guard>>;

这意味着，线程获取锁会返回一个Result，这个Result成功时包含MutexGuard\<T>，失败时包含的是一个中毒的提示。Rust在锁的持有者崩溃时，会设置锁为中毒的状态，这是一种帮助阻止多线程程序只有一个线程崩溃时传播多个线程的策略。因为这个原因，你会发现许多Rust程序不会检查lock的返回值，而是直接进行unwrap（因为只要返回，就不会是当前线程出现问题）。

当MutexGuard\<T>被drop时，会释放锁。一旦Mutex guard是非上锁状态，就不能在通过它获取其中的数据了，所以，这里没有线程之间进行通信的方式。

Mutex: Send+Sync。这已经很完善了，为什么我们还需要将其包括进入Arc？Arc\<T>是原子引用计数指针。Rc\<T>是非线程安全的，因为在其中的强/弱引用计数器上有竞争。Arc\<T>是建立在Atomic\<Integer>上的，这会在之后的章节介绍。不管怎么说，Arc\<T>能够作为引用计数指针而在多线程状态下不会引发数据竞争（因为计数器是原子操作）。

为什么使用Arc\<Mutex\<Ring>>？Mutex\<Ring>可以移动但是不能被克隆。看一下Mutex：
```Rust
    pub struct Mutex<T: ?Sized> {
        inner: sys::Mutex,
        poison: poison::Flag,
        data: UnsafeCell<T>,
    }
```
我们看到T可以实现Sized或者!Sized，T被存储在UnsafeCell\<T>，其与sys::Mutex一起存储。每个Rust运行的平台会提供它自己对Mutex的实现，通常与操作系统的环境紧绑在一起。在Rust源码中,你可以找到sys::Mutex包裹了依赖系统实现的Mutex。Unix对Mutex的实现在src/libstd/sys/unix/mutex.rs中，如下：
```Rust
    pub struct Mutex{
        inner: UnsafeCell<libc::pthread_mutex_t>,
    }
```
但是Mutex并非必须在系统基础上实现，后续章节我们会看到当实现我们的lcok时，会使用atomic primitives。

当我们clone Mutex时，会发生什么？我们会需要新的system mutex的分配地址，如果T本身是可以克隆的话，还需要新的UnsafeCell。这个新的system mutex是问题所在，线程必须在内存中同样的数据结构上进行同步。将Mutex丢到Arc中可以解决这个问题。克隆Arc，就像克隆Rc一样，创建一个新的对Mutex的强引用。这些新的引用是不可变引用。
在Rust提供的抽象模型中，Mutex本身没有内部状态。Rust的Mutex使用内部UnsafeCell提供的内部可变性。Mutex本身是不可变的，而内部T通过MutexGuard可变地引用。这在Rust的内存模型中是安全的，因为由于互斥，在任何给定时间都只有一个对T的可变引用。

经过多次运行，程序本身也能顺利运行完成。不幸的是，从理论上讲，它并不是特别有效。对Mutex上锁是有开销的，当我们线程的操作简短时，一个线程持有锁，其他的都在愚蠢地等待。

使用perf可以证明这一点：
```
> perf stat --event task-clock,context-switches,page-faults,cycles,instructions,branches,branch-misses,cache-references,cache-misses ./data_race02

 Performance counter stats for './data_race02':

          4,694.34 msec task-clock                       #    1.696 CPUs utilized
           190,739      context-switches                 #   40.632 K/sec
               111      page-faults                      #   23.646 /sec
    16,716,392,784      cycles                           #    3.561 GHz                         (83.54%)
    12,370,949,135      instructions                     #    0.74  insn per cycle              (83.47%)
     2,646,307,332      branches                         #  563.723 M/sec                       (83.60%)
       302,586,307      branch-misses                    #   11.43% of all branches             (83.00%)
       172,326,200      cache-references                 #   36.709 M/sec                       (83.24%)
        15,770,640      cache-misses                     #    9.15% of all cache refs           (83.16%)

       2.767760772 seconds time elapsed

       1.285505000 seconds user
       3.246514000 seconds sys
```
## 使用MPSC
Ring究竟做了什么？首先，它有一个固定的容量和一对reader/writer。其次，Ring是一个数据结构，旨在通过两个或多个线程来操作，这些线程有两个动作：reading和writing。Ring是一种在线程之间传递u32,按照写入的顺序读取数据的一种手段。

庆幸的是，只要我们能够接受只有一个reader线程存在的情况，Rust可以使用标准库的一些手段来满足这个需求，就是std::sync::mpsc。The Multi Producer Single Consumer queue中，writer端叫做Sender\<T>，可以被克隆并且转移到多线程中;reader端，叫做Revceier\<T>，只能被转移，不能被克隆。这里有两种Sender变体可以使用，Sender\<T>和SyncSender\<T>。前者代表无限的MPSC，意味着这个通道需要多少内存就会开辟多少内存来存储到来的T数据。后者表示有限的MPSC，说明通道有一个固定的上限。

Rust文档描述Sender\<T>和SyncSender\<T>分别是异步和同步的，严格上来说，不是这样的。SyncSender\<T>::send当管道没有空间可用时，会阻塞，但是，还有一个方法SyncSender<T>::try_send，其中try_send(&self, t: T) -> Result\<(), TrySendError\<T>>，不会阻塞。在有限的内存中使用Rust的MPSC是可能的，请记住，调用者必须有一个策略来处理由于缺少放置空间而被拒绝的输入。

使用MPSC改写我们的程序如下：
```Rust
    use std::thread;
    use std::sync::mpsc;

    fn writer(chan: mpsc::SyncSender<u32>) -> () {
        let mut cur: u32 = 0;
        while let Ok(()) = chan.send(cur) {
            cur = cur.wrapping_add(1);
        }
    }

    fn reader(read_limit: usize, chan: mpsc::Receiver<u32>) -> () {
        let mut cur: u32 = 0;
        while (cur as usize) < read_limit {
            let num = chan.recv().unwrap();
            assert_eq!(num, cur);
            cur = cur.wrapping_add(1);
        }
    }

    fn main() {
        let capacity = 10;
        let read_limit = 1_000_000;
        let (snd, rcv) = mpsc::sync_channel(capacity);

        let reader_jh = thread::spawn(move || {
            reader(read_limit, rcv);
        });
        let _writer_jh = thread::spawn(move || {
            writer(snd);
        });

        reader_jh.join().unwrap();
    }
```
代码变得简短并且一点也不容易出现算术错误。Send类型SyncSender是send(&self, t: T) -> Result<(), SendError<T>>，意味着需要关注SendError来防止程序崩溃。SendError只是当远程的MPSC通道端关闭时返回的值，即在Reader达到read_limit时发生。使用perf来探测一下这个程序的性能：
```
Performance counter stats for './data_race03':

          1,144.71 msec task-clock                       #    1.207 CPUs utilized
           131,725      context-switches                 #  115.073 K/sec
               127      page-faults                      #  110.945 /sec
     2,505,925,283      cycles                           #    2.189 GHz                         (82.64%)
     2,125,401,811      instructions                     #    0.85  insn per cycle              (84.58%)
       441,429,077      branches                         #  385.626 M/sec                       (81.78%)
        30,015,550      branch-misses                    #    6.80% of all branches             (84.09%)
       206,544,370      cache-references                 #  180.434 M/sec                       (83.01%)
         4,919,336      cache-misses                     #    2.38% of all cache refs           (83.92%)

       0.948122293 seconds time elapsed

       0.165672000 seconds user
       0.882092000 seconds sys
```
可以看到标准库的实现，程序的性能远远优于我们的实现。

## 实现一个遥测服务器

***此处建议看英文版，因为译者也不知道这个server有什么特殊作用***

让我们构建一个描述性统计服务器。这经常出现在下面的情况：需要一个东西来消耗事件，对这些东西进行某种描述性统计计算，然后将描述多路复用到其他系统。

首先，Cargo.toml：
```toml
    [package]
    name = "telem"
    version = "0.1.0"

    [dependencies]
    quantiles = "0.7"
    seahash = "3.0"

    [[bin]]
    name = "telem"
    doc = false
```
我们需要，quantiles和seahash。实际执行文件在src/bin/telem.rs，这个项目是库和执行文件分离的构造。
在src/bin/telem.rs中：
```Rust
    extern crate telem;

    use std::{thread, time};
    use std::sync::mpsc;
    use telem::IngestPoint;
    use telem::egress::{CKMSEgress, CMAEgress, Egress};
    use telem::event::Event;
    use telem::filter::{Filter, HighFilter, LowFilter};

    fn main() {
        let limit = 100;
        let (lp_ic_snd, lp_ic_rcv) = mpsc::channel::<Event>();
        let (hp_ic_snd, hp_ic_rcv) = mpsc::channel::<Event>();
        let (ckms_snd, ckms_rcv) = mpsc::channel::<Event>();
        let (cma_snd, cma_rcv) = mpsc::channel::<Event>();

        let filter_sends = vec![lp_ic_snd, hp_ic_snd];
        let ingest_filter_sends = filter_sends.clone();
        let _ingest_jh = thread::spawn(move || {
            IngestPoint::init("127.0.0.1".to_string(), 1990, ingest_filter_sends).run();
        });
        let _low_jh = thread::spawn(move || {
            let mut low_filter = LowFilter::new(limit);
            low_filter.run(lp_ic_rcv, vec![ckms_snd]);
        });
        let _high_jh = thread::spawn(move || {
            let mut high_filter = HighFilter::new(limit);
            high_filter.run(hp_ic_rcv, vec![cma_snd]);
        });
        let _ckms_egress_jh = thread::spawn(move || {
            CKMSEgress::new(0.01).run(ckms_rcv);
        });
        let _cma_egress_jh = thread::spawn(move || {
            CMAEgress::new().run(cma_rcv);
        });

        let one_second = time::Duration::from_millis(1_000);
        loop {
            for snd in &filter_sends {
                snd.send(Event::Flush).unwrap();
            }
            thread::sleep(one_second);
        }
    }
```
主线程建立了工作线程，将合适的管道读\写端给予它们。有些线程有多个发送端。这就是我们使用Rust MPSC进行输出的方式。

下面是IngestPoint：在src/ingest_point.rs中
```Rust
use event;
use std::{net, thread};
use std::net::ToSocketAddrs;
use std::str;
use std::str::FromStr;
use std::sync::mpsc;
use util;

pub struct IngestPoint {
    host: String,
    port: u16,
    chans: Vec<mpsc::Sender<event::Event>>,
}

impl IngestPoint {
    pub fn init(
        host: String,
        port: u16,
        chans: Vec<mpsc::Sender<event::Event>>,
    ) -> IngestPoint {
        IngestPoint {
            chans: chans,
            host: host,
            port: port,
        }
    }

    pub fn run(&mut self) {
        let mut joins = Vec::new();

        let addrs = (self.host.as_str(), self.port).to_socket_addrs();
        if let Ok(ips) = addrs {
            let ips: Vec<_> = ips.collect();
            for addr in ips {
                let listener =
                    net::UdpSocket::bind(addr).expect("Unable to bind to UDP socket");
                let chans = self.chans.clone();
                joins.push(thread::spawn(move || handle_udp(chans, &listener)));
            }
        }

        for jh in joins {
            jh.join().expect("Uh oh, child thread panicked!");
        }
    }
}

fn parse_packet(buf: &str) -> Option<event::Telemetry> {
    let mut iter = buf.split_whitespace();
    if let Some(name) = iter.next() {
        if let Some(val) = iter.next() {
            match u32::from_str(val) {
                Ok(int) => {
                    return Some(event::Telemetry {
                        name: name.to_string(),
                        value: int,
                    })
                }
                Err(_) => return None,
            };
        }
    }
    None
}

fn handle_udp(mut chans: Vec<mpsc::Sender<event::Event>>, socket: &net::UdpSocket) {
    let mut buf = vec![0; 16_250];
    loop {
        let (len, _) = match socket.recv_from(&mut buf) {
            Ok(r) => r,
            Err(e) => panic!(format!("Could not read UDP socket with error {:?}", e)),
        };
        if let Some(telem) = parse_packet(str::from_utf8(&buf[..len]).unwrap()) {
            util::send(&mut chans, event::Event::Telemetry(telem));
        }
    }
}
```
IngestPoint是一个主机名，可以是IP地址或者是DNS域名，每个ToSocketAddrs，含有一个端口和装有mpsc::Sender\<event::Event>的Vector。Event是我们自己定义的：
```Rust
#[derive(Clone)]
pub enum Event {
    Telemetry(Telemetry),
    Flush,
}

#[derive(Debug, Clone)]
pub struct Telemetry {
    pub name: String,
    pub value: u32,
}
```
telem项目含有两种事件：一种是Telemetry，可能来自IngresPoint，另一种是Flush。它们由主线程发出，Flush就像系统的时钟一样，允许项目的各个子系统跟踪系统时间。

回到IngestPoint中：
init方法只是进行初始化，run函数调用to_socket_addrs，检索所有关联的IP地址。这些地址中的每一个都有一个绑定到它们的UdpSocket和一个OS线程来侦听来自该套接字的数据报。最重要的方法是handle_udp，这个函数是无限循环，不断将数据报pull到16KB的缓冲区中，然后在数据报上调用parse_packet。如果这是有效的数据报，我们就调用util::send方法将Event::Telemetry发送到管道中。util::send如下：
```Rust
pub fn send(chans: &[mpsc::Sender<event::Event>], event: event::Event) {
    if chans.is_empty() {
        return;
    }

    for chan in chans.iter() {
        chan.send(event.clone()).unwrap();
    }
}
```
下面是识别数据的方法：可以看出要识别的数据非常简单，只有一个字符串和一个数u32
```Rust
fn parse_packet(buf: &str) -> Option<event::Telemetry> {
    let mut iter = buf.split_whitespace();
    if let Some(name) = iter.next() {
        if let Some(val) = iter.next() {
            match u32::from_str(val) {
                Ok(int) => {
                    return Some(event::Telemetry {
                        name: name.to_string(),
                        value: int,
                    })
                }
                Err(_) => return None,
            };
        }
    }
    None
}
```
至于filter，HighFilter和LowFilter都实现了Filter的trait：在src/filter/mod.rs
```Rust
use event;
use std::sync::mpsc;
use util;

mod high_filter;
mod low_filter;

pub use self::high_filter::*;
pub use self::low_filter::*;

pub trait Filter {
    fn process(
        &mut self,
        event: event::Telemetry,
        res: &mut Vec<event::Telemetry>,
    ) -> ();

    fn run(
        &mut self,
        recv: mpsc::Receiver<event::Event>,
        chans: Vec<mpsc::Sender<event::Event>>,
    ) {
        let mut telems = Vec::with_capacity(64);
        for event in recv.into_iter() {
            match event {
                event::Event::Flush => util::send(&chans, event::Event::Flush),
                event::Event::Telemetry(telem) => {
                    self.process(telem, &mut telems);
                    for telem in telems.drain(..) {
                        util::send(&chans, event::Event::Telemetry(telem))
                    }
                }
            }
        }
    }
}
```
下面是src/filter/low_filter.rs:
```Rust
use event;
use filter::Filter;

pub struct LowFilter {
    limit: u32,
}

impl LowFilter {
    pub fn new(limit: u32) -> Self {
        LowFilter { limit: limit }
    }
}

impl Filter for LowFilter {
    fn process(
        &mut self,
        event: event::Telemetry,
        res: &mut Vec<event::Telemetry>,
    ) -> () {
        if event.value <= self.limit {
            res.push(event);
        }
    }
}
```
下面是src/filter/high_filter.rs:
```Rust
use event;
use filter::Filter;

pub struct HighFilter {
    limit: u32,
}

impl HighFilter {
    pub fn new(limit: u32) -> Self {
        HighFilter { limit: limit }
    }
}

impl Filter for HighFilter {
    fn process(
        &mut self,
        event: event::Telemetry,
        res: &mut Vec<event::Telemetry>,
    ) -> () {
        if event.value >= self.limit {
            res.push(event);
        }
    }
}
```
可见，过滤器是判断所来的信息是不是Telemetry，查看当前的limit值，到来的信息的data字段满足条件才会将信息加入到管道中，否则过滤掉。

Egress定义和Filter很像，在src/egress/mod.rs中：
```Rust
use event;
use std::sync::mpsc;

mod cma_egress;
mod ckms_egress;

pub use self::ckms_egress::*;
pub use self::cma_egress::*;

pub trait Egress {
    fn deliver(&mut self, event: event::Telemetry) -> ();

    fn report(&mut self) -> ();

    fn run(&mut self, recv: mpsc::Receiver<event::Event>) {
        for event in recv.into_iter() {
            match event {
                event::Event::Telemetry(telem) => self.deliver(telem),
                event::Event::Flush => self.report(),
            }
        }
    }
}
```
若到来信息是Flush，则当前对象调用report方法进行一些输出，若是Telemetry，则调用deliver方法将Telemetry存储a起来。

下面是ckms_egress，cma_egress与它很像：
```Rust
use egress::Egress;
use event;
use quantiles;
use util;

pub struct CKMSEgress {
    error: f64,
    data: util::HashMap<String, quantiles::ckms::CKMS<u32>>,
    new_data_since_last_report: bool,
}

impl Egress for CKMSEgress {
    fn deliver(&mut self, event: event::Telemetry) -> () {
        self.new_data_since_last_report = true;
        let val = event.value;
        let ckms = self.data
            .entry(event.name)
            .or_insert(quantiles::ckms::CKMS::new(self.error));
        ckms.insert(val);
    }

    fn report(&mut self) -> () {
        if self.new_data_since_last_report {
            for (k, v) in &self.data {
                for q in &[0.0, 0.25, 0.5, 0.75, 0.9, 0.99] {
                    println!("[CKMS] {} {}:{}", k, q, v.query(*q).unwrap().1);
                }
            }
            self.new_data_since_last_report = false;
        }
    }
}

impl CKMSEgress {
    pub fn new(error: f64) -> Self {
        CKMSEgress {
            error: error,
            data: Default::default(),
            new_data_since_last_report: false,
        }
    }
}
```
实验一下：
```
> cargo run --release
```
在另外一个终端窗口上，发送UDP包。在MacOS或Linux，你可以使用nc。下面是在linux上使用
```
> echo "a 10" | nc -u 127.0.0.1 1990
> echo "a 55" | nc -u 127.0.0.1 1990
```
结果：
```
[CKMS] a 0:10
[CKMS] a 0.25:10
[CKMS] a 0.5:10
[CKMS] a 0.75:10
[CKMS] a 0.9:10
[CKMS] a 0.99:10
[CKMS] a 0:10
[CKMS] a 0.25:10
[CKMS] a 0.5:10
[CKMS] a 0.75:55
[CKMS] a 0.9:55
[CKMS] a 0.99:55
```

```
> echo "b 1000" | nc -u 127.0.0.1 1990
> echo "b 2000" | nc -u 127.0.0.1 1990
> echo "b 3000" | nc -u 127.0.0.1 1990
```
结果
```
[CMA] b 1000
[CMA] b 1500
[CMA] b 2000
```