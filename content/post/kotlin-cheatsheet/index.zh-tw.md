---
title: "我自己留著看的 Kotlin 速查表"
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

_註：筆者居住於韓國，部分內容包含韓國特有的背景。_

- TIP：寫數字時可以使用 `var number = 1_000L` 這種寫法

### 1. 變數

- 即使是 `val` 集合，也可以加入 element
- 不區分 primitive 與 reference type，但實際運算時會轉換為 primitive type 進行運算

### 2. Null 處理

- Safe Call（`?.`）：不為 null 時執行，為 null 時不執行

```kotlin
val str: String? = "ABC"
str.length // ERROR
str?.length // OK
```

- Elvis 運算子（`?:`）：若前面的運算結果為 null，則使用後面的值

```kotlin
val str: String? = "ABC"
str?.length ?: 0 // str.length 或 0 
str?.length ?: throw IllegalArgumentException("null이 들어왔습니다.") // 或拋出例外
str?.length ?: return 0 // 也可用於 Early return
```

- 平台型別：在 Kotlin 中引入 Java 程式碼時
    - null、non-null 兩種型別都可以使用
    - 盡量直接確認 Java 程式碼，或以 Kotlin 包裝後使用

```kotlin
// 有 @Nullable、@NonNull 時會被轉換，沒有時則為平台型別
// 在 java 中 
private final String name; << 沒有 Annotation 時

val name : String? = javaPerson.name // OK
val name : String = javaPerson.name  // OK
```

### 3. Type

- 基本 Type 轉換：基本型別之間使用 `toXX` 之類的函式

```kotlin
val number1 : Int = 4
val number2 : Long = number1.toLong()

val number1 : Int? = 3 
val number2 : Long = number1?.toLong() ?: 0L // 若是 nullable 變數則靈活處理
```

- 自訂 Type：利用智慧轉型（Smart Cast）

```kotlin
fun printAgeIfPerson(obj: Any){
	// is = Java 的 instanceof
	if(obj is Person){
			// 經過智慧轉型，obj 的 type 變為 Person
			println(obj.age)
	}
}

// 或者
// 等同於 java 的 (Person) obj
val person = obj as Person

// !is、as? 等也可以使用
// as?：整體為 null
```

- Any：相當於 Java 的 Object + 還包含 Primitive Type 的最頂層
    - 不包含 null，若想包含 null 請使用 `Any?`
    - 具有 equals、hashCode、toString
- Unit：類似 void，但 Unit 本身可作為泛型的型別實參
- Nothing：表示函式不會正常結束

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

### 4. 運算子

- Kotlin 中使用 `>`, `<`, `≥`, `≤` 時會自動呼叫 `compareTo`
- `==`（值比較，等價性：自動呼叫 equals），`===`（位址比較，同一性：相同的 0x101 位址）

- `in`、`!in`：是否包含於集合 / 範圍中？
- `a..b`：建立從 a 到 b 的範圍物件

```kotlin
val numbers = 1..100 // 建立從 1 到 100 的 IntRange 物件
println(1 in numbers) // true
println(0 in numbers) // false

// 以下兩個式子相同
if(0 <= score && score <= 100) {..}
if(score in 0..100)
```

- 運算子多載

```kotlin
data class Money(val amount : Long){
	
	operator fun plus(other : Money): Money{
		return Money(this.amount + other.amount)
	}
}

// 可以像下面這樣使用
val sumMoney = money1 + money2
```


### 5. 條件式

- if 在 Kotlin 中是 expression（在 Java 中是 statement）
    - expression：可以求值為一個值的陳述

```kotlin
// 在 Java 中不可行，但在 Kotlin 中 if 是 expression 所以 OK 
fun getPassOrFail(score : Int): String{
	return if (score >= 50){ "P" } else { "F" }
}
```

- 在 Java 中三元運算子是 expression，但在 Kotlin 中 if 本身就是 expression，因此不需要三元運算子
    - 因此 Kotlin 沒有三元運算子

- when

```kotlin
// 基本用法，代替 Switch
fun getGradeWithSwitch(score: Int): String{
	return when(score / 10){
		9 -> "A"
		8 -> "B"
		7 -> "C"
		else -> "D"
	}
}

// 左側也可以使用 expression，is Type 等也可以使用
when(score){
in 90..99 -> "A"
in 80..89 -> "B"
...

// 同時檢查多個條件
when (number){
1,0,-1 -> println("1,0,-1입니다")
else -> println("아닙니다")
}

// 像條件式一樣使用
fun printOdd(score: Int): Int{
	when{
		score % 2 == 1 -> println("홀수")
		score % 2 == 0 -> println("짝수")
		else -> println("넌머임")
	}
	return 1
}

```

