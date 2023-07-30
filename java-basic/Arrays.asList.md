## 摘要

先总结要点，接下来详细讲解

1. 返回由指定数组支持的长度不可变的列表，可以看做是传入数组的 list 视图，对 list 的修改其实是在修改该数组，所以 list 中元素可以修改，但是不可以增加或删除元素
2. 返回的列表是可序列化的，并实现 RandomAccess 可以随机访问。
3. 此方法与 collection.toArray 结合，充当基于数组和基于集合的 API 之间的桥梁。

```java
// 使用Arrays.asList创建列表
List<String> list1 = Arrays.asList("apple", "banana", "orange");
// 或者
// List<String> list1 = Arrays.asList(new String[] {"apple", "banana", "orange"});
System.out.println(list1); // 输出: [apple, banana, orange]
// 尝试修改列表
list1.set(0, "pear");
System.out.println(list1); // 输出: [pear, banana, orange]
// 尝试修改列表大小
list1.remove(0); // 抛出UnsupportedOperationException异常
System.out.println(list1); // 输出: [pear, banana, orange]
list1.add("grape"); // 抛出UnsupportedOperationException异常
```

## 详解

返回由指定数组支持的长度不可变的列表，可以看做是传入数组的 list 视图。（因为对返回列表的更改其实是对传入的参数数组进行的修改，所以长度不可变，但是数组中存的元素是可以变得）
图中我们可以看到 Arrays.asList()的底层其实调用了 new ArrayList<>(a)，但是此 ArrayList 不是我们经常使用的那个（java.util.ArrayList），而是 Arrays 类中自己实现了一个 ArrayList 类（java.util.Arrays$ArrayList），不然也不会长度不能变了，毕竟我们当时学习的时候说使用 list 而不使用 array 的一个很重要原因就是 list 长度可变而 array 长度不可变
![image.png](https://raw.githubusercontent.com/233zzh/images/main/PicGo/202307211805353.png)

```java
// 使用Arrays.asList创建列表
List<String> list1 = Arrays.asList("apple", "banana", "orange");
// 使用new ArrayLis创建列表
List<String> list2 = new ArrayList();
// 输出list1的类java.util.Arrays$ArrayList
System.out.println(list1.getClass());
// 输出list2的类java.util.ArrayList
System.out.println(list2.getClass());
```

我们接着往下看，看看 Arrays 类自己实现的 ArrayList 类是怎么样的，我们发现他维护了一个数组，而这个数组是 final 定义的，表明一旦被初始化以后，就不能再赋新的值，而我们可以看到他初始化使用的正是我们传入的那个数组，所以因为数组长度不可变，所以 list 长度也不可变，但是可以修改数组中的元素
![image.png](https://raw.githubusercontent.com/233zzh/images/main/PicGo/202307211808132.png)

## 我们再去看看 java.util.ArrayList 为什么可变的呢？

首先他底层维护的也是数组，并且没有使用 final 修饰，说明他是可以重复被赋值的（暗示）
![image.png](https://raw.githubusercontent.com/233zzh/images/main/PicGo/202307211817376.png)

然后，他在增加元素的时候，当目前长度已无法继续插入新元素，就会进行长度的扩容
![image.png](https://raw.githubusercontent.com/233zzh/images/main/PicGo/202307211832265.png)

然后我们发现 grow 方法中会将旧数组中的元素全部 copy 到新数组（是原数组长度的 1.5 倍）中，并且把 arraylist 中保存元素的数组重新复制为新数组（Arrays.copyOf()会把旧数组的元素拷贝到新数组，并且返回新数组）
![image.png](https://raw.githubusercontent.com/233zzh/images/main/PicGo/202307211833904.png)

## Arrays.asList()和 Collections.singletonList()

`Collections.singletonList()` 和 `Arrays.asList()` 都是 Java 集合框架中常用的实用方法，用于创建列表。但它们之间存在一些重要的区别：

1. **元素数量**：
   - `Collections.singletonList(T item)`：仅用于创建一个只有一个元素的列表。
   - `Arrays.asList(T... items)`：可以用于创建包含任意数量元素的列表。
2. **不可变性**：
   - `Collections.singletonList(T item)`：返回的列表是完全不可变的，这意味着你不能添加、删除或修改列表中的元素。
   - `Arrays.asList(T... items)`：返回的列表的大小是固定的，这意味着你不能添加或删除元素。但是，你可以更改列表中的元素，因为其内部数组是可变的。
3. **内部实现**：
   - `Collections.singletonList(T item)`：返回的是一个维护了单一元素的实例。
   - `Arrays.asList(T... items)`：返回的是一个维护一个数组的实例
4. **使用场景**：
   - `Collections.singletonList(T item)`：当你确定需要一个只有一个元素的不可变列表时使用。
   - `Arrays.asList(T... items)`：当你需要一个包含多个元素的固定大小的列表时使用，且可能需要更改元素。
5. **效率**：
   - `Collections.singletonList(T item)`：由于它是为单一元素优化的，所以通常会比 `Arrays.asList()` 更加高效（当只有一个元素时）。比如当只有一个元素时`Collections.singletonList`只维护一个变量，而 `Arrays.asList()`需要维护一个数组去存放这个单一元素
6. 例子：

```java
java List<String> singleItemList = Collections.singletonList("one");// 只包含一个元素 "one" 的不可变列表
List<String> multipleItemList = Arrays.asList("one", "two", "three"); // 包含三个元素的列表
```

总结：因此，推荐使用 Arrays.asList 还是 Collections.singletonList 取决于你的具体需求。如果需要一个可变的列表或需要对列表进行批量操作，那么使用 Arrays.asList 更为合适。如果需要一个不可变的只包含一个元素的列表，那么使用 Collections.singletonList 更为合适。

## 额外：Collections.singletonList()

Collections.singletonList()是 Java 集合框架中的一个静态方法，用于创建一个不可变的只包含一个元素的列表。

该方法的签名如下：

```
public static <T> List<T> singletonList(T o)
```

它接受一个参数 o，表示要包含在列表中的元素，然后返回一个不可变的 List 对象，该列表只包含一个元素 o。

该方法的特点如下：

1. 返回的列表是不可变的，即列表的大小和元素都是固定的，不支持添加、删除和修改操作。
2. 返回的列表是线程安全的，可以在多线程环境下使用，无需额外的同步措施。
3. 返回的列表不允许包含 null 元素，如果传入的参数 o 为 null，则会抛出 NullPointerException 异常。

使用示例：

```java
String element = "Hello";
List<String> list = Collections.singletonList(element);
System.out.println(list); // 输出: [Hello]
```

需要注意的是，由于返回的列表是不可变的，如果需要对列表进行修改操作，那么就不适合使用 Collections.singletonList()，而应该使用其他可变的列表实现，如 ArrayList。

![](https://raw.githubusercontent.com/233zzh/images/main/PicGo/202307212048065.png)
