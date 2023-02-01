---
title: 자바 1.8 과 코틀린 요약
published: false
tags: java, kotlin
---

# 1. 요약
## 1.1 하나의 파일에 여러개의 클래스를 정의

```java
public class UserDto {
  private Long idx;
  
  public static class UserResponseDto {
    private Long idx;
    private String id;
  }
}
```
```kotlin
class UserDto {
    val idx: Long
    constructor(idx: Long) {
        this.idx = idx
    }
}

class UserResponseDto {
    val idx: Long
    val id: String
    constructor(idx: Long, id: String) {
        this.idx = idx
        this.id = id
    }
}
```

하나의 코틀린 파일(*.kt)에 여러개의 클래스와 인터페이스 등을 자유롭게 선언하면 됩니다. 자바는 하나의 파일에 탑 클래스는 하나만 존재하는게 좋고 그 외 클래스들은
`static` 키워드와 함께 중첩 클래스로 표현하지만 코틀린은 자유롭습니다.

## 1.2 최상위에 함수 정의
```java
public final class Utils {
  private Utils() {}

  public static byte[] intToBytes(BigInteger bigInteger) {
    byte[] array = bigInteger.toByteArray();
    if (array[0] == 0) {
      byte[] tmp = new byte[array.length - 1];
      System.arraycopy(array, 1, tmp, 0, tmp.length);
      array = tmp;
    }
    return array;
  }
}
```
```kotlin
fun intToBytes(bigInteger: BigInteger): ByteArray {
    var array = bigInteger.toByteArray()
    if (array[0].toInt() == 0) {
        val tmp = ByteArray(array.size - 1)
        System.arraycopy(array, 1, tmp, 0, tmp.size)
        array = tmp
    }
    return array
}
```

클래스가 없더라도 어디서든지 함수를 작성합니다. 
자바는 유틸성 메소드를 제공하기 위해 상속과 인스턴스를 금지한 클래스 혹은 인터페이스가 필요하지만 코틀린은 어디서든지 함수를 정의합니다.

## 1.3 널이 불가능한 타입과 가능한 타입의 구분
```java
  public String stringOrNull() {
    if (ThreadLocalRandom.current().nextBoolean()) {
      return "hello";
    } else {
      return null;
    }
  }
```
```kotlin
fun stringOrNull(): String? {
    if (Random.nextBoolean()) {
        return "hello"
    } else {
        return null
    }
}
```
코틀린은 널이 불가능한 타입과 널이 가능한 타입을 엄격히 구분하여 사용합니다. 
위의 자바 코드는 함수가 `String` 을 반환하지만, 아래의 코틀린은 `String?`을 반환하고 있습니다.
널이 가능한 타입은 타입명칭 끝에 물음표를 붙입니다. 위 코틀린 코드에서 반환 타입 `String?`의 물음표를 제거하면 컴파일 자체가 불가능합니다.

## 1.4 if, try 키워드는 값을 생성하는 표현식
```java
  public int stringToInt(String s) {
    int num;
    try {
      num = Integer.parseInt(s);
    } catch (NumberFormatException e) {
      num = 0;
    }
    return num;
  }
```
```kotlin
fun stringToInt(s: String): Int {
    val num = try {
        s.toInt()
    } catch (e: NumberFormatException) {
        0
    }
    return num
}
```
코틀린의 `if` 와 `try` 키워드는 값을 생성하는 표현식(expression)입니다.
위의 자바 코드는 'num' 변수를 먼저 선언한 뒤 `try`문 안에서 별도의 표현식을 통해 num 변수에 값을 대입하고 있습니다.
하지만 아래의 코틀린 코드는 변수 num 선언과 동시에 `try`표현식을 통하여 변수를 초기화 했습니다.

```kotlin
fun stringOrNull(): String? {
    return if (Random.nextBoolean()) "hello" else null
}
```
마찬가지로 `if` 키워드도 값을 만드는 표현식이므로, 1.4항목의 코틀린 코드를 다음과 같이 바꿀 수 있습니다.
이 형태가 삼항연산자와 매우 비슷함을 알수 있습니다. 그래서 코틀린에는 별도의 삼항연산자를 지원할 필요가 없습니다.

