---
title: "我自己留着看的 Kotlin 速查表"
slug: f8a9d54607e901b4
date: 2023-01-17T14:24:20Z
image: cover.png
categories:
    - Kotlin
tags:
    - Kotlin
    - JVM
    - 速查表
---

_注：作者居住在韩国，部分内容包含韩国特有的背景。_

- TIP：写数字时可以使用 `var number = 1_000L` 这种写法

### 1. 变量

- 即使是 `val` 集合，也可以添加 element
- 不区分 primitive 和 reference type，但实际运算时会转换为 primitive type 进行运算

### 2. Null 处理

- Safe Call（`?.`）：不为 null 时执行，为 null 时不执行

```kotlin
val str: String? = "ABC"
str.length // ERROR
str?.length // OK
```

- Elvis 运算符（`?:`）：如果前面的运算结果为 null，则使用后面的值

```kotlin
val str: String? = "ABC"
str?.length ?: 0 // str.length 或 0 
str?.length ?: throw IllegalArgumentException("null이 들어왔습니다.") // 或抛出异常
str?.length ?: return 0 // 也可用于 Early return
```

- 平台类型：在 Kotlin 中引入 Java 代码时
    - null、non-null 类型都可以使用
    - 尽量直接确认 Java 代码，或用 Kotlin 进行包装后使用

```kotlin
// 有 @Nullable、@NonNull 时会被转换，没有时则是平台类型
// 在 java 中 
private final String name; << 没有 Annotation 时

val name : String? = javaPerson.name // OK
val name : String = javaPerson.name  // OK
```

### 3. Type

- 基本 Type 转换：基本类型之间使用 `toXX` 之类的函数

```kotlin
val number1 : Int = 4
val number2 : Long = number1.toLong()

val number1 : Int? = 3 
val number2 : Long = number1?.toLong() ?: 0L // 如果是 nullable 变量则灵活处理
```

- 用户自定义 Type：利用智能转换（Smart Cast）

```kotlin
fun printAgeIfPerson(obj: Any){
	// is = Java 的 instanceof
	if(obj is Person){
			// 经过智能转换，obj 的 type 变为 Person
			println(obj.age)
	}
}

// 或者
// 等同于 java 的 (Person) obj
val person = obj as Person

// !is、as? 等也可以使用
// as?：整体为 null
```

- Any：相当于 Java 的 Object + 还包括 Primitive Type 的最顶层
    - 不包含 null，如果想包含 null 请使用 `Any?`
    - 拥有 equals、hashCode、toString
- Unit：类似 void，但 Unit 本身可以作为泛型的类型实参
- Nothing：表示函数不会正常结束

```kotlin
fun fail(message : String): Nothing{
	throw IllegalArgumentException(message)
}
```

- String interpolation

```kotlin
val person = Person(name = "lemon", age = 20)
val log = "이름 : ${person.name}, 나이 : ${person.age}"
```

- String indexing

```kotlin
val str = "ABC"
// 取代 Java 中的 str.get(0)
str[1] //ok
```

### 4. 运算符

- Kotlin 中使用 `>`, `<`, `≥`, `≤` 时会自动调用 `compareTo`
- `==`（值比较，等价性：自动调用 equals），`===`（地址比较，同一性：相同的 0x101 地址）

- `in`、`!in`：是否包含在集合 / 范围内？
- `a..b`：创建从 a 到 b 的范围对象

```kotlin
val numbers = 1..100 // 创建从 1 到 100 的 IntRange 对象
println(1 in numbers) // true
println(0 in numbers) // false

// 以下两个表达式相同
if(0 <= score && score <= 100) {..}
if(score in 0..100)
```

- 运算符重载

```kotlin
data class Money(val amount : Long){
	
	operator fun plus(other : Money): Money{
		return Money(this.amount + other.amount)
	}
}

// 可以像下面这样使用
val sumMoney = money1 + money2
```


### 5. 条件语句

- if 语句在 Kotlin 中是 expression（在 Java 中是 statement）
    - expression：可以求值为一个值的语句

```kotlin
// 在 Java 中不可行，但在 Kotlin 中 if 是 expression 所以 OK 
fun getPassOrFail(score : Int): String{
	return if (score >= 50){ "P" } else { "F" }
}
```

