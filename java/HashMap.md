# Java中的HashMap深入理解

## 介绍
HashMap是Java中非常重要且常用的一种数据结构，它实现了Map接口，主要通过哈希表来存储键值对。
### 什么是HashMap
HashMap是一个包含键值对的数据结构，每个键都对应着一个值。HashMap的键是唯一的，不能有重复。如果尝试用已存在的键插入新值，HashMap则会接受新值并替换原来的值。
### 如何工作
HashMap通过哈希函数将键转化为一个哈希码，这个哈希码然后被转化为数组的索引，值则被存放在这个索引对应的位置。这就是为什么我们可以快速访问HashMap中的元素：我们知道值在内存中的具体位置。
### 如何使用HashMap
创建一个HashMap：
```java
HashMap<String, Integer> map = new HashMap<>();
```
向HashMap中添加元素：
```java
map.put("one", 1);
map.put("two", 2);
map.put("three", 3);
```

从HashMap中获取元素：
```java
int value = map.get("one");  // value现在是1
```
检查HashMap中是否包含一个键：
```java
boolean exists = map.containsKey("one");  // exists现在是true
```

删除HashMap中的元素：
```java
map.remove("one");

```
遍历HashMap：

```java
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println("Key = " + entry.getKey() + ", Value = " + entry.getValue());
}
```

## 深入理解HashMap

### HashMap的成员变量

```java
// 用于对象序列化，允许版本化的类有不同版本的类存在
private static final long serialVersionUID = 362498820763181265L;

// HashMap的默认初始化容量为16，必须是2的幂
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 

// HashMap的最大容量，如果构造函数中指定了更高的值，将使用这个值，最大为2的30次方
static final int MAXIMUM_CAPACITY = 1 << 30;

// 默认的加载因子为0.75，当HashMap的75%的容量被使用时，将自动扩容，这是为了减少哈希冲突，提高查询效率
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 当一个bin的节点数达到这个阈值时，将链表结构转化为红黑树结构，以提高查找效率，默认值为8
static final int TREEIFY_THRESHOLD = 8;

// 当HashMap进行resize操作时，如果一个bin的节点数量少于这个阈值，将红黑树结构转化为链表结构，默认值为6
static final int UNTREEIFY_THRESHOLD = 6;

// HashMap的最小树化容量，默认值为64，当HashMap的容量小于这个值时，不会进行树化，而是优先扩容
static final int MIN_TREEIFY_CAPACITY = 64;


// Node类是HashMap的基础组成单元，每个Node包含一个键、一个值、该键值对的哈希值，以及指向下一个Node的链接
static class Node<K,V> implements Map.Entry<K,V> {
        // 键值对的哈希值
        final int hash;
        // 键
        final K key;
        // 值
        V value;
        // 指向下一个节点的链接
        Node<K,V> next;

        // Node构造函数
        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        // 获取键
        public final K getKey()        { return key; }
        // 获取值
        public final V getValue()      { return value; }
        // 转化为字符串
        public final String toString() { return key + "=" + value; }

        // 计算哈希值，使用键和值的hashCode进行异或操作
        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        // 设置新值，并返回旧值
        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        // 判断是否相等，如果o是一个Entry，并且键和值都相等，则返回true
        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }

/**
 * 这是哈希表的主体。表的大小总是 2 的幂。当表为空时（在第一次使用时初始化），长度为零。
 */
Node<K,V>[] table;

/**
 * 保存了 HashMap 的 entrySet 的缓存，这是 HashMap 的视图函数所返回的 Set 对象。AbstractMap 类的 fields 被用于 keySet() 和 values() 方法。
 */
Set<Map.Entry<K,V>> entrySet;

/**
 * 保存了 HashMap 中当前 key-value 映射的数量。
 */
int size;

/**
 * 用来记录 HashMap 结构修改的次数。结构修改是那些改变了 HashMap 中映射数量或者其他方式修改其内部结构的修改（例如，rehash）。这个字段被用来使得对 HashMap 的 Collection-views 的迭代器快速失败（详见 ConcurrentModificationException）。
 */
int modCount;

/**
 * 下一次需要调整大小的阈值（容量 * 负载因子）。如果表格数组尚未被分配，该字段保存了初始数组的容量，或者零代表默认的初始容量。
 */
int threshold;

/**
 * 哈希表的负载因子，用于确定何时需要扩容。当实际大小（节点数量）超过阈值时，会进行扩容。
 */
final float loadFactor;
```

