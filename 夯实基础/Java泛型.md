![](https://upload-images.jianshu.io/upload_images/8387919-9d73ce47824dca39.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 泛型概述以及泛型类
- 泛型就是类型参数化，处理的数据类型不是固定的，而是可以作为参数传入；
- 泛型的核心: 告诉编译器想使用什么类型，然后编译器帮你处理一切；
```
public class GenericClass {

    private static class Pair<U,V>{
        private U first;
        private V second;

        public Pair(U first, V second) {
            this.first = first;
            this.second = second;
        }
        public U getFirst() {
            return first;
        }
        public V getSecond() {
            return second;
        }
    }

    public static void main(String[] args){
        Pair<String,Integer>pair = new Pair<>("zhangsan", 23);
    }
}
```
为什么Java不直接使用普通的Object类呢 ？
```
public class GenericClass2 {

    private static class Pair{ //　Generic Class
        private Object first;
        private Object second;

        public Pair(Object first, Object second) {
            this.first = first;
            this.second = second;
        }

        public Object getFirst() {
            return first;
        }

        public Object getSecond() {
            return second;
        }
    }

    public static void main(String[] args){
        Pair pair = new Pair("zhangsan", 23);
        String name = (String) pair.getFirst();
        Integer age = (Integer) pair.getSecond();
    }
}
```
> 其实是可以这样的，而且Java的内部就是这样实现的。
- Java有Java编译器和Java虚拟机，编译器将Java源代码转换为.class文件，虚拟机加载并运行.class文件。
- 对于泛型类，Java编译器会将泛型代码转换为普通的非泛型代码，就像上面的普通Pair类代码及其使用代码一样，将类型参数T擦除，替换为Object，插入必要的强制类型转换。Java虚拟机实际执行的时候，它是不知道泛型这回事的，它只知道普通的类及代码。
- 再次强调，Java泛型是通过擦除实现的，类定义中的类型参数如T会被替换为Object，在程序运行过程中，不知道泛型的实际类型参数，比如Pair<Integer>，运行中只知道Pair，而不知道Integer。

> 那为什么还要使用泛型呢? 泛型有两个好处:
- 更好的安全性；
- 更高的可读性；
### 泛型方法
- 要定义泛型方法，只需要将泛型参数列表置于返回值前；
- 注意: 一个方法是不是泛型的, 和它所在的类是不是泛型没有任何关系；
- 泛型方法调用的时候，不需要指定类型参数的实际类型，Java编译器会推断出来；
```
public class GenericMethod {

    public static <T> int indexOf(T[] arr, T ele){ // Generic Method
        for(int i = 0; i < arr.length; i++){
            if(arr[i].equals(ele))
                return i;
        }
        return -1;
    }

    public static void main(String[] args){
        System.out.println(indexOf(new Integer[]{1, 3, 5, 7}, 5));
        System.out.println(indexOf(new String[]{"zhangsan", "lisi", "wangwu"}, "lisi"));
    }
}
```
实际上泛型类和泛型方法没有联系:
```
//泛型类的泛型和泛型方法的泛型没有一点关系
public class GenericClassMethod<T> {

    public <T> void testMethod(T t){
        System.out.println(t.getClass().getName());
    }

    public <T> T testMethod1(T t){
        return t;
    }
    
    public static void main(String[] args){
        GenericClassMethod<String>gcm = new GenericClassMethod<>();
        gcm.testMethod("generic");
        Integer res = gcm.testMethod1(new Integer(10));
        System.out.println(res);
    }
}

// java.lang.String
// 10
```
### 三、泛型接口
接口也可以是泛型的，例如，Java中的`Comparable`和`Comparator`：
```
public interface Comparable<T> {
    public int compareTo(T o);
}
public interface Comparator<T> {
    int compare(T o1, T o2);
    boolean equals(Object obj);
}
```
### 四、extends、<=、参数类型必须是给定的或者子类型
#### 1、上界为某个具体类
- 可以使用extends来限定一个上界，此时参数类型必须是给定的类型或者其子类型；
- 比如定义一个NumberPair类，限定两个参数类型必须是Number或者子类型，这样限定之后，在子类中，first、second变量就可以当做Number进行处理了，比如调用Number类中的方法doubleValue()、intValue等；
```
// 示例代码，(省略了上面的Pair<U,V>类)
public class GenericExtends {

    private static class NumberPair<U extends Number, V extends Number> extends Pair<U, V>{

        public NumberPair(U first, V second) { // must realize (achieve)
            super(first, second);
        }

        public double getSum(){
            return getFirst().doubleValue() + getSecond().intValue();
        }
    }

    public static void main(String[] args){
        NumberPair<Double, Integer>np = new NumberPair<>(3.3, 3); // <U, V>可以是 Number的子类，即 <=
        System.out.println(np.getSum());
    }
}
```
#### 2.上界为某个接口
在泛型方法中，一种常见的限定类型是必须实现Comparable接口:
- 下面的例子，要进行元素的比较，要求元素必须实现Comparable接口， 所以给类型参数设置了一个上边界Comparable 必须实现Comparable接口；
- 可以理解为： T是一种数据类型，必须实现Comparable,且必须可以与相同类型的元素进行比较；
```
ublic class GenericExtends2 {

    // 要进行元素的比较，要求元素必须实现Comparable接口
    // 所以给类型参数设置了一个上边界Comparable,T 必须实现Comparable接口
    public static <T extends Comparable> T getMax(T[] arr){
        T max = arr[0];
        for(int i = 0; i < arr.length; i++){
            if(arr[i].compareTo(max) > 0){
                max = arr[i];
            }
        }
        return max;
    }

    // 不过上面这么写会有警告 因为Comparable是一个泛型接口，它也需要一个类型参数，所以下面的写法比较好
    // 理解: T是一种数据类型，必须实现Comparable,且必须可以与相同类型的元素进行比较
    public static <T extends Comparable<T> > T getMax2(T[] arr){
        T max = arr[0];
        for(int i = 0; i < arr.length; i++){
            if(arr[i].compareTo(max) > 0){
                max = arr[i];
            }
        }
        return max;
    }
}
```
#### 3.上界为其他参数类型
- 这里模仿ArrayList来创建一个类， 并想着实现其中的addAll()方法，但是如果不使用一个上界的话，会出现无法添加子类的情况，看下面的代码，Number的集合理应可以添加Integer类型的元素。
```
public class GenericExtends3 {

    // seems like ArrayList
    private static class DynamicArray<E>{

        private static final int DEFAULT_CAPACITY = 10;
        private int size;
        private Object[] data;

        public DynamicArray() {
            this.data = new Object[DEFAULT_CAPACITY];
        }
        private void ensureCapacity(int minCapacity){  // simulate ArrayList
            int oldCapacity = data.length;
            if(oldCapacity >= minCapacity)
                return;
            int newCapacity = oldCapacity * 2;
            if(newCapacity < minCapacity) //如果扩展2倍还是小于minCapacity，就直接扩展成为minCapacity
                newCapacity = minCapacity;
            data = Arrays.copyOf(data, newCapacity);
        }

        public void add(E e){
            ensureCapacity(size + 1);
            data[size++] = e;
        }

        public E get(int index){
            return (E)data[index];
        }

        public int size(){
            return size;
        }

        public E set(int index, E e){
            E oldValue = get(index);
            data[index] = e;
            return oldValue;
        }
        
        public void addAll(DynamicArray<E>arr){
            for(int i = 0; i < arr.size; i++){
                add(arr.get(i));
            }
        }
    }

    public static void main(String[] args){
        DynamicArray<Number>numbers = new DynamicArray<>();
        DynamicArray<Integer>ints = new DynamicArray<>();
        ints.add(10);
        ints.add(20);
//        numbers.addAll(ints); // compile error
    }
}
```
那个需求感觉上是可以，但是通过反证法可以发现是行不通的，看下面代码以及解释:
```
DynamicArray<Number>numbers = new DynamicArray<>();
numbers = ints; // 假设合法
numbers.add(new Double(3.3)); // 那么这一样也可以，此时因为numbers和ints指向的同一个堆区空间，则ints中出现double类型值，显然不合理

//再看一个例子
List<Object>olist = null;
List<String>slist = new ArrayList<>();
olist = slist; // err
//如果上述假设合理
olist.add(111);
//则slist中就会出现Integer类型的值，显然不合理
```
所以，可以使用上界类型将addAll方法改进如下:
```
//传入的是T类型，限定为是E类型或者E的子类类型
public <T extends E>void addAll(DynamicArray<T>arr){
    for(int i = 0; i < arr.size; i++){
        add(arr.get(i));
    }
}
```
### 五、通配符?
#### 1、有限定类型通配符的简单使用
```
public void addAll(DynamicArray<? extends E>arr){
    for(int i = 0; i < arr.size; i++){
        add(arr.get(i));
    }
}
```
`<? extends E>`表示有限定通配符，匹配E或E的某个子类型，具体是什么子类型是未知的。 看一下`public <T extends E>void addAll(DynamicArray<T>arr)` 和`public void addAll(DynamicArray<? extends E>arr)`的区别:
- `<T extends E>`用于定义类型参数，它声明了一个类型参数T，可放在泛型类中类名的后面、泛型方法返回值前面；
- `<? extends E>`用于实例化类型参数，它用于实例化泛型变量中的类型参数，只是这个具体类型是未知的，只知道它是E或E的子类型；
#### 2、无限定类型通配符
简单使用: 第一种方式使用通配符，第二种方式使用类型参数，可以达到同样的目的:
```
//使用通配符  
public static int indexOf(DynamicArray<?> arr, Object elm){
    for(int i = 0; i < arr.size(); i++){
        if(arr.get(i).equals(elm))
            return i;
    }
    return -1;
}

//使用类型参数 type parameter
public static <T> int indexOf2(DynamicArray<T> arr, Object elm){
    for(int i = 0; i < arr.size(); i++){
        if(arr.get(i).equals(elm))
            return i;
    }
    return -1;
}
```
但是通配符也有一些限制
- 1)、第一条限制: 只能读，不能写
比如 ，下面三行代码就会报错 :
```
public class WildcardCharacter {
    public static void main(String[] args){
        ArrayList<Integer> ints = new ArrayList<>();
        ArrayList<? extends Number> numbers = ints; // 使用extends通配符指定上界

        Integer a = 10;
//        numbers.add(a); // err
//        numbers.add(Object(a)); //err
//        numbers.add(Number(a)); //err
    }
}
```
- 解释: ?表示类型安全无知，? extends Number表示是Number的某个子类型，但不知道具体子类型，如果允许写入，Java就无法确保类型安全性，所以干脆禁止；
- 这种限制关系是好的，但是这使得很多理应可以完成的操作可能会出现错误；
比如: 下面的代码中最后两行会报错，原因就是不能修改？通配符的值：
```
public static void swap(ArrayList<?> arr, int i, int j){
     Object tmp = arr.get(i);
     arr.set(i, arr.get(j)); // can't change the value
     arr.set(j, tmp);
 }
```
再看一个例子：在方法传递参数的时候，不能往参数中添加元素:
```
public class GenericExtends4 {

	private static class Fruit {}
	private static class Apple extends Fruit{}
	private static class Pear extends Fruit{}
	private static class FuShiApple extends Apple{}

    static class Clazz<T extends Fruit>{  //创建的类必须是Fruit的子类//为了自己类中使用这个类
	}

	public static void main(String[] args) {
		Clazz<Fruit>t = new Clazz<>();  // <= 关系
		Clazz<Apple>t2 = new Clazz<>();
		Clazz<Pear>t3 = new Clazz<>();
		Clazz<FuShiApple>t4= new Clazz<>();


		//调用方法
		List<? extends Fruit> list1 = new ArrayList<>();
		add(list1);
		List<Fruit> list2 = new ArrayList<>();
		add(list2);
		List<Apple> list3 = new ArrayList<>();
		add(list3);
		List<? extends Apple> list4 = new ArrayList<FuShiApple>();  //存放Apple以及它的子类
		add(list4);
		List<FuShiApple> list5 = new ArrayList<>();
		add(list5);


		//？为什么错误 : 因为 ? 等同于？ extends Object :不是<= Fruit的 下面两个是一样的
		List<?>list6 = new ArrayList<>();
		List<? extends Object>list7 = new ArrayList<>();
		//add(list6); // err
		//add(list7); // err
	}

	// 为了保证向下兼容的一致性，不能添加元素
	public static void add(List<? extends Fruit> list) {
		/** 不能往里面加这样的对象 不能用于添加数据
		 list.add(new Fruit());
		 list.add(new Apple());
		 list.add(new Pear());
		 */
		list.add(null);
	}
}
```
- 2)、第二条限制: 参数类型间的依赖关系
如果参数类型之间有依赖关系，也只能用类型参数，比如下面的例子:
```
// S和D要么相同，要么S是D的子类，否则类型不兼容，有编译错误
public static <D,S extends D> void copy(ArrayList<D> dest, ArrayList<S> src){
    for(int i=0; i<src.size(); i++)
        dest.add(src.get(i));
}

// 可以使用通配符简化一下
public static <D> void copy2(ArrayList<D> dest, ArrayList<? extends D> src){
    for(int i=0; i<src.size(); i++)
        dest.add(src.get(i));
}
```
- 3)、第三条限制: 如果返回值依赖于类型参数，也不能用通配符
```
//不能使用通配符，只能用类型参数，因为要返回
public static <T extends Comparable<T> > T max(ArrayList<T> arr){
    T max = arr.get(0);
    for(int i = 1; i < arr.size(); i++){
        if(arr.get(i).compareTo(max)>0){
            max = arr.get(i);
        }
    }
    return max;
}
```
那么到底该用通配符还是类型参数呢?
> - 通配符形式都可以用类型参数的形式来替代，通配符能做的，用类型参数都能做。

