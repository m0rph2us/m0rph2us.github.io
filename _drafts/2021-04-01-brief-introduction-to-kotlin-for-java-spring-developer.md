---
layout: post
excerpt_separator: <!--more-->
title: 'Brief introduction to kotlin for java-spring developer'
categories: spring kotlin
---

# 자바-스프링 개발자를 위한 코틀린 간략 소개
## 개요

이번 포스팅에서는 자바 언어를 기반으로 스프링 애플리케이션 개발을 해온 개발자에 포커싱을 맞춰 코틀린을 소개해 보려고 한다.
따라서 적어도 자바 언어에 대해서는 경험이 있다고 가정한다. 그리고, 따분한 역사에 대한 내용은 따로 얘기하지 않겠지만 관심이 있다면
[Kotlin](https://en.wikipedia.org/wiki/Kotlin_(programming_language)) 을 방문해보기 바란다.

<!--more-->
이 포스팅에서는 코틀린의 모든 것들을 총 망라하지는 않을 것이며, 처음 코틀린을 시작할 때 필수로 알아야 하는 부분들 위주로 쓰윽 훑어볼 수 있도록
소개하려고 한다.

## 코틀린 개발을 위한 첫 발

먼저 코틀린으로 개발을 아주 쾌적하게 하고 싶다면, 인텔리제이 IDE가 거의 필수라고 보면 된다. 코틀린 역시 인텔리제이를 만든
젯브레인에서 만들었다. 커뮤니티 에디션은 [여기](https://www.jetbrains.com/idea/download/#section=mac)에서 다운로드 할 수 있다.

그리고, 스프링 프로젝트 생성은 [spring initializr](https://start.spring.io/) 를 사용하면 쉽고 간편하게 프로젝트를 생성할 수 있다.
참고로 커뮤니티 에디션은 GUI 기반의 스프링 프로젝트 생성 기능을 제공하지 않는다.

![spring initializr](/assets/spring-initializr-kotlin.png)

여기에서 소개할 내용은 다른 의존성이 필요하지 않기 때문에 위와 같이 선택하고 프로젝트를 다운로드 한다. 그런 다음 압축을 풀어 인텔리제이에서 열면 된다.
그리고, 아래의 내용들을 직접 타이핑 해보면서 이해하면 이해가 더 쉬울 것이다. 사실 다음의 예제들은 굳이 IDE 가 필요하진 않다. IDE 를 설치하고 싶지 않다면
[Kotlin play ground](https://play.kotlinlang.org) 에서 바로 해볼 수가 있다.

## 기본적인 특징
### 세미콜론

자바에서 넘어온 경우라면 처음에는 잘 적응되지 않는 부분이다. 구문의 맨 끝에서 세미콜론(;)을 사용하지 않는다.
사용한다고 해서 오류가 발생하는 것은 아니지만 보통 사용하지 않는다.

```kotlin
val value = ""
```

### 클래스 이름과 파일 이름이 일치하지 않아도 된다

제목 그대로 클래스 이름과 파일 이름은 달라도 된다.

### 굳이 클래스가 없어도 된다

자바처럼 무조건 클래스 내부에 모든 것이 위치하지 않아도 된다. 클래스 없이 선언이 가능하다(사실 내부적으로 클래스가 만들어지고 거기에 때려넣는다).

### 기본 가시성은 public 이다

가시성을 명시적으로 지정하지 않으면 `public` 이 기본 가시성이다.

### 자바에는 있지만 코틀린에는 없다

다음 목록은 자바에는 있지만 코틀린에는 없는 것들에 대한 목록이다.

* `static` 키워드가 없다
    * 가장 당황하는 내용이 아닐까 싶다. 우리의 친구 `static` 이 없다니? 하지만 코틀린으로 코드를 작성하다 보면 굳이 `static` 키워드가 필요 없다.
    왜 그런지는 이어지는 내용을 보다보면 알 수 있다.
* `new` 키워드가 없다
    * 인스턴스를 생성할 때 `new` 키워드를 사용하지 않는다.
* `void` 키워드가 없다
    * 반환 타입을 지정하지 않으면 암묵적으로 `Unit` 타입을 지정한 것과 같은데, 이는 자바의 `void` 와 같다고 보면 된다.
* 삼항 연산자가 없다
    * 즉, `a ? b : c` 형태의 연산자가 없다.
* `switch` 키워드가 없다
    * 조건 분기를 위해 흔히 사용하는 `switch` 가 없다.
* `final` 키워드가 없다
    * 보면 알겠지만 필요하지가 않다.

당장 생각나는 것들만 정리했기 때문에 항목이 더 존재할 수도 있다.

## 변수
### 변수의 선언

다음은 변이불가(immutable) 변수이다. 변수 앞에 `val` 을 붙인다.

```kotlin
val value = ""
value = "1234" // 재할당 불가
```

다음은 변이가능(mutable) 변수이다. 변수 앞에 `var` 을 붙인다.

```kotlin
var value = ""
value = "1234" // 재할당 가능
```

기본적으로 변이불가를 우선으로 고려하여 개발하기 바란다.

### 타입 지정

변수 타입은 변수 이름 뒤에 `:` 으로 구분하여 지정한다.

```kotlin
val value: String = ""
```

위와 같이 명시적으로 지정이 가능하지만, 우변을 보면 문자열 타입임을 누가봐도 알 수 있기 때문에 다음과 같이 추론이 가능하다.

```kotlin
val value = "" // String 타입으로 컴파일러가 추론
```

### 널가능성(nullability)

여러 가지 이유로 코틀린 역시 `null` 을 고려해야 하는데, 코틀린은 `null` 이 될 수 있는 변수와 그렇지 않은 변수를 따로 구분해서 선언해야 한다.
다음처럼 변수 선언 시 아무것도 지정하지 않으면 `null` 이 될 수가 없다.

```kotlin
var value = ""
value = null // 불가
```

`null` 이 될 수 있으려면, 변수의 타입을 지정할 때 `?` 를 붙여줘야 한다. `null` 이 될 가능성이 있는 변수에만 사용하도록 한다.

```kotlin
var value: String? = ""
value = null // 불가
```

### 상수

상수를 선언하는 방법은 다음과 같다. `const` 키워드를 사용하여 선언한다. 하지만, `const` 키워드는 일반 클래스 내부에서는 사용할 수 없기 때문에
일반 클래스 내부에 상수를 선언하고 싶다면 동반자 객체를 사용한다.

```kotlin
// top level 에 선언
const val CONST_VAL = "1234"

// 오브젝트 내부에 선언
object Constants {
    const val CONST_VAL = "5678"
}
```

## 함수
### 함수의 선언

코틀린은 함수를 선언할 때 키워드로 `fun` 을 사용한다.

```kotlin
fun test() {
}
```

### 매개변수

매개변수 지정은 다음과 같다. `val` 또는 `var` 키워드는 사용하지 못한다. 기본적으로 변이불가 매개변수이기 때문에 내부에서 재할당을 허용하지 않는다.

```kotlin
fun test(param: String) {
    param = "" // 불가
}
```

### 가변 매개변수

매개변수 개수가 가변인 매개변수도 선언이 가능하다.

```kotlin
fun test(vararg args: String): Int {
    return args.size
}
```

### 기본 매개변수

기본 매개변수 지정도 가능하다.

```kotlin
fun test(param: String = "1234"): String {
    return param
}
```

### 반환 타입

다음과 같이 반환 타입을 지정한다. 지정하지 않으면 `Unit` 이 지정된 것과 같다.

```kotlin
fun test(param: String): String {
    return param
}
```

### 축약 표현

```kotlin
fun test(param: String): String {
    return param
}
```

위의 함수는 다음과 같이 축약이 가능하다.

```kotlin
fun test(param: String) = param
```

### 람다 함수

## 클래스
### 클래스의 선언

```kotlin
class SomeClass {
    fun somefun() = "1234"
}

// 내용이 없다면 다음처럼 선언도 가능하다
class SomeClass2

fun main() {
    // 인스턴스 생성은 new 키워드 없이 생성자를 호출하면 된다.
    println(SomeClass().somefun())
}
```

### 생성자

다음은 주생성자의 예시이다.

```kotlin
// class SomeClass constructor(value: String) 와도 같은 표현이다
class SomeClass(value: String) {
    fun somefun() = "1234"
}
```

위의 코드를 자바로 표현하면 다음과 같다.

```java
class SomeClass {
    public SomeClass(String value) {
    }
    public String somefun() {
        return "1234";
    }
}
```

자 이제 주생성자 매개변수에 `val` 키워드를 붙이면 어떻게 되는지 보자.

```kotlin
class SomeClass(val value: String) {
    fun somefun() = "1234"
}
```

위의 코드를 자바로 표현하면 다음과 같다.

```java
class SomeClass {
    public String value;
    public SomeClass(String value) {
        this.value = value;
    }
    public String somefun() {
        return "1234";
    }
}
```

`val` 나 `var` 키워드를 붙이고 안붙이고의 차이는 매개변수를 선언할 때 멤버변수의 초기화를 함께 선언하느냐 안하느냐의 차이이다.
이 방식은 주생성자에서만 유효하다. 가시성을 지정하지 않으면 기본 가시성이 `public` 이므로, 은닉이 필요하면 `private` 같은 키워드로 가시성을 지정해주면 된다.

부생성자는 다음과 같이 `constructor` 키워드를 사용해서 생성자를 추가 정의하면 된다.

```kotlin
class SomeClass(val value: String) {
    private var value2: String = ""
    constructor(value: String, value2: String): this(value) {
        this.value2 = value2
    }

    fun somefun() = value
}
```

### init 블록

주생성자로 전달된 매개변수의 경우에는 `init` 블록을 통해서 별도로 초기화가 가능하다.

```kotlin
class SomeClass(value: String) {
    private val value: String

    init {
        this.value = value
    }

    fun somefun() = value
}
```

### 클래스의 상속

클래스의 상속, 구현 모두 `:` 하나면 된다. 여기서 주의할 점은 클래스가 상속이 가능하려면 `open` 키워드를 붙여줘야 한다.
자바로 작성된 클래스는 기본으로 모두 `open` 으로 간주하면 된다.

```kotlin
open class Parent

class Child : Parent() // extends. 상위 클래스의 생성자를 이곳에서 호출

interface MyInterface // 인터페이스는 구현이 필요하기 때문에 명시적인 open 지정이 불필요하다

class MyConcrete : MyInterface // implements.
```

### 데이터 클래스

유추할 수 있듯이 주로 데이터를 담기 위한 용도로 사용한다. 이 클래스의 특징은 내부적으로 `toString`, `hashCode`, `equals` 를 기본 구현해 준다는 점이다.
한 가지 단점은 상속이 되지 않는다는 점인데(부모 데이터 클래스에 `open` 키워드를 사용할 수 없다), 이 부분은 데코레이터 같은 포함 패턴을 사용하면 된다.

```kotlin
data class Data(
    val id: String,
    val value: String
)
```

이 클래스는 롬복의 대안으로 사용할 수 있다.

### 싱글턴 인스턴스

코틀린은 싱글턴 인스턴스를 위한 특별한 구조를 제공한다. 스프링의 싱글턴 인스턴스와는 개념이 다르기 때문에 혼동하지 않도록 한다.
명시적인 인스턴스화가 필요없고, 바로 사용하면 된다.

```kotlin
object SomeClass {
    fun somefun() = "1234"
}

fun main() {
    println(SomeClass.somefun())
}
```

### 동반자(companion) 클래스

다음처럼 클래스 내부에 선언할 수 있다. `const` 키워드는 가장 바깥 수준(top level)이나 object 내부에서만 사용할 수 있기 때문에
네임스페이싱이 필요하다면 `static` 대용으로 사용하기에는 다소 부족한 측면이 있다. 따라서 다음과 같이 동반자 객체를 선언하고 그 내부에
상수 변수를 두는 형태로 `static` 변수 대안으로 사용할 수 있다.

```kotlin
class SomeClass {
    companion object {
        const val SOME_VALUE: String = "1234"
    }
}

fun main() {
    println(SomeClass.SOME_VALUE)
    // 위와 동일하다
    println(SomeClass.Companion.SOME_VALUE)
}
```

다음과 같이 일반 object 를 사용해도 되겠지만, 추가적으로 이름이 필요하기 때문에 참조 이름이 길어질 수 있다.

```kotlin
class SomeClass {
    object NeedName {
        const val SOME_VALUE: String = "1234"
    }
}

fun main() {
    // println(SomeClass.SOME_VALUE) // 이렇게는 안된다
    println(SomeClass.NeedName.SOME_VALUE)
}
```

### 함수 재정의

함수의 구현 및 재정의는 `override` 키워드를 사용하면 된다. 재밌는 점은 함수 단위로도 `open` 키워드를 지정해서 재정의 가능한 함수를 지정해줘야 한다는 점이다.
그리고, 자바에서 사용했던 `@Override` 애너테이션은 사용하지 않아도 된다.

```kotlin
open class Parent {
    open fun somefunc() = ""
}

class Child : Parent() {
    override fun somefunc() = ""
}

interface MyInterface {
    fun somefunc(): String
}

class MyConcrete : MyInterface {
    override fun somefunc() = ""
}
```

### 함수 확장

코틀린의 굉장히 편리한 기능중 하나이다. 기존 클래스를 손쉽게 확장할 수 있는 수단을 제공하며, 확장 메커니즘은 데코레이터 패턴으로 보면 이해하기 쉽다.

예를 들어, `LocalDateTime` 클래스에 `Date` 로 변환을 수행하는 `toDate()` 라는 함수를 추가하고 싶다면 다음과 같이 하면 된다.

```kotlin
fun LocalDateTime.toDate(): Date = Date.from(this.atZone(ZoneId.systemDefault()).toInstant())

fun main() {
    println(LocalDateTime.now().toDate())
}
```

자바로 동일하게 하려면 `LocalDateTime` 을 감싸는 래퍼 클래스를 만들어 확장해야 한다. 한 눈에 봐도 코딩량을 얼마나 줄여주는지 알 수 있다.

```java
class DateConvertableLocalDateTime {
    private LocalDateTime ldt;
    public DateConvertableLocalDateTime(LocalDateTime ldt) {
        this.ldt = ldt;
    }
    public Date toDate() {
        ...
    }
}
```

### 연산자 재정의
### 위임
### 익명 클래스

## 타입
### 캐스팅
### 변환
### 제네릭

## 널 처리

코틀린은 `null` 을 처리할 수 있는 매우 편리한 수단을 언어 자체에서 제공하기 때문에 자바에서 `null` 을 처리하기 위해 흔히 사용했던
`@NotNull`, `Optional` 이 불필요하다.

### 널불가 체크

특정한 상황에서는 변수를 널가능으로 선언해도 프로그램 흐름상 `null` 이 될 수 없음을 확신할 수 있는 경우가 생기는데, 이럴 때 사용할 수 있는 연산자가 `!!` 이다.

`!!` 로 체크하면, 만약의 경우 변수가 `null` 이라면 NPE가 발생한다. 체크 이후의 변수 참조는 모두 `null` 불가 변수인 것처럼 참조하여 코딩할 수 있다.

```kotlin
var value: String? = "1234"
val result = somefunc(value!!)
// 이후 라인에서 value 변수를 참조하는 코드가 있다면 번거롭게 null 처리를 하지 않아도 된다.
```

### 엘비스 연산자

엘비스 연산자 `?:` 는 `null` 처리를 위해 꽤 자주 사용하는 연산자이다. 좌변이 `null` 인 경우에만 우변을 취한다. 그렇지 않으면 좌변을 취한다.
예를 들어 변수가 `null` 인 경우 기본값을 돌려주도록 할 때 다음과 같이 간략하게 표현할 수 있다.

```kotlin
fun main() {
    val value: String? = null
    println(value ?: "hello")
}
```

`value ?: "hello"` 를 자바로 표현하면 다음과 같다.

```java
(value == null) ? value : "hello"
```

### 안전호출 연산자

안전호출 연산자 `?.` 는 좌측의 대상을 참조할 때 안전하게 참조할 수 있도록 도와준다. `?.` 이전의 참조 대상이 `null` 이 아닌 경우에만 우측을 수행한다.
그렇지 않으면 `null` 으로 평가된다.

```kotlin
class SomeClass {
    fun somefunc(): String? = null
}
fun main() {
    println(SomeClass().somefunc())
}
```

## 스코핑 함수

## 동시성

## 기타
### 늦은 초기화