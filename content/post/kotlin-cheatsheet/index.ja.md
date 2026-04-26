---
title: "自分用の Kotlin チートシート"
slug: f8a9d54607e901b4
date: 2023-01-17T14:24:20Z
image: cover.png
categories:
    - Kotlin
tags:
    - Kotlin
    - JVM
    - チートシート
---

_注：筆者は韓国在住のため、本文には韓国特有の文脈が含まれることがあります。_

- TIP：数値を書くとき `var number = 1_000L` のように書けます

### 1. 変数

- `val` のコレクションでも要素は追加できます
- primitive と reference type の区別はありませんが、実際の演算時には primitive type に変換されて演算されます

### 2. Null 処理

- Safe Call（`?.`）：null でなければ実行、null なら実行しない

```kotlin
val str: String? = "ABC"
str.length // ERROR
str?.length // OK
```

- Elvis 演算子（`?:`）：左側の式が null なら右側の値を使う

```kotlin
val str: String? = "ABC"
str?.length ?: 0 // str.length または 0 
str?.length ?: throw IllegalArgumentException("null이 들어왔습니다.") // または例外を投げる
str?.length ?: return 0 // Early return にも使えます
```

- プラットフォーム型：Kotlin で Java コードを取り込んだとき
    - null、non-null どちらの型でも使えてしまう
    - 可能な限り Java コードを直接確認するか、Kotlin でラップして使いましょう

```kotlin
// @Nullable, @NonNull があれば変換されますが、なければプラットフォーム型
// java で 
private final String name; << アノテーションがない場合

val name : String? = javaPerson.name // OK
val name : String = javaPerson.name  // OK
```

### 3. Type

- 基本型変換：基本型同士は `toXX` のような関数を使う

```kotlin
val number1 : Int = 4
val number2 : Long = number1.toLong()

val number1 : Int? = 3 
val number2 : Long = number1?.toLong() ?: 0L // nullable な変数は適宜処理
```

- ユーザー定義型：スマートキャスト（Smart Cast）の活用

```kotlin
fun printAgeIfPerson(obj: Any){
	// is = Java の instanceof
	if(obj is Person){
			// スマートキャストされて obj の型は Person
			println(obj.age)
	}
}

// または
// java の (Person) obj と同じ
val person = obj as Person

// !is, as? なども使えます
// as? : 全体が null
```

- Any：Java の Object の役割 + Primitive Type の最上位も含む
    - null は含まれません。null も含めたい場合は `Any?` を使う
    - equals, hashCode, toString が存在
- Unit：void に近いですが、Unit はそれ自体でジェネリクスの型引数として使えます
- Nothing：関数が正常に終了しないことを示す

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
// Java の str.get(0) の代わりに
str[1] //ok
```

### 4. 演算子

- Kotlin では `>`, `<`, `≥`, `≤` を使うと自動的に `compareTo` が呼ばれます
- `==`（値比較、等価性：equals 自動呼び出し）、`===`（アドレス比較、同一性：同じ 0x101 アドレス）

- `in`, `!in`：コレクション／範囲に含まれているか？
- `a..b`：a から b までの範囲オブジェクトを生成

```kotlin
val numbers = 1..100 // 1～100 までの IntRange オブジェクトを生成
println(1 in numbers) // true
println(0 in numbers) // false

// 次の 2 つは同じ
if(0 <= score && score <= 100) {..}
if(score in 0..100)
```

- 演算子オーバーロード

```kotlin
data class Money(val amount : Long){
	
	operator fun plus(other : Money): Money{
		return Money(this.amount + other.amount)
	}
}

// 次のように使えます
val sumMoney = money1 + money2
```


### 5. 条件文

- if 文は Kotlin では expression（Java では statement）
    - expression：1 つの値に評価できる文

```kotlin
// Java では不可能ですが Kotlin では if が expression なので OK 
fun getPassOrFail(score : Int): String{
	return if (score >= 50){ "P" } else { "F" }
}
```

- Java では三項演算子が expression ですが、Kotlin では if 文が expression なので三項演算子は不要
    - したがって三項演算子はありません

- when 文

```kotlin
// 基本の使い方、switch の代わりに
fun getGradeWithSwitch(score: Int): String{
	return when(score / 10){
		9 -> "A"
		8 -> "B"
		7 -> "C"
		else -> "D"
	}
}