- 在 Java 中三元运算符是 expression，但在 Kotlin 中 if 语句就是 expression，因此不需要三元运算符
    - 因此 Kotlin 没有三元运算符

- when 语句

```kotlin
// 基础用法，代替 Switch
fun getGradeWithSwitch(score: Int): String{
	return when(score / 10){
		9 -> "A"
		8 -> "B"
		7 -> "C"
		else -> "D"
	}
}

// 左侧也可以使用 expression，is Type 等也可以使用
when(score){
in 90..99 -> "A"
in 80..89 -> "B"
...

// 同时检查多个条件
when (number){
1,0,-1 -> println("1,0,-1입니다")
else -> println("아닙니다")
}

// 像条件语句一样使用
fun printOdd(score: Int): Int{
	when{
		score % 2 == 1 -> println("홀수")
		score % 2 == 0 -> println("짝수")
		else -> println("넌머임")
	}
	return 1
}

```

### 6. 循环语句

- forEach

```kotlin
val numbers = listOf(1L, 2L, 3L)
for (number in numbers){
	println(number)
}
```

- Progression：等差数列、Range（Range 继承自 Progression）

```kotlin
val intRange = 1..100
```

### 7. 异常

- try-catch-finally：照常使用（但在 Kotlin 中是 expression，可以使用 return 等）
- 在 Kotlin 中所有错误都是 Unchecked Exception
    - Checked：必须处理的 Exception
    - Unchecked Exception：不一定要处理的..
- try with resources

```kotlin
// JAVA 的 try with resources
// try 结束后会自动关闭。但必须是 AutoCloseable 接口的实现类。 
public void readFile(String path) throws IOException{
	try(BufferedReader reader = new BufferedReader(new FileReader(path))){
		System.out.println(reader.readLine());
	}
}

// 在 Kotlin 中使用，use 的工作方式类似于 try with resources
fun readFile(path: String){
	BufferedReader(FileReader(path)).use { reader ->
		println(reader.readLine())
	}
}
```

### 8. 函数

- 基础

```kotlin
// 如果函数体仅由表达式（Expression）组成，可以使用 =
fun max(a: Int, b: Int) : Int = if(a > b) {a} else {b}
```

- 默认参数

```kotlin
fun repeat(
	str : String,
	num : Int = 3,
	useNewLine : Boolean = true
){
	for(i in 1..num){
	if(useNewLine) { println(str)} else{ print(str) }
	}
}
```

- 命名参数（可以明确指定参数名，类似 builder 一样使用）

```kotlin
repeat(str = "Hello, World", useNewLine = true)
// 这种情况下未指定 num，因此使用默认值（3）
// 不用创建 builder 也可以像 builder 一样使用
```

- 可变参数（vararg）

```kotlin
// Java
public void printAll(String... strings){
	for(String str : strings){
		System.out.println(str);	
	}
}
String[] array = new String[]{"Hello", "World", "!!"};
printAll(array)
printAll("Hello", "World", "!!")

// Kotlin
printAll(vararg strings : String){
	for (str in strings){
		println(str)
	}
}

val array = arrayOf("Hello", "World", "!!")
printAll(*array) // spread 运算符
printAll("Hello", "World", "!!")
```


### 9. 类

- 类与属性

```kotlin
class Person(
	val name:String,
	var age:Int
)

// 像访问属性一样访问 getter、setter
person.age = 10
println(person.age)
```

- 构造函数校验、init

```kotlin
class Person(
	val name:String = "Patrick",
	var age:Int = 1
){
	init{
		if(age <= 0){
			throw IllegalArgumentException("나이는 ${age}일 수 없습니다.")
		}
	}
}
```

- 自定义 getter、Setter

```kotlin
class Person(
	val name:String,
	var age:Int
){
	
	// 像 Property 一样使用，可以用单个 Expression 表示的内容用 = 表示
	val isAdult: Boolean
		get() = this.age >= 20
	
	// 与上面是相同的表达式
	fun isAdult(): Boolean{
		return this.age >= 20
	}
}
```

### 10. 继承

- 抽象类

```kotlin
// 默认是 public class 
abstract class Animal(
	protected val species:String,
	protected val legCount:Int
){
	abstract fun move()
}
```

- 继承的子类

