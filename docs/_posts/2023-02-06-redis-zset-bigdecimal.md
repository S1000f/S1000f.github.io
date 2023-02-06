---
title: Redis - Sorted Set 부동소수점 값 저장, 정렬, 조회하기
published: false
tags: redis
---

# 1. Sorted Set
레디스의 'Sorted Set' 자료구조는 기본적으론 'Set' 자료구조와 동일하지만, 각 요소에 'score(이하 스코어)' 라는 연관값을 지정하여 정렬된 상태로 유지합니다.
기본적으로 집합 자료구조이므로 서로 다른 스코어 값을 할당하더라도 동일한 요소는 중복으로써 제거됩니다.

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

스코어 값의 타입은 double-precision 부동소수점입니다.



# 2. lexicographical sorting 사용하기