## 1.5 반복문
```java
  public void loop(List<Integer> list) {
    for (int i = 0; i < list.size(); i++) {
      if (list.get(i) > 2) {
        System.out.println(i);
      }
    }
  }
```
```kotlin
fun loop(list: List<Int>) {
    for ((index, item) in list.withIndex()) {
        if (item > 2) {
            println(index)
        }
    }
}
```
코틀린에는 `while`과 for-each 형식의 `for`문만 지원합니다. 
만약 인덱스가 필요하다면 `withIndex()` 메소드를 사용하여 (index, element) 형식으로 사용하면 됩니다.

## 1.6 val 과 var
```java
  public static void main(String[] args) {
    String mut = "mutable";
    final String imut = "immutable";
    mut = "mutable2";
    imut = "immutable2"; // error
  }
```
```kotlin
fun main() {
    var mut = "mutable"
    val imut = "immutable"
    mut = "mutable2"
    imut = "immutable2" // error
}
```
`var` 키워드로 선언한 변수는 값을 바꿀 수 있습니다. `val` 키워드로 선언된 변수는 자바의 `final` 처럼 한번 값이 초기화된 이후로는 값을 변경할 수 없습니다.
코틀린은 대부분의 경우 변수의 타입을 추론할 수 있습니다. 위 예제에서는 컴파일러가 'mut' 와 'imut' 변수의 타입을 `String` 으로 추론합니다.

## 1.7 변경 가능 혹은 변경 불가능한 컬렉션
```java
  public static void main(String[] args){
    Map<Integer, String> map=new HashMap<>();
    Map<Integer, String> map1=Collections.unmodifiableMap(map);

    map.put(1,"one");
    map1.put(1,"one"); // exception
  }
```
```kotlin
fun main() {
    val map: MutableMap<Int, String> = mutableMapOf()
    val map1: Map<Int, String> = mapOf()
    
    map.put(1, "one")
    map1.put(1, "one") // error
}
```
코틀린은 변경가능한 컬렉션과 변경 불가능한 컬렉션이 완전히 분리되어 있습니다.

자바에서 변경불가능한 컬렉션을 생성하려면 `Collections` 클래스에 있는 'unmodifiable...'로 시작하는 정적 팩토리 메소드를 사용해야 합니다.
이 메소드들이 반환하는 타입은 각 컬렉션 타입(List, Set, Map 등)의 인터페이스 입니다. 변경불가능한 컬렉션을 위한 별도의 인터페이스를 제공하지 않습니다.
자바 코드에서 `map1.put(1, "one")`을 호출하면 `UnsupportedOperationException`이 발생합니다.

반면에 코틀린은 변경가능한 컬렉션 타입을 위한 별도의 인터페이스를 제공합니다.
위 코틀린 코드에서 `map1.put(1, "one")`부분은 예외를 발생시키는게 아니라 애초에 컴파일 자체가 불가능합니다.
왜냐하면 `MutableMap`인터페이스에는 `put()`이라는 행위 자체를 정의하지 않았기 때문입니다.

## 1.8 문자열 편의기능
```java
  public String log(String prefix, String message) {
    return Instant.now() + ":" + prefix + ": " + message;
  }
```
```kotlin
fun log(prefix: String, message: String): String {
    return "${Instant.now()}:$prefix: $message"
}
```
코틀린은 문자열 템플릿을 지원합니다.
문자열 안에서 '${}' 을 사용해서 변수나 표현식의 값을 바로 사용할 수 있습니다.

```java
  public static void main(String[] args) {
    String greeting = "hi";
    String body = "<body>"
    + "<p>" + greeting + "</p>"
    + "</body>";

    String json = 
        "{\"jsonrpc\":\"2.0\",\"method\":\"subtract\",\"params\":[42,23],\"id\":1}";
  }
```
```kotlin
fun main() {
    val greeting = "hi"
    val body = """
        <body>
          <p>$greeting</p>
        </body>
    """.trimIndent()

    val json = """{"jsonrpc":"2.0","method":"subtract","params":[42,23],"id":1}"""
}
```

