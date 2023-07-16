# 背景介绍
由于jdk nio自带的ByteBuffer使用起来太复杂繁琐，netty提供ByteBuf做了替代，既解决了 JDK API 的局限性， 又为网络应用程序的开发者提供了更好的 API。
下面是一些 ByteBuf API 的优点：

- 它可以被用户自定义的缓冲区类型扩展
- 通过内置的复合缓冲区类型实现了透明的零拷贝
- 容量可以按需增长（类似于 JDK 的 StringBuilder）
- 在读和写这两种模式之间切换不需要调用 ByteBuffer 的 flip()方法
- 读和写使用了不同的索引
- 支持方法的链式调用

# 结构
ByteBuf主要是通过readerIndex 和 writerIndex两个指针进行数据的读和写，整个ByteBuf被这两个指针最多分成三个部分，分别是可丢弃部分(discardable bytes)，可读部分(readable bytes)和可写部分(writable bytes)。需要注意的是**调用以 read 或者 write 开头的 ByteBuf 方法，将会推进其对应的索引，而名称以 set 或 者 get 开头的操作则不会。**
![image.png](https://cdn.nlark.com/yuque/0/2023/png/935856/1689489505353-c2743199-db2b-47a6-b9db-12fe3f755d44.png#averageHue=%23f1f1f1&clientId=u63c274a1-93a8-4&from=paste&height=136&id=uff140588&originHeight=177&originWidth=508&originalType=binary&ratio=2&rotation=0&showTitle=false&size=28111&status=done&style=none&taskId=u8b6c4b51-f7b1-47b6-b547-39f855d875a&title=&width=389)
 ByteBuf内部分段(图自netty实战)
我们可以调用discardReadBytes()方法丢弃可丢弃部分， 回收对应的空间， 但是这会导致readerIndex和writerIndex分别向前移动，比如可丢弃字节数为m, 则两个指针分别向前移动m个字节， 确保readerIndex为0， 这当可读部分字节数大于0即直接有数据时， 会导致内存复制，影响性能，因此建议只在有真正需要的时候才这样做，比如当内存非常宝贵的时候。
具体关于 ByteBuf内部分段的讲解可以看ByteBuf对应类代码的注释，讲得非常详细，这里拷贝出来供参考
```java
/**
 * A random and sequential accessible sequence of zero or more bytes (octets).
 * This interface provides an abstract view for one or more primitive byte
 * arrays ({@code byte[]}) and {@linkplain ByteBuffer NIO buffers}.
 *
 * <h3>Creation of a buffer</h3>
 *
 * It is recommended to create a new buffer using the helper methods in
 * {@link Unpooled} rather than calling an individual implementation's
 * constructor.
 *
 * <h3>Random Access Indexing</h3>
 *
 * Just like an ordinary primitive byte array, {@link ByteBuf} uses
 * <a href="http://en.wikipedia.org/wiki/Zero-based_numbering">zero-based indexing</a>.
 * It means the index of the first byte is always {@code 0} and the index of the last byte is
 * always {@link #capacity() capacity - 1}.  For example, to iterate all bytes of a buffer, you
 * can do the following, regardless of its internal implementation:
 *
 * <pre>
 * {@link ByteBuf} buffer = ...;
 * for (int i = 0; i &lt; buffer.capacity(); i ++) {
 *     byte b = buffer.getByte(i);
 *     System.out.println((char) b);
 * }
 * </pre>
 *
 * <h3>Sequential Access Indexing</h3>
 *
 * {@link ByteBuf} provides two pointer variables to support sequential
 * read and write operations - {@link #readerIndex() readerIndex} for a read
 * operation and {@link #writerIndex() writerIndex} for a write operation
 * respectively.  The following diagram shows how a buffer is segmented into
 * three areas by the two pointers:
 *
 * <pre>
 *      +-------------------+------------------+------------------+
 *      | discardable bytes |  readable bytes  |  writable bytes  |
 *      |                   |     (CONTENT)    |                  |
 *      +-------------------+------------------+------------------+
 *      |                   |                  |                  |
 *      0      <=      readerIndex   <=   writerIndex    <=    capacity
 * </pre>
 *
 * <h4>Readable bytes (the actual content)</h4>
 *
 * This segment is where the actual data is stored.  Any operation whose name
 * starts with {@code read} or {@code skip} will get or skip the data at the
 * current {@link #readerIndex() readerIndex} and increase it by the number of
 * read bytes.  If the argument of the read operation is also a
 * {@link ByteBuf} and no destination index is specified, the specified
 * buffer's {@link #writerIndex() writerIndex} is increased together.
 * <p>
 * If there's not enough content left, {@link IndexOutOfBoundsException} is
 * raised.  The default value of newly allocated, wrapped or copied buffer's
 * {@link #readerIndex() readerIndex} is {@code 0}.
 *
 * <pre>
 * // Iterates the readable bytes of a buffer.
 * {@link ByteBuf} buffer = ...;
 * while (buffer.isReadable()) {
 *     System.out.println(buffer.readByte());
 * }
 * </pre>
 *
 * <h4>Writable bytes</h4>
 *
 * This segment is a undefined space which needs to be filled.  Any operation
 * whose name starts with {@code write} will write the data at the current
 * {@link #writerIndex() writerIndex} and increase it by the number of written
 * bytes.  If the argument of the write operation is also a {@link ByteBuf},
 * and no source index is specified, the specified buffer's
 * {@link #readerIndex() readerIndex} is increased together.
 * <p>
 * If there's not enough writable bytes left, {@link IndexOutOfBoundsException}
 * is raised.  The default value of newly allocated buffer's
 * {@link #writerIndex() writerIndex} is {@code 0}.  The default value of
 * wrapped or copied buffer's {@link #writerIndex() writerIndex} is the
 * {@link #capacity() capacity} of the buffer.
 *
 * <pre>
 * // Fills the writable bytes of a buffer with random integers.
 * {@link ByteBuf} buffer = ...;
 * while (buffer.maxWritableBytes() >= 4) {
 *     buffer.writeInt(random.nextInt());
 * }
 * </pre>
 *
 * <h4>Discardable bytes</h4>
 *
 * This segment contains the bytes which were read already by a read operation.
 * Initially, the size of this segment is {@code 0}, but its size increases up
 * to the {@link #writerIndex() writerIndex} as read operations are executed.
 * The read bytes can be discarded by calling {@link #discardReadBytes()} to
 * reclaim unused area as depicted by the following diagram:
 *
 * <pre>
 *  BEFORE discardReadBytes()
 *
 *      +-------------------+------------------+------------------+
 *      | discardable bytes |  readable bytes  |  writable bytes  |
 *      +-------------------+------------------+------------------+
 *      |                   |                  |                  |
 *      0      <=      readerIndex   <=   writerIndex    <=    capacity
 *
 *
 *  AFTER discardReadBytes()
 *
 *      +------------------+--------------------------------------+
 *      |  readable bytes  |    writable bytes (got more space)   |
 *      +------------------+--------------------------------------+
 *      |                  |                                      |
 * readerIndex (0) <= writerIndex (decreased)        <=        capacity
 * </pre>
 *
 * Please note that there is no guarantee about the content of writable bytes
 * after calling {@link #discardReadBytes()}.  The writable bytes will not be
 * moved in most cases and could even be filled with completely different data
 * depending on the underlying buffer implementation.
 *
 * <h4>Clearing the buffer indexes</h4>
 *
 * You can set both {@link #readerIndex() readerIndex} and
 * {@link #writerIndex() writerIndex} to {@code 0} by calling {@link #clear()}.
 * It does not clear the buffer content (e.g. filling with {@code 0}) but just
 * clears the two pointers.  Please also note that the semantic of this
 * operation is different from {@link ByteBuffer#clear()}.
 *
 * <pre>
 *  BEFORE clear()
 *
 *      +-------------------+------------------+------------------+
 *      | discardable bytes |  readable bytes  |  writable bytes  |
 *      +-------------------+------------------+------------------+
 *      |                   |                  |                  |
 *      0      <=      readerIndex   <=   writerIndex    <=    capacity
 *
 *
 *  AFTER clear()
 *
 *      +---------------------------------------------------------+
 *      |             writable bytes (got more space)             |
 *      +---------------------------------------------------------+
 *      |                                                         |
 *      0 = readerIndex = writerIndex            <=            capacity
 * </pre>
 *
 * <h3>Search operations</h3>
 *
 * For simple single-byte searches, use {@link #indexOf(int, int, byte)} and {@link #bytesBefore(int, int, byte)}.
 * {@link #bytesBefore(byte)} is especially useful when you deal with a {@code NUL}-terminated string.
 * For complicated searches, use {@link #forEachByte(int, int, ByteProcessor)} with a {@link ByteProcessor}
 * implementation.
 *
 * <h3>Mark and reset</h3>
 *
 * There are two marker indexes in every buffer. One is for storing
 * {@link #readerIndex() readerIndex} and the other is for storing
 * {@link #writerIndex() writerIndex}.  You can always reposition one of the
 * two indexes by calling a reset method.  It works in a similar fashion to
 * the mark and reset methods in {@link InputStream} except that there's no
 * {@code readlimit}.
 *
 * <h3>Derived buffers</h3>
 *
 * You can create a view of an existing buffer by calling one of the following methods:
 * <ul>
 *   <li>{@link #duplicate()}</li>
 *   <li>{@link #slice()}</li>
 *   <li>{@link #slice(int, int)}</li>
 *   <li>{@link #readSlice(int)}</li>
 *   <li>{@link #retainedDuplicate()}</li>
 *   <li>{@link #retainedSlice()}</li>
 *   <li>{@link #retainedSlice(int, int)}</li>
 *   <li>{@link #readRetainedSlice(int)}</li>
 * </ul>
 * A derived buffer will have an independent {@link #readerIndex() readerIndex},
 * {@link #writerIndex() writerIndex} and marker indexes, while it shares
 * other internal data representation, just like a NIO buffer does.
 * <p>
 * In case a completely fresh copy of an existing buffer is required, please
 * call {@link #copy()} method instead.
 *
 * <h4>Non-retained and retained derived buffers</h4>
 *
 * Note that the {@link #duplicate()}, {@link #slice()}, {@link #slice(int, int)} and {@link #readSlice(int)} does NOT
 * call {@link #retain()} on the returned derived buffer, and thus its reference count will NOT be increased. If you
 * need to create a derived buffer with increased reference count, consider using {@link #retainedDuplicate()},
 * {@link #retainedSlice()}, {@link #retainedSlice(int, int)} and {@link #readRetainedSlice(int)} which may return
 * a buffer implementation that produces less garbage.
 *
 * <h3>Conversion to existing JDK types</h3>
 *
 * <h4>Byte array</h4>
 *
 * If a {@link ByteBuf} is backed by a byte array (i.e. {@code byte[]}),
 * you can access it directly via the {@link #array()} method.  To determine
 * if a buffer is backed by a byte array, {@link #hasArray()} should be used.
 *
 * <h4>NIO Buffers</h4>
 *
 * If a {@link ByteBuf} can be converted into an NIO {@link ByteBuffer} which shares its
 * content (i.e. view buffer), you can get it via the {@link #nioBuffer()} method.  To determine
 * if a buffer can be converted into an NIO buffer, use {@link #nioBufferCount()}.
 *
 * <h4>Strings</h4>
 *
 * Various {@link #toString(Charset)} methods convert a {@link ByteBuf}
 * into a {@link String}.  Please note that {@link #toString()} is not a
 * conversion method.
 *
 * <h4>I/O Streams</h4>
 *
 * Please refer to {@link ByteBufInputStream} and
 * {@link ByteBufOutputStream}.
 */
public abstract class ByteBuf implements ReferenceCounted, Comparable<ByteBuf> {
    ...
}
```
成员变量的定义是在子类AbstractByteBuf中
```java
/**
 * A skeletal implementation of a buffer.
 */
public abstract class AbstractByteBuf extends ByteBuf {
	// 读索引，当前读取的位置
    int readerIndex;
    // 写索引，当前写入的位置
    int writerIndex;
	// 标记读索引，用于后续的操作。
    private int markedReaderIndex;
    // 标记写索引，用于后续的操作。
    private int markedWriterIndex;
    // 缓冲区的最大容量
    private int maxCapacity;
}
```

# 实现
## 内存回收分类

- UnpooledByteBuf，没有使用对象池的普通bytebuf， 
- PooledByteBuf:使用了对象池的bytebuf，避免了ByteBuf的重复分配，回收，减少了GC的负担，但是内存池的管理和维护更加复杂，netty4.1默认使用对象池缓冲区。4.0默认使用非对象池缓冲区。
## 底层实现分类
![image.png](https://cdn.nlark.com/yuque/0/2023/png/935856/1689495516895-75415db8-fa0a-4e4a-906c-5e197a798980.png#averageHue=%233e454d&clientId=u63c274a1-93a8-4&from=paste&height=291&id=ua4a061e8&originHeight=395&originWidth=564&originalType=binary&ratio=2&rotation=0&showTitle=false&size=91567&status=done&style=none&taskId=u43b33785-b355-4a56-a31f-612ebb8602a&title=&width=416)
可以看到， 按底层实现分类， 可以分为堆缓冲区，直接缓冲区和复合缓冲区三类
### 堆缓冲区HeapByteBuf

- 底层实现基于Java堆内存byte[]字节数组，可以有GC回收
- 优势
   - 简单高效：HeapByteBuf 的底层实现使用字节数组，相比于其他的 ByteBuf 实现，HeapByteBuf 的实现较为简单，可以提供高效的读写操作
   - 不需要自己管理内存，它能在没有使用池化的情况下提供快速的分配和释放。
      - 由于池化的情况下涉及对象的复用和管理，这会引入一些额外的开销和处理步骤，从而导致分配和释放的速度略慢于没有使用池化的情况
- 劣势
   - 数据复制：如果进行socket的io读写，需要额外做一次内存复制，将堆内存的内容复制到内核内存中，性能会有一定程度的下降（如果你的数据包含在一个在堆上分配的缓冲区中，在通过套接字发送它之前，JVM将会在内部把你的缓冲区复制到一个直接缓冲区中）
   - 内存占用：由于 HeapByteBuf 使用的是堆内存，它会占用 Java 堆的内存空间。在大规模并发和数据量较大的情况下，堆内存的使用可能会对 JVM 的内存限制产生影响
   - 垃圾回收压力：由于 HeapByteBuf 的底层存储是 Java 堆上的字节数组，因此这些字节数组的创建和回收将由 Java 的垃圾回收器负责。在高频率的数据读写场景下，可能会导致较高的垃圾回收压力。

非池化的UnpooledHeapByteBuf是AbstractByteBuf的子类，它是基于内存的非池化的字节缓冲区。
```java
/**
 * Big endian Java heap buffer implementation. It is recommended to use
 * {@link UnpooledByteBufAllocator#heapBuffer(int, int)}, {@link Unpooled#buffer(int)} and
 * {@link Unpooled#wrappedBuffer(byte[])} instead of calling the constructor explicitly.
 */
public class UnpooledHeapByteBuf extends AbstractReferenceCountedByteBuf {
    // 用于内存分配
	private final ByteBufAllocator alloc;
    // 字节数组缓冲区
    byte[] array;
    // 用于实现ByteBuf到JDK ByteBuffer的转换
    private ByteBuffer tmpNioBuf;
	...
}
```

池化的HeapByteBuf，可以看到它用的是netty自己实现的内存对象池
```java
class PooledHeapByteBuf extends PooledByteBuf<byte[]> {
	
    // Recycler是 netty 实现的内存池
    private static final Recycler<PooledHeapByteBuf> RECYCLER = new Recycler<PooledHeapByteBuf>() {
        @Override
        protected PooledHeapByteBuf newObject(Handle<PooledHeapByteBuf> handle) {
            return new PooledHeapByteBuf(handle, 0);
        }
    };

    //从“池”里借一个用
    static PooledHeapByteBuf newInstance(int maxCapacity) {
        PooledHeapByteBuf buf = RECYCLER.get();
        buf.reuse(maxCapacity);
        return buf;
    }

    PooledHeapByteBuf(Recycler.Handle<? extends PooledHeapByteBuf> recyclerHandle, int maxCapacity) {
        super(recyclerHandle, maxCapacity);
    }
}
```

Recycler的关键实现如下， 其get 方法和recycle是实现内存池的根本方法
```java
/**
 * Light-weight object pool based on a thread-local stack.
 *
 * @param <T> the type of the pooled object
 */
public abstract class Recycler<T> {

    ...
    
    public final T get() {
        if (maxCapacityPerThread == 0) {
            //表明没有开启池化
            return newObject((Handle<T>) NOOP_HANDLE);
        }
        Stack<T> stack = threadLocal.get();
        DefaultHandle<T> handle = stack.pop();
        //试图从“池”中取出一个，没有就新建一个
        if (handle == null) {
            handle = stack.newHandle();
            handle.value = newObject(handle);
        }
        return (T) handle.value;
    }

    static final class DefaultHandle<T> implements Handle<T> {
        private int lastRecycledId;
        private int recycleId;

        boolean hasBeenRecycled;

        private Stack<?> stack;
        private Object value;

        DefaultHandle(Stack<?> stack) {
            this.stack = stack;
        }

        @Override
        public void recycle(Object object) {
            if (object != value) {
                throw new IllegalArgumentException("object does not belong to handle");
            }

            Stack<?> stack = this.stack;
            if (lastRecycledId != recycleId || stack == null) {
                throw new IllegalStateException("recycled already");
            }
            //释放用完的对象到池里面去
            stack.push(this);
        }
    }
    ...
}
```

还对象的方法
```java
abstract class PooledByteBuf<T> extends AbstractReferenceCountedByteBuf {
	...
    private final Recycler.Handle<PooledByteBuf<T>> recyclerHandle;

    //归还对象到“池”里去，pipeline的tail会调用
    @Override
    protected final void deallocate() {
        if (handle >= 0) {
            final long handle = this.handle;
            this.handle = -1;
            memory = null;
            chunk.arena.free(chunk, tmpNioBuf, handle, maxLength, cache);
            tmpNioBuf = null;
            chunk = null;
            recycle();
        }
    }
    
    private void recycle() {
        recyclerHandle.recycle(this);
    }
}
```

### 直接缓冲区DirectByteBuf

- 直接内存字节缓冲区，非堆内存，它在堆外进行内存分配，避免在每次调用本地 I/O 操作之前（或者之后）将缓冲区的内容复制到一个中间缓冲区（或者从中间缓冲区把内容复制到缓冲区）
- 优势
   - 降低内存占用：DirectByteBuf 使用的是堆外直接内存，它不会占用 Java 堆的内存空间。在大规模并发和数据量较大的情况下，DirectByteBuf 可以减少对 JVM 堆内存的占用，从而降低了对垃圾回收器的压力。
   - 提高数据传输性能：由于数据存储在直接内存中，DirectByteBuf 在进行网络传输时可以直接使用操作系统提供的零拷贝机制，减少了数据复制的开销，提高了数据传输的性能。
   - 适用于 I/O 操作：DirectByteBuf 在进行 I/O 操作时，可以直接将数据传输到操作系统的套接字缓冲区中，减少了数据在用户空间和内核空间之间的复制，提高了 I/O 操作的效率
- 劣势
   - 创建和销毁开销较大：相比于 HeapByteBuf，DirectByteBuf 的创建和销毁可能会更加昂贵。直接内存的分配和释放需要涉及到与操作系统的交互(内核空间的字节数组，native堆中，由操作系统申请和管理，所以分配和释放速度较慢)，因此在频繁创建和销毁 DirectByteBuf 对象时，可能会产生较大的开销。
   - 不受 JVM 垃圾回收管理：DirectByteBuf 使用的直接内存不受 JVM 的垃圾回收器管理，需要手动释放。这就需要开发者注意及时释放不再使用的 DirectByteBuf 对象，否则可能会导致内存泄漏。
   - 内存对齐要求：直接内存的分配和使用有一些内存对齐的要求，因此在使用 DirectByteBuf 时需要注意正确地处理内存对齐问题，否则可能会导致性能下降或者运行时错误。

非池化的UnpooledDirectByteBuf 类表示了一个基于直接内存的 ByteBuf 实例，它提供了对直接内存的访问和操作能力。其最终调用JDK的ByteBuffer类的allocateDirect来分配堆外内存的
```java
/**
 * A NIO {@link ByteBuffer} based buffer. It is recommended to use
 * {@link UnpooledByteBufAllocator#directBuffer(int, int)}, {@link Unpooled#directBuffer(int)} and
 * {@link Unpooled#wrappedBuffer(ByteBuffer)} instead of calling the constructor explicitly.
 */
public class UnpooledDirectByteBuf extends AbstractReferenceCountedByteBuf {
	// 创建此缓存区的ByteBufAllocator 对象
    private final ByteBufAllocator alloc;
	//缓存区，用来储存此缓存区的内容数据。
    ByteBuffer buffer; // accessed by UnpooledUnsafeNoCleanerDirectByteBuf.reallocateDirect()
    // 临时的NIO 缓存区ByteBuffer 对象，其实它是 buffer 的一个 duplicate 对象。
    private ByteBuffer tmpNioBuf;
    // 此缓存区的当前容量。
    private int capacity;
    // 是否不用释放资源。如果这个值为 true，那么最后此缓存区不用释放持有的NIO 缓存区buffer 资源
    private boolean doNotFree;

    /**
     * Creates a new direct buffer.
     *
     * @param initialCapacity the initial capacity of the underlying direct buffer
     * @param maxCapacity     the maximum capacity of the underlying direct buffer
     */
    public UnpooledDirectByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
        super(maxCapacity);
        if (alloc == null) {
            throw new NullPointerException("alloc");
        }
        checkPositiveOrZero(initialCapacity, "initialCapacity");
        checkPositiveOrZero(maxCapacity, "maxCapacity");
        if (initialCapacity > maxCapacity) {
            throw new IllegalArgumentException(String.format(
                    "initialCapacity(%d) > maxCapacity(%d)", initialCapacity, maxCapacity));
        }

        this.alloc = alloc;
        //allocateDirect分配堆外内存
        setByteBuffer(allocateDirect(initialCapacity), false);
    }
    
    /**
     * Allocate a new direct {@link ByteBuffer} with the given initialCapacity.
     */
    protected ByteBuffer allocateDirect(int initialCapacity) {
        //此处调用JDK的allocateDirect来分配堆外内存
        return ByteBuffer.allocateDirect(initialCapacity);
    }
}
```
池化的方式跟堆缓冲区的大同小异， 在此不做分析
### 复合缓冲区CompositeByteBuf

   - 上面两种方式的组合，也是一种零拷贝技术。为多个 ByteBuf 提供一个聚合视图，这些ByteBuf在逻辑上连续，实际物理地址不一定连续。在这里你可以根据需要添加或者删除 ByteBuf 实例，这是一个 JDK 的 ByteBuffer 实现完全缺失的特性
   - 优势
      - 集成多个 ByteBuf：CompositeByteBuf 可以集成多个 ByteBuf，将它们视为一个逻辑缓冲区。这些 ByteBuf 可以是 HeapByteBuf 或 DirectByteBuf，甚至是其他类型的 ByteBuf。
      - 零拷贝操作：通过使用 CompositeByteBuf，可以实现零拷贝的操作。当需要对连续的数据块进行读取或写入时，不需要将数据从一个缓冲区复制到另一个缓冲区，而是直接操作它们的引用，减少了数据复制的开销。
      - 复合多个缓冲区：CompositeByteBuf 可以在逻辑上将多个缓冲区组合在一起，并提供了统一的访问接口。这使得对复合缓冲区的读写操作更加方便，不需要关心实际的底层缓冲区的细节。
   - 劣势
      - 不可扩展：CompositeByteBuf 的容量是固定的，无法动态扩展。一旦创建了CompositeByteBuf，其容量就是固定的，并且无法再添加新的缓冲区。
      - 内存管理：CompositeByteBuf 不会直接分配内存，而是使用底层的 ByteBuf 来管理实际的内存分配和释放。因此，在使用 CompositeByteBuf 时，需要确保底层的 ByteBuf 能够适应实际的内存需求。
      - 对象复用：CompositeByteBuf 内部维护了一个内存管理器和对象池，用于管理底层的 ByteBuf 对象。这意味着在使用 CompositeByteBuf 时，需要关注对象的重用和回收，以避免资源泄漏和内存占用过高。

以下对CompositeByteBuf源码进行讲解
```java
/**
 * A virtual buffer which shows multiple buffers as a single merged buffer.  It is recommended to use
 * {@link ByteBufAllocator#compositeBuffer()} or {@link Unpooled#wrappedBuffer(ByteBuf...)} instead of calling the
 * constructor explicitly.
 */
public class CompositeByteBuf extends AbstractReferenceCountedByteBuf implements Iterable<ByteBuf> {
	
 	private static final ByteBuffer EMPTY_NIO_BUFFER = Unpooled.EMPTY_BUFFER.nioBuffer();
    private static final Iterator<ByteBuf> EMPTY_ITERATOR = Collections.<ByteBuf>emptyList().iterator();
	// 保存了 ByteBuf 分配器的引用，用于后续的内存分配操作
    private final ByteBufAllocator alloc;
    // 表示是否使用直接内存
    private final boolean direct;
    // 复合缓冲区中最大的组件数量
    private final int maxNumComponents;
	// 当前复合缓冲区中的组件数量
    private int componentCount;
    // 用于保存组成复合缓冲区的各个组件的数组。
    private Component[] components; // resized when needed

    private boolean freed;

    public CompositeByteBuf addComponents(int cIndex, ByteBuf... buffers) {
        // 首先对输入参数进行非空检查
        checkNotNull(buffers, "buffers");
        // 实际执行添加组件
        addComponents0(false, cIndex, buffers, 0);
        // 在需要的情况下对复合缓冲区进行合并操作
        consolidateIfNeeded();
        return this;
    }

    private CompositeByteBuf addComponents0(boolean increaseWriterIndex,
            final int cIndex, ByteBuf[] buffers, int arrOffset) {
        final int len = buffers.length, count = len - arrOffset;
        // only set ci after we've shifted so that finally block logic is always correct
        int ci = Integer.MAX_VALUE;
        try {
            // 校验传入的 cIndex 参数，确保它是合法的组件索引
            checkComponentIndex(cIndex);
            // 通过调用 shiftComps 方法将原有组件向后移动以腾出空间，为新的组件腾出位置
            shiftComps(cIndex, count); // will increase componentCount
            int nextOffset = cIndex > 0 ? components[cIndex - 1].endOffset : 0;
            // 根据传入的 buffers 数组，逐个创建新的组件，并将其存储在 components 数组中。
            for (ci = cIndex; arrOffset < len; arrOffset++, ci++) {
                ByteBuf b = buffers[arrOffset];
                if (b == null) {
                    break;
                }
                Component c = newComponent(ensureAccessible(b), nextOffset);
                components[ci] = c;
                // 在添加组件的过程中，不断更新组件的偏移量信息
                nextOffset = c.endOffset;
            }
            return this;
        } finally {
            // ci is now the index following the last successfully added component
            if (ci < componentCount) {
                if (ci < cIndex + count) {
                    // we bailed early
                    removeCompRange(ci, cIndex + count);
                    for (; arrOffset < len; ++arrOffset) {
                        ReferenceCountUtil.safeRelease(buffers[arrOffset]);
                    }
                }
                updateComponentOffsets(ci); // only need to do this here for components after the added ones
            }
            if (increaseWriterIndex && ci > cIndex && ci <= componentCount) {
                writerIndex += components[ci - 1].endOffset - components[cIndex].offset;
            }
        }
    }
}
```

## 获取ByteBuf实例的方式
### ByteBufAllocator
为了降低分配和释放内存的开销，Netty 通过 ByteBufAllocator 实现了ByteBuf 的池化，它可以用来分配我们所描述过的任意类型的 ByteBuf 实例。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/935856/1689498475368-78fad5d9-996d-4c99-8c75-e4acc3600666.png#averageHue=%23f4f3f2&clientId=u63c274a1-93a8-4&from=paste&height=307&id=ue22b38ab&originHeight=407&originWidth=793&originalType=binary&ratio=2&rotation=0&showTitle=false&size=125587&status=done&style=none&taskId=u4d524274-86fd-421d-82d3-664b21d96e3&title=&width=598.5)
ByteBufAllocator 的方法(图自netty实战)
Netty提供了两种ByteBufAllocator的实现：PooledByteBufAllocator和UnpooledByteBufAllocator。

- 前者池化了ByteBuf的实例以提高性能并最大限度地减少内存碎片。
   - 使用了一种称为jemalloc的已被大量现代操作系统所采用的高效方法来分配内存。
   - 后者的实现不池化ByteBuf实例，并且在每次它被调用时都会返回一个新的实例。

进入到ByteAllocator接口，可以发现有一个默认的实现类
```java
/**
 * Implementations are responsible to allocate buffers. Implementations of this interface are expected to be
 * thread-safe.
 */
public interface ByteBufAllocator {

    ByteBufAllocator DEFAULT = ByteBufUtil.DEFAULT_ALLOCATOR;
	...
}
```

可以看到会根据参数或平台采取不同的实现
```java
/**
 * A collection of utility methods that is related with handling {@link ByteBuf},
 * such as the generation of hex dump and swapping an integer's byte order.
 */
public final class ByteBufUtil {
	...
    static final ByteBufAllocator DEFAULT_ALLOCATOR;

    static {
        //以io.netty.allocator.type为准，没有的话，安卓平台用非池化实现，其他用池化实现
        String allocType = SystemPropertyUtil.get(
                "io.netty.allocator.type", PlatformDependent.isAndroid() ? "unpooled" : "pooled");
        allocType = allocType.toLowerCase(Locale.US).trim();

        ByteBufAllocator alloc;
        if ("unpooled".equals(allocType)) {
            alloc = UnpooledByteBufAllocator.DEFAULT;
            logger.debug("-Dio.netty.allocator.type: {}", allocType);
        } else if ("pooled".equals(allocType)) {
            alloc = PooledByteBufAllocator.DEFAULT;
            logger.debug("-Dio.netty.allocator.type: {}", allocType);
        } else {
            //io.netty.allocator.type设置的不是"unpooled"或者"pooled"，就用池化实现。
            alloc = PooledByteBufAllocator.DEFAULT;
            logger.debug("-Dio.netty.allocator.type: pooled (unknown: {})", allocType);
        }

        DEFAULT_ALLOCATOR = alloc;

        THREAD_LOCAL_BUFFER_SIZE = SystemPropertyUtil.getInt("io.netty.threadLocalDirectBufferSize", 0);
        logger.debug("-Dio.netty.threadLocalDirectBufferSize: {}", THREAD_LOCAL_BUFFER_SIZE);

        MAX_CHAR_BUFFER_SIZE = SystemPropertyUtil.getInt("io.netty.maxThreadLocalCharBufferSize", 16 * 1024);
        logger.debug("-Dio.netty.maxThreadLocalCharBufferSize: {}", MAX_CHAR_BUFFER_SIZE);
    }
}
```
### Unpooled 缓冲区
在没办法获取到一个ByteBufAllocator 的引用的情况下，Netty 提供了一个简单的称为 Unpooled 的工具类，它提供了静态的辅助方法来创建未池化的 ByteBuf实例。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/935856/1689498927091-915b4a84-0548-4194-8025-25be9eca2adc.png#averageHue=%23f2f1f0&clientId=u55a90805-48d5-4&from=paste&height=189&id=u87c8ff57&originHeight=238&originWidth=782&originalType=binary&ratio=2&rotation=0&showTitle=false&size=72173&status=done&style=none&taskId=u86417c4a-5148-44b1-bafb-46aff35a391&title=&width=621)
Unpooled 的方法(图自netty实战)

# 参考文章
[netty源码之ByteBuf详解_波波仔86的博客-CSDN博客](https://blog.csdn.net/bobozai86/article/details/123698827)
《Netty实战 第五章ByteBuf》