> - 通配符形式可以减少类型参数，形式上往往更为简单，可读性也更好，所以，能用通配符的就用通配符。

> - 如果类型参数之间有依赖关系，或者返回值依赖类型参数，或者需要写操作，则只能用类型参数。

通配符形式和类型参数往往配合使用，比如，上面的copy2方法，定义必要的类型参数，使用通配符表达依赖，并接受更广泛的数据类型。
### 六、super、>=、超类型通配符
- 简单的来说，super和extends刚好相反，匹配的是>= E的类型；
- 相当于是规定了一个下界，可以匹配 >=的类型；
#### 1、使用场景
看它的使用场景， 在DynamicArray中添加一个copyTo方法，功能是将当前对象容器中的数拷贝到传入的参数dest容器中:
```
//add current value to the dest collection
 public void copyTo(DynamicArray<E>dest){
     for(int i = 0; i < dest.size(); i++)
         dest.add(this.get(i));
 }
```
然后不使用super，看下面的代码，最后一行就会报错，但是将Integer数组拷贝到Number数组理应是可以的:
```
 public static void main(String[] args){
       DynamicArray<Integer>ints = new DynamicArray<>();
       ints.add(3);
       ints.add(4);

       DynamicArray<Number>nums = new DynamicArray<>();
       ints.copyTo(nums);  // 将ints 中的元素拷贝到nums，本应该是可以的，但是如果没有? super E就不行
   }
```
使用超类型通配符就可以解决上面的问题:
```
public void copyTo(DynamicArray<? super E>dest){
    for(int i = 0; i < dest.size(); i++)
        dest.add(this.get(i));
}

```
#### 2、没有< T super E>(有<T extend E>)
比较类型参数限定与超类型通配符，类型参数限定只有extends形式，没有super形式，比如前面的copyTo方法，它的通配符形式的声明为：
```
public void copyTo(DynamicArray<? super E> dest)
```
如果类型参数限定支持super形式，则应该是：
```
public <T super E> void copyTo(DynamicArray<T> dest)
```
但是，Java并不支持这种语法。对于有限定的通配符形式<? extends E>，可以用类型参数限定替代，但是对于类似上面的超类型通配符，则无法用类型参数替代。
再看和extends使用方法传递参数的对比: (在方法传递中可以添加自己和子类的数据，　区别于extends，extends都不可以添加)
```
public class GenericSuper2 {

    private static class Fruit {}
    private static class Apple extends Fruit{}
    private static class Pear extends Fruit{}
    private static class FuShiApple extends Apple{}

    static class Clazz<T extends Fruit>{  //创建的类必须是Fruit的子类//为了自己类中使用这个类

    }

	public static void main(String[] args) {
		List<Apple>list1 = new ArrayList<>();
		add(list1);
		List<Fruit>list2 = new ArrayList<>();
		add(list2);
		List<Object>list3 = new ArrayList<>();
		add(list3);

		//？super的使用
		List<? super Apple>list4 = new ArrayList<>();
		add(list4);
		List<? super Apple>list5 = new ArrayList<>();
		add(list5);
		
		List<FuShiApple>list6 = new ArrayList<>();  // < 的不行
//		add(list6); // err
		
	}

	//只要是Apple的祖先都可以调用这个方法 >= 
	public static void add(List <? super Apple> list) {  
		/*** 不能用于添加父类对象的数据
		 * list.add(new Fruit());
		 */
		//区别于extends, 可以添加自己和子类的数据
		list.add(new Apple());
		list.add(new FuShiApple());
	}
}
```