### HashMap的数据结构
HashMap 是 Java 中的一个重要的数据结构，它实现了 Map 接口，并基于哈希表的原理，存储键值对（Key-Value）。其主要的组成部分包括数组（Bucket）和链表，当冲突发生时，还会将链表转为红黑树以提高性能。

1. 数组（Bucket）：这是 HashMap 的主体，它实际上是一个 Node 数组，数组的每个元素都是一个 Node（链表的头节点）。在初始化后，数组的大小为初始容量，数组的每个元素被称为一个桶（Bucket），Bucket 的索引用于存储与检索值。

2. 链表：当发生哈希冲突时，HashMap 使用链表来处理。在 Java 8 之前，一旦在一个 Bucket 位置发生冲突，所有的元素都被存储在一个链表中。在查找或插入一个元素时，我们需要遍历整个链表。因此，如果有太多的冲突，性能会降低。

3. 红黑树：从 Java 8 开始，如果链表的长度超过一个阈值（默认为 8），那么链表就会被转换为红黑树。红黑树是一种自平衡的二叉搜索树，相比于链表，它可以大大提高搜索的速度。

总的来说，HashMap 是一个“链表数组”或者叫做“哈希表”的数据结构，即数组和链表的结合体。在特定条件下，链表会转为红黑树。这些都是为了提供更高效的数据存储和检索。

### HashMap的put操作流程
下面的代码创建一个HashMap对象，并且随机put10个对象。打断点观察put的流程。
```java
@Test
    public void tes1(){
        HashMap<String, Integer> map = new HashMap<>();
        Random rand = new Random();

        // 插入10个键值对
        for (int i = 0; i < 10; i++) {
            // 生成一个随机键和值
            String key = "key" + rand.nextInt(100);
            Integer value = rand.nextInt(100);

            // 插入键值对并输出结果
            map.put(key, value);
            System.out.println("After inserting " + key + " = " + value);
            System.out.println("Size of map: " + map.size());
            System.out.println("Contents of map: " + map);
            System.out.println();
        }
    }
#### put第一个元素。
```
首先进入HashMap源码中的put方法。key="key60" value=51
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230708151323.png)

进入hash方法
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230708151454.png)

进入String类型的hashCode方法，计算String对象的hashcode
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230708151532.png)

返回HashMap源码中的put方法
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230708151658.png)
可以看到此时已经计算出"key60"的hashcode
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230708151914.png)

然后进入putVal方法
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230708152011.png)

可以看到第一次put的时候会触发此条件，并且调用resize方法
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230708152240.png)
总体来说，resize() 方法的作用是调整 HashMap 的容量，以保持高效的存储和查找性能。当 HashMap 的大小超过其阈值时，就会创建一个新的、容量更大的表，并将旧表中的元素复制到新表中。在这个过程中，还会根据新的容量计算新的阈值。此方法用来初始化或对table的大小进行翻倍操作。
第一次(本次)初始化table的时候，会执行下面的代码。初始化操作创建了一个Node<K,V>数组。数组的大小是newCap = DEFAULT_INITIAL_CAPACITY = 16。同时还修改了HashMap的成员变量threshold = newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY) = 12。
下面给出resize实际执行的代码：
```java
int oldCap = (oldTab == null) ? 0 : oldTab.length;  // oldCap = 0
int oldThr = threshold;  
int newCap, newThr = 0; 


else {               // zero initial threshold signifies using defaults
    newCap = DEFAULT_INITIAL_CAPACITY;   // newCap变为默认初始容量，通常是16
    newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);  // newThr变为默认负载因子（通常是0.75）乘以默认初始容量
}



threshold = newThr;  // 将新计算的阈值赋给threshold变量
@SuppressWarnings({"rawtypes","unchecked"})
Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap]; // 创建一个新的Node数组，大小为newCap
table = newTab; // 将table指向新的数组

```

