# Sync 和 Send , Rust并发的基础
Rust旨在成为一个无俱并发的编程语言。这是什么意思？它如何工作？在第二章，我们讨论了顺序执行的程序的性能，故意将并行程序的讨论放在一边。在第三章，我们大致了解了Rust掌管内存的方式，特别是关于如何组合成为高性能的数据结构。在本章，我们会继续之前所学的，深入挖掘Rust并发的编程方式。

在本章，我们会：
- [x]  讨论Sync和Send这两个trait
- [x]  使用Helgrind工具在ring数据结构中检查多线程竞争 
- [x]  使用锁Mutex来解决竞争
- [x]  使用标准库的MPSC
- [x]  建立一个有意义的数据复用项目

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


