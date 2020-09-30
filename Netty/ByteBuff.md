![](https://upload-images.jianshu.io/upload_images/8387919-3d38d8dc3509e5bb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

二进制数据抽象ByteBuf是字节容器,容器里面的的数据分为三个部分,第一个部分是已经丢弃的字节,这部分数据是无效的;第二部分是可读字节,这部分数据是ByteBuf的主体数据,从ByteBuf里面读取的数据都来自这一部分;最后一部分的数据是可写字节,所有写到 ByteBuf的数据都会写到这一段.最后一部分虚线表示的是该ByteBuf最多还能扩容多少容量.这三部分内容是被两个指针给划分出来的,从左到右依次是读指针(readerIndex)、写指针(writerIndex),然后还有容量capacity表示ByteBuf底层内存的总容量.
 读数据是从ByteBuf里每读取一个字节,readerIndex自增1,ByteBuf里面共有writerIndex-readerIndex个字节可读, 由此推论当readerIndex与writerIndex相等的时候,ByteBuf不可读.
 写数据是从writerIndex指向的部分开始写,每写一个字节,writerIndex 自增1,直到增加至容量capacity,此时表示ByteBuf不可写.
 ByteBuf里容量上限maxCapacity表示当向ByteBuf写数据时容量capacity不足进行扩容,直至扩容到最大容量maxCapacity,超过 maxCapacity报错.
 推荐API:markReaderIndex()&resetReaderIndex()/markWriterIndex()& resetWriterIndex():前者表示把当前的读/写指针保存起来,后者表示把当前的读/写指针恢复到之前保存的值,常见使用场景为解析自定义协议的数据包.
 解析数据注意:get/set()方法不会改变读写指针,read/write()方法会改变读写指针.
 Netty使用堆外内存,堆外内存是不被JVM直接管理的,申请到的内存无法被垃圾回收器直接回收需要手动回收,即申请到的内存必须手工释放,否则造成内存泄漏.
 ByteBuf通过引用计数方式管理,如果ByteBuf没有地方被引用到,需要回收底层内存.默认情况下,当创建完ByteBuf其引用为1,然后每次调用retain()方法引用加1,release()方法原理是将引用计数减1,减完发现引用计数为0回收ByteBuf底层分配内存.
 slice()/duplicate()/copy():
 1.slice()方法从原始ByteBuf截取一段,这段数据是从readerIndex到writeIndex,返回的新的ByteBuf的最大容量maxCapacity为原始ByteBuf的readableBytes();
 2.duplicate()方法把整个ByteBuf都截取出来包括所有的数据、指针信息;
 3.slice()方法与duplicate()方法的相同点是:底层内存以及引用计数与原始的ByteBuf共享即经过slice()或者duplicate()返回的ByteBuf调用 write系列方法都会影响到原始的ByteBuf,但是它们都维持着与原始 ByteBuf相同的内存引用计数和不同的读写指针;
 4.slice()方法与duplicate()方法的不同点是:slice()只截取从readerIndex到writerIndex之间的数据,返回的ByteBuf最大容量被限制到原始 ByteBuf的readableBytes(), duplicate()是把整个ByteBuf都与原始的 ByteBuf共享;
 5.slice()方法与duplicate()方法不会拷贝数据,它们只是通过改变读写指针来改变读写的行为,copy()直接从原始的ByteBuf拷贝所有的信息包括读写指针以及底层对应的数据,因此往copy()返回的ByteBuf写数据不会影响到原始的ByteBuf;
 6.slice()方法与duplicate()方法不会改变ByteBuf的引用计数,所以原始的ByteBuf调用release()之后发现引用计数为零开始释放内存,调用这两个方法返回的ByteBuf也会被释放,此时如果再对它们进行读写报错.因此通过调用一次retain()方法增加引用,表示它们对应的底层的内存多一次引用,引用计数为2,在释放内存的时候需要调用两次 release()方法将引用计数降到零才会释放内存;
 7.slice()/duplicate()/copy()方法均维护着自身的读写指针,与原始的 ByteBuf的读写指针无关,相互之间不受影响.
 函数体里面只要增加引用计数包括ByteBuf的创建和手动调用 retain()方法必须调用release()方法.多个ByteBuf引用同一段内存通过引用计数来控制内存的释放,遵循谁retain()谁release()的原则.



### 示例代码

```java
public class ByteBufTest {
    public static void main(String[] args) {
        ByteBuf buffer = ByteBufAllocator.DEFAULT.buffer(9, 100);

        print("allocate ByteBuf(9, 100)", buffer);

        // write 方法改变写指针，写完之后写指针未到 capacity 的时候，buffer 仍然可写
        buffer.writeBytes(new byte[]{1, 2, 3, 4});
        print("writeBytes(1,2,3,4)", buffer);

        // write 方法改变写指针，写完之后写指针未到 capacity 的时候，buffer 仍然可写, 写完 int 类型之后，写指针增加4
        buffer.writeInt(12);
        print("writeInt(12)", buffer);

        // write 方法改变写指针, 写完之后写指针等于 capacity 的时候，buffer 不可写
        buffer.writeBytes(new byte[]{5});
        print("writeBytes(5)", buffer);

        // write 方法改变写指针，写的时候发现 buffer 不可写则开始扩容，扩容之后 capacity 随即改变
        buffer.writeBytes(new byte[]{6});
        print("writeBytes(6)", buffer);

        // get 方法不改变读写指针
        System.out.println("getByte(3) return: " + buffer.getByte(3));
        System.out.println("getShort(3) return: " + buffer.getShort(3));
        System.out.println("getInt(3) return: " + buffer.getInt(3));
        print("getByte()", buffer);


        // set 方法不改变读写指针
        buffer.setByte(buffer.readableBytes() + 1, 0);
        print("setByte()", buffer);

        // read 方法改变读指针
        byte[] dst = new byte[buffer.readableBytes()];
        buffer.readBytes(dst);
        print("readBytes(" + dst.length + ")", buffer);

    }

    private static void print(String action, ByteBuf buffer) {
        System.out.println("after ===========" + action + "============");
        System.out.println("capacity(): " + buffer.capacity());
        System.out.println("maxCapacity(): " + buffer.maxCapacity());
        System.out.println("readerIndex(): " + buffer.readerIndex());
        System.out.println("readableBytes(): " + buffer.readableBytes());
        System.out.println("isReadable(): " + buffer.isReadable());
        System.out.println("writerIndex(): " + buffer.writerIndex());
        System.out.println("writableBytes(): " + buffer.writableBytes());
        System.out.println("isWritable(): " + buffer.isWritable());
        System.out.println("maxWritableBytes(): " + buffer.maxWritableBytes());
        System.out.println();
    }
}
```

