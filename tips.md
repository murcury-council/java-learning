# Java 小贴士

## 如何防止通过反射破坏单例模式

传统的单例模式很容易通过反射破解, 从而使破解者创造新对象, 如:

```java
public class Singleton {

  public static final Singleton INSTANCE = new Singleton();

  public static Singleton getInstance() {
    return INSTANCE;
  }

  private final int id;

  private Singleton() {
    this.id = 1;
  }

  public int getId() {
    return id;
  }

  public static void main(String[] args) throws Exception {
    Singleton instance1 = Singleton.getInstance();

    Class<Singleton> clz = Singleton.class;
    Constructor<Singleton> constructor = clz.getDeclaredConstructor(null);
    Singleton instance2 = constructor.newInstance();

    Field field = clz.getDeclaredField("id");
    field.setAccessible(true);
    field.setInt(instance2, 10);

    Preconditions.checkState(instance1.getId() == instance2.getId(), "InstanceNotTheSame");
  }
}
```

通过反射不但可以创造新对象, 还可以任意修改内部属性值, 不论其是否是final的.

解决办法是通过enum创造单例对象. 虽然enum可以在其构造方法里加入各种逻辑, 但是无法通过反射获取到enum的构造方法. 

```java
public enum Singleton {

  INSTANCE;

  private final int id;

  Singleton() {
    this.id = 1;
  }

  public int getId() {
    return id;
  }

  public static void main(String[] args) throws Exception {
    Singleton instance1 = Singleton.INSTANCE;

    Class<Singleton> clz = Singleton.class;
    Constructor<Singleton> constructor = clz.getDeclaredConstructor(null);
    Singleton instance2 = constructor.newInstance();

    Field field = clz.getDeclaredField("id");
    field.setAccessible(true);
    field.setInt(instance2, 10);

    Preconditions.checkState(instance1.getId() == instance2.getId(), "InstanceNotTheSame");
  }
}
```

报错信息:
```java
Exception in thread "main" java.lang.NoSuchMethodException: com.qyer.commons.web.Singleton.<init>()
	at java.lang.Class.getConstructor0(Class.java:3082)
	at java.lang.Class.getDeclaredConstructor(Class.java:2178)
	at com.qyer.commons.web.Singleton.main(Singleton.java:32)
```

## 合成方法(SYNTHETIC)对GuiceAOP中拦截器的影响
合成方法由编译器生成, 在.java层面看不到. 在某些特定条件下, 编译器会自动生成合成方法, 以确保某些语言特性可以执行(如多态).

其中遇到一个典型的例子是, 子类覆盖带泛型类型参数的方法.

例子:

父类定义

```java
// 拦截器类
public class SimpleInterceptor implements MethodInterceptor {

  @Override
  public Object invoke(MethodInvocation mi) throws Throwable {
    Method method = mi.getMethod();
    RequireIntercept nlc = method.getAnnotation(RequireIntercept.class);
    if (nlc == null) {
      return mi.proceed();
    }
    System.out.println("Intercepted");
    return mi.proceed();
  }

}

// 业务类, 其中业务参数类型为泛型
public abstract class Father<P extends FatherParam> {
  abstract void doSth(P p) throws Exception;
}
// 父参数
public class FatherParam {}

// 子类, 业务方法改变了父类定义, 由泛型变为具体类型
public class Child2 extends Father<ChildParam> {
  @Override
  @RequireIntercept
  void doSth(ChildParam childParam) throws Exception {
    System.out.println("Child2");
  }
}

// 后续标记需要拦截的注解
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface RequireIntercept {}

// 运行程序
public class Runner {

  public static class NoSyntheticMethod extends AbstractMatcher<Method> {

    public static final NoSyntheticMethod Instance = new NoSyntheticMethod();

    @Override
    public boolean matches(Method method) {
      return !method.isSynthetic();
    }
  }

  public static void main(String[] args) throws Exception {
    AbstractModule am = new AbstractModule() {
      @Override
      protected void configure() {
        bindInterceptor(any(), annotatedWith(RequireIntercept.class),
                        //                        NoSyntheticMethod.Instance.and(annotatedWith
                        // (RequireIntercept
                        // .class)),
                        new SimpleInterceptor());
      }
    };
    Injector injector = Guice.createInjector(PRODUCTION, am);
    Child2 c2 = injector.getInstance(Child2.class);
    c2.doSth(new ChildParam());
  }
}
```

在上述程序中, 定义了拦截器SimpleInterceptor, 期望其可以拦截所有被标注为RequireIntercept的方法. 但是在执行的时候会出现warning:

```java
Oct 12, 2018 4:25:27 PM com.google.inject.internal.ProxyFactory <init>
警告: Method [void aop.Child2.doSth(aop.FatherParam) throws java.lang.Exception] 
is synthetic and is being intercepted by [aop.SimpleInterceptor@4a87761d]. 
This could indicate a bug. The method may be intercepted twice, or may not be intercepted at all.
```

警告提示出1个方法为:

```java
[void aop.Child2.doSth(aop.FatherParam) throws java.lang.Exception]
```

这个方法我们在Child2中并没有定义(Child2的参数类型是ChildParam). 事实上是编译器生成的合成方法. 通过javap命令观察发现:

```java
Classfile /Users/WuZijing/git_project_home/qyer-commons/commons-api/target/test-classes/aop/Child2.class
  Last modified Oct 12, 2018; size 835 bytes
  MD5 checksum 7c342b1c75a1effc3970f787bf9ca39e
  Compiled from "Child2.java"
public class aop.Child2 extends aop.Father<aop.ChildParam>
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #8.#29         // aop/Father."<init>":()V
   #2 = Fieldref           #30.#31        // java/lang/System.out:Ljava/io/PrintStream;
   #3 = String             #32            // Child2
   #4 = Methodref          #33.#34        // java/io/PrintStream.println:(Ljava/lang/String;)V
   #5 = Class              #35            // aop/ChildParam
   #6 = Methodref          #7.#36         // aop/Child2.doSth:(Laop/ChildParam;)V
   #7 = Class              #37            // aop/Child2
   #8 = Class              #38            // aop/Father
   #9 = Utf8               <init>
  #10 = Utf8               ()V
  #11 = Utf8               Code
  #12 = Utf8               LineNumberTable
  #13 = Utf8               LocalVariableTable
  #14 = Utf8               this
  #15 = Utf8               Laop/Child2;
  #16 = Utf8               doSth
  #17 = Utf8               (Laop/ChildParam;)V
  #18 = Utf8               childParam
  #19 = Utf8               Laop/ChildParam;
  #20 = Utf8               Exceptions
  #21 = Class              #39            // java/lang/Exception
  #22 = Utf8               RuntimeVisibleAnnotations
  #23 = Utf8               Laop/RequireIntercept;
  #24 = Utf8               (Laop/FatherParam;)V
  #25 = Utf8               Signature
  #26 = Utf8               Laop/Father<Laop/ChildParam;>;
  #27 = Utf8               SourceFile
  #28 = Utf8               Child2.java
  #29 = NameAndType        #9:#10         // "<init>":()V
  #30 = Class              #40            // java/lang/System
  #31 = NameAndType        #41:#42        // out:Ljava/io/PrintStream;
  #32 = Utf8               Child2
  #33 = Class              #43            // java/io/PrintStream
  #34 = NameAndType        #44:#45        // println:(Ljava/lang/String;)V
  #35 = Utf8               aop/ChildParam
  #36 = NameAndType        #16:#17        // doSth:(Laop/ChildParam;)V
  #37 = Utf8               aop/Child2
  #38 = Utf8               aop/Father
  #39 = Utf8               java/lang/Exception
  #40 = Utf8               java/lang/System
  #41 = Utf8               out
  #42 = Utf8               Ljava/io/PrintStream;
  #43 = Utf8               java/io/PrintStream
  #44 = Utf8               println
  #45 = Utf8               (Ljava/lang/String;)V
{
  public aop.Child2();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method aop/Father."<init>":()V
         4: return
      LineNumberTable:
        line 9: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       5     0  this   Laop/Child2;

  void doSth(aop.ChildParam) throws java.lang.Exception;
    descriptor: (Laop/ChildParam;)V
    flags:
    Code:
      stack=2, locals=2, args_size=2
         0: getstatic     #2                  // Field java/lang/System.out:Ljava/io/PrintStream;
         3: ldc           #3                  // String Child2
         5: invokevirtual #4                  // Method java/io/PrintStream.println:(Ljava/lang/String;)V
         8: return
      LineNumberTable:
        line 14: 0
        line 15: 8
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Laop/Child2;
            0       9     1 childParam   Laop/ChildParam;
    Exceptions:
      throws java.lang.Exception
    RuntimeVisibleAnnotations:
      0: #23()

  void doSth(aop.FatherParam) throws java.lang.Exception;
    descriptor: (Laop/FatherParam;)V
    flags: ACC_BRIDGE, ACC_SYNTHETIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: checkcast     #5                  // class aop/ChildParam
         5: invokevirtual #6                  // Method doSth:(Laop/ChildParam;)V
         8: return
      LineNumberTable:
        line 9: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Laop/Child2;
    Exceptions:
      throws java.lang.Exception
    RuntimeVisibleAnnotations:
      0: #23()
}
Signature: #26                          // Laop/Father<Laop/ChildParam;>;
SourceFile: "Child2.java"
```