调用完resize方法后，返回到putVal方法。在插入一个元素的时候，先通过hash计算得到此hash在table中的索引。如果table此索引目前还没有元素，则将创建一个新元素放在table的索引处，否则会执行另一段代码，把新加入的节点放在一个合适的位置。
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230708160410.png)

实际put一个元素到table的源码：
这段代码是 `HashMap` 的 `putVal` 方法的一部分，这部分代码主要用于处理键值对的插入操作。
1. **计算哈希值和索引**：
   首先，通过键的哈希值和数组的长度计算出在数组中的索引，然后检查该索引在数组中是否已有节点。

2. **节点不存在**：
   如果该索引位置没有节点，那么就在这个位置直接创建一个新的节点。

3. **节点存在**：
   如果在该索引位置已有节点存在，则需要进一步处理：

   - **键相同**：如果插入的键和已存在的节点的键是相同的（通过哈希值和 `equals` 方法来确定），那么这个节点就是我们要找的节点。此时，如果允许替换值（`!onlyIfAbsent || oldValue == null`），就替换这个节点的值，并返回旧值。

   - **键不相同**：如果插入的键和已存在的节点的键不相同，那么有可能是哈希冲突，也有可能这个键在链表或者红黑树的后面。此时，对节点的类型进行判断：
     - **TreeNode**：如果该节点是 `TreeNode` 类型（即已经转化为红黑树），那么就按照红黑树的方式来处理插入。
     - **Node**：如果是普通的 `Node`，那么就按照链表的方式来处理。遍历链表，如果找到和插入键相同的节点，就退出循环。否则，就在链表的末尾插入新节点。如果链表的长度超过了阈值（`TREEIFY_THRESHOLD`，默认为 8），那么就把这个链表转化为红黑树。
```java
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            Node<K,V> e; K k;
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
```
然后执行下面的代码，修改次数modCount+1。size++，如果size大于threshold，则执行resize方法，也就是扩容。然后执行插入节点后的操作afterNodeInsertion(evict)。
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230708161319.png)


### HashMap的get操作流程
执行下面的代码进行测试
```java
    @Test
    public void tes1(){
        HashMap<String, Integer> map = new HashMap<>();
        Random rand = new Random();

        // 插入10个键值对
        for (int i = 0; i < 100; i++) {
            // 生成一个随机键和值
            String key = "key" + rand.nextInt(100);
            Integer value = rand.nextInt(100);

            // 插入键值对并输出结果
            map.put(key, value);
            System.out.println("After inserting " + key + " = " + value);
            System.out.println("Size of map: " + map.size());
            System.out.println("Contents of map: " + map);
            System.out.println();
        }
        //插入一个固定的键值对
        map.put("key-static", 110);
        int value = map.get("key-static");
        System.out.println(value);
    }
```

首先进入HashMap的get方法
这段代码是 `HashMap` 的 `getNode` 方法，它是 `HashMap` 中获取元素的核心方法。它通过 `hash` 值和 `key` 来定位并返回相应的节点 `Node`。

以下是代码的详细解释：

1. 用给定的 `hash` 值和当前的 `table` 数组长度计算出数组索引，然后取出这个索引位置的第一个节点。这里使用 `(n - 1) & hash` 这个技巧来代替 `hash % n` 来提高效率。

2. 判断这个节点是否就是要找的节点。首先比较 `hash` 值，如果 `hash` 值相等，再比较 `key`。这里注意，`key` 的比较先使用 `==` 判断是否是同一个对象，如果不是再使用 `equals` 方法比较。

3. 如果第一个节点不是要找的节点，那么查看这个节点是否有下一个节点。如果有，那么需要进一步判断这个链表的类型。
   
   - 如果是 `TreeNode`，即已经转化为红黑树，那么就按照红黑树的方式来搜索节点。
   
   - 如果是普通的 `Node`，那么就按照链表的方式来搜索。遍历链表，如果找到和给定 `key` 相同的节点，就返回这个节点。

4. 如果遍历完链表（或红黑树）都没有找到相应的节点，那么返回 `null`。

通过这个方法，可以看出 `HashMap` 是通过链表（或红黑树）和数组结合的方式来解决哈希冲突的。当哈希冲突导致链表长度超过阈值时，链表会转化为红黑树，以此来提高搜索效率。