```kotlin
//Java
public class JavaPenguin extends JavaAnimal{
	
	// 子对象中新增的字段
	private final int wingCount;
	
	public JavaPenguin(String species){
		super(species, 2)
		this.wingCount = 2;
	}

	@Override
	public void move(){
		System.out.println("penguin is moving");
	}

	// 重写父类方法的 getter
	@Override
	public int getLegCount(){
		return super.legCount + this.wingCount;
	}
}

//Kotlin
class penguin(
	species : String // 主构造函数
) : Animal(species, 2) // 继承 Animal 并直接调用 Animal 的构造函数 {

	private val wingCount: Int = 2
	
	// 使用关键字代替 Annotation
	override fun move(){
		println("cat is moving")
	}

	override val legCount: Int
		get() = super.legCount + this.wingCount
}

// 但 Kotlin 默认所有类都加了 final 关键字，因此要 override 必须加 open
abstract class Animal(
	protected val species:String,
	protected open val legCount: Int
){
	abstract fun move()
}
```

- 接口

```kotlin
// Java
public interface JavaSwimable{
	default void act(){System.out.println("swim!"); }
}
public interface JavaFlyable{
	default void act(){System.out.println("swim!"); }
}

// Kotlin
interface Swimable{
	// 不需要 default 关键字也可以实现
	fun act(){ println("swim!") } 
}
interface Flyable{
	fun act(){ println("fly!") }
}
```

- 实现接口的类

```kotlin
// Java
public final class JavaPenguin extends JavaAnimal implements JavaFlyable, JavaSwimable{
	
	// 构造函数实现..（省略）
	
	@Override
	public void act(){
		JavaSwimable.super.act();
		JavaFlyable.super.act();
	}
}

// Kotlin
class Penguin(): Animal(species, 2), Swimable, Flyable{
	override fun act(){
		super<Swimable>.act()
		super<Flyable>.act()
	}
}
```

- 可以在 Interface 中声明抽象 Property（但 Interface 不能拥有 Backing Field）

```kotlin
// Kotlin
interface Swimable{
	// 期望子类实现的 property
	val swimAbility: Int 
}

class Penguin(): Swimable{
	// 像下面这样实现
	override val swimAbility: Int
		get() = 3
}
```

- 类继承时的注意事项（初始化执行顺序）
    - 父类的 Constructor、Init 块中不要访问子类的 Field！
    - 父类的 Constructor、Init 块中使用的 Property 不要使用 open！

```kotlin
open class Base(
	open val number : Int = 100
){
	init{
		println("Base Class")
		println(number)
	}
}

class Derived(
	override val number: Int
): Base(number){
	init{
		println("Derived class")
	}
}

// 实例化 Derived 类时的输出
/*
	Output : 
	Base Class
	0
	Derived Class 
*/

// 原因：在父类（Base）调用 number 时，会取子类的 Number。
// 但此时父类（Base）的 constructor 正在执行，子类（Derived）尚未初始化
// 因此输出 0
```

- 关键字整理
    - final（默认，可省略）：阻止 override
    - open：开放 override
    - abstract：必须 override
    - override：重写父类型（在 Java 中是 Annotation，在 Kotlin 中是关键字）

### 11. 访问控制

- Java、Kotlin 的访问控制
    - Public
        - Java/Kotlin：随处可访问
    - Protected
        - Java：同包或子类可访问
        - Kotlin：仅**声明该类**或子类可访问
    - default（Java）／ internal（Kotlin）
        - Java：仅同包可访问
        - Kotlin：仅同 Module 可访问
            - Module：一次编译的 Kotlin 代码（同一个 Gradle 项目等）
    - private
        - Java/Kotlin：仅在声明它的类内部可访问

- Property 访问修饰符

```kotlin
class Car(
	// 将 name 的 getter 设为 internal
	internal val name : String,
	var owner : String,
	_price : Int
){
	// 仅 Setter 设为 private
	var price = _price
		private set
}

```

- Java/Kotlin 混用时的注意事项
    - Internal 关键字在字节码转换（转为 Java 时）会被转换为 public
    - 因此在 Java 代码中可以访问 Kotlin 中的 Internal 关键字
    - Kotlin 的 Protected 也是如此（Java 中同包也可访问）

### 12. object

- static 函数和变量
    - static：静态地在实例间共享值
    - companion object：与类相伴的唯一 object

