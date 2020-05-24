### 核心语法

##### 条件表达式

~~~scala
if(x>0){
  true
} else{
  false
}
~~~

##### 循环表达式

~~~scala
1 to 10
Range(1,10)
Range(1,10,2) //调整步长为2  不能为0
1 until 10  1.until(10)
~~~

### 面向对象

- 类的定义
- 主构造器和附属构造器
- 集成和重写
- 抽象类
- 伴生类和伴生对象

> 同名的calss和object

- apply方法

- case calss  

>  不用new 通常用在模式匹配中
- trait 类似接口

### Scala集合

- 数组（可变/不可变）

~~~scala
var a = new Array[String](5)
a(0) = "hello"

val b = Array("hadoop","spark","strom") //实际上是apply方法

val d = scala.collection.mutable.ArrayBuffer[Int]()
~~~



- List
- Set
- Map
- OPtion Some None
- Tuple

