---
title: Java中的retry
date: 2020-10-19 11:38:44
categories: [Java]
tags: [Java]
---
### 概念
最近在看线程池源码的时候看到有关阻塞队列源码中使用了 *retry* ，有点懵逼，查阅资料总结一下。

*retry* 是作用于循环中，用于提前中断循环，可以搭配 *continue* 和 *break* 使用。使用时在循环之前声明 *retry:* ，在需要中断的地方使用 *break retry* 或者 *continue retry*。看起来很像goto。

搭配continue：中断整个嵌套循环循环，从 *retry:* 标记的循环继续执行，注意这里可能有坑，你标记写在第几层循环，就是从哪里开始。

搭配break：中断标记开始的循环，注意标记外层的循环继续执行。

### code
```
    //retry写最外层循环
    public static void testBreak() {
        retry:
        for (int i = 0; i < 3; i++) {
            for (int j = 0; j < 3; j++) {
                for (int k = 0; k < 3; k++) {
                    System.out.println(i + " " + j + " " + k);
                    if (k == 1) {
                        break retry;
                    }
                }
            }
        }
    }
    
    /*
    输出：
    0 0 0
    0 0 1
    */
    
    //retry写第二层循环
    public static void testBreak2() {
//        retry:
        for (int i = 0; i < 3; i++) {
            retry:
            for (int j = 0; j < 3; j++) {
                for (int k = 0; k < 3; k++) {
                    System.out.println(i + " " + j + " " + k);
                    if (k == 1) {
                        break retry;
                    }
                }
            }
        }
    }
    /*
    输出：
    0 0 0
    0 0 1
    1 0 0
    1 0 1
    2 0 0
    2 0 1
    */
    
    //retry写最外层循环
    public static void testContinue1() {
        retry:
        for (int i = 0; i < 5; i++) {
            for (int j = 0; j < 5; j++) {
                for (int k = 0; k < 5; k++) {
                    System.out.println(i + " " + j + " " + k);
                    if (k == 1) {
                        continue retry;
                    }
                }
            }
        }
    }
    /*
    输出：
    0 0 0
    0 0 1
    1 0 0
    1 0 1
    2 0 0
    2 0 1
    */
    
    //retry写第二层循环
    public static void testContinue2() {
//        retry:
        for (int i = 0; i < 3; i++) {
            retry:
            for (int j = 0; j < 3; j++) {
                for (int k = 0; k < 3; k++) {
                    System.out.println(i + " " + j + " " + k);
                    if (k == 1) {
                        continue retry;
                    }
                }
            }
        }
    }
    /*
    输出：
    0 0 0
    0 0 1
    0 1 0
    0 1 1
    0 2 0
    0 2 1
    1 0 0
    1 0 1
    1 1 0
    1 1 1
    1 2 0
    1 2 1
    2 0 0
    2 0 1
    2 1 0
    2 1 1
    2 2 0
    2 2 1
    */
```