```kotlin
//Java
public class JavaPerson{
    private static final int MIN_AGE = 1;

    // 静态工厂方法
    public static JavaPerson newBaby(String name){
        return new JavaPerson(name, MIN_AGE);
    }

    private String name;
    private int age;

    private JavaPerson(String name, int age){ this.name = name; this.age = age}
}

//Kotlin
class Person private constructor(
    var name : String,
    var age : Int,
){
		// 用 companion object 代替 Static
    companion object{
				// 有 const 时编译时分配，没有时运行时分配
				// 仅可分配于基本类型 + String
        const val MIN_AGE = 1
        fun newBaby(name: String): Person{
            return Person(name, MIN_AGE)
        }
    }
}

// 可通过 Person.Companion.newBaby("ABC") 调用，
// 或通过 @JvmStatic 注解像 Java Static 一样直接访问

/*
	@JvmStatic
	fun newBaby(name: String) : Person{...}

	之后
	可以使用 Person.newBaby("ABC")
*/
```

- Companion Object 的运用（与 Java 的不同点）
    - 被视为一个对象
    - 因此可以命名或实现 Interface

```kotlin
// Kotlin
interface Log{ fun log() }

class Person private constructor(
    var name : String,
    var age : Int,
){
		// 可以设置名称
    companion object Factory{
        const val MIN_AGE = 1
        fun newBaby(name: String): Person{
            return Person(name, MIN_AGE)
        }
				override fun log(){ println("I am companion object!") }
    }
}

// 如果有名称，可以基于名称访问
// Person.Factory.newBaby("ABC");
```

- Singleton
    - 使用 object 关键字

```kotlin
//Java
public class JavaSingleton{
	private static final JavaSingleton INSTANCE = new JavaSingleton();
	private JavaSingleton(){}

	public static Javasingleton getInstance(){ return INSTANCE; }
}

//Kotlin
object Singleton{
	var a: Int = 0
}

// 之后可以像 Singleton.a 这样使用
```

- 匿名类（Anonymous class）
    - 一次性使用某个 Interface、Class 的实现时

```kotlin
// Java
private static void moveSomething(Movable movable){ movable.move(); movable.fly(); }

// 匿名类实现
moveSomething(new Movable(){
	@Override
	public void move(){System.out.println("move~");}
	@Override
	public void fly(){System.out.println("fly~");}
}

// Kotlin
private fun moveSomething(movable:Movable){ movable.move(); movable.fly();}

// 使用 Object 关键字实现匿名类
moveSomething(object : Movable{
	override fun move(){ println("move~") }
	override fun fly(){println("fly~")}
})
```

### 13. 嵌套类（nested class）

- 感觉用得不多就略过..

### 14. Data class、Enum class、Sealed Class/Interface

- Data class
    - 可以方便地使用 DTO
    - 默认提供 field、constructor、getter、equals、hashCode、toString

```kotlin
//Kotlin
data class PersonDto(val name: String, val age: Int)

// 使用命名参数效果等同于使用 Builder 模式
PersonDto(name="Patrick", age=20)
```

- Enum class

```kotlin
//Java
public enum JavaCountry{
	KOREA("KO"),
	AMERICA("US");

	private final String code;
	JavaCountry(String code) { this.code=code; }
	public String getCode() { return this.code; }
}

// Kotlin（等效代码）
enum class Country(
	private val code: String
){
	KOREA("KO"),
	AMERICA("US");
}

// 利用 When 子句方便地处理
// 此外，这样使用时如果 ENUM CLASS 增加新值，编译器会提醒
fun handleCountry(country: Country){
	when(country){
		Country.KOREA -> LOGIC()
		Country.AMERICA -> LOGIC()
		// 不写 ELSE 也能用！
	}
}
```

- Sealed Class/Interface
    - 想创建一个可继承的抽象类，但不希望外部继承
    - 编译期记住所有子类类型，运行时无法添加其他类类型
    - 子类必须在同一包中
    - 与 Enum 的区别
        - 可以继承类
        - Enum 的实例是单例，但 Sealed Class 可以有多个实例

```kotlin
//Kotlin
sealed class Car(
	val name:String,
	val price:Long
)

class Avante: Car("Avante",1_000L)
class Sonata: Car("Sonata",2_000L)
class Grandeur: Car("Grandeur",3_000L)
// 像上面的 Enum class 一样可以使用 when 子句
```