### 6. 迴圈

- forEach

```kotlin
val numbers = listOf(1L, 2L, 3L)
for (number in numbers){
	println(number)
}
```

- Progression：等差數列、Range（Range 繼承自 Progression）

```kotlin
val intRange = 1..100
```

### 7. 例外

- try-catch-finally：照常使用（但在 Kotlin 中是 expression，可以使用 return 等）
- 在 Kotlin 中所有錯誤都是 Unchecked Exception
    - Checked：必須處理的 Exception
    - Unchecked Exception：不一定要處理的..
- try with resources

```kotlin
// JAVA 的 try with resources
// try 結束後會自動關閉。但必須是 AutoCloseable 介面的實作類別。 
public void readFile(String path) throws IOException{
	try(BufferedReader reader = new BufferedReader(new FileReader(path))){
		System.out.println(reader.readLine());
	}
}

// 在 Kotlin 中，use 的運作方式類似於 try with resources
fun readFile(path: String){
	BufferedReader(FileReader(path)).use { reader ->
		println(reader.readLine())
	}
}
```

### 8. 函式

- 基本

```kotlin
// 若函式主體僅由表示式（Expression）組成，可以使用 =
fun max(a: Int, b: Int) : Int = if(a > b) {a} else {b}
```

- 預設參數

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

- 具名引數（可以明確指定參數名稱，類似 builder 一樣使用）

```kotlin
repeat(str = "Hello, World", useNewLine = true)
// 此情況下 num 未指定，因此使用預設值（3）
// 不必建立 builder 也可以像 builder 一樣使用
```

- 可變參數（vararg）

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
printAll(*array) // spread 運算子
printAll("Hello", "World", "!!")
```


### 9. 類別

- 類別與屬性

```kotlin
class Person(
	val name:String,
	var age:Int
)

// 像存取屬性一樣存取 getter、setter
person.age = 10
println(person.age)
```

- 建構子驗證、init

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

- 自訂 getter、Setter

```kotlin
class Person(
	val name:String,
	var age:Int
){
	
	// 像 Property 一樣使用，可用單一 Expression 表達的內容以 = 表示
	val isAdult: Boolean
		get() = this.age >= 20
	
	// 與上面相同的表達式
	fun isAdult(): Boolean{
		return this.age >= 20
	}
}
```

### 10. 繼承

- 抽象類別

```kotlin
// 預設為 public class 
abstract class Animal(
	protected val species:String,
	protected val legCount:Int
){
	abstract fun move()
}
```

- 繼承的子類別

```kotlin
//Java
public class JavaPenguin extends JavaAnimal{
	
	// 子物件中新增的欄位
	private final int wingCount;
	
	public JavaPenguin(String species){
		super(species, 2)
		this.wingCount = 2;
	}

	@Override
	public void move(){
		System.out.println("penguin is moving");
	}

	// 覆寫父類別方法的 getter
	@Override
	public int getLegCount(){
		return super.legCount + this.wingCount;
	}
}

//Kotlin
class penguin(
	species : String // 主建構子
) : Animal(species, 2) // 繼承 Animal 並直接呼叫 Animal 的建構子 {

	private val wingCount: Int = 2
	
	// 使用關鍵字代替 Annotation
	override fun move(){
		println("cat is moving")
	}

	override val legCount: Int
		get() = super.legCount + this.wingCount
}

// 不過 Kotlin 預設所有類別都加上了 final 關鍵字，因此要 override 必須加 open
abstract class Animal(
	protected val species:String,
	protected open val legCount: Int
){
	abstract fun move()
}
```

- 介面

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
	// 不需要 default 關鍵字也可以實作
	fun act(){ println("swim!") } 
}
interface Flyable{
	fun act(){ println("fly!") }
}
```

- 實作介面的類別

```kotlin
// Java
public final class JavaPenguin extends JavaAnimal implements JavaFlyable, JavaSwimable{
	
	// 建構子實作..（省略）
	
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

- 可在 Interface 中宣告抽象 Property（但 Interface 不能擁有 Backing Field）

```kotlin
// Kotlin
interface Swimable{
	// 期望子類別實作的 property
	val swimAbility: Int 
}

