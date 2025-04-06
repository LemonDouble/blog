---
title: "내가 볼려고 올리는 Kotlin Cheat Sheet"
slug: f8a9d54607e901b4
date: 2023-01-17T14:24:20Z
image: cover.png
categories:
    - 코틀린
tags:
    - 코틀린
---

- TIP : 숫자 작성할 때 var number = 1_000L 과 같이 작성 가능

### 1. 변수

- val collections에도 element는 추가할 수 있음
- primitive, reference type 구분 없지만, 실제 연산할떈 primitive type으로 변환되어 연산됨

### 2. Null 처리

- Safe Call ( ?. ) : null이 아니면 실행, null이면 실행하지 않음

```kotlin
val str: String? = "ABC"
str.length // ERROR
str?.length // OK
```

- Elvis 연산자 ( ?: ) : 앞의 연산 결과가 null이면 뒤의 값을 사용

```kotlin
val str: String? = "ABC"
str?.length ?: 0 // str.length 혹은 0 
str?.length ?: throw IllegalArgumentException("null이 들어왔습니다.") // 혹은 예외 발생
str?.length ?: return 0 // Early return에도 사용 가능
```

- 플랫폼 타입 : Kotlin에서 Java 코드를 가져왔을 때
    - null, nonnull 타입 둘 다 사용 가능
    - 가능하면 Java 코드 직접 확인하거나, Kotlin으로 wrapping하여 사용하자.

```kotlin
// @Nullable, @NonNull 있으면 변환됨, 하지만 없으면 플랫폼 타입
// java 에서 
private final String name; << Annotation 없을 때

val name : String? = javaPerson.name // OK
val name : String = javaPerson.name  // OK
```

### 3. Type

- 기본 Type 변환 : 기본 타입끼리는 toXX 같은 함수를 사용

```kotlin
val number1 : Int = 4
val number2 : Long = number1.toLong()

val number1 : Int? = 3 
val number2 : Long = number1?.toLong() ?: 0L // nullable 변수면 재주껏 처리
```

- 사용자 정의 Type : 스마트 캐스트 (Smart Cast) 활용

```kotlin
fun printAgeIfPerson(obj: Any){
	// is = Java의 instanceof
	if(obj is Person){
			// 스마트 캐스트 되어서 obj 의 type은 Person
			println(obj.age)
	}
}

// OR
// java의 (Person) obj와 같음
val person = obj as Person

// !is, as? 등도 사용 가능
// as? : 전체가 null
```

- Any : Java의 Object 역할 + Primitive Type의 최상위도 포함
    - null은 포함 X, null도 포함하고 싶다면 Any?로 사용
    - equals, hashCode, toString 존재
- Unit : void 비슷, 하지만 Unit은 그 자체로 제네릭에서 타입 인자로 사용 가능
- Nothing : 함수가 정상적으로 끝나지 않았음을 표시

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
// Java에서의 str.get(0) 대신
str[1] //ok
```

### 4. 연산자

- Kotlin에선 >, < , ≥, ≤ 사용하면 자동으로 compateTo 호출해줌
- == (값 비교,동등성 : equals 자동 호출 ) , === (주소 비교, 동일성 : 같은 0x101 주소)

- in, !in : 컬렉션 / 범위에 포함되어 있는지?
- a..b : a부터 b까지의 범위 객체를 생성

```kotlin
val numbers = 1..100 // 1~100까지의 IntRange 객체 생성
println(1 in numbers) // true
println(0 in numbers) // false

// 두 식은 같음
if(0 <= score && score <= 100) {..}
if(score in 0..100)
```

- 연산자 오버로딩

```kotlin
data class Money(val amount : Long){
	
	operator fun plus(other : Money): Money{
		return Money(this.amount + other.amount)
	}
}

//아래와 같이 사용 가능
val sumMoney = money1 + money2
```

---

### 5. 조건문

- if문은 Kotlin에선 expression (Java에선 statement)
    - expression : 하나의 값으로 평가될 수 있는 문장

```kotlin
// Java에선 불가능하지만 Kotlin에선 if문도 expression이라 OK 
fun getPassOrFail(score : Int): String{
	return if (score >= 50){ "P" } else { "F" }
}
```

- Java에선 삼항연산자가 expression이지만, Kotlin은 if문이 expression이므로 삼항 연산자가 필요x
    - 따라서 삼항 연산자가 없음

- When문

```kotlin
// 기본 사용, Switch 대신
fun getGradeWithSwitch(score: Int): String{
	return when(score / 10){
		9 -> "A"
		8 -> "B"
		7 -> "C"
		else -> "D"
	}
}

