---
title: Redis - Sorted Set 부동소수점 값 저장, 정렬, 조회하기
published: false
tags: redis
---

우선 제가 필요로 하는 요구사항은 아래와 같습니다.

- 자바의 `BigDecimal` 을 키로 사용하고 그에 매핑된 리스트 구조의 튜플를 저장.
  - 자바 타입으로 표현하면 `TreeMap<BigDecimal, List<String>>` 으로 될것 같습니다.
- 저장된 값들이 키값을 기준으로 정렬되어 있어야 함
- 원하는 `BigDecimal`을 사용하여 조회가능
- 원하는 `BigDecimal`의 범위로 검색가능
- 메모리 캐시를 사용하며 기본적인 영속성을 만족할 것(레디스 사용)

```yaml
###
25500:
  .12345: user1, user2
  .12346: user2, user3, user2
25505:
  .05443: user2, user1
  .233: user3
25506:
  .12345: user3, user5
###
```
위의 예시에서는 총 5개의 `BigDecimal` 키가 존재하고 각 키에 매핑된 문자열 리스트가 있습니다.
5개의 키들은 오름차순으로 정렬되어 있습니다.

만약 (25500.12346, 25505.2) 으로 검색하면 [[25500.12346:user2, user3, user2], [25505.05443:user2, user1]] 의 결과가 나와야 합니다. 
또한 새로운 `BigDecimal`키를 추가하면 소수점 아래까지 손실없이 저장되며 다음의 범위 검색을 위해 정렬되어 있어야합니다.

---

# 1. Sorted Set
레디스의 'Sorted Set' 자료구조는 기본적으론 'Set' 자료구조와 동일하지만, 각 요소에 'score(이하 스코어)' 라는 연관값을 지정하여 정렬된 상태로 유지합니다.
기본적으로 집합 자료구조이므로 서로 다른 스코어 값을 할당하더라도 동일한 요소는 중복으로써 제거할 수 있습니다(옵션 조정필요).

```redis
127.0.0.1:6379> zadd ss 10 a
(integer) 1
127.0.0.1:6379> zadd ss 20 b
(integer) 1
127.0.0.1:6379> zrange ss 0 -1 withscores
1) "a"
2) "10"
3) "b"
4) "20"
127.0.0.1:6379>
```
기본적인 사용법은 위와 같습니다. `zadd [key] [score] [value]` 를 통해 스코어와 요소를 추가할 수 있습니다.
조회는 다른 자료구조와 비슷하게 `zrange [key] [start] [end]` 를 통해 조회할 수 있습니다.
위 예제는 마지막에 `withscores` 옵션을 추가하여 스코어도 함께 조회하도록 했습니다.

```redis
127.0.0.1:6379> zadd ss 30.3 c
(integer) 1
127.0.0.1:6379> zrange ss 0 -1 withscores
1) "a"
2) "10"
3) "b"
4) "20"
5) "c"
6) "30.300000000000001"
127.0.0.1:6379>
```

스코어 값의 타입은 double-precision 부동소수점입니다. 입력한 스코어값 30.3 이 30.300000000000001 으로 표현되었습니다.
우선은 스코어에 우리가 원하는 `BigDecimal` 값을 스코어에 저장할 수 는 없어보입니다.

---

# 2. lexicographical sorting
## 2.1 부동소수점을 고정소수점 방식으로 표현

`Sorted Set`에 값을 저장할때 스코어값을 모두 동일한 값으로 지정할 경우 문자열 요소들은 스코어값으로 정렬되는것이 아니라 사전편찬순(lexicographic)으로 정렬됩니다.

```redis
127.0.0.1:6379> zadd lex 0 44
(integer) 1
127.0.0.1:6379> zadd lex 0 355
(integer) 1
127.0.0.1:6379> zadd lex 0 5
(integer) 1
127.0.0.1:6379> zrange lex 0 -1
1) "355"
2) "44"
3) "5"
127.0.0.1:6379>
```
위의 세 개의 요소들은 모두 스코어값이 0으로 동일하게 저장되었습니다.

보시다시피 위 예제의 요소들인 44나 355등은 숫자의 형식을 하고있지만 `Sorted Set`에 문자열로 저장되면서 동시에 스코어값이 모두 0으로 동일하므로
사전편찬순으로 정렬된 것을 확인할 수 있습니다. 그래서 정렬 순서가 오름차순으로 355, 44, 5 순으로 정렬되었습니다.

만약 숫자로 인식되었다면 오름차순으로 정렬시 5, 44, 355 순서가 될겁니다.

---

이제 `BigDecimal`값에서 정수부분과 소수점 아래 부분을 분리하여, 정수파트와 소수점 부분을 별도의 집합에 저장하겠습니다.
소수점 아래 부분은 스코어를 0으로 통일하여 사전편찬순으로 정렬되도록 하겠습니다.

