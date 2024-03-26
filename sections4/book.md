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
