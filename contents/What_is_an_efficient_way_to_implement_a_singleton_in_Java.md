# 如何创建单例 ？

## 问题
Java 创建单例有哪些方式 ?

## 解答
实现单例，从加载方式来看，有两种:

- 预加载
- 懒加载

先看一下实现单例最简单的方式(预加载):
```
public class Foo {

    private static final Foo INSTANCE = new Foo();

    private Foo() {
        if (INSTANCE != null) {
            throw new IllegalStateException("Already instantiated");
        }
    }

    public static Foo getInstance() {
        return INSTANCE;
    }
}
```

再来看一下懒加载的方式:
```
class Foo {

    private static Foo INSTANCE = null;

    private Foo() {
        if (INSTANCE != null) {
            throw new IllegalStateException("Already instantiated");
        }
    }

    public static Foo getInstance() {
        if (INSTANCE == null) {
            INSTANCE = new Foo();
        }
        return INSTANCE;
    }
}
```

以上方式在单线程的情况可以很好的满足需要，换言之，若是在多线程，还需要作一定的改进,如下所示:
```
class Foo {
	// 请注意 volatile 关键字的使用
    private static volatile Foo INSTANCE = null;

    private Foo() {
        if (INSTANCE != null) {
            throw new IllegalStateException("Already instantiated");
        }
    }

    public static Foo getInstance() {
        if (INSTANCE == null) { // Check 1
            synchronized (Foo.class) {
                if (INSTANCE == null) { // Check 2
                    INSTANCE = new Foo();
                }
            }
        }
        return INSTANCE;
    }
}
```

上述代码运用了 [Double-Checked Locking idiom](http://www.cs.umd.edu/~pugh/java/memoryModel/DoubleCheckedLocking.html)。

解决了多线程环境下的单例，可以进一步思考如何实现可序列化的单例 ? 反序列化可以不通过构造函数直接生成一个对象，所以反序列化时，我们需要保证其不再创建新的对象。

```
class Foo implements Serializable {

    private static final long serialVersionUID = 1L;

    private static volatile Foo INSTANCE = null;

    private Foo() {
        if (INSTANCE != null) {
            throw new IllegalStateException("Already instantiated");
        }
    }

    public static Foo getInstance() {
        if (INSTANCE == null) { // Check 1
            synchronized (Foo.class) {
                if (INSTANCE == null) { // Check 2
                    INSTANCE = new Foo();
                }
            }
        }
        return INSTANCE;
    }

    @SuppressWarnings("unused")
    private Foo readResolve() {
        return INSTANCE;
    }
}
```

readResolve 方法可以保证，即使程序在上一次运行时序列化过此单例，也只会返回全局唯一的单例。对于 Java 对象序列化机制，可参考[附录拓展](#appendix)。

java 创建单例的方法基本实现了，不过我们还可以作进一步的改进 —— 代码重构:
```
public final class Foo implements Serializable {

    private static final long serialVersionUID = 1L;

    // 使用内部静态 class 实现懒加载
    private static class FooLoader {
        // 保证在多线程环境下无差错运行
        private static final Foo INSTANCE = new Foo();
    }

     private Foo() {
        if (INSTANCE != null) {
            throw new IllegalStateException("Already instantiated");
        }
    }

    public static Foo getInstance() {
        return FooLoader.INSTANCE;
    }

    @SuppressWarnings("unused")
    private Foo readResolve() {
        return FooLoader.INSTANCE;
    }
}
```

好了，现在已经很完美实现了单例的创建，是不是很高兴。单例实线的基本原理，我们已经基本清楚里，最后提供一种更加简洁方法,如下:
```
public enum Foo {
   INSTANCE;
}
```

为什么可以这么简洁？因为 Java 中每一个枚举类型都默认继承了 java.lang.Enum ，而 Enum 实现了 Serializable 接口，所以枚举类型对象都是默认可以被序列化的。通过反编译，也可以知道枚举常量本质上就是一个 
```
public static final xxx
````

对于枚举的进一步理解，请参考[附录拓展](#appendix)。

<a id="appendix" name="appendix" />
附录拓展:
[深入理解 Java 对象序列化](http://developer.51cto.com/art/201202/317181.htm)
[对象的序列化和反序列化](http://www.blogjava.net/lingy/archive/2008/10/10/233630.html)
[通过反编译字节码来理解 Java 枚举](http://unmi.cc/understand-java-enum-with-bytecode/)

stackoverflow原址：http://stackoverflow.com/questions/70689/what-is-an-efficient-way-to-implement-a-singleton-pattern-in-java