```java
public void save() {
    Set<BigDecimal> set = new HashSet<>(Arrays.asList(
        BigDecimal.valueOf(25500.44),
        BigDecimal.valueOf(25500.355),
        BigDecimal.valueOf(25500.05),
        BigDecimal.valueOf(25505.44)
    ));

    for (BigDecimal price : set) {
      double score = Math.floor(price.doubleValue());
      String[] split = price.stripTrailingZeros()
          .toPlainString()
          .split("\\.");

      String priceBase = split[0];
      String fragment = split[1];
      String keyFractionZset = keyPriceFractionZset(priceBase);

      redisRepository.zAdd(keyFractionZset, fragment, 0);
      redisRepository.zAdd(keyPriceZset(), keyFractionZset, score);
    }
}
```
우선 `BigDecimal`값을 정수부분과 소수점 아래부분으로 분리해줍니다. 예제의 방법 이외에도 다른 방법도 많을 것 같습니다.
그리고 소수점 아래 부분을 저장할 키를 생성합니다. 이 키에는 정수부분의 값을 포함해줍니다.
저장할때는 스코어값을 0으로 통일합니다. 사실 반드시 0일 필요는 없고 모두 동일한 값이면 됩니다.

마지막으로 이 소수점 아래 부분을 저장한 키들을 정수부 값으로 정렬해줄 집합을 하나 더 저장합니다.
이 집합의 키는 단 한개의 키만 있으면 됩니다. 그리고 정수부 값들을 이 집합의 스코어로 지정하여 정렬시키면 됩니다.

결론적으로 이러한 구조를 자바 타입으로 표현하자면 `Map<Long, Map<Long, List<String>>>`의 다소 복잡해 보이는 구조가 되겠습니다.
앞의 `Long`타입은 정수부분, 뒤의 `Long`은 소수점 아래부분을 의미합니다.

```redis
127.0.0.1:6379> zrange price.zset 0 -1
1) "price.fraction.zset:25500"
2) "price.fraction.zset:25505"
127.0.0.1:6379> zrange price.fraction.zset:25500 0 -1
1) "05"
2) "355"
3) "44"
127.0.0.1:6379> zrange price.fraction.zset:25505 0 -1
1) "44"
127.0.0.1:6379>
```
위 예제코드로 저장한 값을 직접 조회해보겠습니다. 의도한대로 정수부분과 소수점 아래부분이 분리되어 잘 정렬된 것을 확인 할 수 있습니다.

---

```redis
127.0.0.1:6379> zrangebyscore price.zset 25500 25500 withscores
1) "price.fraction.zset:25500"
2) "25500"
127.0.0.1:6379>
```

이제 구체적인 조건을 바탕으로 검색을 해보겠습니다.

우선 25500.x 부분에 어떠한 구체적인 값들이 있는지 검색하겠습니다.
우선 위 처럼 `zrangebyscore` 명령을 사용해서 스코어값이 25500과 일치하는 요소만 검색합니다.
검색결과가 있다면 그 검색결과는 소수점 아래부분들을 저장하고있는 집합의 키를 의미합니다. 반환된 그 키값으로 다시 검색을 이어나갑니다.

```redis
127.0.0.1:6379> zrangebylex price.fraction.zset:25500 - +
1) "05"
2) "355"
3) "44"
127.0.0.1:6379>
```

모든 소수점 부분들을 검색하고 싶다면 위와 같은 명령어를 실행하면 됩니다. 우리가 원한대로 정렬이 올바르게 되어있습니다.

> `zrangebylex`명령어는 스코어로 검색하는 방식과 약간의 차이점이 있습니다. 해당 명령어의 사용법은 레디스 공식사이트에서 레퍼런스 문서를 참고하시면 됩니다.

```redis
127.0.0.1:6379> zrangebylex price.fraction.zset:25500 [355 [355
1) "355"
127.0.0.1:6379>
```

정확히 25500.355 라는 값이 있는지 알고싶다면 위와 같이 최소, 최대 조건을 입력하여 검색합니다.
'[' 는 해당 값을 포함하고(inclusive), '(' 는 해당 값을 포함하지 않습니다(exclusive).

```redis
127.0.0.1:6379> zrangebylex price.fraction.zset:25500 [1 (5
1) "355"
2) "44"
127.0.0.1:6379> zrangebylex price.fraction.zset:25500 (04 [4
1) "05"
2) "355"
127.0.0.1:6379>
```

이번에는 여러 범위의 값을 검색했습니다.

위의 명령어는 숫자로 표현하자면 '25500.1 <= x < 25500.5' 를 의미합니다. 따라서 결과로 [355, 44] 가 반환되었습니다.
아래는 숫자로 표현시 '25500.04 < x <= 25500.40' 일 것입니다. 그래서 [05, 355] 가 반환되었습니다.

---

## 2.2 저장

부동소수점으로 저장되어있는 값을 고정소수점으로 저장하겠습니다.
그리고 정수부분과 소수점아래 부분을 두 개의 별도의 계층으로 구분하여 `Sorted Set`에 저장해보겠습니다.

