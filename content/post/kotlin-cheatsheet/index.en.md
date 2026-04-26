---
title: "Kotlin Cheat Sheet (For My Own Reference)"
slug: f8a9d54607e901b4
date: 2023-01-17T14:24:20Z
image: cover.png
categories:
    - Kotlin
tags:
    - Kotlin
    - JVM
    - Cheat Sheet
---

_Note: I'm based in Korea, so some context here is Korea-specific._

- TIP: When writing numbers, you can use formats like `var number = 1_000L`

### 1. Variables

- You can still add elements to a `val` collection
- There's no distinction between primitive and reference types, but during actual computation, values are converted to primitive types

### 2. Null Handling

- Safe Call (`?.`): executes if not null, skips if null

```kotlin
val str: String? = "ABC"
str.length // ERROR
str?.length // OK
```

- Elvis operator (`?:`): if the expression on the left is null, use the value on the right

```kotlin
val str: String? = "ABC"
str?.length ?: 0 // either str.length or 0 
str?.length ?: throw IllegalArgumentException("null이 들어왔습니다.") // or throw exception
str?.length ?: return 0 // can also be used for early return
```

- Platform Type: When pulling Java code into Kotlin
    - Both null and non-null types are usable
    - When possible, check the Java code directly or wrap it in Kotlin

```kotlin
// If @Nullable, @NonNull are present they get converted, otherwise platform type
// In Java 
private final String name; << When no annotation

val name : String? = javaPerson.name // OK
val name : String = javaPerson.name  // OK
```

### 3. Type

- Basic type conversion: use functions like `toXX` between primitive types

```kotlin
val number1 : Int = 4
val number2 : Long = number1.toLong()

val number1 : Int? = 3 
val number2 : Long = number1?.toLong() ?: 0L // for nullable variables, handle as needed
```

- User-defined types: leverage Smart Cast

```kotlin
fun printAgeIfPerson(obj: Any){
	// is = Java's instanceof
	if(obj is Person){
			// Smart cast makes obj's type Person
			println(obj.age)
	}
}

// OR
// Same as Java's (Person) obj
val person = obj as Person

// !is, as? are also usable
// as? : the whole expression becomes null
```

- Any: plays the role of Java's Object + also includes the top of primitive types
    - Doesn't include null; use `Any?` if you want null too
    - Has equals, hashCode, toString
- Unit: similar to void, but Unit can itself be used as a type argument in generics
- Nothing: indicates that the function does not return normally

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
// Instead of Java's str.get(0)
str[1] //ok
```

### 4. Operators

- In Kotlin, using `>`, `<`, `≥`, `≤` automatically calls `compareTo`
- `==` (value comparison, equality: auto-calls equals), `===` (address comparison, identity: same 0x101 address)

- `in`, `!in`: is this in a collection / range?
- `a..b`: creates a range object from a to b

```kotlin
val numbers = 1..100 // creates an IntRange object from 1 to 100
println(1 in numbers) // true
println(0 in numbers) // false

// These two are equivalent
if(0 <= score && score <= 100) {..}
if(score in 0..100)
```

- Operator overloading

```kotlin
data class Money(val amount : Long){
	
	operator fun plus(other : Money): Money{
		return Money(this.amount + other.amount)
	}
}

// Can be used as below
val sumMoney = money1 + money2
```


### 5. Conditionals

- In Kotlin, `if` is an expression (statement in Java)
    - Expression: a statement that evaluates to a single value

```kotlin
// Not possible in Java, but works in Kotlin since if is an expression
fun getPassOrFail(score : Int): String{
	return if (score >= 50){ "P" } else { "F" }
}
```

- In Java, the ternary operator is an expression, but in Kotlin since `if` is already an expression, no ternary is needed
    - Therefore Kotlin has no ternary operator

- `when`

```kotlin
// Basic usage, in place of Switch
fun getGradeWithSwitch(score: Int): String{
	return when(score / 10){
		9 -> "A"
		8 -> "B"
		7 -> "C"
		else -> "D"
	}
}

