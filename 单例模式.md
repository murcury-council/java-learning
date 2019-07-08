#单例模式

```java
public class Singleton {

  private Singleton() {
  }

  private static class SingletonContainer {

    private static final Singleton INSTANCE = new Singleton();
  }

  public static final Singleton getInstance() {
    return SingletonContainer.INSTANCE;
  }
}
```

* JVM内部保证一个类加载的时候是线程互斥的

* final字段保证INSTANCE的初始化过程

静态内部类只有在getInstance()方法第一次被调用时**才会被加载**（懒汉模式）



Effective Java中提供如下写法

```java
public enum SingletonEnum {
  INSTNCE;
}
```

Javac 编译后

```java
public enum SingletonEnum {
  INSTNCE;

  private SingletonEnum() {
  }
}
```

enum无常提供了序列化并绝对防止多次实例化



An `enum` type is a special type of `class`.

Your `enum` will actually be compiled to something like

```java
public final class MySingleton {
    public final static MySingleton INSTANCE = new MySingleton();
    private MySingleton(){} 
}
```