### 七、通配符extends、super比较
通配符比较:

- 共同点: 目的都是为了使方法接口更为灵活，可以接受更为广泛的类型。

- <? super E>用于灵活写入或比较，使得对象可以写入父类型的容器(>=)，使得父类型的比较方法可以应用于子类对象。

- <? extends E>用于灵活读取，使得方法可以读取E或E的任意子类型的容器对象。
Java容器类的实现中，有很多这种用法，比如，Collections中就有如下一些方法：
```
public static <T extends Comparable<? super T>> void sort(List<T> list)

public static <T> void sort(List<T> list, Comparator<? super T> c)

public static <T> void copy(List<? super T> dest, List<? extends T> src)

public static <T> T max(Collection<? extends T> coll, Comparator<? super T> comp)
```
### 八、泛型擦除
- 泛型信息只存在于代码编译阶段，在进入 JVM 之前，与泛型相关的信息会被擦除掉，专业术语叫做类型擦除。

- 通俗地讲，泛型类和普通类在 java 虚拟机内是没有什么特别的地方；
看下面代码：
```
public class GenericWipe {
    public static void main(String[] args){

        List<String> slist = new ArrayList<>();
        List<Integer> ilist = new ArrayList<>();

        System.out.println(slist.getClass() == ilist.getClass());
    }
}
```
这段代码的输出结果是true。正如一开始说的，编译器会将T擦除，然后替换成为Object(并不完全正确)，在必要的时候进行强制类型转换。