class Penguin(): Swimable{
	// 像下面這樣實作
	override val swimAbility: Int
		get() = 3
}
```

- 類別繼承時的注意事項（初始化執行順序）
    - 在父類別的 Constructor、Init 區塊中，不要存取子類別的 Field！
    - 父類別的 Constructor、Init 區塊中使用的 Property，不要使用 open！

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

// 將 Derived 類別實例化時的輸出
/*
	Output : 
	Base Class
	0
	Derived Class 
*/

// 原因：父類別（Base）呼叫 number 時，會取到子類別的 Number。
// 但此時父類別（Base）的 constructor 正在執行，子類別（Derived）尚未初始化
// 因此輸出 0
```

- 關鍵字整理
    - final（預設，可省略）：阻止 override
    - open：開放 override
    - abstract：必須 override
    - override：覆寫父型別（在 Java 中是 Annotation，在 Kotlin 中是關鍵字）

### 11. 存取控制

- Java、Kotlin 的存取控制
    - Public
        - Java/Kotlin：任何位置皆可存取
    - Protected
        - Java：同套件或子類別可存取
        - Kotlin：僅**宣告該類別**或子類別可存取
    - default（Java）／ internal（Kotlin）
        - Java：僅同套件可存取
        - Kotlin：僅同 Module 可存取
            - Module：一次編譯的 Kotlin 程式碼（同一個 Gradle 專案等）
    - private
        - Java/Kotlin：僅在宣告它的類別內部可存取

- Property 存取修飾子

```kotlin
class Car(
	// 將 name 的 getter 設為 internal
	internal val name : String,
	var owner : String,
	_price : Int
){
	// 僅 Setter 設為 private
	var price = _price
		private set
}

```

- Java/Kotlin 混用時的注意事項
    - Internal 關鍵字在位元組碼轉換時（轉為 Java 時）會被轉換為 public
    - 因此在 Java 程式碼中，可以存取 Kotlin 中的 Internal 關鍵字
    - Kotlin 的 Protected 也是如此（Java 中同套件也可存取）

### 12. object

- static 函式與變數
    - static：靜態地在實例間共享值
    - companion object：與類別相伴的唯一 object

```kotlin
//Java
public class JavaPerson{
    private static final int MIN_AGE = 1;

    // 靜態工廠方法
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
		// 用 companion Object 代替 Static
    companion object{
				// 有 const 時於編譯時配置，沒有時於執行階段配置
				// 僅可配置於基本型別 + String
        const val MIN_AGE = 1
        fun newBaby(name: String): Person{
            return Person(name, MIN_AGE)
        }
    }
}

// 可透過 Person.Companion.newBaby("ABC") 呼叫，
// 或透過 @JvmStatic 註解像 Java Static 一樣直接存取

/*
	@JvmStatic
	fun newBaby(name: String) : Person{...}

	之後
	可以使用 Person.newBaby("ABC")
*/
```

- Companion Object 的運用（與 Java 的差異點）
    - 被視為一個物件
    - 因此可以命名或實作 Interface

```kotlin
// Kotlin
interface Log{ fun log() }

class Person private constructor(
    var name : String,
    var age : Int,
){
		// 可以設定名稱
    companion object Factory{
        const val MIN_AGE = 1
        fun newBaby(name: String): Person{
            return Person(name, MIN_AGE)
        }
				override fun log(){ println("I am companion object!") }
    }
}

// 若有名稱，可基於名稱存取
// Person.Factory.newBaby("ABC");
```

- Singleton
    - 使用 object 關鍵字

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

// 之後可像 Singleton.a 這樣使用
```

- 匿名類別（Anonymous class）
    - 一次性使用某個 Interface、Class 的實作時

```kotlin
// Java
private static void moveSomething(Movable movable){ movable.move(); movable.fly(); }