// 左辺に expression を使うことも可能、is Type なども OK
when(score){
in 90..99 -> "A"
in 80..89 -> "B"
...

// 複数条件を同時にチェック
when (number){
1,0,-1 -> println("1,0,-1입니다")
else -> println("아닙니다")
}

// 条件文のように使う
fun printOdd(score: Int): Int{
	when{
		score % 2 == 1 -> println("홀수")
		score % 2 == 0 -> println("짝수")
		else -> println("넌머임")
	}
	return 1
}

```

### 6. 繰り返し文

- forEach

```kotlin
val numbers = listOf(1L, 2L, 3L)
for (number in numbers){
	println(number)
}
```

- Progression：等差数列。Range（Range は Progression を継承）

```kotlin
val intRange = 1..100
```

### 7. 例外

- try-catch-finally：そのまま使えます（ただし Kotlin では expression、return 等も使用可）
- Kotlin ではすべてのエラーが Unchecked Exception
    - Checked：必ず処理しなければならない例外
    - Unchecked Exception：必ず処理する必要はない例外
- try with resources

```kotlin
// JAVA の try with resources
// try が終わると自動でクローズ。ただし AutoCloseable インタフェースの実装である必要あり。 
public void readFile(String path) throws IOException{
	try(BufferedReader reader = new BufferedReader(new FileReader(path))){
		System.out.println(reader.readLine());
	}
}

// Kotlin では use が try with resources のように動作
fun readFile(path: String){
	BufferedReader(FileReader(path)).use { reader ->
		println(reader.readLine())
	}
}
```

### 8. 関数

- 基本

```kotlin
// 関数本体が式（Expression）だけで構成されていれば = が使える
fun max(a: Int, b: Int) : Int = if(a > b) {a} else {b}
```

- デフォルトパラメータ

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

- 名前付き引数（パラメータ名を明示できる、builder のように使える）

```kotlin
repeat(str = "Hello, World", useNewLine = true)
// この場合、num は指定されていないのでデフォルト値（3）が使われる
// builder を作らずに builder のように使える
```

- 可変長引数（vararg）

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
printAll(*array) // spread 演算子
printAll("Hello", "World", "!!")
```


### 9. クラス

- クラスとプロパティ

```kotlin
class Person(
	val name:String,
	var age:Int
)

// プロパティアクセスのように getter, setter にアクセス
person.age = 10
println(person.age)
```

- コンストラクタ検証、init

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

- カスタム getter、setter

```kotlin
class Person(
	val name:String,
	var age:Int
){
	
	// プロパティのように使える、1 つの式で表現できるものは = で表記
	val isAdult: Boolean
		get() = this.age >= 20
	
	// 上と同じ表現
	fun isAdult(): Boolean{
		return this.age >= 20
	}
}
```

### 10. 継承

- 抽象クラス

```kotlin
// デフォルトで public class 
abstract class Animal(
	protected val species:String,
	protected val legCount:Int
){
	abstract fun move()
}
```

- 継承するクラス

```kotlin
//Java
public class JavaPenguin extends JavaAnimal{
	
	// 子オブジェクトで追加されたフィールド
	private final int wingCount;
	
	public JavaPenguin(String species){
		super(species, 2)
		this.wingCount = 2;
	}

	@Override
	public void move(){
		System.out.println("penguin is moving");
	}

	// 親メソッドの getter をオーバーライド
	@Override
	public int getLegCount(){
		return super.legCount + this.wingCount;
	}
}

//Kotlin
class penguin(
	species : String // プライマリコンストラクタ
) : Animal(species, 2) // Animal を継承して Animal のコンストラクタを直接呼び出す {

	private val wingCount: Int = 2
	
	// アノテーションの代わりに予約語を使用
	override fun move(){
		println("cat is moving")
	}

	override val legCount: Int
		get() = super.legCount + this.wingCount
}

// ただし Kotlin では基本的にすべて final が付いているので、override するには open が必要
abstract class Animal(
	protected val species:String,
	protected open val legCount: Int
){
	abstract fun move()
}
```

- インタフェース

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
	// default キーワードなしで実装可能
	fun act(){ println("swim!") } 
}
interface Flyable{
	fun act(){ println("fly!") }
}
```

- インタフェースを実装するクラス

```kotlin
// Java
public final class JavaPenguin extends JavaAnimal implements JavaFlyable, JavaSwimable{
	
