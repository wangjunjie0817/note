# DCL

DCL ，即 Double Check Lock ，中文称为“双重检查锁定”。

### 问题分析

我们先看单例模式里面的懒汉式：
```java
public class Singleton {
    
    private static Singleton singleton;

    private Singleton(){}

    public static Singleton getInstance(){
        if (singleton == null) {
            singleton = new Singleton();
        }
        return singleton;
    } 
}
```

我们都知道这种写法是错误的，因为它无法保证线程的安全性。优化如下：

```java
public class Singleton {

    private static Singleton singleton;

    private Singleton(){}

    public static synchronized Singleton getInstance() {
        if (singleton == null) {
            singleton = new Singleton();
        }

        return singleton;
    }
    
}
```

优化非常简单，就是在 #getInstance() 方法上面做了同步，但是 synchronized 就会导致这个方法比较低效，导致程序性能下降，那么怎么解决呢？聪明的人们想到了双重检查 DCL：

```
public class Singleton {

    private static Singleton singleton;

    private Singleton() {}

    public static Singleton getInstance(){
        if(singleton == null){                              // 1
            synchronized (Singleton.class){                 // 2
                if(singleton == null){                      // 3
                    singleton = new Singleton();            // 4
                }
            }
        }
        return singleton;
    }
}
```

就如上面所示，这个代码看起来很完美，理由如下：

- 如果检查第一个 singleton 不为 null ，则不需要执行下面的加锁动作，极大提高了程序的性能。
- 如果第一个 singleton 为 null ，即使有多个线程同一时间判断，但是由于 synchronized 的存在，只会有一个线程能够创建对象。
- 当第一个获取锁的线程创建完成后 singleton 对象后，其他的在第二次判断 singleton 一定不会为 null ，则直接返回已经创建好的 singleton 对象。

通过上面的分析，DCL 看起确实是非常完美，但是可以明确地告诉你，这个错误的。上面的逻辑确实是没有问题，分析也对，但是就是有问题，那么问题出在哪里呢？在回答这个问题之前，我们先来复习一下创建对象过程，实例化一个对象要分为三个步骤：

```java
memory = allocate();   //1：分配内存空间
ctorInstance(memory);  //2：初始化对象
instance = memory;     //3：将内存空间的地址赋值给对应的引用
```

但是由于重排序的原因，步骤 2、3 可能会发生重排序，其过程如下：

```
memory = allocate();   // 1：分配内存空间
instance = memory;     // 3：将内存空间的地址赋值给对应的引用
                                    // 😈 注意，此时对象还没有被初始化！
ctorInstance(memory);  // 2：初始化对象
```

如果 2、3 发生了重排序，就会导致第二个判断会出错，singleton != null，但是它其实仅仅只是一个地址而已，此时对象还没有被初始化，所以 return 的 singleton 对象是一个没有被初始化的对象，


通过上面的阐述，我们可以判断 DCL 的错误根源在于步骤 4：

singleton = new Singleton();

知道问题根源所在，那么怎么解决呢？有两个解决办法：

- 不允许初始化阶段步骤 2、3 发生重排序。

- 允许初始化阶段步骤 2、3 发生重排序，但是不允许其他线程“看到”这个重排序。

### 解决方案

解决方案依据上面两个解决办法即可。

#### 1. 基于 volatile 解决方案

对于上面的DCL其实只需要做一点点修改即可：将变量singleton生命为volatile即可：

```java
public class Singleton {

    // 通过volatile关键字来确保安全
    private volatile static Singleton singleton;

    private Singleton(){}

    public static Singleton getInstance(){
        if(singleton == null){
            synchronized (Singleton.class){
                if(singleton == null){
                    singleton = new Singleton();
                }
            }
        }
        return singleton;
    }
}
```

当 singleton 声明为 volatile后，步骤 2、3 就不会被重排序了，也就可以解决上面那问题了。

#### 2. 基于类初始化的解决方案

该解决方案的根本就在于：利用 ClassLoder 的机制，保证初始化 instance 时只有一个线程。JVM 在类初始化阶段会获取一个锁，这个锁可以同步多个线程对同一个类的初始化。

public class Singleton {

    private static class SingletonHolder{
        public static Singleton singleton = new Singleton();
    }

    public static Singleton getInstance(){
        return SingletonHolder.singleton;
    }
}
这种解决方案的实质是：运行步骤 2 和步骤 3 重排序，但是不允许其他线程看见。

Java 语言规定，对于每一个类或者接口 C ，都有一个唯一的初始化锁 LC 与之相对应。从C 到 LC 的映射，由 JVM 的具体实现去自由实现。JVM 在类初始化阶段期间会获取这个初始化锁，并且每一个线程至少获取一次锁来确保这个类已经被初始化过了。
