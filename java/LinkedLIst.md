# LinkedList 深入理解
- [LinkedList 深入理解](#linkedlist-深入理解)
  - [LinkedList源码解读](#linkedlist源码解读)
  - [LinkedList的继承和实现关系](#linkedlist的继承和实现关系)
    - [LinkedList的成员变量](#linkedlist的成员变量)
    - [LinkedList的辅助类Node](#linkedlist的辅助类node)
    - [LinkedList实例的创建](#linkedlist实例的创建)
    - [LinkedList的add(E e)源码分析](#linkedlist的adde-e源码分析)
    - [LinkedList的get(index)源码分析](#linkedlist的getindex源码分析)
  - [LinkedList常见面试题TODO](#linkedlist常见面试题todo)

## LinkedList源码解读

## LinkedList的继承和实现关系
LinkedList的类图如下
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710233809.png)

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable
```
1. LinkedList<E> 继承自 AbstractSequentialList<E>。AbstractSequentialList 是一个抽象类，实现了 List<E> 接口。它提供了基于链表的顺序访问列表的通用实现。
2. LinkedList<E> 实现List<E>接口。提供了基本的列表操作方法，例如添加、删除、获取元素等。
3. LinkedList<E> 实现Deque<E>接口。提供了队列的双端操作，允许在队列的两端进行插入、删除、获取元素。
4. LinkedList<E> 实现Cloneable接口。表示该类可以进行克隆操作。
5. LinkedList<E> 实现java.io.Serializable接口。表示该类的实例可以被序列化和反序列化。

### LinkedList的成员变量
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710234314.png)
1. size。该变量表示链表中元素的数量，即链表的大小。
2. first。该变量是一个指向链表第一个节点的指针。
3. last。该变量是一个指向链表最后一个节点的指针。
4. serialVersionUID。用于序列化和反序列化对象的版本控制，确保序列化和反序列化的一致性。
5. modCount。用于记录列表被修改的次数的计数器。

### LinkedList的辅助类Node
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710235645.png)
可以看到Node类中有三个成员变量
1. item。表示Node携带的信息。
2. next。表示Node的下一个节点。
3. prev。表示Node的上一个节点
同时可以看到Node的构造函数有且仅有一个。创建Node实例时需要需要给出Node的三个成员变量才可以创建。
```java
        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
```


### LinkedList实例的创建
第一步：执行这段代码。
```java
    @Test
    public void tes3(){
        LinkedList<Integer> list = new LinkedList<>();
    }
```
第二步：观察debug信息
可以看到LinkedList的无参构造函数就是一个空代码块。创建一个LinkedList实例，如果不加参数，LinkedList内部不会进行额外的操作。
![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710234851.png)


### LinkedList的add(E e)源码分析
**add(E e)
Appends the specified element to the end of this list.将一个特定的元素追加到链表的尾部**


第一步：执行这段代码。
```java
    @Test
    public void tes3(){
        LinkedList<Integer> list = new LinkedList<>();
        for (int i = 0; i < 100; i++){
            list.add(i);
        }
    }
```
第二步：观察debug信息——第一次add
* 首先会进行Integer的自动装箱操作，这里省略介绍自动装箱。
* 然后进入LinkedList的add(E e)方法

* ![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710235334.png)
* 可以看到这里会调用linkLast()方法。
* 进入linkLast方法：
* ![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230710235436.png)
  * 此方法用于将元素插入到链表尾部。
  * 将last设为新节点。
  * 同时如果插入前的last为null，即代表插入前链表为空。所以将first设为此新节点。
  * 否则将插入前的last的next设为新节点。
后续的add和第一次一样。

### LinkedList的get(index)源码分析
第一步：执行这段代码。
```java
    public void tes3(){
        LinkedList<Integer> list = new LinkedList<>();
        for (int i = 0; i < 100; i++){
            list.add(i);
        }
        list.get(5);
    }
```
第二步：观察debug信息——get方法
* 可以看到跳转到get方法后，会先执行checkElementIndex方法来判断index是否合法。
* 然后调用node方法来获取相应的Node，最后返回其item。
* ![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230711001023.png)
* 判断要获取的节点位置在前半部分还是后半部分。
  * 如果 index 小于链表大小的一半 (size >> 1)，说明目标节点在链表的前半部分。
  * 如果 index 大于等于链表大小的一半 (size >> 1)，说明目标节点在链表的后半部分。
* 如果目标节点在链表的前半部分，从头节点开始遍历链表，直到达到目标索引位置的节点。遍历的过程中，通过 x = x.next 将当前节点指针后移，直到达到目标位置。
* 如果目标节点在链表的后半部分，从尾节点开始遍历链表，倒着查找到目标索引位置的节点。遍历的过程中，通过 x = x.prev 将当前节点指针前移，直到达到目标位置。
* 返回找到的目标节点。
* ![](https://tallestdaisy.oss-cn-beijing.aliyuncs.com/20230711022134.png)


## LinkedList常见面试题TODO




