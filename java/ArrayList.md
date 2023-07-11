
# ArrayList 深入理解
- [ArrayList 深入理解](#arraylist-深入理解)
  - [ArrayList源码解读](#arraylist源码解读)
    - [ArrayList的继承和实现关系](#arraylist的继承和实现关系)
    - [ArrayList的成员变量](#arraylist的成员变量)
    - [ArrayList实例的创建](#arraylist实例的创建)
    - [ArrayList的add(E e)和扩容源码分析](#arraylist的adde-e和扩容源码分析)
    - [ArrayList的add(int index, E element)分析](#arraylist的addint-index-e-element分析)
    - [ArrayList的remove(int index)分析](#arraylist的removeint-index分析)
  - [ArrayList常见面试题TODO](#arraylist常见面试题todo)
## ArrayList源码解读

### ArrayList的继承和实现关系
ArrayList的类图如下
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710212943.png)
```
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable
```

1. ArrayList\<E>继承自 AbstractList<E> 类，同时实现了 List<E> 接口。这个类提供了可变大小的数组，并支持对元素的快速访问、添加、删除和修改。
2. ArrayList\<E>实现 List<E> 接口。List<E> 接口是 Java Collections Framework 中定义的一种有序、可重复的集合。通过实现 List<E> 接口，ArrayList<E> 类提供了对列表的操作，包括元素的添加、删除、访问、遍历等。
3. ArrayList\<E>实现了RandmoAccess接口，即提供了随机访问功能。
4. ArrayList\<E>实现了Cloneable接口，即覆盖了函数clone()，能被克隆。
5. ArrayList\<E>实现java.io.Serializable接口，这意味着ArrayList支持序列化，能通过序列化去传输。

### ArrayList的成员变量
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710214157.png)
1. serialVersionUID。用于序列化和反序列化对象的版本控制，确保序列化和反序列化的一致性。
2. DEFAULT_CAPACITY。默认的初始容量，当没有指定容量时，将使用该值。
3. EMPTY_ELEMENTDATA。一个空的、不可变的 Object 数组实例，用于表示空的 ArrayList 对象。
4. DEFAULTCAPACITY_EMPTY_ELEMENTDATA。一个空的、不可变的 Object 数组实例，用于表示默认大小的空 ArrayList 对象。当添加第一个元素时，将会扩容到 DEFAULT_CAPACITY。
5. elementData。ArrayList 内部用于存储元素的数组缓冲区。elementData 的容量即为 ArrayList 的容量。
6. size。ArrayList。 的大小，即列表中当前元素的数量。
7. MAX_ARRAY_SIZE。是一个常量，表示在 Java 虚拟机中可以分配的最大数组大小。
8. modCount。用于记录列表被修改的次数的计数器。


### ArrayList实例的创建
第一步：执行这段代码。
```java
    @Test
    public void tes2(){
        ArrayList<Integer> list = new ArrayList<>();
    }
```
第二步：观察debug信息

进入ArrayList类中的无参构造函数。
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710215216.png)
将成员变量elementData赋值为DEFAULTCAPACITY_EMPTY_ELEMENTDATA。由DEFAULTCAPACITY_EMPTY_ELEMENTDATA的定义可知，它是一个空的Object数组对象。
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710215338.png)
由此可知目前list是一个ArrayList对象，但是ArrayList的elementData本质上是空的。
### ArrayList的add(E e)和扩容源码分析
**add(E e)**
**Appends the specified element to the end of this list.将一个元素追加到尾部。**

第一步：执行这段代码
```java
    @Test
    public void tes2(){
        ArrayList<Integer> list = new ArrayList<>();
        for (int i = 0; i < 100; i++){
            list.add(i);
        }
    }
```
第二步：观察debug信息——第一次add
* 在此处设置断点，并且观察下一步运行什么代码。
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710215949.png)

* 首先跳转到了Integer类中。执行了自动装箱操作。简单来说就是自动将基本类型int转化为int的包装类Integer。具体来说就是遇到这种类型不一致的情况，就会自动调用Integer的valueOf方法，用int的值创建一个Integer对象。
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710220046.png)

* 完成了自动装箱后，跳转至ArrayList的add方法中。
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710220523.png)

* 然后调用了ensureCapacityInternal方法。
  * 其中的参数是size+1.由于当前ArrayList对象中没有元素，所以size为0，size+1的含义就是进行了add操作后，所需的最小容量是1。
  * 此方法用来确保目前的elementData至少有指定的最小容量。
  * 在内部调用了calculateCapacity，来计算出新的容量，这里判断elementData是否为DEFAULTCAPACITY_EMPTY_ELEMENTDATA。
    * 第一次add的时候，elementData是DEFAULTCAPACITY_EMPTY_ELEMENTDATA，所以此方法返回DEFAULT_CAPACITY=10。
    * 后续调用此函数的时候，会返回minCapacity,也就是size+1。
  * 然后调用ensureExplicitCapacity来进行实际的扩容检查。
    * 首先将修改次数modCount自增
    * 然后进行判断，如果minCapacity比elementData还要大（也就是进行add操作后，所需的最小容量比当前的缓冲数组elementData的length还要大），就要进行扩容，扩容的函数是grow。
    * 当前的minCapacity是10，而elementData的length是0，所以需要进行扩容。（第一次add就需要扩容）
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710220914.png)

* 然后进入了grow函数
  * `int oldCapacity = elementData.length;`获取旧的容量`oldCapacity`，即当前 `elementData` 数组的长度。
  * 根据旧容量计算新的容量 `newCapacity`。这里使用了 `(oldCapacity >> 1)`，将旧容量右移一位（相当于除以2），然后加上旧容量，即 `newCapacity = oldCapacity + (oldCapacity >> 1)`。这样，新容量大约是旧容量的1.5倍，用于进行数组的扩容。
  * 检查新容量与所需的最小容量 `minCapacity` 的关系。如果 `newCapacity - minCapacity < 0`，即新容量小于所需的最小容量，说明新容量不足以满足需求，这时将新容量设置为所需的最小容量。
  * 接下来，检查新容量是否超过了 `ArrayList` 允许的最大容量 `MAX_ARRAY_SIZE`。如果 `newCapacity - MAX_ARRAY_SIZE > 0`，即新容量超过了最大容量限制，那么将调用 `hugeCapacity(minCapacity)` 来处理超出范围的情况。
    * 首先如果最小容量需求`minCapacity` 小于 0，表示发生了溢出，因为 `minCapacity` 应该是非负数。在这种情况下，会抛出`OutOfMemoryError `异常，表示无法分配足够的内存空间。
    * 如果`minCapacity` 非负数，则继续执行。
      * 如果` minCapacity` 大于 `MAX_ARRAY_SIZE`，则表示容量超出了范围。根据规定，`ArrayList `允许的最大容量是 `MAX_ARRAY_SIZE`。在这种情况下，将返回 `Integer.MAX_VALUE`，即返回 `Integer `类型的最大值，表示容量达到了 Java 虚拟机所允许的最大值。返回 `Integer.MAX_VALUE` 的结果是一种特殊情况处理，表示 `ArrayList` 已经达到了容量的最大限制。这种情况下，虽然无法分配更多的内存来容纳更多的元素，但仍然可以继续使用 `ArrayList `中已有的元素。
      * 如果 `minCapacity` 小于等于 `MAX_ARRAY_SIZE`，则表示容量在合理范围内，返回 `MAX_ARRAY_SIZE`，表示容量不超过` MAX_ARRAY_SIZE`。
      * ![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710224109.png)
  * 最后，通过调用 `Arrays.copyOf() `方法，将原有的元素拷贝到新的数组中，将 `elementData` 引用指向新的数组，完成数组的扩容操作，给出copyOf方法的文档。
    * ![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710225836.png)
    阅读文档得知copyOf需要指定newCapacity，然后返回一个长度为newCapacity的新数组。当原数组的length小于newCapacity时，多余的部分会被截断，剩余的部分会复制给新数组。当原数组的length大于newCapacity时，多余的位置会被设为null。
  * ![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710222705.png)
* 随后返回到add方法中继续执行。此时的elementData数组已经是一个长度为10的Object数组。然后进行赋值操作。最后size自增。
  * ![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710225207.png)

第三步：观察debug信息——第二次add
第二次add的时候，ArrayList中的elementData的length是10，而所需的最小容量minCapacity = size+1 = 2,所以不需要扩容。

第四步：观察debug信息——第十次add
第十一次add的时候，ArrayList中minCapacity = size+1 = 11 大于 elementData的length，会进行扩容。
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710231308.png)
可以看到进入了grow函数。
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710231343.png)
最终会利用Arrays.copyOf方法将elementData扩容为15大小的新数组。

### ArrayList的add(int index, E element)分析
**add(int index, E element)
Inserts the specified element at the specified position in this list.将这个特殊的元素插入到顺序表中的特定位置**

```java
    public void add(int index, E element) {
        rangeCheckForAdd(index);

        ensureCapacityInternal(size + 1);  // Increments modCount!!
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        elementData[index] = element;
        size++;
    }
```
顺序表将元素插入到中间位置的操作比较繁琐。需要将后续的元素依次后移。
可以看到源码中使用了System.arraycopy（）方法将elementData的index之后的元素右移。
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710232718.png)
然后将新的元素放在index位置上。

### ArrayList的remove(int index)分析
**remove(int index)
Removes the element at the specified position in this list.将一个特定位置的元素移除顺序表**


```java
    public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
```
在顺序表中移除一个元素，需要将此元素后面的元素前移。
同add(int index, E element)，前移操作也使用了System.arraycopy来实现。



## ArrayList常见面试题TODO
