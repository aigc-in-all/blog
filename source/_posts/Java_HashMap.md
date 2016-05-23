title: HashMap工作原理
date: 2016-03-14 19:02:30
tags: 
- Java 
- HashMap
comments: true
categories: Java
---

> 原文 https://www.javacodegeeks.com/2014/03/how-hashmap-works-in-java.html
> 作者 Arpit Mandliya
>
> 内容有改动

面试的时候经常会遇见诸如：“java中的HashMap是怎么工作的”，“HashMap的get和put内部的工作原理”这样的问题。本文将用一个简单的例子来解释下HashMap内部的工作原理。首先我们从一个例子开始，而不仅仅是从理论上，这样，有助于更好地理解，然后，我们来看下get和put到底是怎样工作的。

我们来看个非常简单的例子。有一个”国家”(Country)类，我们将要用Country对象作为key，它的首都的名字（String类型）作为value。下面的例子有助于我们理解key-value对在HashMap中是如何存储的。

Country.java
```java
public class Country {

    private String name;
    private int population;

    public Country(String name, int population) {
        this.name = name;
        this.population = population;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getPopulation() {
        return population;
    }

    public void setPopulation(int population) {
        this.population = population;
    }
    
    // If length of name in country object is even then return 31(any random number) and if odd then return 95(any random number).
    // This is not a good practice to generate hashcode as below method but I am doing so to give better and easy understanding of hashmap.
    @Override
    public int hashCode() {
        if (this.name.length() % 2 == 0) {
            return 31;
        } else {
            return 95;
        }
    }

    @Override
    public boolean equals(Object obj) {
        Country other = (Country) obj;
        return name.equalsIgnoreCase(other.getName());
    }
}
```

