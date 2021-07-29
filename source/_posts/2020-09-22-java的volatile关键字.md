---
title: java的volatile关键字
date: 2020-09-22 10:20:41
categories: [java]
tags: [java]
---
volatile保证可见行，有序性，不保证原子性。
<br/></br>
涉及的计算机知识：为了使得程序能够高速的运行，cpu会将内存中的值复制一份到高速缓冲区（Cache），后续直接使用Cache中的值。这种做法在单核cpu是ok，在多核cpu涉及多线程的操作时，就可能会出现数据不一致的情况。解决办法有：
</br></br>
（1）在总线上加LOCK#
</br>
（2）使用MESI缓存一致性协议
</br></br>
两种办法都是在硬件层面实现。早期的时候是使用第一种方法来解决，通过在总线上加锁，阻塞其他cpu对其他硬件的访问（如内存），直到当前cpu使用后释放锁，这样就保证了数据的一致性。但是这样的操作会出现同一时刻只有一个cpu访问内存，导致程序执行效率大大降低。
</br></br>
于是Intel推出了MESI缓存一致性协议。缓存一致性协议保证了所有cpu中的变量副本都是一致的，具体的实现是cpu写数据的时候，如果发现该变量是共享变量，会发出信号通知其他cpu将该变量置为失效，当其他cpu再次访问该变量时，发现变量失效了就会进入内存中获取最新的值。
</br></br>
JMM（java memory model）中规定所有的变量都是存在于主存，每个线程都有自己的工作内存（类似cpu中的Cache），线程只能操作自己的工作内存，不能访问和操作其他线程的工作内存。如果线程操作一个非共享变量，首先将变量从内存中拷贝到cpu的Cache中再进行操作。volatile实际上通过将变量标记为共享变量，被标记为共享的变量一旦被修改，就会强制更新内存，其他cpu在总线上通过嗅探这个变量的地址被改变就会将缓存该变量的缓存行设为无效，下次其他线程就会从内存中重新加载这个变量，这样就保证了数据的一致性。
</br></br>
先看一段代码
```
    public static boolean flag = false;

    public static void main(String[] args) {
        Thread t = new Thread(() -> test());
        t.start();
        while (!flag) {
        }
    }

    public static void test() {
        try {
            //停止100毫秒是为了保证子线程在主线程之后执行
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        flag = true;
    }
```
上面的代码会进入死循环，新建线程对flag的修改对主线程并不可见，对flag用volatile修饰则能够解决死循环。
</br></br>
volatile解决有序性：volatile会禁止指令重排序，通过类似内存屏障的方式，在变量赋值后进行load addl $0x0, (%esp)操作。指令重排序的时候不能将后面的指令重排序到内存屏障之前的位置。
</br></br>
拓展的相关知识：false sharing（有关MESI），synchronized，lock。
