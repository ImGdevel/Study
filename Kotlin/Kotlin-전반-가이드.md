# Kotlin 전반 가이드 — Java 개발자 기준

> 태그: `#kotlin` `#jvm` `#coroutines`<br>
> 작성일: 2026-07-13<br>
> 최종 수정일: 2026-07-13

---

## 목차

1. [기본 문법](#1-기본-문법)
2. [클래스와 OOP](#2-클래스와-oop)
3. [함수형 프로그래밍](#3-함수형-프로그래밍)
4. [제네릭과 타입 시스템](#4-제네릭과-타입-시스템)
5. [Coroutines](#5-coroutines)
6. [Java 상호운용성](#6-java-상호운용성)

---

## 1. 기본 문법

### 1-1. 변수: val / var

```kotlin
val name: String = "Kotlin"    // 재할당 불가 (Java final)
var count: Int = 0              // 재할당 가능
val list = mutableListOf(1,2,3) // val이지만 내부 상태는 변경 가능
list.add(4)  // OK — val은 참조만 고정
```

- `val` ↔ Java `final` (참조 고정, 객체 내부 필드 변경은 가능)
- `var` ↔ 일반 Java 변수
- 타입 추론: `val x = 42` (Int 자동 추론), 명시도 가능 `val x: Int = 42`

### 1-2. 타입 시스템

| Kotlin | Java | 비고 |
|---|---|---|
| `Int`, `Long`, `Double` | `int`, `long`, `double` | JVM은 primitive로 컴파일, Kotlin 소스는 객체처럼 씀 |
| `Int?`, `Long?` | `Integer`, `Long` | nullable 버전 = JVM boxed type |
| `Any` | `Object` | Kotlin 타입 계층의 최상위 |
| `Unit` | `void` | 값으로 사용 가능한 void — Unit 타입의 인스턴스 존재 |
| `Nothing` | 없음 | 정상 리턴이 없는 함수 타입 (`throw`, `TODO()`) |
| `String` | `String` | Kotlin String은 **non-nullable** — Java String은 Kotlin의 `String?`에 더 가깝다 |

### 1-3. Null Safety — Kotlin의 가장 큰 차이

Java에서 NPE가 발생하는 이유: 컴파일러가 null 가능성을 타입 수준에서 추적하지 않는다.
Kotlin은 타입 자체를 `String`(non-null)과 `String?`(nullable)로 분리해 컴파일 시점에 강제한다.

```kotlin
val a: String  = "hello"    // null 대입 불가 — 컴파일 에러
val b: String? = null       // OK

// safe call ?.
val len: Int? = b?.length   // b가 null이면 null 반환

// Elvis operator ?:
val len2: Int = b?.length ?: 0  // null이면 0

// not-null assertion !! — null이면 NPE 발생, 신중히
val len3: Int = b!!.length

// Smart Cast: null 체크 이후 컴파일러가 자동 캐스팅
if (b != null) {
    println(b.length)   // 여기서 b는 String으로 스마트 캐스팅됨
}
```

Java의 `Optional`이 런타임 래퍼라면, Kotlin의 `?`는 컴파일 타임 타입 제약이다 — 성능 오버헤드 없음.

### 1-4. 함수 선언

```kotlin
// 기본
fun add(a: Int, b: Int): Int {
    return a + b
}

// 단일 표현식 — return 생략, 리턴 타입 추론 가능
fun add(a: Int, b: Int) = a + b

// 기본 파라미터 (Java에서는 오버로딩으로 처리)
fun greet(name: String = "World") = "Hello, $name"

// 네임드 아규먼트
greet(name = "Kotlin")
```

Java와의 차이:
- 리턴 타입이 파라미터 **뒤**에 온다 (`fun name(): Type`)
- 세미콜론 불필요
- 기본값 파라미터로 Java 메서드 오버로딩 다수를 단일 함수로 대체

### 1-5. 제어문 — Expression vs Statement

Kotlin에서 `if`와 `when`은 **expression** (값을 반환한다).

```kotlin
// if expression
val abs = if (x >= 0) x else -x

// when — Java switch보다 훨씬 강력
val result = when (x) {
    0       -> "zero"
    1, 2    -> "one or two"
    in 3..9 -> "3 to 9"
    is String -> "it's a string"
    else    -> "other"
}

// for
for (i in 1..10) println(i)               // inclusive range
for (i in 1 until 10) println(i)          // exclusive (1~9)
for (i in 10 downTo 1 step 2) println(i)
for ((index, value) in list.withIndex()) println("$index: $value")
```

### 대표 질문 — 기본 문법

- `val`로 선언한 객체의 내부 필드를 변경할 수 있는가?
  → 가능. `val`은 참조만 고정. Java `final`과 동일한 의미.
- Java의 `String`은 Kotlin의 `String`인가 `String?`인가?
  → `String?`에 더 가깝다. Java는 모든 참조 타입에 null이 가능하지만 컴파일러가 추적하지 않는다.
- `fun add(a: Int, b: Int): Int = a + b`에서 Java 대비 어색한 점은?
  → 리턴 타입 위치(파라미터 뒤), `=`로 바로 표현식 반환, 세미콜론 없음.
- `when`과 Java `switch`의 차이는?
  → `when`은 expression(값 반환), 타입 체크/범위 체크/스마트 캐스트 지원, fall-through 없음.

---

## 2. 클래스와 OOP

### 2-1. 클래스 선언과 생성자

```java
// Java — boilerplate 많음
public class Person {
    private final String name;
    private int age;
    public Person(String name, int age) { this.name = name; this.age = age; }
    public String getName() { return name; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
}
```

```kotlin
// Kotlin — primary constructor에서 프로퍼티 선언까지 한 줄
class Person(val name: String, var age: Int)
```

- `val`/`var`를 파라미터에 붙이면 자동으로 프로퍼티(getter/setter)가 생성된다.
- Kotlin의 **프로퍼티**는 Java의 필드 + getter + (var이면) setter의 묶음이다.
- Kotlin 클래스는 기본적으로 **final** — Java와 반대. 상속 허용하려면 `open` 키워드 필요.

```kotlin
class Person(val name: String) {
    constructor(name: String, age: Int) : this(name) { }

    init {
        require(name.isNotBlank()) { "name must not be blank" }
    }
}
```

### 2-2. data class

```kotlin
data class Point(val x: Int, val y: Int)

val p1 = Point(1, 2)
val p2 = p1.copy(y = 10)     // 불변 복사본 생성

// 자동 생성: equals(), hashCode(), toString(), copy(), component1/2()
val (x, y) = p1              // 구조 분해 (componentN() 기반)
```

Java 16+ `record`와 유사하지만 `copy()`가 있고 `var` 필드도 허용한다.

### 2-3. sealed class

```kotlin
sealed class Result<out T>
data class Success<T>(val data: T)    : Result<T>()
data class Failure(val error: Throwable) : Result<Nothing>()
object Loading                         : Result<Nothing>()

fun handle(result: Result<String>) = when (result) {
    is Success -> println(result.data)
    is Failure -> println(result.error.message)
    Loading    -> println("loading...")
    // else 불필요 — 컴파일러가 exhaustive 보장
}
```

- `sealed class`: 같은 파일 안에서만 하위 타입 정의 가능 → 닫힌 계층
- `enum class`와 차이: enum은 각 상수가 동일 타입의 단일 인스턴스, sealed는 각 서브클래스가 서로 다른 상태(필드)를 가질 수 있다.

### 2-4. object / companion object

```kotlin
// object — 싱글턴 (언어 수준, thread-safe 보장)
object AppConfig {
    val baseUrl = "https://api.example.com"
}

// companion object — Java의 static에 해당
class MyClass {
    companion object {
        fun create(): MyClass = MyClass()
        const val TAG = "MyClass"   // 컴파일 타임 상수
    }
}
MyClass.create()

// object expression — Java 익명 클래스에 해당
val comparator = object : Comparator<Int> {
    override fun compare(a: Int, b: Int) = a - b
}
```

### 대표 질문 — 클래스와 OOP

- Kotlin 클래스가 기본 final인 이유는?
  → Effective Java: "상속을 위해 설계하고 문서화하거나, 금지하라." 의도적으로 `open`해야 상속 허용.
- `data class`가 자동 생성하는 메서드는?
  → `equals()`, `hashCode()`, `toString()`, `copy()`, `componentN()` (구조 분해용)
- `sealed class`와 `enum class`의 차이는?
  → enum: 각 상수가 동일 타입의 인스턴스(상태 고정). sealed: 각 서브클래스가 다른 필드/상태 보유 가능.
- `companion object` 함수를 Java에서 static처럼 쓰려면?
  → `@JvmStatic` 필요. 없으면 Java에서 `MyClass.Companion.create()` 형태가 된다.

---

## 3. 함수형 프로그래밍

### 3-1. 람다

```kotlin
val sum: (Int, Int) -> Int = { a, b -> a + b }

// 마지막 파라미터가 람다면 괄호 밖으로 (trailing lambda)
list.forEach { println(it) }     // it = 단일 파라미터 암묵적 이름

// 고차함수
fun operate(a: Int, b: Int, op: (Int, Int) -> Int) = op(a, b)
operate(3, 4) { x, y -> x + y }   // 7
```

### 3-2. 확장함수 (Extension Function)

기존 클래스를 수정하지 않고 함수를 "추가"하는 것처럼 보이게 한다.
내부적으로는 수신 객체를 첫 번째 인자로 받는 **정적 함수**로 컴파일된다 — 리플렉션 없음, 런타임 비용 없음.

```kotlin
fun String.isPalindrome(): Boolean = this == this.reversed()
"racecar".isPalindrome()   // true

fun String?.orEmpty(): String = this ?: ""
```

Java 관점: `StringUtils.isPalindrome(str)` → `str.isPalindrome()` — 가독성 향상.
오버라이드 불가, 다형성 적용 안 됨 — 정적 디스패치임을 명심.

### 3-3. inline 함수

람다를 받는 고차함수는 매 호출마다 Function 객체를 생성한다.
`inline`을 붙이면 호출부에 함수 본문을 직접 삽입 — 객체 생성과 가상 디스패치 제거.

```kotlin
inline fun <T> measureTime(block: () -> T): T {
    val start = System.nanoTime()
    val result = block()
    println("${System.nanoTime() - start}ns")
    return result
}
```

### 3-4. 컬렉션 API

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6)

numbers.filter { it % 2 == 0 }             // [2, 4, 6]
numbers.map { it * 2 }                     // [2, 4, 6, 8, 10, 12]
numbers.flatMap { listOf(it, it * 10) }
numbers.fold(0) { acc, x -> acc + x }      // 21
numbers.groupBy { if (it % 2 == 0) "even" else "odd" }
numbers.partition { it > 3 }               // Pair([4,5,6], [1,2,3])
```

**Kotlin 컬렉션 vs Java Stream:**

| 항목 | Kotlin 컬렉션 | Java Stream |
|---|---|---|
| 평가 방식 | 즉시 평가 (eager) | 지연 평가 (lazy) |
| 재사용 | 결과가 List — 재사용 가능 | 한 번 소비하면 재사용 불가 |
| Lazy 필요 시 | `.asSequence()` | 기본이 lazy |

```kotlin
// Sequence — 지연 평가, Java Stream과 유사
(1..1_000_000)
    .asSequence()
    .filter { it % 2 == 0 }
    .map { it * 2 }
    .take(10)
    .toList()   // 이 시점에 실제 계산, 중간 리스트 미생성
```

### 대표 질문 — 함수형 프로그래밍

- 확장함수는 실제로 클래스를 수정하는가?
  → 아니다. 컴파일 시 정적 함수로 변환된다. 오버라이드 불가, 다형성 적용 안 됨.
- `inline` 함수의 목적은?
  → Function 객체 생성 비용과 가상 디스패치 제거. 성능 민감한 유틸리티에 적합.
- Kotlin 컬렉션과 Sequence의 차이는?
  → 컬렉션은 즉시 평가(각 연산마다 새 리스트 생성), Sequence는 지연 평가. 원소가 많거나 중간 take가 있으면 Sequence 유리.
- Stream의 장단점은?
  → 장: 지연 평가로 불필요한 중간 연산 생략, 병렬 처리 용이. 단: 재사용 불가, 디버깅 어려움.

---

## 4. 제네릭과 타입 시스템

### 4-1. 공변성 / 반공변성 (Variance)

| Java | Kotlin | 의미 |
|---|---|---|
| `? extends T` | `out T` | 공변 (covariant) — 읽기/생산 전용 |
| `? super T` | `in T` | 반공변 (contravariant) — 쓰기/소비 전용 |
| `T` | `T` | 무공변 (invariant) |

```kotlin
// out T — 생산자, 읽기만 허용
interface Source<out T> { fun next(): T }
val strings: Source<String> = TODO()
val any: Source<Any> = strings   // OK — String은 Any의 서브타입

// in T — 소비자, 쓰기만 허용
interface Sink<in T> { fun accept(value: T) }
val anySink: Sink<Any> = TODO()
val stringSink: Sink<String> = anySink  // OK
```

기억법: PECS (Producer Extends, Consumer Super) → Kotlin: Producer=`out`, Consumer=`in`

### 4-2. reified 타입 파라미터

Java 제네릭은 런타임에 타입이 소거(type erasure)된다. 일반 제네릭 함수에서 `T::class`를 쓸 수 없다.

```kotlin
// inline + reified — 런타임에도 T 타입 정보 보존
inline fun <reified T> parse(json: String): T =
    gson.fromJson(json, T::class.java)

val user = parse<User>(jsonString)
```

`reified`는 반드시 `inline`과 함께 써야 한다.
함수 본문이 호출부에 삽입될 때 T가 실제 타입으로 대체되어 타입 소거를 피한다.

### 대표 질문 — 제네릭

- Java의 `? extends T`에 해당하는 Kotlin 문법은?
  → `out T` (공변, 생산자 위치)
- `reified`가 필요한 이유는?
  → JVM 제네릭 타입 소거 때문에 런타임에 T 정보가 없다. `inline + reified`로 실제 타입이 inlining되어 타입 정보가 유지된다.

---

## 5. Coroutines

### 5-1. 왜 Coroutines인가

Thread-per-request의 한계(블로킹 I/O 시 스레드 낭비)를 해결하는 두 가지 방법:
1. 비동기 콜백 / ReactiveX — 코드가 복잡해지고 에러 처리가 어렵다.
2. **Coroutines** — 동기 코드처럼 보이지만 블로킹 없이 중단/재개한다.

### 5-2. suspend 함수

```kotlin
suspend fun fetchUser(id: Long): User {
    return httpClient.get("/users/$id")  // I/O 대기 중 스레드를 점유하지 않음
}
```

- `suspend` 키워드: 함수가 일시 중단될 수 있다는 선언.
- 컴파일러가 CPS(Continuation Passing Style)로 변환: `Continuation<T>` 파라미터가 내부적으로 추가된다.
- 중단 시 현재 스레드는 반환되어 다른 코루틴이 사용할 수 있다.
- `suspend` 함수는 코루틴 또는 다른 `suspend` 함수 내에서만 호출 가능.

### 5-3. CoroutineScope과 Structured Concurrency

```kotlin
val scope = CoroutineScope(Dispatchers.IO)

scope.launch {
    val a = async { fetchUser(1) }
    val b = async { fetchUser(2) }
    println(a.await().name + b.await().name)  // 병렬 실행 후 결과 합산
}

scope.cancel()   // 산하 모든 코루틴 취소

// coroutineScope 빌더 — 자식 완료 시까지 suspend
suspend fun fetchBoth() = coroutineScope {
    val user = async { fetchUser(1) }
    val post = async { fetchPost(1) }
    user.await() to post.await()
}
```

**Structured Concurrency 규칙:**
- 자식 코루틴은 부모 Scope보다 오래 살 수 없다.
- 한 자식 예외 → 부모와 다른 형제 코루틴 취소.
- `SupervisorScope` 사용 시 형제 코루틴 독립 실행 가능.

### 5-4. Dispatcher

```kotlin
launch(Dispatchers.IO) {        // I/O 블로킹 작업 — 스레드 풀 최대 64
    val data = readFile()
}
launch(Dispatchers.Default) {   // CPU 집약 작업 — CPU 코어 수만큼 스레드
    val result = heavyComputation()
}
withContext(Dispatchers.IO) {   // 블록 내에서만 Dispatcher 전환
    database.query()
}
```

### 5-5. launch vs async

```kotlin
// launch — 결과 불필요, Job 반환 (fire-and-forget)
val job: Job = launch {
    delay(1000)
    println("done")
}
job.cancel()

// async — 결과 필요, Deferred<T> 반환
val deferred: Deferred<Int> = async { 42 }
val result: Int = deferred.await()
```

### 5-6. Flow — 비동기 스트림

```kotlin
fun temperatures(): Flow<Double> = flow {
    while (true) {
        emit(readSensor())
        delay(1000)
    }
}

temperatures()
    .filter { it > 37.0 }
    .map { "Fever: $it" }
    .collect { println(it) }   // 터미널 연산, 여기서 실행 시작 (cold)
```

- **Cold**: `collect` 호출 전까지 실행 안 됨.
- **Hot**: `SharedFlow`, `StateFlow` — 구독자 없어도 발행.

### 5-7. 예외 처리

```kotlin
// launch — CoroutineExceptionHandler 필요
val handler = CoroutineExceptionHandler { _, e -> println("Error: $e") }
launch(handler) { throw RuntimeException("fail") }

// async — await() 호출 시 예외 전파
try {
    async { throw RuntimeException() }.await()
} catch (e: RuntimeException) {
    println("caught")
}
```

### 대표 질문 — Coroutines

- `suspend` 함수가 스레드를 블록하지 않는 원리는?
  → 컴파일러가 CPS로 변환. 중단 시 현재 스레드에 Continuation 콜백만 등록하고 스레드를 반환. 재개 시 Dispatcher가 스레드 풀에서 다시 스케줄.
- `launch`와 `async`의 차이는?
  → `launch`는 결과값 없이 Job 반환(fire-and-forget), `async`는 `Deferred<T>` 반환 후 `await()`로 결과 획득.
- Structured Concurrency가 없으면 어떤 문제가 생기는가?
  → 부모 취소 시 자식이 계속 실행되거나, 예외가 누락될 수 있음.
- `Dispatchers.IO`와 `Dispatchers.Default`의 차이는?
  → IO는 블로킹 I/O용(최대 64 스레드), Default는 CPU bound용(CPU 코어 수로 제한).
- Flow가 cold인 이유는?
  → `collect` 호출 전까지 `flow { }` 블록이 실행되지 않기 때문. 구독마다 독립 스트림.
- Event Loop가 Thread 하나로 동작 가능한 이유는?
  → I/O 대기 시 스레드를 블록하지 않고 이벤트 큐에 콜백 등록 후 반환, I/O 완료 시 콜백 실행. Coroutines도 같은 원리 — suspend 시 스레드 반환, 완료 시 재스케줄.

---

## 6. Java 상호운용성

### 6-1. Platform Type (T!)

Java에서 오는 타입은 Kotlin 컴파일러가 null 여부를 알 수 없다 → **platform type** (`T!`).

```kotlin
val name = javaClass.findName()   // 타입: String! (platform type)
name.length   // 컴파일 되지만 NPE 위험

// 안전하게: 명시적으로 nullable 처리
val name: String? = javaClass.findName()
name?.length
```

Java 쪽에서 `@NonNull` / `@Nullable` 애노테이션을 붙이면 Kotlin이 이를 인식해 타입을 확정한다.

### 6-2. @JvmStatic / @JvmField / @JvmOverloads

```kotlin
class Config {
    companion object {
        @JvmStatic
        fun create(): Config = Config()
        // 없으면 Java에서: Config.Companion.create()
        // 있으면 Java에서: Config.create() (static처럼)

        @JvmField
        val DEFAULT_TIMEOUT = 30    // Java에서 getter 없이 필드 직접 접근
    }

    @JvmOverloads
    fun connect(host: String, port: Int = 8080, timeout: Int = 30) { }
    // Java에서 connect(host), connect(host,port), connect(host,port,timeout) 세 오버로딩 생성
}
```

### 6-3. Checked Exception

```kotlin
// Java checked exception은 Kotlin에서 강제 안 함
// IOException도 try-catch 없이 컴파일됨 — 실수로 누락 가능
fun readFile(path: String): String = File(path).readText()
```

### 대표 질문 — 상호운용성

- Kotlin에서 Java 코드를 쓸 때 NPE 위험이 있는 이유는?
  → Java는 null 여부를 타입에 표현하지 않으므로 Kotlin이 platform type으로 처리, 컴파일러 경고 없음.
- `companion object` 함수를 Java에서 static처럼 쓰려면?
  → `@JvmStatic` 필요. 없으면 `ClassName.Companion.method()` 형태.
- Kotlin에서 Java의 checked exception을 try-catch 안 해도 컴파일되는 이유는?
  → Kotlin은 checked exception 개념이 없다. 모든 예외가 unchecked처럼 취급된다.

---

## 참고

- [Kotlin 공식 문서 — kotlinlang.org](https://kotlinlang.org/docs/home.html) — 확인일 2026-07-13
  - [Basic types](https://kotlinlang.org/docs/basic-types.html)
  - [Null safety](https://kotlinlang.org/docs/null-safety.html)
  - [Classes](https://kotlinlang.org/docs/classes.html)
  - [Extensions](https://kotlinlang.org/docs/extensions.html)
  - [Generics: in, out, where](https://kotlinlang.org/docs/generics.html)
  - [Coroutines basics](https://kotlinlang.org/docs/coroutines-basics.html)
  - [Flow](https://kotlinlang.org/docs/flow.html)
  - [Java interop](https://kotlinlang.org/docs/java-interop.html)

## 관련 내용

- [가비지-컬렉션](../Java/가비지-컬렉션.md) — JVM 공유, ZGC/G1 동일 적용
- [JDK-버전별-주요-변경사항](../Java/JDK-버전별-주요-변경사항.md) — Kotlin 1.9+는 JDK 17/21 타겟 권장
- [Blocking-Nonblocking-Sync-Async](../운영체제/Blocking-Nonblocking-Sync-Async.md) — Coroutines의 non-blocking 동작 원리
