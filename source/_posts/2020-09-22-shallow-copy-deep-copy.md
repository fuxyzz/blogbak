---
title: shallow copy&deep copy
date: 2020-09-22 09:24:23
categories: [java]
tags: [java]
---
#### 拷贝前提
拷贝需要实现Cloneable接口，实现clone方法，基本是调用super.clone()方法（会调用Object的clone方法，这个方法是native，由jvm进行调用）。
#### shallow copy（浅拷贝）
直接进行拷贝对象的成员变量，对基本数据类型不影响，对对象类型有影响（直接拷贝了对象引用，拷贝后操作的成员变量是同一个对象引用，可能出现与期望不一致的情况，类似Context）。

#### deep copy（深拷贝）
对对象的每一层进行对象拷贝，拷贝后操作的成员变量不是同一个对象引用。

（1）可以在实现clone方法的时候自行实现所有关联成员变量的拷贝，如果对象的层级很多需要实现大量的成员变量的clone方法。
（2）可以使用序列化和反序列化的方式进行，用到IO，可能会慢一点。需要注意的是被transient修饰的成员无法被拷贝。这种方式需要实现Serializable接口，不需要实现Cloneable接口和clone方法。

实现Serializable的code
```
import java.io.*;

public class Main {

    public static volatile boolean flag = false;
    static ReentrantLock lock = new ReentrantLock();

    public static void main(String[] args) throws CloneNotSupportedException, IOException, ClassNotFoundException {
        Test a = new Test();
        Sub er = new Sub();
        er.setB(10);
        a.setEr(er);
//        Per b = (Per) a.clone();
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bos);
        oos.writeObject(a);
        oos.flush();
        ObjectInputStream ois = new ObjectInputStream(new ByteArrayInputStream(bos.toByteArray()));
        Test b = (Test) ois.readObject();
        System.out.println(a.getSub().getB());
        System.out.println(b.getSub().getB());
        er.setB(1);
        System.out.println(a.getSub().getB());
        System.out.println(b.getSub().getB());

    }

}

class Test implements Serializable {
    private Sub sub;

    public Sub getSub() {
        return sub;
    }

    public void setEr(Sub sub) {
        this.sub. = sub;
    }
}

class Sub implements Serializable {
    private int b;

    public int getB() {
        return b;
    }

    public void setB(int b) {
        this.b = b;
    }
}
```
