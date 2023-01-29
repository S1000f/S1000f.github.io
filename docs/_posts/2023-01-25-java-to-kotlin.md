---
title: 자바와 코틀린의 차이점 요약
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

## 2.2 확장함수

## 2.3 null-safe 연산자

## 2.4 표현식 함수

## 2.5 이터레이션

## 2.6 영역함수: let, apply, also, with

## 2.7 컬렉션의 확장함수