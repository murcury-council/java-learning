普通类型与包装类

1，自动装箱与拆箱

​      原始类型byte,short,char,int,long,float,double和boolean对应的封装类为Byte,Short,Character,Integer,Long,Float,Double,Boolean。

​     通过调用封装类的静态方法valueOf(...)实现普通类型自动装箱；

​     通过调用封装类的对象方法intValue()实现自动拆箱。

2，两种类型装箱与拆箱的==与equals方法比较

```java
public class TestIntLong {

  public static void main(String[] args) throws Exception {
    int _int1 = 1;
    int _int2 = 128;
    Integer _integer3 = 1;
    Integer _integer4 = 1;
    Integer _integer5 = new Integer(1);
    Integer _integer6 = new Integer(1);
    Integer _integer7 = 128;
    Integer _integer8 = 128;

    long _long1 = 1;
    long _long2 = 128;
    Long _Long3 = 1l;
    Long _Long4 = new Long(1);

    //==是一个比较运算符，基本数据类型比较的是值，引用数据类型比较的是地址值。
    boolean r1 = _int1 == _long1;
    boolean r2 = _int1 == _integer3;
    boolean r3 = _int1 == _integer5;
    boolean r4 = _integer3 == _integer4;
    boolean r5 = _integer7 == _integer8;
    boolean r6 = _integer3 == _integer5;
    boolean r7 = _integer5 == _integer6;

    /*编译不通过
    boolean r8 = _integer3 == _Long3;*/

    //equals()是一个方法，只能比较引用数据类型。重写前比较的是地址值，重写后比一般是比较对象的属性。
    boolean r9 = _integer3.equals(_integer4);
    boolean r10 = _integer3.equals(_Long3);
    boolean r11 = _Long3.equals(_Long4);

    System.out.println("r1 = " + r1);  //true
    System.out.println("r2 = " + r2);  //true  _integer3自动拆箱，比较的是值
    System.out.println("r3 = " + r3);  //true  _integer5自动拆箱，比较的是值
    /*java Integer类 自动装箱源码
    public static Integer valueOf(int i) {
      if (i >= Integer.IntegerCache.low && i <= Integer.IntegerCache.high)
        return Integer.IntegerCache.cache[i + (-Integer.IntegerCache.low)];
      return new Integer(i);
    }*/
    //同理 附Long的valueOf源码
    /* java Long类 自动装箱源码
    public static Long valueOf(long l) {
      final int offset = 128;
      if (l >= -128 && l <= 127) { // will cache
        return Long.LongCache.cache[(int)l + offset];
      }
      return new Long(l);
    }*/
    System.out.println("r4 = " + r4);  //true  调用自动装箱方法，取的是缓存中的地址
    System.out.println("r5 = " + r5);  //false 调用自动装箱方法，新建对象

    System.out.println("r6 = " + r6);  //false 比较地址
    System.out.println("r7 = " + r7);  //false 比较地址

    /*java Integer类的 equals方法
    public boolean equals(Object obj) {
      if (obj instanceof Integer) {
        return value == ((Integer)obj).intValue();
      }
      return false;
    }*/
    System.out.println("r9 = " + r9);   //true
    System.out.println("r10 = " + r10);  //false
    System.out.println("r11 = " + r11);  //true

  }
}
```

3，HashMap涉及到的装箱和拆箱

```java
public class TestMapIntLong {

  public static void main(String[] args) throws Exception {

    HashMap<Integer, String> map = new HashMap<Integer, String>();
    int a = 1;
    long b = 1;
    map.put(a, "1");
    //map放入对象调用自动装箱
    boolean r12 = Integer.valueOf(a).hashCode() ==Long.valueOf(b).hashCode();
    String r13 = map.get(b);

    /*Integer
    public int hashCode() {
      return Integer.hashCode(value);
    }*/
   /* Long
    public static int hashCode(long value) {
      return (int)(value ^ (value >>> 32));
    }*/
    System.out.println("r12 = " + r12); //true
    /*final HashMap.Node<K,V> getNode(int hash, Object key) {
      HashMap.Node<K,V>[] tab; HashMap.Node<K,V> first, e; int n; K k;
      if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {
        if (first.hash == hash && // always check first node
          ((k = first.key) == key || (key != null && key.equals(k))))
          return first;
        if ((e = first.next) != null) {
          if (first instanceof HashMap.TreeNode)
            return ((HashMap.TreeNode<K,V>)first).getTreeNode(hash, key);
          do {
            if (e.hash == hash &&
              ((k = e.key) == key || (key != null && key.equals(k))))
              return e;
          } while ((e = e.next) != null);
        }
      }
      return null;
    }*/
    //虽然两者hashCode相等，但调用 key.equals(k)==false 参见r10
    System.out.println("r13 = " + r13); //null

  }
}

```

  

​          

​         