再看以下代码的输出结果:
```
public class GenericWipe<T> {

    private T obj;

    public GenericWipe(T obj){
        this.obj = obj;
    }

    public static void main(String[] args){
        GenericWipe<String>gw = new GenericWipe<>("wipe");
        Class gwClass = gw.getClass();
        System.out.println(gwClass.getName()); // 得到运行时的状态信息,运行时是真实的类型

        System.out.println("--------------------------");

        Field[] fs = gwClass.getDeclaredFields();  //得到在JVM中的类型 
        for ( Field f:fs)
            System.out.println("Field name " + f.getName() + " type:" + f.getType().getName());
    }
}
```
```
JavaPrimary.Generic.GenericWipe
--------------------------
Field name obj type:java.lang.Object
```
第一种类型是`Class` 的类型是 `GenericWipe`，并不是 `GenericWipe<T>` 这种形式，第二种类型是`Jvm`中的类型； 那是不是泛型类被类型擦除后，相应的类型就被替换成 `Object` 类型呢？这种说法不是完全正确的。

更改一下代码:
```
public class GenericWipe<T extends String> {  // <= String

    private T obj;

    public GenericWipe(T obj){
        this.obj = obj;
    }

    public static void main(String[] args){
        GenericWipe<String>gw = new GenericWipe<>("wipe");
        Class gwClass = gw.getClass();
        System.out.println(gwClass.getName()); // 得到运行时的状态信息,运行时是真实的类型

        System.out.println("--------------------------");

        Field[] fs = gwClass.getDeclaredFields();  //得到在JVM中的类型
        for ( Field f:fs)
            System.out.println("Field name " + f.getName() + " type:" + f.getType().getName());
    }
}
```
输出结果：
```
JavaPrimary.Generic.GenericWipe
--------------------------
Field name obj type:java.lang.String
```
可以看到，第二个输出变成了String。所以结论如下:
- 在泛型类被类型擦除的时候，之前泛型类中的类型参数部分如果没有指定上限，如<T>则会被转译成普通的 Object 类型；
- 如果指定了上限如 <T extends String> 则类型参数就被替换成类型上限。
所以，在反射中，add() 这个方法对应的 Method 的签名应该是 Object.class。也就是说，如果你要在反射中找到 add 对应的 Method，你应该调用 getDeclaredMethod("add",Object.class) 否则程序会报错，提示没有这么一个方法，原因就是类型擦除的时候，T 被替换成 Object 类型了。
```
public class GenericWipe<T> {  // <= String
    // public class GenericWipe<T extends String> {  // <= String
    private T obj;

    public GenericWipe(T obj){
        this.obj = obj;
    }

    public void add(T obj){

    }

    public static void main(String[] args){
        GenericWipe<String>gw = new GenericWipe<>("wipe");
        Class gwClass = gw.getClass();
        System.out.println(gwClass.getName()); // 得到运行时的状态信息,运行时是真实的类型

        System.out.println("--------------------------");
        Method[] methods = gwClass.getDeclaredMethods();
        for ( Method m:methods ){
            System.out.println(" method:" + m.toString());
        }
    }
}
```
[更加详细的解释见这里]([https://blog.csdn.net/briblue/article/details/76736356#t11](https://blog.csdn.net/briblue/article/details/76736356#t11)
)
### 九、泛型注意事项
#### 1、基本类型不能用于实例化类型参数，也就是泛型类或者泛型方法中，不接受 8 种基本数据类型。
比如:
```
List<int> li = new ArrayList<>(); // err
List<boolean> li = new ArrayList<>(); // err
List<Integer> li = new ArrayList<>();  //ok
List<Boolean> li1 = new ArrayList<>(); // ok
```
#### 2、运行时类型信息不适用于泛型
这个也就是上面说的泛型擦除，泛型不支持运行时的信息(和反射有关)。

instanceof后面是接口或类名，instanceof是运行时判断，也与泛型无关，所以，Java也不支持类似如下写法：
```
if(p1 instanceof Pair<Integer>)
```
#### 3、Java 不能创建具体类型的泛型数组
例如下面的list1和list2创建是错误的，但是后面的?可以，因为?代表的是未知类型:
```
public class GenericOther {
    public static void main(String[] args){
//        List<Integer>[] list1 = new ArrayList<Integer>[]; // complier err
//        List<Boolean> list2 = new ArrayList<Boolean>[]; // complier err
        List<?>[] list3 = new ArrayList<?>[10]; // 这个却可以 ? 代表的是未知类型
        list3[1] = new ArrayList<String>();
        List<?> tmp = list3[1];
        System.out.println(tmp.get(0));
//        tmp.set(1, 2); complier err
    }
}
```
- `List<Boolean>` 和 `List<Boolean>` 在 Jvm 中等同于`List<Object>` ，所有的类型信息都被擦除，程序也无法分辨一个数组中的元素类型具体是 `List<Integer>`类型还是 `List<Boolean>` 类型。
-  `？` 代表未知类型，涉及的操作都基本上与类型无关，Jvm 不针对它对类型作判断，因此它能编译通过，但是，它只能读，不能写。比如，上面的 tmp 这个局部变量，它只能进行 get() 操作，不能进行 add() 操作。
再从如果可以创建泛型数组会出现什么样的问题来看: 数组可以进行不同类型之间的转换，但是也需要注意使用，使用不当就会造成运行时异常，而如果运行创建泛型数组也会产生类似的问题，所以Java干脆禁止。
```
public class NoGenericClassArray {
     private static class Pair { //　Generic Class
        private Object first;
        private Object second;
        public Pair(Object first, Object second) {
            this.first = first;
            this.second = second;
        }
        public Object getFirst() {
            return first;
        }
        public Object getSecond() {
            return second;
        }
    }
    public static void main(String[] args) {

        // 数组是Java直接支持的概念，它知道数组元素的实际类型，
        // 它知道Object和Number都是Integer的父类型，所以这个操作是允许的。
        Integer[] ints = new Integer[10];
        Number[] numbers = ints; //  is ok
        Object[] objs = ints;

        // 虽然Java允许这种转换，但是如果使用不恰当，就有可能引起运行时异常
        Integer[] ints2 = new Integer[10];
        Object[] objs2 = ints2;
        objs2[0] = "hello"; // RuntimeException

//        Pair<Object, Integer>[] options = new Pair<Object, Integer>[3]; //如果可以，那最后一行就会不会编译错误，这样显然是不行的
//        Object[] objs = options;
//        objs[0] = new Pair<Double, String>(12.34, "hello");
        
    }
}
```
#### 4、不能通过类型参数创建对象
下面的写法是非法的。
```
T elm = new T();
T[] arr = new T[10];
```
如果允许，本来你以为创建的就是对应类型的对象，但由于类型擦除，Java只能创建Object类型的对象，而无法创建T类型的对象。 那如果确实希望根据类型创建对象呢？需要设计API接受类型对象，即Class对象，并使用Java中的反射机制，如果类型有默认构造方法，可以调用Class的newInstance方法构建对象：
```
public static <T> T create(Class<T> type){
    try {
        return type.newInstance();
    } catch (Exception e) {
        return null;
    }
}
```
#### 5、泛型类类型参数不能用于静态变量和方法，泛型类中泛型只能用在成员变量上，只能使用引用类型，在接口中泛型只能只能用在抽象方法中，全局常量不能使用泛型
对于泛型类声明的类型参数，可以在实例变量和方法中使用，但在静态变量和静态方法中是不能使用的。下面的写法是非法的:
```
public class Singleton<T> {
    private static T instance;
    public synchronized static T getInstance(){
        if(instance==null){
             // 创建实例
        }
        return instance;
    }
}    
```
如果合法的话，那么对于每种实例化类型，都需要有一个对应的静态变量和方法。但由于类型擦除，Singleton类型只有一份，静态变量和方法都是类型的属性，且与类型参数无关，所以不能使用泛型类类型参数。
但是，对于静态方法，它可以是泛型方法，可以声明自己的类型参数，这个参数与泛型类的类型参数是没有关系的。
#### 6、子类继承父类泛型
注意子类继承泛型的注意事项: 可以有四种方式，可以按需实现，或者定义子类自己的泛型等
```
public class SubClass {

    public abstract class Father<T1,T2> {     //注意实际过程中一般定义为抽象的父类
        T1 age;
        public abstract void test(T2 name);  //抽象方法
    }

    //1)全部保留
    class C1<T1,T2,A,B> extends Father<T1,T2>{  //除了继承父类,可以自己"加""富二代"(不是负二代)

        @Override
        public void test(T2 name) {
            // this.age -->  T1类型
        }
    }

    //2)部分保留
    class C2<T2,A,B> extends Father<Integer,T2>{
        @Override
        public void test(T2 name) {
            // this.age -->  Integer类型
        }
    }

    //不保留: -->按需实现
    class C3<A,B> extends Father<Integer,String>{
        @Override
        public void test(String name) { //注意这里是String 不是T2
            // this.age -->  Integer类型
        }
    }

    //2)没有类型 : 擦除 (类似于Object)//相当于
    class C4<A,B> extends Father {  //相当于  class C4<A,B> extends Father<Object,Object>{}

        @Override
        public void test(Object name) { //注意这里是Object(完全没有类型(擦除))
            // this.age -->  Object类型
        }
    }
}
```