### 15. Collections（集合）

- Collections
    - 需要设置集合是不可变还是可变
    - Mutable（可变）集合：可添加、删除 element
    - Immutable（不可变）集合：不可添加、删除 element
        - 但即使是不可变集合，也可以修改 Reference Type 元素的字段
        - 例如，访问 Index 1 的数据并将 Price 字段从 1000 改为 2000 韩元是可行的

- List

```kotlin
/* 默认实现：ArrayList */
// 创建不可变 List
val numbers = listOf(100,200)
// 创建可变 List
val mutableNumbers = mutableListOf(100,200)
// 创建空 List
val emptyList = emptyList<Int>()

// 等同于 numbers.get(0)
numbers[0]

for(number in numbers){println(number)}
// 需要同时取得 Index 时
for((idx, number) in numbers.withIndex()){println("${idx}, ${number}")}
```

- Set

```kotlin
/* 默认实现：LinkedHashSet */
val numbers = setOf(100,200)
```

- Map

```kotlin
val weekMap= mutableMapOf<Int, String>()
// 可以像数组一样访问
weekMap[1] = "MONDAY"
weekMap[2] = "TUESDAY"

// 也可使用 to 进行初始化
mapOf(1 to "MONDAY", 2 to "TUESDAY")

for(key in weekMap.keys){println(key) println(weekMap[key]) }
for((key, value) in weekMap.entries){println(key) println(value)} 
```

- 集合的 null 可能性
    - `List<Int?>`：List 本身非 null，列表中可为 null
    - `List<Int>?`：List 本身可为 null，列表中非 null
    - `List<Int?>?`：List 本身和列表中都可为 null

- 与 Java 混用时的注意事项
    - Java 不区分不可变 / 可变 List
        - Kotlin 的不可变 List 在 Java 中可被当作可变 List 使用
        - Kotlin 的 Non-null List 在 Java 中可被添加 null
        - 在 Kotlin 中使用 `Collections.unmodifiable` 或编写防御性逻辑
    - Platform Type 可能引起类型问题

### 16. Extension（扩展）、Infix（中缀）、inline、Local（局部）函数

- 扩展函数
    - Kotlin：以与 Java 100% 兼容为目标
        - 因此希望可以在用 Java 编写的库上添加 Kotlin 代码
        - 像类内部方法一样使用，但代码可以写在外部！
    - 限制：
        - 扩展函数是 public 的，如果取用 private、protected 成员则破坏封装..
            - 为了维持封装性，无法取用 private、protected 成员
        - 成员函数 / 扩展函数的签名（名称）相同时
            - 优先调用成员函数
            - 因此之后增加成员函数时可能会出错

```kotlin
// Kotlin

// 扩展 String 类
fun String.lastChar(){ return this[this.length - 1] }

// 之后在 Kotlin 中可像下面这样使用
val str = "ABC"
println(str.lastChar());
```

- 中缀函数
    - 仅有一个变量、一个参数时
    - 用 `var functionName argument` 替代 `var.functionName(argument)`
    - 即可调用

```kotlin
infix fun Int.add(other: Int): Int{
	return this + other
}

// 可以像下面这样调用
3 add 4
```

- 内联函数
    - 我们熟知的那个内联函数..
    - 可以减少将函数作为参数传递的开销

```kotlin
inline fun Int.add(other: Int): Int{
```

- 局部函数
    - 函数中的函数

```kotlin
// Kotlin
fun createPerson(firstName: String, lastName:String): Person{
	if(firstName.isEmpty()){ throw IllegalArgumentException("에러!!")}
	if(lastName:String.isEmpty()){ throw IllegalArgumentException("에러!!")}
	return Person(firstName, lastName)
}

// 使用 Local 函数
fun createPerson(firstName: String, lastName:String): Person{
	// 函数中的函数
	fun validateName(name: String){if(name.isEmpty()){throw IllegalArgumentException("에러!!")}
	
	validateName(firstName)
	validateName(lastName)
	return Person(firstName, lastName)
}

```

### 17. lambda

- Java 中无法将函数赋值给变量、作为参数传递（不是一等公民）
    - 看起来 `Fruit::isApple` 之类是可行的，但实际上传递的是 Predicate 接口
