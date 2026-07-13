# Kotlin 심화 — 실무 함정과 Kotlin다운 코드

> 태그: `#kotlin` `#pitfalls` `#coroutines` `#spring`<br>
> 작성일: 2026-07-13<br>
> 최종 수정일: 2026-07-13

---

## 목차

1. [Null Safety 실무 함정](#1-null-safety-실무-함정)
2. [Scope Functions 선택 기준](#2-scope-functions-선택-기준)
3. [스마트 캐스트 함정](#3-스마트-캐스트-함정)
4. [확장함수 함정](#4-확장함수-함정)
5. [data class / sealed class 심화](#5-data-class--sealed-class-심화)
6. [Coroutines 실무 함정](#6-coroutines-실무-함정)
7. [Spring + Kotlin 통합 함정](#7-spring--kotlin-통합-함정)
8. [Value Class (Inline Class)](#8-value-class-inline-class)
9. [성능 고려사항](#9-성능-고려사항)

---

## 1. Null Safety 실무 함정

### 1-1. !! 과용 — "Kotlin NPE"

`!!`는 null이면 `NullPointerException`을 던진다. Kotlin에서 NPE를 보는 가장 흔한 원인.

```kotlin
// 나쁜 예 — !! 남용
val user = userRepository.findById(id)!!
val name = user.profile!!.name!!

// 좋은 예 — 의도 명확하게
val user = userRepository.findById(id)
    ?: throw NotFoundException("User $id not found")
val name = user.profile?.name ?: "Unknown"
```

규칙: `!!`는 "이 값이 null이면 프로그래밍 버그다"가 확실한 경우에만. 그 외엔 `?:` + 의미있는 예외.

### 1-2. lateinit var — 초기화 전 접근

`lateinit`은 초기화 전 접근 시 `UninitializedPropertyAccessException`을 던진다. NPE와 같은 패턴.

```kotlin
class Service {
    lateinit var repository: Repository   // DI로 주입받을 예정

    fun doSomething() {
        if (::repository.isInitialized) {  // 초기화 확인 가능
            repository.save(data)
        }
    }
}
```

- `lateinit`은 non-null 타입에만 사용 가능 (Int, Boolean 불가 — boxing 필요)
- 테스트에서 `@BeforeEach` 초기화 누락 시 자주 발생
- 생성자 주입(constructor injection)을 쓰면 아예 `lateinit` 불필요

### 1-3. Platform Type (T!) — Java 코드와 연동 시 NPE

Java에서 오는 타입은 컴파일러가 null 여부를 모른다. 명시적으로 처리하지 않으면 런타임 NPE.

```kotlin
// Java: @Nullable String findName(Long id)
val name = javaService.findName(userId)   // 타입: String! (platform type)
println(name.length)   // 컴파일 OK, 런타임 NPE 가능

// 안전하게
val name: String? = javaService.findName(userId)   // 명시적으로 nullable
println(name?.length ?: 0)
```

Java 라이브러리를 쓸 때는 반환값을 항상 `T?`로 받는 습관 권장.

### 1-4. let으로 nullable 체인 처리

```kotlin
// 중첩 null 체크 — 가독성 나쁨
if (user != null && user.profile != null && user.profile.avatar != null) {
    loadImage(user.profile.avatar.url)
}

// let 체이닝
user?.profile?.avatar?.let { avatar ->
    loadImage(avatar.url)
}

// 여러 nullable을 동시에 사용할 때
val result = if (a != null && b != null) combine(a, b) else null
// 또는
val result = a?.let { av -> b?.let { bv -> combine(av, bv) } }
```

---

## 2. Scope Functions 선택 기준

가장 혼란스러운 부분 중 하나. 선택 기준은 두 가지: **수신 객체 참조(it vs this)** + **반환값**.

| 함수 | 수신 객체 | 반환값 | 주 용도 |
|---|---|---|---|
| `let` | it | 람다 결과 | nullable 처리, 변환 |
| `run` | this | 람다 결과 | 객체 설정 후 다른 값 반환 |
| `apply` | this | 수신 객체 | 객체 설정(빌더 패턴) |
| `also` | it | 수신 객체 | 사이드 이펙트(로깅, 검증) |
| `with` | this | 람다 결과 | 수신 객체 없이 블록 실행 |

```kotlin
// let — nullable 처리, 변환
val length = name?.let { it.trim().length }

// apply — 객체 초기화/빌더 (this = 객체, 객체 반환)
val request = HttpRequest().apply {
    url = "https://api.example.com"
    method = "POST"
    headers["Content-Type"] = "application/json"
}

// also — 사이드 이펙트 (it = 객체, 객체 반환, 체이닝 유지)
val user = createUser()
    .also { logger.info("Created user: ${it.id}") }
    .also { validateUser(it) }

// run — 설정 후 다른 값 필요 (this = 객체, 람다 결과 반환)
val result = connection.run {
    connect()
    fetchData()   // 이 값이 반환됨
}

// with — 이미 not-null인 객체에 여러 작업
with(userDto) {
    println(name)
    println(email)
    println(age)
}
```

**실무 함정: 과도한 중첩**

```kotlin
// 나쁜 예 — 콜백 헬처럼 읽힘
user?.let { u ->
    u.profile?.let { p ->
        p.avatar?.let { a ->
            a.url?.also { url ->
                loadImage(url)
            }
        }
    }
}

// 좋은 예 — safe call 체이닝
user?.profile?.avatar?.url?.let { loadImage(it) }
```

---

## 3. 스마트 캐스트 함정

### 3-1. var 필드는 스마트 캐스트 불가

컴파일러는 `var`의 값이 null 체크와 사용 시점 사이에 바뀔 수 있다고 판단한다 (멀티스레드 환경).

```kotlin
class Foo {
    var name: String? = null

    fun printLength() {
        if (name != null) {
            println(name.length)   // 컴파일 에러: Smart cast impossible
            // name이 다른 스레드에서 null로 바뀔 수 있음
        }
    }
}

// 해결책 1: 로컬 변수에 캡처
fun printLength() {
    val n = name
    if (n != null) {
        println(n.length)   // OK — n은 로컬 val, 변경 불가
    }
}

// 해결책 2: let 사용
fun printLength() {
    name?.let { println(it.length) }
}
```

### 3-2. 타입 체크 후 스마트 캐스트

```kotlin
fun process(value: Any) {
    if (value is String) {
        println(value.length)   // OK — value는 String으로 스마트 캐스트
        println(value.uppercase())
    }

    // when에서도 동작
    when (value) {
        is String -> println(value.length)
        is Int    -> println(value * 2)
        is List<*> -> println(value.size)
    }
}
```

---

## 4. 확장함수 함정

### 4-1. 멤버 함수 vs 확장함수 우선순위

**멤버 함수가 항상 이긴다.** 같은 시그니처의 확장함수를 정의해도 멤버 함수가 호출된다.

```kotlin
class Dog {
    fun speak() = "Woof"
}

fun Dog.speak() = "Extended Woof"   // 절대 호출 안 됨

Dog().speak()   // "Woof" — 멤버 함수 호출
```

### 4-2. 확장함수는 오버라이드 안 됨 — 정적 디스패치

확장함수는 런타임 타입이 아닌 **컴파일 타임 타입**으로 디스패치된다.

```kotlin
open class Animal
class Dog : Animal()

fun Animal.sound() = "..."
fun Dog.sound() = "Woof"

val animal: Animal = Dog()
println(animal.sound())   // "..." — Animal.sound() 호출 (Dog가 아님!)
// 멤버 함수였다면 "Woof"가 출력됐을 것
```

확장함수로 다형성을 기대하면 안 된다. 다형적 동작이 필요하면 멤버 함수 또는 인터페이스를 써야 한다.

### 4-3. nullable 수신 객체 확장함수

```kotlin
fun String?.orEmpty(): String = this ?: ""

// null에도 안전하게 호출 가능
val s: String? = null
println(s.orEmpty())   // "" — NPE 없음
```

`toString()`도 nullable에 안전하게 호출 가능한 이유: `Any?.toString()` 확장이 정의되어 있기 때문.

---

## 5. data class / sealed class 심화

### 5-1. data class copy() — 얕은 복사

`copy()`는 **얕은 복사(shallow copy)**다. 중첩 객체는 같은 참조를 가리킨다.

```kotlin
data class Address(var street: String)
data class User(val name: String, val address: Address)

val user1 = User("Kim", Address("Seoul"))
val user2 = user1.copy()

user2.address.street = "Busan"   // user1.address.street도 "Busan"으로 바뀜!
println(user1.address.street)    // "Busan" — 의도치 않은 변경
```

해결책: 중첩 객체도 함께 copy 처리.
```kotlin
val user2 = user1.copy(address = user1.address.copy(street = "Busan"))
```

### 5-2. data class와 상속

`data class`는 다른 클래스를 상속할 수 없다 (인터페이스는 가능).
`sealed class`의 서브클래스로 `data class`를 쓰는 패턴이 관용적.

```kotlin
sealed class ApiResponse<out T>
data class Success<T>(val data: T) : ApiResponse<T>()
data class Error(val code: Int, val message: String) : ApiResponse<Nothing>()
object Loading : ApiResponse<Nothing>()
```

### 5-3. sealed class — when 절 exhaustiveness

sealed class의 장점: `else` 없이 when을 쓰면 새 서브클래스 추가 시 컴파일 에러 발생.

```kotlin
fun handle(response: ApiResponse<User>): String = when (response) {
    is Success -> response.data.name
    is Error   -> "Error ${response.code}: ${response.message}"
    Loading    -> "Loading..."
    // else 없음 — 새 서브클래스 추가 시 여기서 컴파일 에러 → 처리 누락 방지
}
```

단, `when`을 **expression**으로 쓸 때만 exhaustive 검사가 강제된다. statement로 쓰면 경고도 안 남.

```kotlin
// expression — exhaustive 강제 (값 반환 필요)
val result: String = when (response) { ... }

// statement — exhaustive 강제 안 됨 (주의)
when (response) { ... }
```

---

## 6. Coroutines 실무 함정

### 6-1. GlobalScope — 절대 쓰지 말것

`GlobalScope`는 애플리케이션 전체 생명주기와 묶인다. 취소가 안 되고, 예외 핸들러도 없다.
Kotlin 1.5부터 `@DelicateCoroutinesApi`로 마킹되어 경고가 뜬다.

```kotlin
// 나쁜 예
GlobalScope.launch {
    sendEmail(user)   // 앱이 종료돼도 계속 실행, 예외 누락 가능
}

// 좋은 예 — 생명주기 있는 Scope 사용
class UserService(private val scope: CoroutineScope) {
    fun notifyAsync(user: User) {
        scope.launch {
            sendEmail(user)
        }
    }
}
```

### 6-2. runBlocking — 메인/IO 스레드에서 데드락

`runBlocking`은 현재 스레드를 블록한다.
코루틴 내부에서 `runBlocking`을 호출하면 데드락이 발생할 수 있다.

```kotlin
// 데드락 시나리오
fun main() = runBlocking {
    // 현재 스레드를 점유 중
    val result = runBlocking {   // 같은 스레드를 또 블록 시도 → 데드락
        delay(1000)
        42
    }
}

// 올바른 패턴: runBlocking은 최상위 진입점에만, 내부는 coroutineScope
fun main() = runBlocking {
    val result = coroutineScope {   // suspend — 스레드 반환
        async { compute() }.await()
    }
}
```

- `runBlocking`: 스레드 블록 (진입점, 테스트용)
- `coroutineScope`: suspend — 스레드 반환 (내부 로직)

### 6-3. blocking call in suspend function

suspend 함수 안에서 blocking I/O를 그냥 호출하면 **Dispatcher 스레드를 블록**한다.

```kotlin
// 나쁜 예 — Dispatchers.Default(CPU bound용) 스레드를 블록함
suspend fun fetchData(): String {
    return URL("https://api.example.com").readText()   // blocking HTTP
}

// 좋은 예 — IO dispatcher로 명시적 전환
suspend fun fetchData(): String = withContext(Dispatchers.IO) {
    URL("https://api.example.com").readText()
}
```

JDBC도 blocking이다. R2DBC가 없는 환경이면 `withContext(Dispatchers.IO) { }` 필수.

### 6-4. async 예외 — await() 호출 전까지 예외 숨겨짐

```kotlin
// 나쁜 예 — 예외가 숨겨짐
val deferred = async { riskyOperation() }
delay(1000)
// 예외가 발생해도 아직 알 수 없음

// await() 시점에서야 예외 전파
deferred.await()   // 여기서 예외 발생
```

`async`로 시작한 코루틴은 `await()` 호출 전까지 예외를 조용히 보관한다.
`launch`와 달리 `CoroutineExceptionHandler`도 동작하지 않는다.

```kotlin
// 안전한 패턴
val result = try {
    async { riskyOperation() }.await()
} catch (e: Exception) {
    defaultValue
}
```

### 6-5. Flow 예외 처리

```kotlin
// 잘못된 패턴 — try-catch가 collect 밖에 있어 재개 안 됨
flow {
    emit(1)
    throw RuntimeException("oops")
    emit(2)
}
.collect {
    try {
        process(it)
    } catch (e: Exception) {
        // emit 중 발생한 예외는 여기서 못 잡음
    }
}

// 올바른 패턴 — catch 연산자
flow {
    emit(1)
    throw RuntimeException("oops")
}
.catch { e -> emit(-1) }     // 업스트림 예외를 잡아 대체값 발행
.collect { process(it) }
```

### 6-6. structured concurrency 위반 — 코루틴 누수

```kotlin
// 나쁜 예 — scope 외부로 코루틴 탈출
suspend fun processItems(items: List<Item>) {
    items.forEach { item ->
        GlobalScope.launch {   // 부모 scope와 무관하게 실행됨
            processItem(item)
        }
    }
    // 함수가 리턴해도 코루틴들은 계속 실행 중 — 누수
}

// 좋은 예
suspend fun processItems(items: List<Item>) = coroutineScope {
    items.forEach { item ->
        launch { processItem(item) }   // 모든 자식이 끝날 때까지 대기
    }
}
```

---

## 7. Spring + Kotlin 통합 함정

### 7-1. @Transactional + suspend — 동작 안 함 (Spring MVC + JDBC)

`@Transactional`은 ThreadLocal로 트랜잭션 컨텍스트를 전파한다.
코루틴은 suspend/resume 시 스레드가 바뀔 수 있어 ThreadLocal 전파가 끊긴다.

```kotlin
// 문제 있는 코드 (Spring MVC + JDBC 환경)
@Transactional   // ThreadLocal 기반 — coroutine suspend 시 컨텍스트 손실
suspend fun transferMoney(from: Long, to: Long, amount: Long) {
    val sender = accountRepository.findById(from)   // suspend 후 다른 스레드 재개 가능
    val receiver = accountRepository.findById(to)
    // 두 쿼리가 다른 트랜잭션에서 실행될 수 있음!
}
```

**해결책 1: TransactionalOperator (R2DBC/WebFlux 환경)**

```kotlin
@Service
class TransferService(private val operator: TransactionalOperator) {
    suspend fun transfer(from: Long, to: Long, amount: Long) {
        operator.executeAndAwait {
            val sender = accountRepository.findById(from)
            val receiver = accountRepository.findById(to)
            // ...
        }
    }
}
```

**해결책 2: withContext(Dispatchers.IO)로 블로킹 유지 (Spring MVC + JDBC)**

```kotlin
@Transactional   // suspend 제거
fun transferMoney(from: Long, to: Long, amount: Long) {
    // blocking 코드만 — @Transactional 정상 동작
}

// 호출하는 쪽에서 IO dispatcher로 전환
suspend fun transferAsync(from: Long, to: Long, amount: Long) =
    withContext(Dispatchers.IO) {
        transferMoney(from, to, amount)
    }
```

### 7-2. Spring AOP + Kotlin final class

Kotlin 클래스는 기본 `final`. Spring의 `@Transactional`, `@Cacheable` 등은 CGLIB 프록시가 필요한데,
final 클래스는 프록시 생성 불가 → 런타임 에러.

```kotlin
// 문제
@Service
class OrderService {        // final — CGLIB 프록시 생성 실패
    @Transactional
    fun createOrder() { }
}

// 해결책 1: open 키워드
@Service
open class OrderService {
    @Transactional
    open fun createOrder() { }
}

// 해결책 2 (권장): kotlin-spring(all-open) 컴파일러 플러그인
// build.gradle.kts에서:
// id("org.jetbrains.kotlin.plugin.spring") version "..."
// → @Component, @Service, @Repository, @Controller, @Transactional 자동 open 처리
```

### 7-3. Jackson + data class — 기본 생성자 없음

Jackson은 기본적으로 no-arg 생성자로 역직렬화한다. Kotlin data class는 기본 생성자가 없다.

```kotlin
data class UserDto(val name: String, val age: Int)
// Jackson: "No suitable constructor found..." 에러

// 해결책: jackson-module-kotlin 의존성 추가
// build.gradle.kts:
// implementation("com.fasterxml.jackson.module:jackson-module-kotlin")

// ObjectMapper 설정
val mapper = ObjectMapper().registerKotlinModule()

// Spring Boot는 auto-configuration으로 자동 등록됨 (spring-boot-starter-web 포함 시)
```

### 7-4. Connection Leak — @Transactional + Coroutines + Dispatchers.IO 조합

```kotlin
// 위험한 패턴
@Transactional
suspend fun process() {
    withContext(Dispatchers.IO) {   // 트랜잭션 컨텍스트에서 다른 스레드로 전환
        repository.findAll()        // 새 스레드에서 새 커넥션 획득
        // 원래 트랜잭션의 커넥션은 반환 안 됨 → 커넥션 풀 고갈
    }
}
```

트랜잭션 컨텍스트 안에서 `withContext`로 Dispatcher를 바꾸면 커넥션이 반환되지 않는다.
WebFlux + R2DBC를 쓰거나, blocking 레이어를 완전히 분리해야 한다.

---

## 8. Value Class (Inline Class)

래퍼 타입 없이 타입 안전성을 제공한다. JVM에서는 가능한 한 원시 타입으로 언박싱됨.

```kotlin
@JvmInline
value class UserId(val value: Long)

@JvmInline
value class OrderId(val value: Long)

// 실수 방지: UserId와 OrderId를 바꿔 쓰면 컴파일 에러
fun findUser(id: UserId): User = TODO()
fun findOrder(id: OrderId): Order = TODO()

val userId = UserId(1L)
val orderId = OrderId(1L)
findUser(orderId)   // 컴파일 에러 — 타입 혼용 방지
```

**제약사항:**
- 프로퍼티 하나만 허용
- 제네릭 타입으로 쓸 때는 boxing 발생 (이점 사라짐)
- `init` 블록에서 검증 가능
- 인터페이스 구현 가능, 상속 불가

---

## 9. 성능 고려사항

### 9-1. 람다/고차함수 — 인라인 여부

인라이닝 안 된 람다는 매 호출마다 Function 객체를 생성한다.

```kotlin
// 성능 측정에 자주 쓰는 패턴 — inline으로 오버헤드 제거
inline fun <T> measureTimeMs(block: () -> T): Pair<T, Long> {
    val start = System.currentTimeMillis()
    val result = block()
    return result to (System.currentTimeMillis() - start)
}
```

### 9-2. Sequence vs Collection — 중간 리스트 생성 비용

```kotlin
// 컬렉션 — 매 연산마다 새 리스트 생성
listOf(1..1_000_000)
    .filter { it % 2 == 0 }    // 새 리스트 500,000개
    .map { it * 2 }            // 새 리스트 500,000개
    .take(10)                  // 새 리스트 10개

// Sequence — 터미널 연산 시점에 한 번만 처리
(1..1_000_000).asSequence()
    .filter { it % 2 == 0 }
    .map { it * 2 }
    .take(10)
    .toList()    // 실제 계산 — 10개만 처리하고 중단
```

단, 짧은 컬렉션(수십~수백 개)은 Sequence 오버헤드가 오히려 클 수 있다.
`take`, `find`, `firstOrNull`처럼 early termination이 가능한 연산에서 Sequence 효과가 크다.

### 9-3. String template vs StringBuilder

```kotlin
// 루프 안에서 + 연산 — O(n²)
var result = ""
for (i in 1..1000) result += i.toString()   // 매번 새 String 생성

// StringBuilder
val sb = StringBuilder()
for (i in 1..1000) sb.append(i)
val result = sb.toString()

// buildString DSL
val result = buildString {
    for (i in 1..1000) append(i)
}
```

---

## 참고

- [Kotlin 공식 문서 — Null Safety](https://kotlinlang.org/docs/null-safety.html) — 확인일 2026-07-13
- [Kotlin 공식 문서 — Scope Functions](https://kotlinlang.org/docs/scope-functions.html) — 확인일 2026-07-13
- [Kotlin 공식 문서 — Coroutines Exception Handling](https://kotlinlang.org/docs/exception-handling.html) — 확인일 2026-07-13
- [GlobalScope 사용을 피해야 하는 이유 — Roman Elizarov (코루틴 설계자)](https://elizarov.medium.com/the-reason-to-avoid-globalscope-835337445abc) — 확인일 2026-07-13
- [Spring Framework — Kotlin Coroutines 공식 문서](https://docs.spring.io/spring-framework/reference/languages/kotlin/coroutines.html) — 확인일 2026-07-13
- [Spring GitHub Issue #28290 — @Transactional + coroutine context loss](https://github.com/spring-projects/spring-framework/issues/28290) — 확인일 2026-07-13
- [Why @Transactional with Coroutines can cause Connection Leaks](https://medium.com/@erlangga258/why-transactional-with-coroutines-and-reactive-can-break-your-app-c3235dd3da7b) — 확인일 2026-07-13
  - 보조 출처(2차 — 구체적인 커넥션 누수 시나리오 설명)
- [Death by a Thousand Coroutines: 10 Mistakes at Scale](https://androidengineers.substack.com/p/death-by-a-thousand-coroutines-10) — 확인일 2026-07-13
  - 보조 출처(2차 — 프로덕션 코루틴 실수 사례 모음)

## 관련 내용

- [Kotlin-전반-가이드](./Kotlin-전반-가이드.md)
- [Blocking-Nonblocking-Sync-Async](../운영체제/Blocking-Nonblocking-Sync-Async.md)
