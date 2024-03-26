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