其中有2个业务方法. 一个业务方法带有"ACC_SYNTHETIC"合成标记
观察其拥有的注解, 2个业务方法都拥有:
`0: #23()`即`Laop/RequireIntercept;`
因此Guice会弹出"方法可能执行2次或不被执行"的警告.

解决办法为注册拦截器的时候, 除了annotatedWith, 外加一个自定义的Matcher, 自定义的Matcher为不匹配所有合成方法. 见上述代码的NoSyntheticMatcher

## 原生类型, 常量池的string的==比较结果
```java
Integer i1 = new Integer(1);
Integer i2 = new Integer(1);
Integer i3 = 1;
Integer i4 = 1;
int i5 = 1;
System.out.println(i1 == i2);
System.out.println(i1 == i3);
System.out.println(i3 == i4);
System.out.println(i1 == i5);
System.out.println(i3 == i5);
```
输出:
```java
false
false
true
true
true
```
原因: 
* i1和i2不等, 因为比较的是对象, 是内存地址, 不是同一个对象, 因此不等
* i1和i3不等, 因为1会被编译成Integer.valueOf(), 本质上还是创建了新对象, 比较的是对象地址
* i3和i4相等, 是因为在一定范围内(-127到127)的所有通过Integer.valueOf创建的Integer都会被缓存, 因此是相同的对象
* i1和i5相等, 因为比较的是int值(对象和原生比, 比值)
* i3和i5相等, 原因同上, 只是制造对象的方式不同.

**注意: 如果不是1, 而是127以上的数, 则i3==i4是false**

### 对于字符串的情况:

```java
String s1 = "a";
String s2 = s1;
String s3 = new String(s1);
String s4 = new String("a");
System.out.println(s1 == s2);
System.out.println(s1 == s3);
System.out.println(s1 == s4);
System.out.println(s3 == s4);

String s5 = "b";
String s6 = "ab";
String s7 = "a" + "b";
String s8 = s1 + s5;
String s9 = new StringBuilder().append(s1).append(s5).toString();
String s10 = s1.concat(s5);

System.out.println(s6 == s7);
System.out.println(s6 == s8);
System.out.println(s6 == s9);
System.out.println(s6 == s10);
```

其中, s1==s2, s6==s7, 其余都不相等

原因: s1和s2是同一个对象, s6和s7虽然看起来不一样, 但实际都是字符串常量池里的同一个对象, 在编译的时候就知道了



## 类型擦除会额外增加虚拟方法

Java的泛型只在编译的时候生效, 到jvm中会被擦除. 如实现了```Comparable<String>```的类, 实际上运行时比较的是```Comparable<Object>```, 这个对Object比较的方法会强转参数, 然后调用使用者实现的方法.

例如

```java
public class TestComparable implements Comparable<TestComparable>{

  @Override
  public int compareTo(TestComparable o) {
    return 0;
  }
}
```

调用javap反编译, 得到

```java
Constant pool:
   #1 = Methodref          #4.#21         // java/lang/Object."<init>":()V
   #2 = Class              #22            // com/qyer/test/TestComparable
   #3 = Methodref          #2.#23         // com/qyer/test/TestComparable.compareTo:(Lcom/qyer/test/TestComparable;)I
   #4 = Class              #24            // java/lang/Object
   #5 = Class              #25            // java/lang/Comparable
     
  public int compareTo(com.qyer.test.TestComparable);
    descriptor: (Lcom/qyer/test/TestComparable;)I
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=2, args_size=2
         0: iconst_0
         1: ireturn
      LineNumberTable:
        line 13: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       2     0  this   Lcom/qyer/test/TestComparable;
            0       2     1     o   Lcom/qyer/test/TestComparable;

  public int compareTo(java.lang.Object);
    descriptor: (Ljava/lang/Object;)I
    flags: ACC_PUBLIC, ACC_BRIDGE, ACC_SYNTHETIC
    Code:
      stack=2, locals=2, args_size=2
         0: aload_0
         1: aload_1
         2: checkcast     #2                  // class com/qyer/test/TestComparable
         5: invokevirtual #3                  // Method compareTo:(Lcom/qyer/test/TestComparable;)I
         8: ireturn
      LineNumberTable:
        line 9: 0
      LocalVariableTable:
        Start  Length  Slot  Name   Signature
            0       9     0  this   Lcom/qyer/test/TestComparable;
```

其中, 

```java
5: invokevirtual #3                  // Method compareTo:(Lcom/qyer/test/TestComparable;)I
```

表明, 运行时会运行`compareTo(Object)`, 然后调用`compareTo(com.qyer.test.TestComparable)`