	// コンストラクタ実装..（省略）
	
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

- 抽象プロパティをインタフェースに宣言できる（ただしインタフェースは Backing Field を持てない）

```kotlin
// Kotlin
interface Swimable{
	// 子クラスで実装が期待されるプロパティ
	val swimAbility: Int 
}

class Penguin(): Swimable{
	// 次のように実装
	override val swimAbility: Int
		get() = 3
}
```

- クラス継承時の注意点（初期化の実行順序）
    - 親クラスのコンストラクタや init ブロックでは、子クラスのフィールドにアクセスしないこと！
    - 親クラスのコンストラクタや init ブロックで使われるプロパティには open を使わないこと！

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

// Derived クラスをインスタンス化したときの出力
/*
	Output : 
	Base Class
	0
	Derived Class 
*/

// 理由：親クラス（Base）で number を呼び出すと、子クラスの number が呼ばれる。
// しかし親クラス（Base）のコンストラクタ実行段階では、子クラス（Derived）はまだ初期化されていない
// したがって 0 が出力される
```

- キーワード整理
    - final（デフォルト、省略可）：override できないようにする
    - open：override を許可する
    - abstract：必ず override する必要がある
    - override：親型をオーバーライドする（Java ではアノテーションだが Kotlin ではキーワード）

### 11. アクセス制御

- Java、Kotlin のアクセス制御
    - Public
        - Java/Kotlin：どこからでもアクセス可能
    - Protected
        - Java：同じパッケージ、または子クラスからアクセス可能
        - Kotlin：**宣言されたクラス**、または子クラスのみアクセス可能
    - default（Java）／ internal（Kotlin）
        - Java：同じパッケージのみアクセス可能
        - Kotlin：同じモジュール内のみアクセス可能
            - モジュール：一度にコンパイルされる Kotlin コード（同じ Gradle プロジェクトなど）
    - private
        - Java/Kotlin：宣言されたクラス内のみアクセス可能

- プロパティのアクセス指示子

```kotlin
class Car(
	// name の getter を internal に
	internal val name : String,
	var owner : String,
	_price : Int
){
	// setter のみ private に
	var price = _price
		private set
}

```

- Java/Kotlin 混在使用時の注意点
    - internal キーワードはバイトコード変換時（Java 変換時）に public に変換される
    - したがって Java コードからは、Kotlin の internal キーワードを取得できる
    - Kotlin の protected も同様（Java では同じパッケージからもアクセス可能）

### 12. object

- static 関数と変数
    - static：インスタンス間で値を静的に共有
    - companion object：クラスに付随する唯一の object

```kotlin
//Java
public class JavaPerson{
    private static final int MIN_AGE = 1;

