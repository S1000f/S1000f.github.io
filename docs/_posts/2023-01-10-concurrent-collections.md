---
title: 멀티스레드 환경에서 여러 java.util.concurrent 컬렉션을 함께 사용하기
published: true
tags: java concurrency
---

# 1. ConcurrentModificationException

```java
@Test
void concurrentModificationExceptionTest() {
    Map<Integer, String> map = new HashMap<>();
    map.put(1, "a");
    map.put(2, "b");
    map.put(3, "c");
    
    assertThrows(ConcurrentModificationException.class, () ->
        map.forEach((k, v) -> {
          if (k == 2) {
            map.remove(k);
          }
        }));
}
```

위의 코드는 맵 객체를 순회하는 중에 해당 객체를 수정하려 시도하기 때문에 `ConcurrentModificationException`이 발생합니다.
상기의 예제는 `removeIf`같은 메소드를 사용하면 간단하게 해결할 수 있지만, 멀티스레드 환경에서 맵 객체를 공유자원으로 사용할 경우 예외가 발생하는 것을
완전히 막을 순 없습니다.

```java
@Test
void concurrentMapTest() {
    Map<Integer, String> map = new ConcurrentHashMap<>();
    map.put(1, "a");
    map.put(2, "b");
    map.put(3, "c");

    assertDoesNotThrow(() ->
        map.forEach((k, v) -> {
          if (k == 2) {
            map.remove(k);
          }
        }));
}
```

다음과 같이 맵 객체의 구현체를 단순히 `ConcurrentHashMap`으로 변경함으로써 예외 발생을 방지할 수 있습니다.
`ConcurrentHashMap`은 `java.util.concurrent` 패키지에 포함되어 있으며 `Map` 인터페이스를 구현하고 있으므로, 상기 예제처럼 다른 코드를 수정하지 않고
간단하게 'thread-safe' 맵 객체를 사용할 수 있습니다. 또한 이 방법이 `syncronized` 키워드를 사용하는 것보다 더 나은 방식이기도 합니다.(이펙티브자바[아이템81])

# 2. ConcurrentMap 과 다른 컬렉션을 함께 사용하기
제가 겪은 문제는 `ConcurrentMap`과 `Set`을 함께 사용하는 경우였습니다.

```java
@Test
void concurrentMapWithSetTest() {
    ConcurrentMap<Integer, Set<String>> container = new ConcurrentHashMap<>();
    container.put(1, new HashSet<>(Arrays.asList("a", "b", "c")));

    assertThrows(ConcurrentModificationException.class, () ->
        container.forEach((k, v) -> {
          for (String item : v) {
            if (item.equals("b")) {
              v.remove(item);
            }
          }
        })
    );
}
```

위의 코드는 `ConcurrentMap`을 사용하였지만, `Set`은 'tread-safe' 하지 않기 때문에 예외가 발생합니다.
이를 해결하기 위해서 `ConcurrentHashMap`으로 부터 `Set`객체를 생성하는 방법이 있습니다.

```java
@Test
void concurrentMapWithConcurrentSetTest() {
    ConcurrentMap<Integer, Set<String>> container = new ConcurrentHashMap<>();
    Set<String> set = Collections.newSetFromMap(new ConcurrentHashMap<>());
    set.addAll(Arrays.asList("a", "b", "c"));
    container.put(1, set);

    assertDoesNotThrow(() ->
        container.forEach((k, v) -> {
          for (String item : v) {
            if (item.equals("b")) {
              v.remove(item);
            }
          }
        })
    );
}
```

자바의 `Set`객체는 내부적으로 `Map`객체를 사용하여 구현됩니다. 따라서 위의 코드처럼 `Collections.newSetFromMap` 메소드를 사용하여 `ConcurrentMap`
을 기반으로 `Set` 객체를 생성하면 멀티스레드 환경에서 공유자원으로 사용되더라도 예외가 발생하지 않습니다.

# 3. 그 외 thread-safe 컬렉션

```java
@Test
void threadSafeList() throws InterruptedException {
    BlockingQueue<String> arrayQueue = new ArrayBlockingQueue<>(5000);
    BlockingQueue<String> linkedListQueue = new LinkedBlockingQueue<>(5000);

    arrayQueue.add("a");
    arrayQueue.put("b");
    arrayQueue.offer("c");

    assertEquals(3, arrayQueue.size());

    List<String> list = new ArrayList<>();
    arrayQueue.drainTo(list);

    assertEquals(0, arrayQueue.size());
    assertEquals(3, list.size());
}
```

멀티스레드 환경에서 리스트 객체를 공유자원으로 사용할때는 `BlockingQueue`가 하나의 대안이 될 수 있습니다. 또한 `drainTo`같은 메소드를 지원하므로 본래
목적인 큐 자료구조로도 사용할 수 있습니다.