如果你想更多地了解hashCode和equals方法，可以参考[hashcode() and equals() method in java](http://www.java2blog.com/2014/02/hashcode-and-equals-method-in-java.html)。

<!-- more -->

HashMapStructure.java
```java
public class HashMapStructure {

    public static void main(String[] args) {
        Country india = new Country("India", 1000);
        Country japan = new Country("Japan", 10000);

        Country france = new Country("France", 2000);
        Country russia = new Country("Russia", 20000);

        HashMap<Country, String> map = new HashMap<>();
        map.put(india, "Delhi");
        map.put(japan, "Tokyo");
        map.put(france, "Paris");
        map.put(russia, "Moscow");

        Iterator<Country> it = map.keySet().iterator(); // 在这一行加入断点调试
        while (it.hasNext()) {
            Country country = it.next();
            String capital = map.get(country);
            System.out.println(country.getName() + "---" + capital);
        }
    }
}
```

在迭代器那一行加入断点，Debug到此位置。然后我们看一下map里面的内容：

![](http://7xrcq5.com1.z0.glb.clouddn.com/hashmap_1.jpg)

从上图可以观察到以下几点：

* 有一个叫做table大小是16的Entry数组。

* 这个table数组存储了Entry类的对象。HashMap类有一个叫做Entry的内部类。这个Entry类包含了key-value作为实例变量。我们来看一下Entry类的结构：

HashMap$Entry：
```java
static class Entry<K,V> implements Map.Entry<K,V> {
    final K key;
    V value;
    Entry<K,V> next;
    int hash;

    /**
     * Creates new entry.
     */
    Entry(int h, K k, V v, Entry<K,V> n) {
        value = v;
        next = n;
        key = k;
        hash = h;
    }

    ...
}
```

* 每当往HashMap里面存放key-value对的时候，都会为它们实例化一个Entry对象，这个Entry对象就会存储在前面提到的Entry数组table中。现在你一定很想知道，上面创建的Entry对象将会存放在具体哪个位置（在table中的精确位置）。答案就是，根据Key的hashCode()方法计算出来的hash值来决定。hash值用来计算key在Entry数组中的索引。

* 现在，如果你看下上图中数组索引10，它有一个叫做HashMap$Entry的Entry对象。

* 我们往HashMap放了4个key-value对，但是看上去好像只有2个元素！！！这是因为，如果两个元素有相同的hashCode，它们会被放在同一个索引上。问题出现了，该怎么放呢？原来它是以链表（LinkedList）的形式来存储（逻辑上）。

我们来看一下上面几个Country的hashCode.

```
Hashcode for Japan = 95 as its length is odd.
Hashcode for India =95 as its length is odd
HashCode for Russia=31 as its length is even.
HashCode for France=31 as its length is even.
```

下图会清晰的从概念上解释下链表：

![](http://7xrcq5.com1.z0.glb.clouddn.com/hashmap_2.jpg)

所以，现在假如你已经很好地了解了HashMap的结构，让我们看下put和get方法。

HashMap#put

```java
public V put(K key, V value) {
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    }
    if (key == null)
        return putForNullKey(value);
    int hash = hash(key);
    int i = indexFor(hash, table.length);
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        Object k;
        if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
            V oldValue = e.value;
            e.value = value;
            e.recordAccess(this);
            return oldValue;
        }
    }

    modCount++;
    addEntry(hash, key, value, i);
    return null;
}

final int hash(Object k) {
    int h = hashSeed;
    if (0 != h && k instanceof String) {
        return sun.misc.Hashing.stringHash32((String) k);
    }

    h ^= k.hashCode();

    // This function ensures that hashCodes that differ only by
    // constant multiples at each bit position have a bounded
    // number of collisions (approximately 8 at default load factor).
    h ^= (h >>> 20) ^ (h >>> 12);
    return h ^ (h >>> 7) ^ (h >>> 4);
}

/**
 * Returns index for hash code h.
 */
static int indexFor(int h, int length) {
    // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
    return h & (length-1);
}

/**
 * Adds a new entry with the specified key, value and hash code to
 * the specified bucket.  It is the responsibility of this
 * method to resize the table if appropriate.
 *
 * Subclass overrides this to alter the behavior of put method.
 */
void addEntry(int hash, K key, V value, int bucketIndex) {
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }

    createEntry(hash, key, value, bucketIndex);
}

void createEntry(int hash, K key, V value, int bucketIndex) {
    Entry<K,V> e = table[bucketIndex];
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
```

现在一步一步来看上面的代码：

1. 对key做null检查。如果key是null，会被存储到table[0]，因为null的hash值总是0.
2. key的hashCode()方法会被调用，然后计算hash值。hash值用来找到存储Entry对象的数组的索引。有时候hash函数可能写的很不好，所以JDK的设计者添加了另一个叫做hash()的方法，它接收刚才计算的hash值作为参数。
3. indexFor(int h, int length)用来计算在table数组中存储Entry对象的精确位置。
4. 在我们的例子已经看到，如果两个key有相同的hash值（也叫冲突），他们会以链表的形式莱存储。所以，这里我们就迭代链表。

	1. 如果刚才计算出来的索引位置没有元素，则直接把Entry对象放到那个索引上。
	2. 如果索引上有元素，则已有的Entry对象将作为新添加Entry对象的下一个节点。(If There is element present at that index then it will iterate until it gets Entry->next as null.Then current Entry object become next node in that linkedlist，原文好像有问题，根据createEntry方法的逻辑来看，是把当前index上的元素取出来传给了new Entry的next)
	3. 如果我们再次放入同样的key会怎样呢？逻辑上，它应该替换老的value。事实上，它确实是这么做的。在迭代的过程中，会调用equals()方法来检查key的相等性(key.equals(k))，如果这个方法返回true，它就会用当前的Entry的value来替换之前的value。

HashMap#get
```java
public V get(Object key) {
    if (key == null)
        return getForNullKey();
    Entry<K,V> entry = getEntry(key);

    return null == entry ? null : entry.getValue();
}

private V getForNullKey() {
    if (size == 0) {
        return null;
    }
    for (Entry<K,V> e = table[0]; e != null; e = e.next) {
        if (e.key == null)
            return e.value;
    }
    return null;
}

final Entry<K,V> getEntry(Object key) {
    if (size == 0) {
        return null;
    }

    int hash = (key == null) ? 0 : hash(key);
    for (Entry<K,V> e = table[indexFor(hash, table.length)];
         e != null;
         e = e.next) {
        Object k;
        if (e.hash == hash &&
            ((k = e.key) == key || (key != null && key.equals(k))))
            return e;
    }
    return null;
}
```

当你理解了HashMap的put的工作原理，理解get的工作原理就非常简单了。当你传递一个key从HashMap中获取value的时候：

1. 对key进行null检查。如果Key为null，table[0]这个位置的元素将被返回（如果size==0则返回null）。
2. key的hashCode方法被调用，然后计算hash值。
3. indexFor(hash, table.length)用来计算要获取的Entry对象在table数组中精确的位置，使用刚才计算的hash值。
4. 在获取了table数组索引之后，会迭代链表，调用equals()方法检查key的相等性，如果equals()方法返回true，get方法返回Entry对象的value，否则返回null.

**要牢记以下关键点**

* HashMap有一个叫做Entry的内部类，它用来存储kye-value对。
* 上面的Entry对象是存储在一个叫做table的Entry数组中。
* table的索引在逻辑上叫做“桶”(bucket)，它存储了链表的第一个元素。
* key的hashCode方法用来找到Entry对象所在的桶。
* 如果两个key有相同的hash值，他们会被放在table数组的同一个桶里面。
* key的equals()方法用来确保key的唯一性。
* value对象的equals()和hashcode()方法根本一点用也没有。

----------------

问题一：为什么初始化table数组的时候，我们看到的是size==16呢？

构造方法：
```java
/**
 * The default initial capacity - MUST be a power of two.
 */
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

/**
 * The load factor used when none specified in constructor.
 */
static final float DEFAULT_LOAD_FACTOR = 0.75f;

static final Entry<?,?>[] EMPTY_TABLE = {};

/**
 * The table, resized as necessary. Length MUST Always be a power of two.
 */
transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;

/**
 * Constructs an empty <tt>HashMap</tt> with the default initial capacity
 * (16) and the default load factor (0.75).
 */
public HashMap() {
    this(DEFAULT_INITIAL_CAPACITY, DEFAULT_LOAD_FACTOR);
}

public HashMap(int initialCapacity, float loadFactor) {
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);

    this.loadFactor = loadFactor;
    threshold = initialCapacity;
    init();
}

void init() {
}
```

虽然构造方法里面把threshold初始化成16了，并且init是空实现，但是threshold是个什么东西？此时table仍是EMPTY_TABLE。再看put方法：

```java

/**
 * The next size value at which to resize (capacity * load factor).
 * @serial
 */
// If table == EMPTY_TABLE then this is the initial capacity at which the
// table will be created when inflated.
int threshold;

public V put(K key, V value) {
    if (table == EMPTY_TABLE) {
        inflateTable(threshold);
    }
    ...
}

/**
 * Inflates the table.
 */
private void inflateTable(int toSize) {
    // Find a power of 2 >= toSize
    int capacity = roundUpToPowerOf2(toSize);

    threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
    table = new Entry[capacity];
    initHashSeedAsNeeded(capacity);
}

private static int roundUpToPowerOf2(int number) {
    // assert number >= 0 : "number must be non-negative";
    return number >= MAXIMUM_CAPACITY
            ? MAXIMUM_CAPACITY
            : (number > 1) ? Integer.highestOneBit((number - 1) << 1) : 1;
}
```

原来是在inflateTable里面new了一个新的Entry数组。

问题二：根据上面看到的，初始化table数组的size==16（不同的JDK版本可能有所不同），那假设要存储的数据超过16个怎么办呢？再看put方法：

```java
public V put(K key, V value) {
    ...
    addEntry(hash, key, value, i);
    return null;
}

void addEntry(int hash, K key, V value, int bucketIndex) {
    if ((size >= threshold) && (null != table[bucketIndex])) {
        resize(2 * table.length);
        hash = (null != key) ? hash(key) : 0;
        bucketIndex = indexFor(hash, table.length);
    }

    createEntry(hash, key, value, bucketIndex);
}

void resize(int newCapacity) {
    Entry[] oldTable = table;
    int oldCapacity = oldTable.length;
    if (oldCapacity == MAXIMUM_CAPACITY) {
        threshold = Integer.MAX_VALUE;
        return;
    }

    Entry[] newTable = new Entry[newCapacity];
    transfer(newTable, initHashSeedAsNeeded(newCapacity));
    table = newTable;
    threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
}
```
从源码中可以看到，扩容的时候是按照`2 * table.length`作为newCapacity。

另外关于loadFactory，俗称加载因子，在put的时候，判断是否需要扩容是根据当前的size和threshold来判断的，而threshold又跟加载因子有关`threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1)`，可见，并不是等到table数组存满了再扩容。这里的加载因子默认是0.75（`static final float DEFAULT_LOAD_FACTOR = 0.75f`）