- Kotlin 中可以将函数赋值给变量、作为参数传递（是一等公民）

- lambda 的创建与调用

```kotlin
// Kotlin

// 函数的 Type 也可以这样设置
val isApple : (Fruit) -> Boolean 
= fun(fruit: Fruit): Boolean {
	return fruit.name == "사과"
}

// 与上面完全相同的代码
val isApple2 = {fruit: Fruit -> fruit.name == "사과" }

val isApple3 = { fruit ->
println("사과만 받는 함수입니다.")
fruit.name == "사과" // 多行 lambda 中省略 return 时，最后一行会自动作为返回值
}

// 调用 lambda，相同代码
isApple(Fruit("사과", 1000))
// 显式调用
isApple.invoke(Fruit("사과", 1000))
```

- 将函数作为参数接收

```kotlin
// 写出函数 Type 即可接收函数
fun filterFruits(fruits: List<Fruit>, filterFunc : (Fruit) -> Boolean) : List<Fruit>{
	val results = mutableListOf<Fruit>()
	for(fruit in fruits){
		if(filterFunc(fruit)){ results.add(fruit)}
	}
	return results
}

// 调用时可像下面这样传递 lambda
filterFruits(fruit, {fruit -> fruit.name =="사과"} )

// 当 lambda 是最后一个参数时，可以像下面这样使用（与上面代码功能相同）
filterFruits(fruit) {fruit -> fruit.name =="사과"}

// 当只有一个参数时，可用 it 替代箭头前的部分
filterFruits(fruit) {it.name =="사과"}
```

- Closure
    - Kotlin 在 lambda 执行时，会捕获所引用的所有变量并持有其值
    - 下方代码捕获了 `targetFruitName = "수박"` 这个时点
    - 这种数据结构称为 Closure，正是这种结构使得 lambda 能够成为一等公民

```kotlin
//Java
String targetFruitName = "바나나"
targetFruitName = "수박"

// 在 Java 中 Lambda 中使用的变量必须是 final 变量， 
// 或虽未加 final 关键字但实际未变更（Effectively final）的变量
filterFruits(fruits, (fruit) -> targetFruitName.equals(fruit.getName())); // ERROR!

//Kotlin
var targetFruitName = "바나나"
targetFruitName = "수박"
filterFruits(fruits) { it.name == targetFruitName } // It Works!

```

### 18. Collection 内置函数 + FP

- **filter**

```kotlin
// 仅取苹果
val apples = fruits.filter { fruit -> fruit.name == "사과" }
```

- **filterIndexed**：filter 时需要 index

```kotlin
val apples = fruits.filterIndexed { idx, fruit -> 
println(idx)
fruit.name == "사과"
} 

```

- **map、mapIndex**：对每个元素执行 lambda 函数

- **mapNotNull**：仅取出 mapping 结果不为 null 的元素

```kotlin
val values = fruits.filter { it.name == "사과" }
	.mapNotNull { it.nullOrValue() } // 只取非 null 的值
```

- **all**：全部满足条件返回 true，否则 false

```kotlin
// 检查是否全部为苹果
val isAllApple = fruits.all { it.name == "사과" }
```

- **none**：全部不满足条件返回 true，否则 false

```kotlin
// 检查是否一个苹果都没有
val isNoApple = fruits.none { it.name =="사과" }
```

- **any**：只要有一个满足就返回 true，否则 false

```kotlin
// 检查是否有任意一个超过一万韩元的水果
val hasExpensiveFruit = fruits.any { it.price > 10_000 }
```

- **count**：数量

```kotlin
// 等同于 Java 的 list.size()
val fruitCount = fruits.count()
```

- **sortedBy、sortedByDescending**：升序、降序排序

```kotlin
val sortedFruit = fruit.sortedBy{it.price} // 升序
val desendSortedFruit = fruit.sortedByDescending{it.price} // 降序
```

- **distinctBy**：基于变换后的值去重

```kotlin
// 检查存在哪些种类的水果
val distinctFruitNames = fruits.distinctBy{ it.name } // 按名称去重
	.map{it.name}
```

- **first、firstOrNull、last、lastOrNull**：第一个、最后一个值
    - 列表为空时 first、last 没有值可取，会抛出 Exception