삼중따옴표(""")을 사용하면 문자열안에서 따옴표(")을 이스케이프 하지 않아서 편합니다. 또한 여러줄 문자열도 좀 더 가시성 좋게 작성할 수 있습니다.

# 2. 조금 더 살펴보기

## 2.1 생성자와 data class

항목 1.1 에서 보았던 클래스 선언을 좀 더 간략하게 만들어보겠습니다.

```kotlin
class UserDto constructor(idx: Long) {
    val idx: Long
    init {
        this.idx = idx
    }
}
```
클래스 내부에 있던 생성자를 클래스 이름 옆에 지정할 수 있습니다. 이 생성자는 주 생성자(primary constructor)라고 부릅니다.
클래스 내부의 `init`블록에서 프로퍼티 'idx' 를 초기화 해줍니다.

```kotlin
class UserDto(idx: Long) {
    val idx = idx
}
```

주 생성자앞에 `private` 이나 `protected` 같은 접근제어자가 없다면 `constructor` 키워드를 생략할 수 있습니다.
또한 생성자의 매개변수로 프로퍼티로 바로 초기화 할 수 있습니다. 파라미터명 앞의 언더바(_)는 파라미터를 프로퍼티로 바로 초기화 할 때 사용합니다.

```kotlin
class UserDto(val idx: Long)
```
만약 주 생성자 파라미터로 프로퍼티를 바로 초기화 한다면, 주 생성자 파라미터앞에 `val` 키워드를 붙여 간략화 할 수 있습니다.
그리고 본문이 없는 클래스는 중괄호를 생략할 수 있습니다.

이상의 내용들은 만약 '인텔리제이 IDEA' 를 사용한다면 IDE 가 리펙터링 힌트를 주므로 금방 익숙해집니다.

```kotlin
data class UserDto(val idx: Long)
```
특정 값을 담고 그 값으로 정의되는 객체를 데이터 클래스 혹은 DTO(Data Transfer Object)라고 부릅니다.
데이터 클래스들은 내부의 값으로 동치관계나 대소관계를 나타내는게 보통입니다.
코틀린에서 `data class` 키워드를 사용하면 자동으로 해당 클래스의 `equals()`, `hashCode()`, `toString()` 메소드를 오버라이드 해줍니다.
자바 라이브러리 롬복의 `@Data`어노테이션을 생각하시면 됩니다.

## 2.2 확장함수

```kotlin
fun BigInteger.toBytesNoSignBit(): ByteArray {
    var array = this.toByteArray()
    if (array[0].toInt() == 0) {
        val tmp = ByteArray(array.size - 1)
        System.arraycopy(array, 1, tmp, 0, tmp.size)
        array = tmp
    }
    return array
}
```
```kotlin
fun main() {
    val big = BigInteger("255")
    val bytes = big.toBytesNoSignBit()
}
```

항목 1.2 의 함수를 확장함수로 재정의 하였습니다.
우리는 자바의 `BigInteger` 클래스를 직접 확장하지 않았지만, 확장함수를 사용하면 해당 클래스의 멤버 메소드인 것처럼 함수를 호출할 수 있습니다.

위의 확장함수 이름 `BigInteger.toBytesNoSignBit()` 중에서 `BigInteger` 부분을 '수신 객체 타입(receiver type)'이라고 부릅니다.
확장함수 블록안에서 `this` 키워드를 사용하면 수신 객체를 참조할 수 있습니다.

## 2.3 null-safe 연산자
```kotlin
fun main() {
    val placeToGo: String = stringOrNull()
        ?.substring(0..3)
        ?.uppercase() ?: "HEAVEN"
    
    println(placeToGo)
}

fun stringOrNull(): String? {
    return if (Random.nextBoolean()) "hello" else null
}
```
항목 1.3 에서 보았던 'stringOrNull()' 함수는 널이 가능한 `String?`타입을 반환합니다. 널이 가능한 타입은 자바의 `Optional`클래스와 유사한 방법으로 처리가능합니다.

`?.` 연산자는 해당 참조변수가 널을 참조하지 않는 경우에만 연산을 수행합니다. 위 코드의 `?.` 체이닝은 함수가 문자열 'hello' 를 반환한 경우에만 수행됩니다.
체이닝 마지막에 있는 `?:` 는 엘비스 연산자로 불리는 것으로, 반환값이 널일 경우 해당 연산자 우측의 값을 변수에 대입합니다. 결국 `Optional.orElse()`와 역할이 같습니다.

`Optional`은 익셉션 발생 리스크를 감수하며 널 체크없이 `get()`메소드를 호출할 수 있습니다. 하지만 코틀린에서는 불가능합니다.
만약 위 코드에서 엘비스 연산자 `?:`을 제거한다면 변수 'placeToGo' 의 타입은 `String`이 아니라 `String?`으로 강제됩니다.

## 2.4 표현식 함수
```kotlin
fun stringToInt(s: String) = try {
    s.toInt()
} catch (e: NumberFormatException) {
    e.printStackTrace()
    0
}
```
항목 1.4 의 함수를 위와 같이 표현식 함수로 다시 쓸 수 있습니다.
표현식 함수는 함수의 본문에 단 하나의 표현식만 있는 경우로써, 함수 본문 블록과 반환타입 선언을 생략할 수 있습니다.

위 함수는 하나의 표현식이지만 `try`와 `catch`의 두 개의 블록이 있습니다. 하나의 블록에서 가장 마지막에 위치한 표현식의 값이 그 블록의 반환값이 됩니다.
따라서 `catch` 블록에서는 가장 마지막 표현식인 '0' 의 값이 `catch`블록의 반환값이 됩니다.

## 2.5 이터레이션
```kotlin
fun main() {
    for (i in 0..10) println(i)
    for (i in 0 until 10) println(i)
    for (i in 10 downTo 0) println(i)
    for (i in 10 downTo 0 step 2) println(i)
}
```
코틀린에는 `for (;;)` 형식의 루프가 없습니다. 만약 별도의 인덱스가 사용되는 루프가 필요하다면 위와 같이 범위를 사용하면 됩니다.

```kotlin
fun main() {
    val map = sortedMapOf(1 to "one", 3 to "three", 2 to "two")
    for ((key, value) in map) {
        println("$key, $value")
    }

    for ((i, entry) in map.asIterable().withIndex()) {
        println("$i: ${entry.key}:${entry.value}")
    }
}
```
맵에 대한 이터레이션은 구조분해를 사용하면 됩니다. 만약 정렬된 맵 컨테이너를 인덱스와 함께 순회하고 싶다면 `map.asIterable().withIndex()`을 사용합니다.

## 2.6 영역함수: let, apply, also, with
### 2.6.1 let
```kotlin
fun main() {
    val mood = stringOrNull()?.let {
        println(it)
        "${it.substring(0..3).uppercase()}!!"
    } ?: "HEAVEN!!"

    println(mood)
}
```
`let` 함수는 '수신 객체 지정 람다(lambda with receiver)'로 불리는 함수 중 하나입니다. 이 함수들은 모든 제네릭 타입의 확장함수입니다.
코틀린 설계적 관점으로 수신 객체 지정 람다와 2.2 항목에서 본 확장함수는 매우 유사한 개념을 공유하고 있습니다.

우선 `let`함수는 자신의 블록의 마지막 표현식 값을 반환합니다. 따라서 위 예제에서 `let`블록안에는 두 개의 표현식이 있지만, 마지막 표현식의 결과값인 문자열이 반환됩니다.
그리고 위 예제는 `?.let`과 앨비스 연산자 `?:`을 함께 사용하고 있습니다.
`?.let`은 `stringOrNull()`메소드의 반환값이 널이 아닌 경우에만 블록을 실행하고, `?:`는 `null`인 경우에 대한 기본값을 지정합니다.

이는 자바의 `Optional.ifPresent()` 혹은 `Optional.orElseGet()`과 유사한 방식임을 알 수 있습니다.

```kotlin
fun randomUser() = Random.nextLong(1, 100)
    .let {
        log("randomUser", "idx:$it")
        UserDto(it)
    }
```
또다른 용례는 위와 같이 `let`을 임시변수로 사용하는 것입니다.
위 예제에서 랜덤으로 생성된 `Long`값을 로깅과 인스턴스 생성 모두에 사용하기 위해 별도의 변수를 선언할 필요없이, `let`블록 안에서 `it`으로 참조할 수 있습니다.


## 2.7 연산자 오버로딩


# 3. 마무리