    // 静的ファクトリメソッド
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
		// static の代わりに companion object を使用
    companion object{
				// const があればコンパイル時に割り当て、なければランタイムに割り当て
				// 基本型 + String にのみ割り当て可能
        const val MIN_AGE = 1
        fun newBaby(name: String): Person{
            return Person(name, MIN_AGE)
        }
    }
}

// Person.Companion.newBaby("ABC") として呼び出し可能、
// あるいは @JvmStatic アノテーションを通じて Java の static のように直接アクセス可能

/*
	@JvmStatic
	fun newBaby(name: String) : Person{...}

	その後
	Person.newBaby("ABC") として使用可能
*/
```

- Companion object の活用（Java との違い）
    - 1 つのオブジェクトとして扱われる
    - したがって名前を付けたりインタフェースを実装したりできる

```kotlin
// Kotlin
interface Log{ fun log() }

class Person private constructor(
    var name : String,
    var age : Int,
){
		// 名前を付けられる
    companion object Factory{
        const val MIN_AGE = 1
        fun newBaby(name: String): Person{
            return Person(name, MIN_AGE)
        }
				override fun log(){ println("I am companion object!") }
    }
}

// 名前があれば名前を介してアクセス可能
// Person.Factory.newBaby("ABC");
```

- Singleton
    - object キーワードを使用

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

// その後 Singleton.a のように使用可能
```

- 匿名クラス（Anonymous class）
    - 特定のインタフェースやクラスを継承した実装を一回限り使うとき

```kotlin
// Java
private static void moveSomething(Movable movable){ movable.move(); movable.fly(); }

// 匿名クラス実装
moveSomething(new Movable(){
	@Override
	public void move(){System.out.println("move~");}
	@Override
	public void fly(){System.out.println("fly~");}
}

// Kotlin
private fun moveSomething(movable:Movable){ movable.move(); movable.fly();}

// object キーワードを使って匿名クラスを実装
moveSomething(object : Movable{
	override fun move(){ println("move~") }
	override fun fly(){println("fly~")}
})
```

### 13. 入れ子クラス（nested class）

- あまり使わなさそうなので省略..

### 14. Data class、Enum class、Sealed Class/Interface

- Data class
    - DTO を簡単に使える
    - field, constructor, getter, equals, hashCode, toString が標準で提供される

```kotlin
//Kotlin
data class PersonDto(val name: String, val age: Int)

// 名前付き引数を使うと Builder パターンを使うのと同じ効果
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

// Kotlin（同等のコード）
enum class Country(
	private val code: String
){
	KOREA("KO"),
	AMERICA("US");
}

// when 節を活用して簡単に処理
// またこのように使うと、ENUM CLASS に値が増えるとコンパイラが教えてくれる
fun handleCountry(country: Country){
	when(country){
		Country.KOREA -> LOGIC()
		Country.AMERICA -> LOGIC()
		// ELSE なしでも使える！
	}
}
```

- Sealed Class/Interface
    - 継承可能な抽象クラスを作りたいけれど、外部からは継承できないようにしたい
    - コンパイル時にすべての子クラスの型を覚えておき、ランタイムに別のクラス型を追加できない
    - 子クラスは同じパッケージにある必要がある
    - Enum との違い
        - クラスの継承が可能
        - Enum はインスタンスがシングルトンだが、Sealed Class は複数のインスタンスを持てる

```kotlin
//Kotlin
sealed class Car(
	val name:String,
	val price:Long
)

class Avante: Car("Avante",1_000L)
class Sonata: Car("Sonata",2_000L)
class Grandeur: Car("Grandeur",3_000L)
// 上の Enum class のように when 節が使える
```


### 15. Collections（コレクション）

- Collections
    - コレクションが不変か可変かを設定する必要がある
    - Mutable（可変）コレクション：要素の追加・削除が可能
    - Immutable（不変）コレクション：要素の追加・削除が不可
        - ただし不変コレクションでも、Reference Type の要素のフィールドは変更可能
        - 例えば、Index 1 のデータにアクセスして Price フィールドを 1000 → 2000 ウォンに変更することは可能

- List

```kotlin
/* デフォルト実装：ArrayList */
// 不変リストの生成
val numbers = listOf(100,200)
// 可変リストの生成
val mutableNumbers = mutableListOf(100,200)
// 空のリストの生成
val emptyList = emptyList<Int>()

// numbers.get(0) と同じ
numbers[0]

for(number in numbers){println(number)}
// インデックスも一緒に取得したいとき
for((idx, number) in numbers.withIndex()){println("${idx}, ${number}")}
```

- Set

```kotlin
/* デフォルト実装：LinkedHashSet */
val numbers = setOf(100,200)
```

- Map

```kotlin
val weekMap= mutableMapOf<Int, String>()
// 配列のようにアクセス可能
weekMap[1] = "MONDAY"
weekMap[2] = "TUESDAY"

// to を使って初期化も可能
mapOf(1 to "MONDAY", 2 to "TUESDAY")

for(key in weekMap.keys){println(key) println(weekMap[key]) }
for((key, value) in weekMap.entries){println(key) println(value)} 
```

- コレクションの null 可能性
    - `List<Int?>`：List 自体は null X、リストの中は nullable
    - `List<Int>?`：List 自体は nullable、リストの中は null X
    - `List<Int?>?`：List 自体も nullable、リストの中も nullable

- Java と併用するときの注意点
    - Java は不変・可変リストを区別しない
        - Kotlin の不変リストを Java では可変リストのように扱える
        - Kotlin の non-null リストに、Java から null を追加できる
        - Kotlin で `Collections.unmodifiable` を使うか、防御的ロジックを使う
    - Platform Type で型の問題が生じることがある

### 16. Extension（拡張）、Infix（中置）、inline、Local（ローカル）関数

- 拡張関数
    - Kotlin：Java との 100% 互換性を目指している
        - そこで Java で作られたライブラリに Kotlin コードを追加できると嬉しい
        - クラス内部のメソッドのように使うが、コードは外部に書けるようにしよう！
    - 制限：
        - 拡張関数が public で、private、protected メンバを取得すると、カプセル化が破られるので..
            - カプセル化を維持するために、private、protected メンバを取得できない
        - メンバ関数と拡張関数のシグネチャ（名前）が同じ場合
            - メンバ関数が優先的に呼ばれる
            - したがってメンバ関数があとから追加されると、エラーが起きる可能性がある

```kotlin
// Kotlin

// String クラスを拡張
fun String.lastChar(){ return this[this.length - 1] }

// その後 Kotlin では次のように使える
val str = "ABC"
println(str.lastChar());
```

- 中置関数
    - 変数と引数がそれぞれ 1 つずつしかない場合
    - `var.functionName(argument)` の代わりに
    - `var functionName argument` で呼び出せる

```kotlin
infix fun Int.add(other: Int): Int{
	return this + other
}

// 次のように呼び出せる
3 add 4
```

- インライン関数
    - 我々が知っているあのインライン関数..
    - 関数をパラメータとして渡すオーバーヘッドを減らせる

```kotlin
inline fun Int.add(other: Int): Int{
```

- ローカル関数
    - 関数の中の関数

```kotlin
// Kotlin
fun createPerson(firstName: String, lastName:String): Person{
	if(firstName.isEmpty()){ throw IllegalArgumentException("에러!!")}
	if(lastName:String.isEmpty()){ throw IllegalArgumentException("에러!!")}
	return Person(firstName, lastName)
}

// ローカル関数を使用
fun createPerson(firstName: String, lastName:String): Person{
	// 関数の中の関数
	fun validateName(name: String){if(name.isEmpty()){throw IllegalArgumentException("에러!!")}
	
	validateName(firstName)
	validateName(lastName)
	return Person(firstName, lastName)
}

```

### 17. lambda

- Java では関数を変数に代入したりパラメータとして渡したりできない（第一級オブジェクト X）
    - `Fruit::isApple` のように可能なように見えるが、実際には Predicate というインタフェースを渡している
- Kotlin では関数を変数に代入したりパラメータとして渡したりできる（第一級オブジェクト O）

- lambda の生成と呼び出し

```kotlin
// Kotlin

// 関数の Type も次のように設定可能
val isApple : (Fruit) -> Boolean 
= fun(fruit: Fruit): Boolean {
	return fruit.name == "사과"
}

// 上とまったく同じコード
val isApple2 = {fruit: Fruit -> fruit.name == "사과" }

val isApple3 = { fruit ->
println("사과만 받는 함수입니다.")
fruit.name == "사과" // 複数行 lambda で return を省略すると、最後の行が自動的に return される
}

// lambda の呼び出し、同じコード
isApple(Fruit("사과", 1000))
// 明示的に呼び出し
isApple.invoke(Fruit("사과", 1000))
```

- パラメータとして関数を受け取る

```kotlin
// 関数の Type を書けば、関数を受け取れる
fun filterFruits(fruits: List<Fruit>, filterFunc : (Fruit) -> Boolean) : List<Fruit>{
	val results = mutableListOf<Fruit>()
	for(fruit in fruits){
		if(filterFunc(fruit)){ results.add(fruit)}
	}
	return results
}

// 呼び出し時に次のように lambda を渡して使える
filterFruits(fruit, {fruit -> fruit.name =="사과"} )

// このとき lambda が最後のパラメータなら次のように使える（上のコードと機能は同じ）
filterFruits(fruit) {fruit -> fruit.name =="사과"}

// パラメータが 1 つの場合は、it で矢印の前の部分を置き換え可能
filterFruits(fruit) {it.name =="사과"}
```

- Closure
    - Kotlin は lambda が実行される時点で、参照している変数をすべて捕捉して値を持つ
    - 下のコードでは `targetFruitName = "수박"` の時点を捕捉して持っている
    - このようなデータ構造を Closure と呼び、こうした構造があってこそ lambda が第一級オブジェクトになれる

```kotlin
//Java
String targetFruitName = "바나나"
targetFruitName = "수박"

// Java では lambda で使われる変数は、final 変数か、 
// final キーワードが付いていなくても値が実際には変更されない（Effectively final）変数のみ使用可能
filterFruits(fruits, (fruit) -> targetFruitName.equals(fruit.getName())); // ERROR!

//Kotlin
var targetFruitName = "바나나"
targetFruitName = "수박"
filterFruits(fruits) { it.name == targetFruitName } // It Works!

```

### 18. Collection 組み込み関数 + FP

- **filter**

```kotlin
// 사과（リンゴ）だけを取得
val apples = fruits.filter { fruit -> fruit.name == "사과" }
```

- **filterIndexed**：filter で index が必要な場合

```kotlin
val apples = fruits.filterIndexed { idx, fruit -> 
println(idx)
fruit.name == "사과"
} 

```

- **map, mapIndex**：それぞれの要素に対して lambda 関数を実行

- **mapNotNull**：mapping の結果が null でないものだけを抽出

```kotlin
val values = fruits.filter { it.name == "사과" }
	.mapNotNull { it.nullOrValue() } // null でない値だけ抽出
```

- **all**：条件をすべて満たせば true、そうでなければ false

```kotlin
// すべてリンゴか確認
val isAllApple = fruits.all { it.name == "사과" }
```

- **none**：条件をすべて満たさなければ true、そうでなければ false

```kotlin
// リンゴが 1 つもないか確認
val isNoApple = fruits.none { it.name =="사과" }
```

- **any**：1 つでも満たせば true、そうでなければ false

```kotlin
// 1 つでも 1 万ウォン超の果物があるか確認
val hasExpensiveFruit = fruits.any { it.price > 10_000 }
```

- **count**：個数

```kotlin
// Java の list.size() と同じ
val fruitCount = fruits.count()
```

- **sortedBy, sortedByDescending**：昇順・降順ソート

```kotlin
val sortedFruit = fruit.sortedBy{it.price} // 昇順
val desendSortedFruit = fruit.sortedByDescending{it.price} // 降順
```

- **distinctBy**：変換した値を基準に重複排除

```kotlin
// 果物の種類を確認
val distinctFruitNames = fruits.distinctBy{ it.name } // 名前を基準に重複排除
	.map{it.name}
```

- **first, firstOrNull, last, lastOrNull**：最初・最後の値
    - 空のリストで first、last が取得できる値がない場合は Exception が発生

```kotlin
fruits.first()
fruits.lastOrNull()
```

- **groupBy**：あるキーを基準にグループ化（List → Map）

```kotlin
// 果物の名前を key にリストの中身をグループ化
// key：사과、value：{사과1, 사과2, 사과3...}
val map: Map<String, List<Fruit>> = fruits.groupBy{fruit -> fruit.name } 

// 次のように key、value を同時に処理することも可能
// 名前を Key に、価格 List を作る例
val map2 : Map<String, List<Long>> = fruits
	.groupBy({it.name}, {it.price})
```

- **associateBy**：重複しないキーを基準に Map に変換（List → Map）
    - **ただし id に重複値があると値が失われる可能性がある。**
    - **id が重複している場合、最後の値が value になる。**

```kotlin
val map: Map<Long, Fruit> = fruits.associateBy { fruit -> fruit.id }

val fruitList = listOf<Fruit>(
			// id が同じ
      Fruit(1,"1"),
      Fruit(1,"2"),
      Fruit(1,"3"),
    )

  val associateBy = fruitList.associateBy { it.id }

  println(associateBy.count()) // 1
  for (entry in associateBy) {
      println(entry) // 1=Fruit(id=1, name=3)
  }

// 同様に key、value を同時に処理可能
val map: Map<Long, Long> = fruits
	.associateBy({it.id}, {it.price})
```

- Map でも上記の関数のほとんどを使える

```kotlin
val map: Map<String, List<Fruit>> = fruits.groupBy{fruit -> fruit.name } 
	.filter { (key, value) -> key == "사과" } 
```

- ネストされた Collection の処理
    - flatMap, flatten

```kotlin
var fruitsInList: List<List<Fruit>> = listOf(
	listOf(Fruit(1,"사과"), Fruit(2,"수박"), Fruit(3,"사과")),
	listOf(Fruit(4,"바나나"), Fruit(5,"사과"), Fruit(6,"바나나")),
)

// リスト構造に関係なく、リンゴをすべて取り出してください
// output: listOf(Fruit(1,"사과"), Fruit(3,"사과"), Fruit(5,"사과"))
val appleList : List<Fruit>
	= fruitsInList.flatMap { list -> list.filter{it.name == "사과"}

// 2 次元リストを 1 次元に
val flattenList : List<Fruit>
	= fruitsInList.flatten();
```


### 19. TakeIf、scope Function

- **TakeIf**：与えられた条件を満たせばその値を、そうでなければ null を返す
- **TakeUnless**：与えられた条件を満たさなければその値を、そうでなければ null を返す

```kotlin
// この関数を
fun getFruitOrNull(fruit: Fruit): Fruit?{
	return if(fruit.name == "사과"){fruit} else {null}
}
// 次のように使える
val getfruit = fruit.takeIf { it.name == "사과" }
Fruit(1L, "사과") // この場合 fruit
Fruit(1L, "바나나") // この場合 null を返す
```

- Scope function：一時的なスコープを形成する関数
    - つまり lambda を使って一時的なスコープを作り
        - コードをより簡潔にしたり
        - method chaining に活用したりする関数

```kotlin
// 次の 2 つの関数は同じ
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

- Scope function の種類
    - **let, run**：lambda の結果を返す
        - let：it を使う（省略不可だが別の名前を付けられる）
        - run：this を使う（省略可能だが別の名前は付けられない）
    - **also, apply**：オブジェクト自体を返す
        - also：it を使う
        - apply：this を使う
    - **with**：唯一拡張関数ではない。this を使う、lambda の結果を返す

```kotlin
val value1 = person.let{ it.age }  // value1 の値：person.age
// person.let{ person -> person.age } のように、別の名前を付けられる
val value2 = person.run{ this.age } // value2 の値：person.age
// person.run{ age } のように、this を省略可能

val value3 = person.also{ it.age } // value3 の値：person
val value4 = person.apply{ this.age } // value4 の値：person 

with(person){
	println(name)
	println(age)
}

/* 
it を使う場合：通常関数をパラメータとして受け取る block : (T) -> R : R
this を使う場合：拡張関数をパラメータとして受け取る block: T.() -> R : R
気になれば実際の実装を確認してみよう....
*/
```
