title: HashMap与HashSet
date: 2016-05-01 14:25:17
tags: 
- HashMap
- HashSet
categories: Java
comments: false
---

HashMap和HashSet的区别是Java面试中最常被问到的问题，如果没有涉及到Collections框架和多线程的面试，可以说是不完整的。而Collections框架的问题不涉及到HashMap和HashSet，也可以说是不完整的。HashMap和HashSet都是Collections框架的一部分，它们让我们能够使用对象的集合。Collections框架有自己的接口和实现，主要分为Set接口，List接口和Queue接口。它们有各自的特点，Set的集合里面不允许对象有重复的值，List允许有重复，它对集合中的对象进行索引，Queue的工作原理是FCFS算法（First Come，First Server）。

<!-- more -->

关于HashMap可以参考[HashMap工作原理](http://www.heqingbao.net/2016/03/14/Java_HashMap/)。这里主要分析HashSet，以及它与HashMap的区别。

### 什么是HashSet

HashSet实现了Set接口，它不允许集合中有重复的值，当我们提到HashSet时，第一件事情就是在将对象存储在HashSet之前，要先确保对象重写equals()和hashCode()方法，这样才能比较对象的值是否相等，以确保Set中没有存储相等的对象，如果我们没有重写这两个方法，将会使用这两个方法的默认实现。

`public boolean add(Object o)`方法用来在Set中添加元素，当元素值重复时会立即返回False，如果添加成功则返回True。

### 什么是HashMap

HashMap实现了Map接口，Map接口对键值对进行映射。Map中不允许重复的键。Map接口有两个基本实现，HashMap和TreeMap。TreeMap保存了对象的排列次序，而HashMap则不能。HashMap允许键和值为null。HashMap是非synchronized的，但Collections框架提供方法能保证HashMap synchronized，这样多个线程同时访问HashMap时，能保证只有一个线程更改Map。

`public Object put(Object key, Object value)`方法用来将元素添加到Map中。

### HashSet和HashMap的区别：

HashMap | HashSet
---|---
HashMap实现了Map接口                | HashSet实现了Set接口
使用put()方法将元素放入map中        | 使用add()方法将元素放入set中
HashMap中使用键对象来计算hashCode值 | HashSet使用成员对象来计算hashCode值，对于两个对象来说hashCode可能相同，所以equals()方法用来判断对象的相等性，如果两个对象不同的话，那么返回false
HashMap比较快，因为是使用唯一的键来获取对象 | HashSet较HashMap来说比较慢

这里从源码的角度来分析下HashSet的内部实现。

> Read the fuck source code —— Linus

```
public class HashSet<E>
    extends AbstractSet<E>
    implements Set<E>, Cloneable, java.io.Serializable
{
    static final long serialVersionUID = -5024744406713321676L;

    private transient HashMap<E,Object> map;

    // Dummy value to associate with an Object in the backing Map
    private static final Object PRESENT = new Object();

    public HashSet() {
        map = new HashMap<>();
    }
    
    public HashSet(Collection<? extends E> c) {
        map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
        addAll(c);
    }
    
    public HashSet(int initialCapacity, float loadFactor) {
        map = new HashMap<>(initialCapacity, loadFactor);
    }

    public HashSet(int initialCapacity) {
        map = new HashMap<>(initialCapacity);
    }

    HashSet(int initialCapacity, float loadFactor, boolean dummy) {
        map = new LinkedHashMap<>(initialCapacity, loadFactor);
    }

    public Iterator<E> iterator() {
        return map.keySet().iterator();
    }

    public int size() {
        return map.size();
    }

    public boolean isEmpty() {
        return map.isEmpty();
    }

    public boolean contains(Object o) {
        return map.containsKey(o);
    }

    public boolean add(E e) {
        return map.put(e, PRESENT)==null;
    }

    public boolean remove(Object o) {
        return map.remove(o)==PRESENT;
    }

    public void clear() {
        map.clear();
    }

    public Object clone() {
        try {
            HashSet<E> newSet = (HashSet<E>) super.clone();
            newSet.map = (HashMap<E, Object>) map.clone();
            return newSet;
        } catch (CloneNotSupportedException e) {
            throw new InternalError();
        }
    }

    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException {
        // Write out any hidden serialization magic
        s.defaultWriteObject();

        // Write out HashMap capacity and load factor
        s.writeInt(map.capacity());
        s.writeFloat(map.loadFactor());

        // Write out size
        s.writeInt(map.size());

        // Write out all elements in the proper order.
        for (E e : map.keySet())
            s.writeObject(e);
    }

    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        // Read in any hidden serialization magic
        s.defaultReadObject();

        // Read in HashMap capacity and load factor and create backing HashMap
        int capacity = s.readInt();
        float loadFactor = s.readFloat();
        map = (((HashSet)this) instanceof LinkedHashSet ?
               new LinkedHashMap<E,Object>(capacity, loadFactor) :
               new HashMap<E,Object>(capacity, loadFactor));

        // Read in size
        int size = s.readInt();

        // Read in all elements in the proper order.
        for (int i=0; i<size; i++) {
            E e = (E) s.readObject();
            map.put(e, PRESENT);
        }
    }
}
```

这个是JDK1.7.0_79里面HashSet的完整源码（去除了部分注释），从源码来看有没有很熟悉？

对，你没有看错，它内部就是通过HashMap实现的！！！

```
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

向HashSet里面add元素的时候，直接把元素作为Key，put到HashMap里面，Value直接是一个new Object()。对于HashMap的put(key, vlaue)方法，如果key存在，则返回对应的value。这里根据map.put方法返回是否为空来标记是否add成功。同理`remove(Object o)`实现也是类似的。