// 匿名類別實作
moveSomething(new Movable(){
	@Override
	public void move(){System.out.println("move~");}
	@Override
	public void fly(){System.out.println("fly~");}
}

// Kotlin
private fun moveSomething(movable:Movable){ movable.move(); movable.fly();}

// 使用 Object 關鍵字實作匿名類別
moveSomething(object : Movable{
	override fun move(){ println("move~") }
	override fun fly(){println("fly~")}
})
```

### 13. 巢狀類別（nested class）

- 感覺用得不多就略過..

### 14. Data class、Enum class、Sealed Class/Interface

- Data class
    - 可以方便地使用 DTO
    - 預設提供 field、constructor、getter、equals、hashCode、toString

```kotlin
//Kotlin
data class PersonDto(val name: String, val age: Int)

// 使用具名引數的效果等同於使用 Builder 模式
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

// Kotlin（等效程式碼）
enum class Country(
	private val code: String
){
	KOREA("KO"),
	AMERICA("US");
}

// 利用 When 子句方便地處理
// 此外，這樣使用時若 ENUM CLASS 增加新值，編譯器會提醒
fun handleCountry(country: Country){
	when(country){
		Country.KOREA -> LOGIC()
		Country.AMERICA -> LOGIC()
		// 不寫 ELSE 也能使用！
	}
}
```

- Sealed Class/Interface
    - 想建立可繼承的抽象類別，但不希望外部繼承
    - 編譯時記住所有子類別型別，執行階段無法新增其他類別型別
    - 子類別必須位於同一套件
    - 與 Enum 的差異
        - 可以繼承類別
        - Enum 的實例為單例，但 Sealed Class 可有多個實例

```kotlin
//Kotlin
sealed class Car(
	val name:String,
	val price:Long
)

class Avante: Car("Avante",1_000L)
class Sonata: Car("Sonata",2_000L)
class Grandeur: Car("Grandeur",3_000L)
// 像上面的 Enum class 一樣可以使用 when 子句
```


### 15. Collections（集合）

- Collections
    - 需設定集合是不可變或可變
    - Mutable（可變）集合：可新增、刪除 element
    - Immutable（不可變）集合：無法新增、刪除 element
        - 但即使是不可變集合，也可以變更 Reference Type 元素的欄位
        - 例如，存取 Index 1 的資料並將 Price 欄位從 1000 改為 2000 韓元是可行的

- List

```kotlin
/* 預設實作：ArrayList */
// 建立不可變 List
val numbers = listOf(100,200)
// 建立可變 List
val mutableNumbers = mutableListOf(100,200)
// 建立空 List
val emptyList = emptyList<Int>()

// 等同於 numbers.get(0)
numbers[0]

for(number in numbers){println(number)}
// 需要同時取得 Index 時
for((idx, number) in numbers.withIndex()){println("${idx}, ${number}")}
```

- Set

```kotlin
/* 預設實作：LinkedHashSet */
val numbers = setOf(100,200)
```

- Map

```kotlin
val weekMap= mutableMapOf<Int, String>()
// 可以像陣列一樣存取
weekMap[1] = "MONDAY"
weekMap[2] = "TUESDAY"

// 也可使用 to 進行初始化
mapOf(1 to "MONDAY", 2 to "TUESDAY")

for(key in weekMap.keys){println(key) println(weekMap[key]) }
for((key, value) in weekMap.entries){println(key) println(value)} 
```

- 集合的 null 可能性
    - `List<Int?>`：List 本身非 null，列表內可為 null
    - `List<Int>?`：List 本身可為 null，列表內非 null
    - `List<Int?>?`：List 本身與列表內都可為 null

- 與 Java 混用時的注意事項
    - Java 不區分不可變 / 可變 List
        - Kotlin 的不可變 List 在 Java 中可被當作可變 List 使用
        - Kotlin 的 Non-null List 在 Java 中可被加入 null
        - 在 Kotlin 中使用 `Collections.unmodifiable` 或撰寫防禦性邏輯
    - Platform Type 可能引起型別問題

### 16. Extension（擴充）、Infix（中置）、inline、Local（區域）函式

- 擴充函式
    - Kotlin：以與 Java 100% 相容為目標
        - 因此希望可以在用 Java 寫成的函式庫上加入 Kotlin 程式碼
        - 像類別內部方法一樣使用，但程式碼可寫在外部！
    - 限制：
        - 擴充函式為 public，若取用 private、protected 成員會破壞封裝..
            - 為了維持封裝性，無法取用 private、protected 成員
        - 成員函式 / 擴充函式的簽章（名稱）相同時
            - 優先呼叫成員函式
            - 因此之後新增成員函式時，可能會出現錯誤

```kotlin
// Kotlin

// 擴充 String 類別
fun String.lastChar(){ return this[this.length - 1] }

// 之後在 Kotlin 中可像下面這樣使用
val str = "ABC"
println(str.lastChar());
```

- 中置函式
    - 僅有一個變數、一個引數時
    - 以 `var functionName argument` 取代 `var.functionName(argument)`
    - 即可呼叫

```kotlin
infix fun Int.add(other: Int): Int{
	return this + other
}

// 可以像下面這樣呼叫
3 add 4
```

- 內聯函式
    - 我們所熟知的那個內聯函式..
    - 可減少將函式當作參數傳遞的開銷

```kotlin
inline fun Int.add(other: Int): Int{
```

- 區域函式
    - 函式內的函式

```kotlin
// Kotlin
fun createPerson(firstName: String, lastName:String): Person{
	if(firstName.isEmpty()){ throw IllegalArgumentException("에러!!")}
	if(lastName:String.isEmpty()){ throw IllegalArgumentException("에러!!")}
	return Person(firstName, lastName)
}

// 使用 Local 函式
fun createPerson(firstName: String, lastName:String): Person{
	// 函式內的函式
	fun validateName(name: String){if(name.isEmpty()){throw IllegalArgumentException("에러!!")}
	
	validateName(firstName)
	validateName(lastName)
	return Person(firstName, lastName)
}

```

### 17. lambda

- 在 Java 中無法將函式指派給變數、作為參數傳遞（不是一級物件）
    - 看起來 `Fruit::isApple` 之類是可行的，但實際上傳遞的是 Predicate 介面
- 在 Kotlin 中可將函式指派給變數、作為參數傳遞（是一級物件）

- lambda 的建立與呼叫

```kotlin
// Kotlin

// 函式的 Type 也可以這樣設定
val isApple : (Fruit) -> Boolean 
= fun(fruit: Fruit): Boolean {
	return fruit.name == "사과"
}

// 與上面完全相同的程式碼
val isApple2 = {fruit: Fruit -> fruit.name == "사과" }

val isApple3 = { fruit ->
println("사과만 받는 함수입니다.")
fruit.name == "사과" // 多行 lambda 中省略 return 時，最後一行會自動成為回傳值
}

// 呼叫 lambda，相同程式碼
isApple(Fruit("사과", 1000))
// 顯式呼叫
isApple.invoke(Fruit("사과", 1000))
```

- 將函式作為參數接收

```kotlin
// 寫出函式 Type 即可接收函式
fun filterFruits(fruits: List<Fruit>, filterFunc : (Fruit) -> Boolean) : List<Fruit>{
	val results = mutableListOf<Fruit>()
	for(fruit in fruits){
		if(filterFunc(fruit)){ results.add(fruit)}
	}
	return results
}

// 呼叫時可像下面這樣傳遞 lambda
filterFruits(fruit, {fruit -> fruit.name =="사과"} )

// 此時 lambda 若是最後一個參數，可以像下面這樣使用（與上面程式碼功能相同）
filterFruits(fruit) {fruit -> fruit.name =="사과"}

// 只有一個參數時，可用 it 取代箭頭前的部分
filterFruits(fruit) {it.name =="사과"}
```

- Closure
    - Kotlin 在 lambda 執行時，會擷取所引用的所有變數並持有其值
    - 下方程式碼擷取了 `targetFruitName = "수박"` 此時點
    - 這種資料結構稱為 Closure，正是這種結構使得 lambda 能成為一級物件

```kotlin
//Java
String targetFruitName = "바나나"
targetFruitName = "수박"

// 在 Java 中 Lambda 中使用的變數必須是 final 變數， 
// 或雖未加 final 關鍵字但實際未被變更（Effectively final）的變數
filterFruits(fruits, (fruit) -> targetFruitName.equals(fruit.getName())); // ERROR!

//Kotlin
var targetFruitName = "바나나"
targetFruitName = "수박"
filterFruits(fruits) { it.name == targetFruitName } // It Works!

```

### 18. Collection 內建函式 + FP

- **filter**

```kotlin
// 僅取蘋果
val apples = fruits.filter { fruit -> fruit.name == "사과" }
```

- **filterIndexed**：filter 時需要 index

```kotlin
val apples = fruits.filterIndexed { idx, fruit -> 
println(idx)
fruit.name == "사과"
} 

```

- **map、mapIndex**：對每個元素執行 lambda 函式

- **mapNotNull**：僅取出 mapping 結果不為 null 的元素

```kotlin
val values = fruits.filter { it.name == "사과" }
	.mapNotNull { it.nullOrValue() } // 僅取非 null 的值
```

- **all**：全部滿足條件回傳 true，否則 false

```kotlin
// 確認是否全部為蘋果
val isAllApple = fruits.all { it.name == "사과" }
```

- **none**：全部不滿足條件回傳 true，否則 false

```kotlin
// 確認是否一個蘋果都沒有
val isNoApple = fruits.none { it.name =="사과" }
```

- **any**：只要有一個滿足就回傳 true，否則 false

```kotlin
// 確認是否有任何超過一萬韓元的水果
val hasExpensiveFruit = fruits.any { it.price > 10_000 }
```

- **count**：數量

```kotlin
// 等同於 Java 的 list.size()
val fruitCount = fruits.count()
```

- **sortedBy、sortedByDescending**：升冪、降冪排序

```kotlin
val sortedFruit = fruit.sortedBy{it.price} // 升冪
val desendSortedFruit = fruit.sortedByDescending{it.price} // 降冪
```

- **distinctBy**：以變換後的值為基準去重

```kotlin
// 確認存在哪些種類的水果
val distinctFruitNames = fruits.distinctBy{ it.name } // 以名稱為基準去重
	.map{it.name}
```

- **first、firstOrNull、last、lastOrNull**：第一個、最後一個值
    - 列表為空時 first、last 沒有可取的值，會拋出 Exception

```kotlin
fruits.first()
fruits.lastOrNull()
```

- **groupBy**：以某 Key 為基準進行分組（List → Map）

```kotlin
// 以水果名稱為 key 對列表內容分組
// key：사과（蘋果），value：{사과1, 사과2, 사과3...}
val map: Map<String, List<Fruit>> = fruits.groupBy{fruit -> fruit.name } 

// 也可同時處理 key、value
// 以名稱為 Key、價格 List 為 value 的範例
val map2 : Map<String, List<Long>> = fruits
	.groupBy({it.name}, {it.price})
```

- **associateBy**：以不重複的 Key 為基準轉換為 Map（List→ Map）
    - **但若 id 有重複值，可能遺失資料。**
    - **若 id 重複，最後一個值會成為 value。**

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

// 同樣可以同時處理 key、value
val map: Map<Long, Long> = fruits
	.associateBy({it.id}, {it.price})
```

- 在 Map 中也可使用上面的大部分函式

```kotlin
val map: Map<String, List<Fruit>> = fruits.groupBy{fruit -> fruit.name } 
	.filter { (key, value) -> key == "사과" } 
```

- 巢狀 Collections 的處理
    - flatMap、flatten

```kotlin
var fruitsInList: List<List<Fruit>> = listOf(
	listOf(Fruit(1,"사과"), Fruit(2,"수박"), Fruit(3,"사과")),
	listOf(Fruit(4,"바나나"), Fruit(5,"사과"), Fruit(6,"바나나")),
)

// 不論列表結構，將所有蘋果取出
// output：listOf(Fruit(1,"사과"), Fruit(3,"사과"), Fruit(5,"사과"))
val appleList : List<Fruit>
	= fruitsInList.flatMap { list -> list.filter{it.name == "사과"}

// 將二維列表展平為一維
val flattenList : List<Fruit>
	= fruitsInList.flatten();
```


### 19. TakeIf、scope Function

- **TakeIf**：滿足給定條件時回傳該值，否則回傳 null
- **TakeUnless**：不滿足給定條件時回傳該值，否則回傳 null

```kotlin
// 這個函式
fun getFruitOrNull(fruit: Fruit): Fruit?{
	return if(fruit.name == "사과"){fruit} else {null}
}
// 可以這樣使用
val getfruit = fruit.takeIf { it.name == "사과" }
Fruit(1L, "사과") // 此情況回傳 fruit
Fruit(1L, "바나나") // 此情況回傳 null
```

- Scope function：建立暫時作用域的函式
    - 即利用 lambda 建立暫時作用域
        - 讓程式碼更簡潔
        - 或用於 method chaining 的函式

```kotlin
// 以下兩個函式相同
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

- Scope function 的種類
    - **let、run**：回傳 lambda 的結果
        - let：使用 it（無法省略，但可另起名稱）
        - run：使用 this（可以省略，但無法另起名稱）
    - **also、apply**：回傳物件本身
        - also：使用 it
        - apply：使用 this
    - **with**：唯一不是擴充函式的。使用 this，回傳 lambda 的結果

```kotlin
val value1 = person.let{ it.age }  // value1 的值：person.age
// 可像 person.let{ person -> person.age } 一樣另起名稱
val value2 = person.run{ this.age } // value2 的值：person.age
// 可像 person.run{ age } 一樣省略 this

val value3 = person.also{ it.age } // value3 的值：person
val value4 = person.apply{ this.age } // value4 的值：person 

with(person){
	println(name)
	println(age)
}

/* 
使用 it 的情況：以一般函式為參數 block : (T) -> R : R
使用 this 的情況：以擴充函式為參數 block: T.() -> R : R
有興趣的話直接看實作吧....
*/
```
