---
title: Spring Scheduler
published: false
tags: spring
---

```java
@EnableScheduling
@SpringBootApplication
public class TestSpringbootApplication {

  public static void main(String[] args) {
    SpringApplication.run(TestSpringbootApplication.class, args);
  }

  @Scheduled(cron = "*/5 * * * * *")
  public void task() {
    System.out.println(LocalDateTime.now());
  }
}
```

위 코드는 아마 스프링 부트(이하 스프링)에서 스케쥴러를 사용하는 가장 간단한 방식일겁니다. 만약 해당 태스크의 크론 스케쥴을 런타임에 변경하고 싶다면 어떻게 해야할까요?

```java
@Scheduled(cron = "${props.cron}")
public void task() {
  System.out.println(LocalDateTime.now());
}
```
```yaml
props:
  cron: */5 * * * * *
```

크론 표현식을 프로퍼티로 이동시키고 `@Scheduled`의 `cron` 속성에 프로퍼티를 참조하고 난 뒤애, 실행중인 서버에 프로퍼티 값을 변경하고 이 스케쥴러를 재시작해야 합니다.
이는 좀 귀찮은 작업이므로 좀 더 간단하게 변경 할 수 있는 방법을 찾아보았습니다.