// Expressions on the left side are also fine, including is Type, etc.
when(score){
in 90..99 -> "A"
in 80..89 -> "B"
...

// Multiple conditions at once
when (number){
1,0,-1 -> println("1,0,-1입니다")
else -> println("아닙니다")
}

// Can be used like an if-chain
fun printOdd(score: Int): Int{
	when{
		score % 2 == 1 -> println("홀수")
		score % 2 == 0 -> println("짝수")
		else -> println("넌머임")
	}
	return 1
}

```

### 6. Loops

- forEach

```kotlin
val numbers = listOf(1L, 2L, 3L)
for (number in numbers){
	println(number)
}
```

- Progression: arithmetic sequence; Range (Range inherits from Progression)

```kotlin
val intRange = 1..100
```

### 7. Exceptions

- try-catch-finally: same as Java (but in Kotlin, it's an expression and supports return, etc.)
- In Kotlin, all errors are Unchecked Exceptions
    - Checked: exceptions that must be handled
    - Unchecked: exceptions that don't have to be handled
- try-with-resources

```kotlin
// Java's try-with-resources
// Auto-closes when try block ends. Must implement AutoCloseable. 
public void readFile(String path) throws IOException{
	try(BufferedReader reader = new BufferedReader(new FileReader(path))){
		System.out.println(reader.readLine());
	}
}

// In Kotlin, use behaves similarly to try-with-resources
fun readFile(path: String){
	BufferedReader(FileReader(path)).use { reader ->
		println(reader.readLine())
	}
}
```

### 8. Functions

- Basics

```kotlin
// If a function body is just a single expression, you can use =
fun max(a: Int, b: Int) : Int = if(a > b) {a} else {b}
```

- Default parameters

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

- Named arguments (you can specify parameter names, kind of like a builder)

```kotlin
repeat(str = "Hello, World", useNewLine = true)
// Here, num isn't specified so its default (3) is used
// Acts like a builder without writing one
```

- Varargs

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
printAll(*array) // spread operator
printAll("Hello", "World", "!!")
```


### 9. Classes

- Classes and properties

```kotlin
class Person(
	val name:String,
	var age:Int
)

// Access getter/setter as if accessing a property
person.age = 10
println(person.age)
```

- Constructor validation, init

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

- Custom getter, setter

```kotlin
class Person(
	val name:String,
	var age:Int
){
	
	// Used like a property; single expression form uses =
	val isAdult: Boolean
		get() = this.age >= 20
	
	// Same expression as above
	fun isAdult(): Boolean{
		return this.age >= 20
	}
}
```

### 10. Inheritance

- Abstract class

```kotlin
// public class by default
abstract class Animal(
	protected val species:String,
	protected val legCount:Int
){
	abstract fun move()
}
```

- Subclass

```kotlin
//Java
public class JavaPenguin extends JavaAnimal{
	
	// Field added in subclass
	private final int wingCount;
	
	public JavaPenguin(String species){
		super(species, 2)
		this.wingCount = 2;
	}

	@Override
	public void move(){
		System.out.println("penguin is moving");
	}

	// Override the parent's getter
	@Override
	public int getLegCount(){
		return super.legCount + this.wingCount;
	}
}

//Kotlin
class penguin(
	species : String // primary constructor
) : Animal(species, 2) // Inherit Animal and call its constructor directly {

	private val wingCount: Int = 2
	
	// Use a keyword instead of an annotation
	override fun move(){
		println("cat is moving")
	}

	override val legCount: Int
		get() = super.legCount + this.wingCount
}

// Note: Kotlin classes are final by default, so you need open to override
abstract class Animal(
	protected val species:String,
	protected open val legCount: Int
){
	abstract fun move()
}
```

- Interface

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
	// No default keyword needed
	fun act(){ println("swim!") } 
}
interface Flyable{
	fun act(){ println("fly!") }
}
```

- Class implementing interfaces

```kotlin
// Java
public final class JavaPenguin extends JavaAnimal implements JavaFlyable, JavaSwimable{
	
