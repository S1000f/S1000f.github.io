---
title: 자바와 코틀린의 차이점
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

클래스가 없더라도 어디서든지 함수를 작성합니다. 자바는 유틸성 메소드를 제공하기 위해 상속과 인스턴스를 금지한 클래스 혹은 인터페이스가 필요하지만 코틀린은 어디서든지 함수를 정의합니다.

## 1.3 널이 불가능한 타입과 가능한 타입의 구분

## 1.4 if, try 키워드는 값을 생성하는 표현식

## 1.5 반복문과 구조분해

## 1.6 val 과 var

## 1.7 변경 가능 혹은 변경 불가능한 컬렉션

## 1.8 문자열 편의기능