// 좌변에 expression 사용도 가능, is Type 등도 사용 OK
when(score){
in 90..99 -> "A"
in 80..89 -> "B"
...

// 여러 조건 동시 검사
when (number){
1,0,-1 -> println("1,0,-1입니다")
else -> println("아닙니다")
}

// 조건문처럼 사용
fun printOdd(score: Int): Int{
	when{
		score % 2 == 1 -> println("홀수")
		score % 2 == 0 -> println("짝수")
		else -> println("넌머임")
	}
	return 1
}

```

### 6. 반복문

- forEach

```kotlin
val numbers = listOf(1L, 2L, 3L)
for (number in numbers){
	println(number)
}
```

- Progression : 등차수열, Range (Range는 Progression을 상속받음)

```kotlin
val intRange = 1..100
```

### 7. 예외

- try-catch-finally : 그대로 (단, kotlin에선 expression, return 등 사용 가능)
- Kotlin에서는 모든 에러가 Unchecked Exception
    - Checked : 무조건 처리해줘야 하는 Exception
    - Unchecked Exception : 꼭 처리하지 않아도 되는..
- try with resources

```kotlin
// JAVA의 try with resources
// Try가 끝나면 자동으로 닫아줌. 단 AutoCloseable 인터페이스의 구현체여야 함. 
public void readFile(String path) throws IOException{
	try(BufferedReader reader = new BufferedReader(new FileReader(path))){
		System.out.println(reader.readLine());
	}
}

//Kotlin에서 사용, use가 try with resources 비슷하게 작동
fun readFile(path: String){
	BufferedReader(FileReader(path)).use { reader ->
		println(reader.readLine())
	}
}
```

### 8. 함수

- 기본

```kotlin
//함수 본문이 식(Expression)으로만 이뤄져 있으면 = 사용 가능
fun max(a: Int, b: Int) : Int = if(a > b) {a} else {b}
```

- Default paramters

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

- named argument ( 파라미터 이름을 명시 가능, builder 비슷하게 사용 가능)

```kotlin
repeat(str = "Hello, World", useNewLine = true)
// 이 경우, num은 지정되지 않았으므로 기본값 (3) 사용됨
// builder 만들지 않고 builder처럼 사용 가능
```

- 가변인자 (vararg)

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
printAll(*array) // spread 연산자
printAll("Hello", "World", "!!")
```

---

### 9. 클래스

- 클래스와 프로퍼티

```kotlin
class Person(
	val name:String,
	var age:Int
)

// 프로퍼티 접근처럼 getter,setter 접근
person.age = 10
println(person.age)
```

- 생성자 검증, init

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

- Custom getter, Setter

```kotlin
class Person(
	val name:String,
	var age:Int
){
	
	// Property처럼 사용, 하나의 Expresson으로 표시되는 것을 =으로 표시
	val isAdult: Boolean
		get() = this.age >= 20
	
	//위와 같은 표현식
	fun isAdult(): Boolean{
		return this.age >= 20
	}
}
```

### 10. 상속

- 추상 클래스

```kotlin
// 기본적으로 public class 
abstract class Animal(
	protected val species:String,
	protected val legCount:Int
){
	abstract fun move()
}
```

- 상속 받는 클래스

```kotlin
//Java
public class JavaPenguin extends JavaAnimal{
	
	// 하위 객체에서 추가된 필드
	private final int wingCount;
	
	public JavaPenguin(String species){
		super(species, 2)
		this.wingCount = 2;
	}

	@Override
	public void move(){
		System.out.println("penguin is moving");
	}

	// 상위 메소드의 getter를 오버라이드
	@Override
	public int getLegCount(){
		return super.legCount + this.wingCount;
	}
}

//Kotlin
class penguin(
	species : String // 주 생성자
) : Animal(species, 2) //Animal을 상속받아 Animal의 생성자를 바로 호출 {

	private val wingCount: Int = 2
	
	// Annotation 대신 예약어 사용
	override fun move(){
		println("cat is moving")
	}

	override val legCount: Int
		get() = super.legCount + this.wingCount
}

// 단, 코틀린은 기본적으로 모두 final 키워드가 붙여져 있기 때문에, override 하려면 open 필요
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
	//Default 키워드 없이 구현 가능
	fun act(){ println("swim!") } 
}
interface Flyable{
	fun act(){ println("fly!") }
}
```

- Interface 구현 클래스

```kotlin
// Java
public final class JavaPenguin extends JavaAnimal implements JavaFlyable, JavaSwimable{
	
	// 생성자 구현.. (생략됨)
	
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

- Backing Field (getter, setter가 제공되는 property) 를 Interface에 선언 가능

```kotlin
// Kotlin
interface Swimable{
	// 하위 클래스에서 구현을 기대하는 property
	val swimAbility: Int 
}

class Penguin(): Swimable{
	// 다음과 같이 구현
	override val swimAbility: Int
		get() = 3
}
```

- 클래스 상속시 주의할 점 (초기화 실행 순서)
    - 상위 Class의 Constructor, Init 블락에선 하위 클래스의 Field에 접근하지 말 것!
    - 상위 Class의 Constructor, Init 블럭에 사용되는 Property는 open 사용하지 말자!

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

// Derived 클래스 인스턴스화시 출력
/*
	Output : 
	Base Class
	0
	Derived Class 
*/

// 이유 : 상위 Class(Base)에서 number 호출 시, 하위 Class의 Number 가져옴.
// 하지만 상위 Class(Base) contstructor 실행 단계라 하위 Class(Dervied) 초기화 안 된 상태
// 따라서 0 출력
```

- 키워드 정리
    - final (기본, 생략가능) : override 할 수 없게 막는다.
    - open : override를 열어 준다.
    - abstract : 반드시 override 해야 한다.
    - override: 상위 타입을 오버라이드 한다. (Java에선 Annotation이지만 Kotlin에선 키워드)

### 11. 접근 제어

- Java, Kotlin의 접근 제어
    - Public
        - Java/Kotlin : 모든 곳에서 접근 가능
    - Protected
        - Java : 같은 패키지, 또는 하위 클래스에서 접근 가능
        - Kotlin : **선언된 클래스**, 또는 하위 클래스에서만 접근 가능
    - default(Java) / internal(Kotlin)
        - Java : 같은 패키지에서만 접근 가능
        - Kotlin : 같은 Module에서만 접근 가능
            - Module : 한 번에 컴파일되는 Kotlin 코드 (같은 Gradle 프로젝트 등..)
    - private
        - Java/Kotlin : 선언된 클래스 내에서만 접근 가능

- Property 접근 지시어

```kotlin
class Car(
	// name에 대한 getter를 internal로
	internal val name : String,
	var owner : String,
	_price : Int
){
	// Setter만 private로
	var price = _price
		private set
}

```

- Java/Kotlin 혼합 사용시 주의점
    - Internal 키워드는 Bytecode 변환 (Java 변환시) public으로 변환됨
    - 따라서 Java 코드에서는, Kotlin에서의 Internal 키워드를 가져올 수 있음
    - Kotlin의 Prtotected 또한 마찬가지 (Java에서는 같은 패키지에서도 접근 가능)

### 12. object

- static 함수와 변수
    - static : 정적으로 인스턴스끼리의 값을 공유
    - companion object : 클래스와 동행하는 유일한 object

```kotlin
//Java
public class JavaPerson{
    private static final int MIN_AGE = 1;

    // 정적 팩토리 메소드
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
		//Static 대신 companion Object 사용
    companion object{
				// const가 있으면 컴파일시 할당, 없으면 런타임에 할당
				// 기본 타입 + String에만 할당 가능
        const val MIN_AGE = 1
        fun newBaby(name: String): Person{
            return Person(name, MIN_AGE)
        }
    }
}

// Person.Companion.newBaby("ABC") 로 호출 가,
// 혹은 다음과 같이 @JvmStatic 어노테이션 통해 Java Static처럼 바로 접근 가능

/*
	@JvmStatic
	fun newBaby(name: String) : Person{...}

	이후
	Person.newBaby("ABC") 사용 가능
*/
```

- Companion Object 활용 (Java와의 차이점)
    - 하나의 객체로 간주됨
    - 따라서 이름을 붙이거나 Interface 만들 수 있음

```kotlin
// Kotlin
interface Log{ fun log() }

class Person private constructor(
    var name : String,
    var age : Int,
){
		// 이름 설정 가능
    companion object Factory{
        const val MIN_AGE = 1
        fun newBaby(name: String): Person{
            return Person(name, MIN_AGE)
        }
				override fun log(){ println("I am companion object!") }
    }
}

// 이름이 있다면 이름을 바탕으로 접근 가능
// Person.Factory.newBaby("ABC");
```

- Singleton
    - object 키워드 사용

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

// 이후 Singleton.a 와 같이 사용 가능
```

- 익명 클래스 (Anonymous class)
    - 특정 Interface, Class 상속받은 구현체를 일회성으로 사용할 때

```kotlin
// Java
private static void moveSomething(Movable movable){ movable.move(); movable.fly(); }

//익명 클래스 구현
moveSomething(new Movable(){
	@Override
	public void move(){System.out.println("move~");}
	@Override
	public void fly(){System.out.println("fly~");}
}

// Kotlin
private fun moveSomething(movable:Movable){ movable.move(); movable.fly();}

// Object 키워드 사용해서 익명 클래스 구현
moveSomething(object : Movable{
	override fun move(){ println("move~") }
	override fun fly(){println("fly~")}
})
```

### 13. 중첩 클래스 (nested class)

- 자주 안 쓸 것 같아서 생략..

### 14. Data class,  Enum class, Sealed Class/Interface

- Data class
    - DTO를 편하게 사용 가능
    - field, constructor, getter, equals, hashCode, toString 를 기본적으로 제공

```kotlin
//Kotlin
data class PersonDto(val name: String, val age: Int)

// named Arguments를 활용하면 Builder 패턴도 사용하는 것과 같은 효과
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

// Kotlin (동일 코드)
enum class Country(
	private val code: String
){
	KOREA("KO"),
	AMERICA("US");
}

// When절을 활용해 간편하게 처리
// 또한, 이렇게 사용하는 경우 ENUM CLASS에 값이 생기면 컴파일러가 알려줌
fun handleCountry(country: Country){
	when(country){
		Country.KOREA -> LOGIC()
		Country.AMERICA -> LOGIC()
		// ELSE 없이도 사용 가능!
	}
}
```

- Sealed Class/Interface
    - 상속이 가능한 추상 클래스를 만들고 싶은데, 외부에서는 상속하지 못 했으면 좋겠다
    - 컴파일 타임에 하위 클래스 타입을 모두 기억해, 런타임때 다른 클래스 타입 추가 불가
    - 하위 클래스는 같은 패키지에 있어야 함
    - Enum과의 차이점
        - Class 상속 가능
        - Enum은 인스턴스가 싱글톤이지만, Sealed Class는 멀티 인스턴스 가능

```kotlin
//Kotlin
sealed class Car(
	val name:String,
	val price:Long
)

class Avante: Car("Avante",1_000L)
class Sonata: Car("Sonata",2_000L)
class Grandeur: Car("Grandeur",3_000L)
// 위 Enum class처럼 when절 사용 가능
```

---

### 15. Collections (컬렉션)

- Collections
    - 컬렉션이 불변인지, 가변인지 설정 필요
    - Mutable(가변) 컬렉션 : element 추가,삭제 가능
    - Immutable(불변) 컬렉션 : element 추가,삭제 불가
        - 단, 불변 컬렉션이라도 Reference Type의 엘리먼트 필드는 바꿀 수 있다.
        - 예를 들어서, Index 1번 데이터에 접근해서 Price 필드를 1000 → 2000원으로 바꾸는건 가능

- List

```kotlin
/* 기본 구현체 : ArrayList */
// 불변 리스트 생성
val numbers = listOf(100,200)
// 가변 리스트 생성
val mutableNumbers = mutableListOf(100,200)
// 빈 리스트 생성
val emptyList = emptyList<Int>()

//numbers.get(0)과 동일
numbers[0]

for(number in numbers){println(number)}
//Index를 같이 가져와야 할 때
for((idx, number) in numbers.withIndex()){println("${idx}, ${number}")}
```

- Set

```kotlin
/* 기본 구현체 : LinkedHashSet */
val numbers = setOf(100,200)
```

- Map

```kotlin
val weekMap= mutableMapOf<Int, String>()
// 배열처럼 접근 가능
weekMap[1] = "MONDAY"
weekMap[2] = "TUESDAY"

// to를 사용해 초기화도 가능
mapOf(1 to "MONDAY", 2 to "TUESDAY")

for(key in weekMap.keys){println(key) println(weekMap[key]) }
for((key, value) in weekMap.entries){println(key) println(value)} 
```

- 컬렉션의 null 가능성
    - `List<Int?>` : List 자체는 null X, 리스트 안에는 nullable
    - `List<Int>?` : List 자체는 nullable, 리스트 안에는 null X
    - `List<Int?>?` : List 자체도 nullable, 리스트 안에도 nullable

- Java와 혼용시 주의할 점
    - Java는 불변/가변 리스트를 구분하지 않음
        - Kotlin의 불변 리스트를, Java에서 가변 리스트처럼 사용할 수 있음
        - Kotlin의 Non-null 리스트를, Java에서 null을 추가할 수 있음
        - Kotlin에서 Collections.unmodifiable을 사용하거나, 방어 로직 사용
    - Platform Type으로 타입 문제 생길 수 있음

### 16. Extension(확장), Infix(중위) , inline, Local(지역) 함수

- 확장 함수
    - Kotlin : Java와의 100% 호환성을 목표로함
        - 그래서 Java로 만들어진 라이브러리에 Kotlin 코드가 추가 가능하면 좋겠다
        - 클래스 내부 메소드처럼 사용하지만, 코드는 외부에 작성할 수 있게 하자!
    - 제한 :
        - 확장함수가 public인데, private, protected 멤버를 가져오면 캡슐화가 깨지므로..
            - 캡슐화 유지 위해, private, proteced 멤버 가져올 수 없음
        - 멤버함수/ 확장함수의 시그니쳐(이름) 같은 경우
            - 멤버 함수가 우선적으로 호출
            - 따라서, 멤버함수가 이후 추가된다면 오류 생길 수도 있다.

```kotlin
// Kotlin

// String 클래스를 확장
fun String.lastChar(){ return this[this.length - 1] }

// 이후 Kotlin에선 다음과 같이 사용 가능
val str = "ABC"
println(str.lastChar());
```

- 중위 함수
    - 변수, argument가 각각 하나씩만 있는 경우
    - var.functionName(argument) 대신
    - var functionName argument로 호출 가능

```kotlin
infix fun Int.add(other: Int): Int{
	return this + other
}

// 다음과 같이 호출 가능
3 add 4
```

- 인라인 함수
    - 우리가 아는 그 인라인 함수..
    - 함수를 파라미터로 전달하는 오버헤드 줄일 수 있음

```kotlin
inline fun Int.add(other: Int): Int{
```

- 지역 함수
    - 함수 안의 함수

```kotlin
// Kotlin
fun createPerson(firstName: String, lastName:String): Person{
	if(firstName.isEmpty()){ throw IllegalArgumentException("에러!!")}
	if(lastName:String.isEmpty()){ throw IllegalArgumentException("에러!!")}
	return Person(firstName, lastName)
}

// Local 함수 사용
fun createPerson(firstName: String, lastName:String): Person{
	// 함수 속 함수
	fun validateName(name: String){if(name.isEmpty()){throw IllegalArgumentException("에러!!")}
	
	validateName(firstName)
	validateName(lastName)
	return Person(firstName, lastName)
}

```

### 17. lambda

- Java에서는 함수를 변수 할당, 파라미터로 전달 불가 (1급 객체 X)
    - Fruit::isApple 처럼 가능한 것 처럼 보이지만, 실제로는 Predicate라는 Interface를 넘김
- Kotlin에서는 함수를 변수 할당,파라미터로 전달 가능 (1급 객체 O)

- lambda 생성, 호출

```kotlin
// Kotlin

// 함수의 Type도 다음과 같이 설정 가능
val isApple : (Fruit) -> Boolean 
= fun(fruit: Fruit): Boolean {
	return fruit.name == "사과"
}

// 바로 위와 동일한 코드
val isApple2 = {fruit: Fruit -> fruit.name == "사과" }

val isApple3 = { fruit ->
println("사과만 받는 함수입니다.")
fruit.name == "사과" // 여러 줄 lambda에서 return 생략 시, 마지막 줄이 자동으로 return됨
}

// labmda 호출, 동일한 코드
isApple(Fruit("사과", 1000))
// 명시적으로 호출
isApple.invoke(Fruit("사과", 1000))
```

- Parameter로 함수 받기

```kotlin
// 함수 Type을 적어서, 함수를 받을 수 잇다.
fun filterFruits(fruits: List<Fruit>, filterFunc : (Fruit) -> Boolean) : List<Fruit>{
	val results = mutableListOf<Fruit>()
	for(fruit in fruits){
		if(filterFunc(fruit)){ results.add(fruit)}
	}
	return results
}

//호출시 다음과 같이 lambda 전달해서 사용 가능
filterFruits(fruit, {fruit -> fruit.name =="사과"} )

// 이 때, lambda가 마지막 파라미터면 다음과 같이 사용 가능  (위 코드와 기능은 동일)
filterFruits(fruit) {fruit -> fruit.name =="사과"}

// 파라미터가 하나인 경우는, it으로 화살표 앞 부분 대체 가능
filterFruits(fruit) {it.name =="사과"}
```

- Closure
    - Kotlin은 lambda가 실행되는 시점에, 참조하고 있는 변수들을 모두 포획해서 값을 가진다.
    - 아래 코드에선 targetFruitName =”수박” 시점을  포획해서 가지고 있는다.
    - 이러한 데이터 구조를 Closure라고 부르고, 이런 구조여야 lambda가 1급 객체일 수 있다.

```kotlin
//Java
String targetFruitName = "바나나"
targetFruitName = "수박"

//Java에서는 Lambda에서 사용되는 변수는, final 변수거나, 
//final 키워드가 붙지 않았어도 값이 실제로 변경되지 않는 (Effectively final) 변수만 사용 가능
filterFruits(fruits, (fruit) -> targetFruitName.equals(fruit.getName())); // ERROR!

//Kotlin
var targetFruitName = "바나나"
targetFruitName = "수박"
filterFruits(fruits) { it.name == targetFruitName } // It Works!

```

### 18. Collection 내장 함수+ FP

- **filter**

```kotlin
// 사과만 받기
val apples = fruits.filter { fruit -> fruit.name == "사과" }
```

- **filterIndexed** : filter에서 index가 필요한 경우

```kotlin
val apples = fruits.filterIndexed { idx, fruit -> 
println(idx)
fruit.name == "사과"
} 

```

- **map, mapIndex :** 각각의 원소에 대해 lambda 함수 실행

- **mapNotNull** : mapping의 결과가 null이 아닌 것만 추출

```kotlin
val values = fruits.filter { it.name == "사과" }
	.mapNotNull { it.nullOrValue() } // null 아닌 값만 추출
```

- **all** : 조건을 모두 만족하면 true, 아니면 false

```kotlin
// 전부 사과인지 확인
val isAllApple = fruits.all { it.name == "사과" }
```

- **none** : 조건을 모두 불만족하면 true, 아니면 false

```kotlin
// 사과가 하나도 없는지 확인
val isNoApple = fruits.none { it.name =="사과" }
```

- **any** : 하나라도 만족하면 true, 아니면 false

```kotlin
// 하나라도 만원 넘는 과일 있는지 확인
val hasExpensiveFruit = fruits.any { it.price > 10_000 }
```

- **count** : 개수

```kotlin
// Java의 list.size()와 동일
val fruitCount = fruits.count()
```

- **sortedBy, sortedByDescending** : 오름차순, 내림차순 정렬

```kotlin
val sortedFruit = fruit.sortedBy{it.price} //오름차순
val desendSortedFruit = fruit.sortedByDescending{it.price} // 내림차순
```

- **distinctBy** : 변경된 값을 기준으로 중복 제거

```kotlin
// 과일이 각각 어떤 종류 있는지 확인
val distinctFruitNames = fruits.distinctBy{ it.name } // 이름을 기준으로 중복 제거
	.map{it.name}
```

- **first, firstOrNull, last, lastOrNull** : 첫번째, 마지막 값
    - 빈 리스트라서 first, last가 가져올 값이 없으면 Exception 발생

```kotlin
fruits.first()
fruits.lastOrNull()
```

- **groupBy** : 어떤 Key를 기준으로 Grouping (List → Map)

```kotlin
// 과일 이름을 key로 리스트의 내용물을 Grouping
// key : 사과, value : {사과1, 사과2, 사과3...}
val map: Map<String, List<Fruit>> = fruits.groupBy{fruit -> fruit.name } 

// 다음과 같이 key, value 동시 처리도 가능
// 이름을 Key로, 가격 List를 만드는 예시
val map2 : Map<String, List<Long>> = fruits
	.groupBy({it.name}, {it.price})
```

- **associateBy** : 어떤 중복되지 않는 Key를 기준으로 Map 변환(List→ Map)
    - **단, id가 중복값 있을 시 값이 유실될 수 있다.**
    - **id가 중복값인 경우, 가장 마지막 값이 value가 된다.**

```kotlin
val map: Map<Long, Fruit> = fruits.associateBy { fruit -> fruit.id }

val fruitList = listOf<Fruit>(
			// id가 동일
      Fruit(1,"1"),
      Fruit(1,"2"),
      Fruit(1,"3"),
    )

  val associateBy = fruitList.associateBy { it.id }

  println(associateBy.count()) // 1
  for (entry in associateBy) {
      println(entry) // 1=Fruit(id=1, name=3)
  }

// 마찬가지로 key, value 동시 처리 가능
val map: Map<Long, Long> = fruits
	.associateBy({it.id}, {it.price})
```

- Map에서도 위의 함수들 대부분 사용 가능

```kotlin
val map: Map<String, List<Fruit>> = fruits.groupBy{fruit -> fruit.name } 
	.filter { (key, value) -> key == "사과" } 
```

- 중첩된 Collections 처리
    - flatMap, flatten

```kotlin
var fruitsInList: List<List<Fruit>> = listOf(
	listOf(Fruit(1,"사과"), Fruit(2,"수박"), Fruit(3,"사과")),
	listOf(Fruit(4,"바나나"), Fruit(5,"사과"), Fruit(6,"바나나")),
)

// 리스트 구조 관계없이, 사과인 과일을 모두 뽑아주세요
// output : listOf(Fruit(1,"사과"), Fruit(3,"사과"), Fruit(5,"사과"))
val appleList : List<Fruit>
	= fruitsInList.flatMap { list -> list.filter{it.name == "사과"}

// 2차원 리스트를 1차원으로
val flattenList : List<Fruit>
	= fruitsInList.flatten();
```

---

### 19. TakeIf, scope Function

- **TakeIf** : 주어진 조건을 만족하면 그 값이, 그렇지 않으면 null 반환
- **TakeUnless** : 주어진 조건 만족하지 않으면 그 값이, 그렇지 않으면 null 반환

```kotlin
// 이 함수를
fun getFruitOrNull(fruit: Fruit): Fruit?{
	return if(fruit.name == "사과"){fruit} else {null}
}
//다음과 같이 사용 가능
val getfruit = fruit.takeIf { it.name == "사과" }
Fruit(1L, "사과") // 이 경우 fruit
Fruit(1L, "바나나") // 이 경우 null 반환
```

- Scope function : 일시적인 영역을 형성하는 함수
    - 즉, lamba를 활용해 일시적 영역을 만들고
        - 코드를 더 간결하게 하거나
        - method chaining에 활용하는 함수

```kotlin
// 두 함수는 같음
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

- Scope function의 종류
    - **let, run** : lambda의 결과를 반환
        - let : it 사용 (생략 불가능한 대신, 다른 이름 붙일 수 있음)
        - run : this 사용 (생략 가능한 대신, 다른 이름 붙일 수 없음.)
    - **also, apply** : 객체 자체를 반환
        - also : it 사용
        - apply : this 사용
    - **with** : 유일하게 확장함수 아님. this 사용, lambda의 결과를 반환

```kotlin
val value1 = person.let{ it.age }  // value1 값 : person.age
// person.let{ person -> person.age } 처럼, 다른 이름 붙일 수 있음
val value2 = person.run{ this.age } // value2 값 : person.age
// person.run{ age } 처럼, this 생략 가능

val value3 = person.also{ it.age } // value3 값 : person
val value4 = person.apply{ this.age } // value4 값 : person 

with(person){
	println(name)
	println(age)
}

/* 
it 쓰는 경우 : 일반 함수를 파라미터로 받음 block : (T) -> R : R
this 쓰는 경우 : 확장 함수를 파라미터로 받음 block: T.() -> R : R
궁금하면 직접 구현체를 확인해 보자....
*/
```

---