```kotlin
fruits.first()
fruits.lastOrNull()
```

- **groupBy**：基于某个 Key 进行分组（List → Map）

```kotlin
// 以水果名称为 key 对列表内容分组
// key：사과（苹果），value：{사과1, 사과2, 사과3...}
val map: Map<String, List<Fruit>> = fruits.groupBy{fruit -> fruit.name } 

// 也可同时处理 key、value
// 以名称为 Key、价格 List 为 value 的示例
val map2 : Map<String, List<Long>> = fruits
	.groupBy({it.name}, {it.price})
```

- **associateBy**：基于不重复的 Key 转换为 Map（List→ Map）
    - **但若 id 有重复值，可能丢失数据。**
    - **若 id 重复，最后一个值会成为 value。**

```kotlin
val map: Map<Long, Fruit> = fruits.associateBy { fruit -> fruit.id }

val fruitList = listOf<Fruit>(
			// id 相同
      Fruit(1,"1"),
      Fruit(1,"2"),
      Fruit(1,"3"),
    )

  val associateBy = fruitList.associateBy { it.id }

  println(associateBy.count()) // 1
  for (entry in associateBy) {
      println(entry) // 1=Fruit(id=1, name=3)
  }

// 同样可以同时处理 key、value
val map: Map<Long, Long> = fruits
	.associateBy({it.id}, {it.price})
```

- 在 Map 中也可以使用上面的大部分函数

```kotlin
val map: Map<String, List<Fruit>> = fruits.groupBy{fruit -> fruit.name } 
	.filter { (key, value) -> key == "사과" } 
```

- 嵌套 Collections 的处理
    - flatMap、flatten

```kotlin
var fruitsInList: List<List<Fruit>> = listOf(
	listOf(Fruit(1,"사과"), Fruit(2,"수박"), Fruit(3,"사과")),
	listOf(Fruit(4,"바나나"), Fruit(5,"사과"), Fruit(6,"바나나")),
)

// 不论列表结构，将所有苹果取出来
// output：listOf(Fruit(1,"사과"), Fruit(3,"사과"), Fruit(5,"사과"))
val appleList : List<Fruit>
	= fruitsInList.flatMap { list -> list.filter{it.name == "사과"}

// 将二维列表展平为一维
val flattenList : List<Fruit>
	= fruitsInList.flatten();
```


### 19. TakeIf、scope Function

- **TakeIf**：满足给定条件时返回该值，否则返回 null
- **TakeUnless**：不满足给定条件时返回该值，否则返回 null

```kotlin
// 这个函数
fun getFruitOrNull(fruit: Fruit): Fruit?{
	return if(fruit.name == "사과"){fruit} else {null}
}
// 可以这样使用
val getfruit = fruit.takeIf { it.name == "사과" }
Fruit(1L, "사과") // 这种情况返回 fruit
Fruit(1L, "바나나") // 这种情况返回 null
```

- Scope function：构造临时作用域的函数
    - 即利用 lambda 形成临时作用域
        - 让代码更简洁
        - 或用于 method chaining 的函数

```kotlin
// 以下两个函数相同
fun printPerson(person: Person?){
	if(person != null){
		println(person.name)
		println(person.age)
	}
}

fun printPerson(person: Person?){
	person?.let{
		println(person.name)
		println(person.age)
	}
}
```

- Scope function 的种类
    - **let、run**：返回 lambda 的结果
        - let：使用 it（不能省略，但可以另起名称）
        - run：使用 this（可以省略，但不能另起名称）
    - **also、apply**：返回对象本身
        - also：使用 it
        - apply：使用 this
    - **with**：唯一不是扩展函数的。使用 this，返回 lambda 的结果

```kotlin
val value1 = person.let{ it.age }  // value1 的值：person.age
// 可以像 person.let{ person -> person.age } 一样另起名称
val value2 = person.run{ this.age } // value2 的值：person.age
// 可以像 person.run{ age } 一样省略 this

val value3 = person.also{ it.age } // value3 的值：person
val value4 = person.apply{ this.age } // value4 的值：person 

with(person){
	println(name)
	println(age)
}

/* 
使用 it 的情况：以普通函数为参数 block : (T) -> R : R
使用 this 的情况：以扩展函数为参数 block: T.() -> R : R
有兴趣的话直接看实现吧....
*/
```
