---
title: Java8之HashMap
date: 2020-08-09 15:25:02
---
HashMap作为Java编程中一种常用的数据结构，虽然我们每天都用，但还是有许多有趣的地方需要深究一下。

## 容量
在我经历的几个项目上，HashMap的容量几乎被所有开发人员都忽略了，虽然HashMap会根据实际使用情况扩容，但容量规划不好，还是有可能会发生意想不到的情况。
### 初始容量
HashMap在创建时会赋予一个初始的容量，我们可以通过构造器来指定大小，或者使用默认大小

``` java
// 创建一个默认容量(16)的HashMap
Map<Integer,Integer> map = new HashMap<>();
// 创建一个容量为8的HashMap
Map<Integer,Integer> map = new HashMap<>(8);

// 创建一个容量为4的HashMap
Map<Integer,Integer> map = new HashMap<>(3);
```
这里你会发现我指定容量为3，但注释写的是4。  
没错，HashMap的容量一定是2的n次方，所以当不足2的n次方时，会补足到2的n次方。

### 容量扩充
HashMap当容量到达一定的阈值时会自动扩充容量，以保存更多的数据。同样的每次会扩充2倍
``` java
Map<Integer,Integer> map = new HashMap<>(4);
map.put(3,3);
map.put(2,2);
map.put(8,8);
map.put(5,5);
map.put(9,5);
Class<?> mapType = map.getClass();
Method capacity = mapType.getDeclaredMethod("capacity");
capacity.setAccessible(true);
System.out.printf("capacity: %s",capacity.invoke(map));

// capacity: 8
```

HashMap的扩充策略是到达一个阈值就会扩充，而这个阈值默认是0.75倍的capacity，我们也可以通过构造函数修改这个比例

``` java
// 这个是HashMap的默认阈值
static final float DEFAULT_LOAD_FACTOR = 0.75f;
// 手动指定阈值
Map<Integer,Integer> map = new HashMap<>(4,0.8f);
```

## 存储原理

太多了 不愿意写，网上有很多，自己找吧。

## lambda表达式
Java8中因为加入的lambda表达式，多了几个API供开发者使用

### compute
``` java
//对传入key的这对键值对进行计算处理，返回值会赋予到value上
public V compute(K key,BiFunction<? super K, ? super V, ? extends V> remappingFunction);

// 例:
HashMap<Integer,Integer> map = new HashMap<>();
map.put(3,4);
map.compute(3,(k,v)-> k+v);
System.out.println(map);

// {3=7}
```

### computeIfAbsent
``` java
// 对传入key的，如果不存在则执行函数，并将返回值存入这对key,value中
public V computeIfAbsent(K key,Function<? super K, ? extends V> mappingFunction);

//例:
HashMap<Integer,Integer> map = new HashMap<>();
map.put(3,4);
map.computeIfAbsent(4,k-> 5);
System.out.println(map);

// {3=4, 4=5}
```

### computeIfPresent
``` java
// 对传入key的，如果存在则执行函数，并将返回值存入value中
public V computeIfPresent(K key,BiFunction<? super K, ? super V, ? extends V> remappingFunction);
// 例:
HashMap<Integer,Integer> map = new HashMap<>();
map.put(3,4);
map.computeIfPresent(3,(k,v)-> ++v); //如果3存在 则执行 并将返回值存入3中
System.out.println(map);

// {3=5}
```

### replaceAll
``` java
// 会替换所有key的value替换为函数的返回值
public void replaceAll(BiFunction<? super K, ? super V, ? extends V> function);

// 例:
HashMap<Integer,Integer> map = new HashMap<>();
map.put(3,4);
map.computeIfAbsent(4,k-> 5);//如果不存在 则将函数返回值存入key为4中
map.replaceAll((k,v) -> k==3 ? 999: 0);
System.out.println(map);

// {3=999, 4=0}
```

## LinkedHashMap
LinkedHashMap继承了HashMap，区别在与其在HashMap的基础上维护了一个双向链表，来保证插入和遍历的顺序。

### 顺序
LinkedHashMap和HashMap使用方式都一样，唯一的区别就在与遍历时的顺序。HashMap会根据hash值再经过一系列运算进行存储，所以顺序不能保证。但LinkedHashMap的遍历顺序会和插入的顺序相同。
``` java
LinkedHashMap<Integer,Integer> map = new LinkedHashMap<>();
map.put(3,4);
map.put(5,6);
map.put(2,3);
map.put(9,10);
map.put(8,9);
System.out.println(map);

// {3=4, 5=6, 2=3, 9=10, 8=9}
```
但是在重新put后不会改变他的顺序,但remove后再put会改变顺序。
``` java
LinkedHashMap<Integer,Integer> map = new LinkedHashMap<>();
map.put(3,4);
map.put(5,6);
map.put(2,3);
map.put(9,10);
map.put(8,9);
map.put(2,4);
System.out.println(map);
        
// {3=4, 5=6, 2=4, 9=10, 8=9}
map.remove(9);
map.put(9,11);
// {3=4, 5=6, 2=4, 8=9, 9=11}
```