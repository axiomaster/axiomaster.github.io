---
layout:     post
title:      Java并发编程
subtitle:   原子性、可见性与有序性
date:       2019-04-27
author:     axiomaster
header-img: img/post-bg-desk.jpg
catalog: true
tags:
    - Java
    - 并发
---

# 缓存一致性与并发问题

我们今天使用的计算机体系结构都属于存储程序型（冯·诺伊曼结构），运行时所有的数据都存储在内存中；指令执行时，数据从内存读入寄存器，运算结束后再写回内存。

```java
// i == 0
i = i + 1
// i == 1
```

以上面代码为例，计算机先将```i```加载到寄存器中，再寄存器中```+1```后再将```i```写回内存。

可以认为运算中，寄存器中的数据是内存的一份缓存。但是，有缓存，就有缓存不一致的问题。

我们只考虑多CPU时的并发问题。单线程执行时，以上执行顺序不存在问题；但并发场景下，假设2个线程A、B同时执行，A读入寄存器的值均为```0```, 计算后都为```1```，在A写入内存前，B此时读取到的值为```0```，最终写入内存的```i```为```1```，显然存在问题。

解决缓存不一致问题，通常有2种方法：

1. 在总线加锁；
2. 使用缓存一致性协议；

## 总线加锁

CPU通过总线读取内存，通过在总线加锁，锁未释放前，CPU只能再次读取```i```所在内存，直到锁释放。早期CPU采用这种方式，这样可以解决不一致问题，但这等于同一块内存锁释放前无法访问。

## 缓存一致性协议

Intel的MESI协议，提供了如下机制：CPU写数据时，如果发现变量是共享变量，会发信号通知其它CPU将该变量的缓存置为无效状态。其它CPU操作时发现缓存失效了，就会重新从内存读取。

# 并发编程中的概念

## 原子性

提到原子性时都会以银行转账为例：用户A向用户B转账100元。

转账事务请求中，银行会进行2项操作：A账户减少100元；B账户增加100元。

不管银行按什么顺序执行，如果2项操作进行到一半时失败了，银行都需要先进行回滚，否则就会出现错误（银行损失或用户A损失）。

换句话说，银行需要保障转账的原子性：这项请求要么成功，要么失败，不能出现任何中间状态（2个操作一个成功，一个失败）。

对应到计算机运行中，一些操作是原子性的，另一些操作是由若干原子性操作组合而成。

比如根据上面分析，```i=i+1```不是原子操作；那```i=9```是原子操作吗？如果计算机分别对```i```的高低16位先后赋值，那就不是原子操作。

## 可见性

可见性是指多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

比如：
```java
// 线程A
int i = 0
i = 5

// 线程B
j = i
```

如果线程A执行完```i = 5```但尚未写入内存时，线程B从内存看到的```i = 0```，无法看到线程A缓存中```i```的值已经发生变化。

## 有序性

以下代码示例来自《七周七并发》

```java
public class Puzzle {

    static boolean answerReady = false;
    static int answer = 0;
    static Thread t1 = new Thread() {
        @Override
        public void run() {
            answer = 42;
            answerReady = true;
        }
    };

    static Thread t2 = new Thread() {
        @Override
        public void run() {
            if (answerReady) {
                System.out.println("The meaning of life is: " + answer);
            } else {
                System.out.println("I don't know the answer.");
            }
        }
    };

    public static void main(String[] args) throws InterruptedException {
        t1.start();
        t2.start();

        t1.join();
        t2.join();
    }
}

```

运行结果除了```"The meaning of life is 42"``` 和 ```"I don't know the answer."```之外，代码还可能输出"```"The meaning of life is 0"```, What the fxxk！

原因是t1在执行中，编译器静态优化，JVM运行时优化，甚至硬件会进行指令重排序已提升性能。

指令重排序可能会打乱原语句执行顺序，但它会保证最终结果正确。

```java
int a = 1; // s1
int r = 2; // s2
a = a + 1; // s3
r = a * a; // s4
```
以上示例中，s1可能和s2调整顺序，但s1, s3, s4一定保序。

指令排序对单线程有效，但对并发执行可能失效。回到```Puzzle```, t1中```answer```与```answerReady```并无依赖关系，但t1并发执行时，就存在相互依赖关系。