	// Constructor implementation.. (omitted)
	
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

- You can declare abstract properties on interfaces (but interfaces can't have backing fields)

```kotlin
// Kotlin
interface Swimable{
	// Property expected to be implemented by subclasses
	val swimAbility: Int 
}

class Penguin(): Swimable{
	// Implement like this
	override val swimAbility: Int
		get() = 3
}
```

- Caveats when inheriting (initialization order)
    - In a superclass's constructor or init block, don't access subclass fields!
    - Don't use `open` on properties that are used in the superclass's constructor or init block!

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

// Output when instantiating Derived
/*
	Output : 
	Base Class
	0
	Derived Class 
*/

// Reason: when the superclass (Base) accesses number, it pulls the subclass's number.
// But the superclass (Base) constructor is still running, so the subclass (Derived) hasn't been initialized yet.
// So 0 is printed.
```

- Keyword summary
    - final (default, can be omitted): blocks overriding
    - open: allows overriding
    - abstract: must be overridden
    - override: overrides the parent type (annotation in Java, keyword in Kotlin)

### 11. Access Control

- Java vs. Kotlin access modifiers
    - Public
        - Java/Kotlin: accessible from everywhere
    - Protected
        - Java: same package or subclass
        - Kotlin: only the **declaring class** or subclass
    - default (Java) / internal (Kotlin)
        - Java: same package only
        - Kotlin: same module only
            - Module: a unit of Kotlin code compiled together (a single Gradle project, etc.)
    - private
        - Java/Kotlin: only inside the declaring class

- Property access modifiers

```kotlin
class Car(
	// internal getter for name
	internal val name : String,
	var owner : String,
	_price : Int
){
	// Only the setter is private
	var price = _price
		private set
}

```

- Caveats when mixing Java/Kotlin
    - The `internal` keyword becomes `public` in bytecode (when interoperating with Java)
    - So Java code can access Kotlin's `internal` members
    - Kotlin's `protected` is similar (in Java, you can access from the same package)

### 12. object

- Static functions and variables
    - static: shares values across instances statically
    - companion object: a unique object that accompanies the class

```kotlin
//Java
public class JavaPerson{
    private static final int MIN_AGE = 1;

    // Static factory method
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
		// companion object instead of static
    companion object{
				// const = assigned at compile time, otherwise at runtime
				// Only allowed for primitive types + String
        const val MIN_AGE = 1
        fun newBaby(name: String): Person{
            return Person(name, MIN_AGE)
        }
    }
}

// Callable as Person.Companion.newBaby("ABC"),
// or via @JvmStatic annotation to access like a Java static directly

/*
	@JvmStatic
	fun newBaby(name: String) : Person{...}

	After this, you can call:
	Person.newBaby("ABC")
*/
```

- Companion object usage (differences from Java)
    - Treated as a single object
    - So you can name it or implement an interface

```kotlin
// Kotlin
interface Log{ fun log() }

class Person private constructor(
    var name : String,
    var age : Int,
){
		// You can name it
    companion object Factory{
        const val MIN_AGE = 1
        fun newBaby(name: String): Person{
            return Person(name, MIN_AGE)
        }
				override fun log(){ println("I am companion object!") }
    }
}

// If named, you can access via the name
// Person.Factory.newBaby("ABC");
```

- Singleton
    - Use the `object` keyword

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

// Use as Singleton.a
```

- Anonymous class
    - For one-off implementations of an interface or class

```kotlin
// Java
private static void moveSomething(Movable movable){ movable.move(); movable.fly(); }

// Anonymous class
moveSomething(new Movable(){
	@Override
	public void move(){System.out.println("move~");}
	@Override
	public void fly(){System.out.println("fly~");}
}

// Kotlin
private fun moveSomething(movable:Movable){ movable.move(); movable.fly();}

// Use the object keyword for an anonymous class
moveSomething(object : Movable{
	override fun move(){ println("move~") }
	override fun fly(){println("fly~")}
})
```

### 13. Nested classes

- Skipping; I don't expect to use these often..

### 14. Data class, Enum class, Sealed Class/Interface

- Data class
    - Convenient for DTOs
    - Provides field, constructor, getter, equals, hashCode, toString out of the box

```kotlin
//Kotlin
data class PersonDto(val name: String, val age: Int)

// With named arguments, it's like using a builder
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

// Kotlin (equivalent)
enum class Country(
	private val code: String
){
	KOREA("KO"),
	AMERICA("US");
}

// Handle easily with when
// Bonus: when adding a new value, the compiler will tell you about non-exhaustive when
fun handleCountry(country: Country){
	when(country){
		Country.KOREA -> LOGIC()
		Country.AMERICA -> LOGIC()
		// works without an else!
	}
}
```

- Sealed Class/Interface
    - You want an inheritable abstract class, but you don't want external code to inherit it
    - All subclass types are remembered at compile time, so no new types can be added at runtime
    - Subclasses must be in the same package
    - Differences from Enum
        - Class can be inherited
        - Enum instances are singletons; sealed classes can have multiple instances

```kotlin
//Kotlin
sealed class Car(
	val name:String,
	val price:Long
)

class Avante: Car("Avante",1_000L)
class Sonata: Car("Sonata",2_000L)
class Grandeur: Car("Grandeur",3_000L)
// Like enum class above, you can use when
```


### 15. Collections

- Collections
    - Need to specify whether the collection is mutable or immutable
    - Mutable: can add/remove elements
    - Immutable: can't add/remove elements
        - Even an immutable collection lets you mutate fields of reference-type elements
        - For example, accessing index 1 and changing the price field from 1000 to 2000 is fine

- List

```kotlin
/* Default implementation: ArrayList */
// Immutable list
val numbers = listOf(100,200)
// Mutable list
val mutableNumbers = mutableListOf(100,200)
// Empty list
val emptyList = emptyList<Int>()

// Same as numbers.get(0)
numbers[0]

for(number in numbers){println(number)}
// When you also need the index
for((idx, number) in numbers.withIndex()){println("${idx}, ${number}")}
```

- Set

```kotlin
/* Default implementation: LinkedHashSet */
val numbers = setOf(100,200)
```

- Map

```kotlin
val weekMap= mutableMapOf<Int, String>()
// Access like an array
weekMap[1] = "MONDAY"
weekMap[2] = "TUESDAY"

// Initialize with `to`
mapOf(1 to "MONDAY", 2 to "TUESDAY")

for(key in weekMap.keys){println(key) println(weekMap[key]) }
for((key, value) in weekMap.entries){println(key) println(value)} 
```

- Nullability with collections
    - `List<Int?>`: list itself is non-null, elements can be null
    - `List<Int>?`: list itself can be null, elements non-null
    - `List<Int?>?`: both can be null

- Caveats when mixing with Java
    - Java doesn't distinguish immutable vs. mutable lists
        - Java code can treat a Kotlin immutable list as mutable
        - Java code can add nulls to a Kotlin non-null list
        - Use `Collections.unmodifiable` in Kotlin or add defensive logic
    - Platform types may cause type issues

### 16. Extension, Infix, inline, Local functions

- Extension functions
    - Kotlin: aiming for 100% Java compatibility
        - It would be nice to add Kotlin code to libraries written in Java
        - Use them like member methods, but write the code outside the class!
    - Restrictions:
        - If an extension function is public but accesses private/protected members, encapsulation breaks..
            - To preserve encapsulation, you can't access private/protected members
        - When member function and extension function have the same signature
            - The member function is called first
            - So if a member function is added later, you might get errors

```kotlin
// Kotlin

// Extending the String class
fun String.lastChar(){ return this[this.length - 1] }

// Use it in Kotlin like this
val str = "ABC"
println(str.lastChar());
```

- Infix functions
    - When you have one variable and one argument
    - Instead of `var.functionName(argument)`
    - You can call as `var functionName argument`

```kotlin
infix fun Int.add(other: Int): Int{
	return this + other
}

// Callable like this
3 add 4
```

- Inline functions
    - The inline function we all know..
    - Can reduce the overhead of passing functions as parameters

```kotlin
inline fun Int.add(other: Int): Int{
```

- Local functions
    - Functions inside functions

```kotlin
// Kotlin
fun createPerson(firstName: String, lastName:String): Person{
	if(firstName.isEmpty()){ throw IllegalArgumentException("에러!!")}
	if(lastName:String.isEmpty()){ throw IllegalArgumentException("에러!!")}
	return Person(firstName, lastName)
}

// With local functions
fun createPerson(firstName: String, lastName:String): Person{
	// Function inside function
	fun validateName(name: String){if(name.isEmpty()){throw IllegalArgumentException("에러!!")}
	
	validateName(firstName)
	validateName(lastName)
	return Person(firstName, lastName)
}

```

### 17. Lambda

- In Java, you can't assign functions to variables or pass them as parameters (not first-class)
    - Things like `Fruit::isApple` look possible, but you're really passing a Predicate interface
- In Kotlin, functions can be assigned to variables and passed as parameters (first-class)

- Creating and calling lambdas

```kotlin
// Kotlin

// You can specify the function type
val isApple : (Fruit) -> Boolean 
= fun(fruit: Fruit): Boolean {
	return fruit.name == "사과"
}

// Same as above
val isApple2 = {fruit: Fruit -> fruit.name == "사과" }

val isApple3 = { fruit ->
println("사과만 받는 함수입니다.")
fruit.name == "사과" // In multi-line lambdas, omitting return makes the last line the return value
}

// Calling the lambda, equivalent
isApple(Fruit("사과", 1000))
// Explicit invoke
isApple.invoke(Fruit("사과", 1000))
```

- Receiving functions as parameters

```kotlin
// You can write the function type to accept functions as parameters
fun filterFruits(fruits: List<Fruit>, filterFunc : (Fruit) -> Boolean) : List<Fruit>{
	val results = mutableListOf<Fruit>()
	for(fruit in fruits){
		if(filterFunc(fruit)){ results.add(fruit)}
	}
	return results
}

// Call by passing a lambda
filterFruits(fruit, {fruit -> fruit.name =="사과"} )

// If the lambda is the last parameter, you can write it like this (same behavior as above)
filterFruits(fruit) {fruit -> fruit.name =="사과"}

// With a single parameter, you can replace the part before the arrow with `it`
filterFruits(fruit) {it.name =="사과"}
```

- Closure
    - In Kotlin, when a lambda runs, it captures all the variables it references along with their values
    - In the code below, the value `targetFruitName = "수박"` is captured
    - This data structure is called a closure, and it's what allows lambdas to be first-class

```kotlin
//Java
String targetFruitName = "바나나"
targetFruitName = "수박"

// In Java, variables used in a lambda must be final, 
// or effectively final (not changed after assignment, even without the final keyword)
filterFruits(fruits, (fruit) -> targetFruitName.equals(fruit.getName())); // ERROR!

//Kotlin
var targetFruitName = "바나나"
targetFruitName = "수박"
filterFruits(fruits) { it.name == targetFruitName } // It Works!

```

### 18. Built-in Collection Functions + FP

- **filter**

```kotlin
// Take only apples
val apples = fruits.filter { fruit -> fruit.name == "사과" }
```

- **filterIndexed**: filter when you also need the index

```kotlin
val apples = fruits.filterIndexed { idx, fruit -> 
println(idx)
fruit.name == "사과"
} 

```

- **map, mapIndexed**: run a lambda on each element

- **mapNotNull**: extract only non-null mapping results

```kotlin
val values = fruits.filter { it.name == "사과" }
	.mapNotNull { it.nullOrValue() } // only non-null values
```

- **all**: true if all match, otherwise false

```kotlin
// Check whether all are apples
val isAllApple = fruits.all { it.name == "사과" }
```

- **none**: true if none match, otherwise false

```kotlin
// Check whether there are no apples
val isNoApple = fruits.none { it.name =="사과" }
```

- **any**: true if any match, otherwise false

```kotlin
// Check whether there's any fruit over 10,000 won
val hasExpensiveFruit = fruits.any { it.price > 10_000 }
```

- **count**: count

```kotlin
// Same as Java's list.size()
val fruitCount = fruits.count()
```

- **sortedBy, sortedByDescending**: ascending, descending sort

```kotlin
val sortedFruit = fruit.sortedBy{it.price} // ascending
val desendSortedFruit = fruit.sortedByDescending{it.price} // descending
```

- **distinctBy**: deduplicate based on a transformed value

```kotlin
// See what kinds of fruits exist
val distinctFruitNames = fruits.distinctBy{ it.name } // dedupe by name
	.map{it.name}
```

- **first, firstOrNull, last, lastOrNull**: first/last value
    - For an empty list, first/last throw an exception

```kotlin
fruits.first()
fruits.lastOrNull()
```

- **groupBy**: group by some key (List → Map)

```kotlin
// Group items by fruit name
// key: 사과, value: {사과1, 사과2, 사과3...}
val map: Map<String, List<Fruit>> = fruits.groupBy{fruit -> fruit.name } 

// You can also process key and value at once
// Example: name as key, list of prices as value
val map2 : Map<String, List<Long>> = fruits
	.groupBy({it.name}, {it.price})
```

- **associateBy**: convert a list to a Map keyed by a unique key (List → Map)
    - **If the id has duplicates, values can be lost.**
    - **If duplicates exist, the last value wins.**

```kotlin
val map: Map<Long, Fruit> = fruits.associateBy { fruit -> fruit.id }

val fruitList = listOf<Fruit>(
			// id is the same
      Fruit(1,"1"),
      Fruit(1,"2"),
      Fruit(1,"3"),
    )

  val associateBy = fruitList.associateBy { it.id }

  println(associateBy.count()) // 1
  for (entry in associateBy) {
      println(entry) // 1=Fruit(id=1, name=3)
  }

// Likewise can process key and value together
val map: Map<Long, Long> = fruits
	.associateBy({it.id}, {it.price})
```

- Most of these functions also work on Maps

```kotlin
val map: Map<String, List<Fruit>> = fruits.groupBy{fruit -> fruit.name } 
	.filter { (key, value) -> key == "사과" } 
```

- Working with nested collections
    - flatMap, flatten

```kotlin
var fruitsInList: List<List<Fruit>> = listOf(
	listOf(Fruit(1,"사과"), Fruit(2,"수박"), Fruit(3,"사과")),
	listOf(Fruit(4,"바나나"), Fruit(5,"사과"), Fruit(6,"바나나")),
)

// Pull all apples regardless of the list structure
// output: listOf(Fruit(1,"사과"), Fruit(3,"사과"), Fruit(5,"사과"))
val appleList : List<Fruit>
	= fruitsInList.flatMap { list -> list.filter{it.name == "사과"}

// Flatten 2D list to 1D
val flattenList : List<Fruit>
	= fruitsInList.flatten();
```


### 19. takeIf, Scope Functions

- **takeIf**: returns the value if the condition is met, otherwise null
- **takeUnless**: returns the value if the condition is NOT met, otherwise null

```kotlin
// This function
fun getFruitOrNull(fruit: Fruit): Fruit?{
	return if(fruit.name == "사과"){fruit} else {null}
}
// Can be written as
val getfruit = fruit.takeIf { it.name == "사과" }
Fruit(1L, "사과") // returns the fruit in this case
Fruit(1L, "바나나") // returns null in this case
```

- Scope functions: functions that create a temporary scope
    - In other words, use lambdas to form a temporary scope
        - to make code more concise, or
        - to use in method chaining

```kotlin
// These two are equivalent
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

- Types of scope functions
    - **let, run**: return the result of the lambda
        - let: uses `it` (can't omit, but can rename)
        - run: uses `this` (can omit, but can't rename)
    - **also, apply**: return the object itself
        - also: uses `it`
        - apply: uses `this`
    - **with**: the only one that's not an extension function. Uses `this`, returns the lambda's result

```kotlin
val value1 = person.let{ it.age }  // value1: person.age
// You can rename it: person.let{ person -> person.age }
val value2 = person.run{ this.age } // value2: person.age
// You can omit this: person.run{ age }

val value3 = person.also{ it.age } // value3: person
val value4 = person.apply{ this.age } // value4: person 

with(person){
	println(name)
	println(age)
}

/* 
Using `it`: takes a regular function as a parameter, block: (T) -> R : R
Using `this`: takes an extension function as a parameter, block: T.() -> R : R
If you're curious, check the actual implementation....
*/
```