![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230708165906.png)

然后进入getNode方法

![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230708170324.png)
## 补充
### 通过hash计算索引为什么不用取余操作而是用hash和数组的最大值做与运算？
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230708160410.png)
主要有两个原因：
1. 性能考虑：位运算在处理器中执行的速度要比除法和取余运算快得多。特别是对于取余运算，这是一种相对昂贵的操作。
2. 特性利用：HashMap的设计者在创建HashMap时，故意将容量设置为2的幂次方。这样，(n - 1) & hash就等同于hash % n。如果n是2的幂，那么n - 1的二进制表示是一串连续的1，这样可以保证hash值被均匀地分布在数组中。

举个例子，假设n=16，那么n-1=15，二进制为1111。无论hash的高位是什么，只要与1111做与操作，都能得到一个0到15之间的值，这样就可以均匀地分布在数组中了。同时，由于使用了位运算，速度也得到了保证。

所以这是一个巧妙的设计，既利用了2的幂次的特性，又利用了位运算的性能优势。

### equals和==的区别
在Java中，"==" 和 "equals()" 方法是用来进行相等性测试的，但它们的用途和行为是有所不同的。

" == "运算符用来比较两个对象的引用是否相等，也就是它们是否指向内存中的同一个对象。
" == "还可以用来比较基本数据类型（如 int, char, float, double 等）的值是否相等。

"equals()"方法是定义在Java的Object类中的，用于比较两个对象的内容是否相等。默认情况下，如果没有在某个类中重写这个方法，那么它的行为和"=="是一样的，也就是比较对象的引用。然而，很多类（比如 String, Integer, Date 等）都重写了这个方法，让它可以根据类的具体内容来判断两个对象是否相等。例如，对于两个不同的String对象，如果它们包含的字符序列相同，那么"equals()"方法就会返回true，即使这两个对象在内存中的位置不同。

总的来说，"=="用于比较对象的引用或基本数据类型的值，而"equals()"用于比较对象的内容。在使用时需要根据具体情况来选择。

### getNode方法中为什么比较出Node的hash相同后还要比较Node的key？

在HashMap中，比较的首要条件是哈希值（hash）。哈希值相同意味着可能是同一个键（key），但是并不能确保就是同一个键。这是因为哈希函数具有一种属性，即"多对一"的映射关系，也就是说可能存在多个不同的输入，但它们的哈希值却是相同的，我们通常称之为哈希冲突（Hash Collision）。

当哈希冲突发生时，HashMap中存储具有相同哈希值的不同键的方式就是链表或红黑树。这也就是为什么在哈希值相同的情况下，还需要继续比较键是否真的相同。比较的方式通常是使用键的equals()方法进行进一步的比较。

所以，哈希值相同只是一个初步的相等条件，最终还需要通过equals()方法来确保键确实相同。这是为了解决哈希冲突的问题。

### getNode方法中为什么要先用==再用equals对Node的key进行比较
在比较键（key）是否相等时，首先使用 == 操作符进行比较的原因在于：

1. 效率：== 操作符的效率高于 equals() 方法。== 操作符只是进行引用的比较，看两个引用是否指向同一个对象，这是一种很快速的操作。而 equals() 方法可能需要比较复杂的逻辑和计算，尤其是在比较复杂的对象时。因此，如果两个引用指向的是同一个对象，使用 == 可以快速得出结果。

2. 空指针：== 操作符对空指针安全，而如果使用 equals() 方法，需要考虑空指针的情况，否则可能会抛出空指针异常。在比较两个对象是否相等时，我们通常会首先使用 == 操作符进行比较，因为这是安全的，不会引发异常。如果 == 操作符无法确定两个对象是否相等（即，两个对象引用不指向同一个对象），那么我们会再使用 equals() 方法进行比较。但在调用 equals() 方法之前，我们需要确认至少一方不为 null，以防止 NullPointerException。

所以，在进行键的比较时，首先使用 == 操作符，如果 == 操作符不能确定结果（即两个引用不指向同一个对象），再使用 equals() 方法。这样可以在保证结果正确的同时，尽可能提高效率。