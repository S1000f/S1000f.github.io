---
title: Spring - 런타임시 스프링 스케쥴러 크론 표현식 변경하기
published: true
tags: spring, scheduler
---

# 1. 실행자 프레임워크와 스프링 스케쥴러

배치(batch)작업이 없는 비교적 간단한 스케쥴러도 스프링 부트(이하 스프링) 프로젝트에서 많이 사용하고 있습니다.
저는 이런 간단한 루틴의 스케쥴러는 자바 동시성 패키지의 '실행자 프레임워크(Executor Framework)'와 `CompletableFuture`를 조합하여 직접 구현하거나,
혹은 스프링이 제공하는 스케쥴러 API 를 사용합니다. 물론 스프링 스케쥴러 API 도 내부적으로 실행자 프레임워크를 사용합니다.

저는 특히 크론(cron) 표현식을 사용해야 하는 스케쥴링은 스프링 기능을 사용합니다. 왜냐하면 자바 실행자 인터페이스는 크론 표현식을 인자로 받지 않기 때문입니다.(자바8 기준)
직접 크론 표현식 파서를 만들어서 실행자 프레임워크에 적용하는 헬퍼를 만들거나 아니면 스프링의 파서만 사용할 수 도 있겠지만, 굳이 그럴거면 그냥
스프링의 기능을 처음부터 사용하는게 낫다고 생각했습니다.

그리고 어느날 문득 스케쥴러의 크론 주기를 런타임시에 변경할 수 있는 요구사항이 추가된다면 어떻게 해야할까 하는 의문이 생겼습니다. 실행자 프레임워크로 직접 작성한
스케쥴러는 변경과 재시작을 쉽게 생각해볼 수 있는 반면에 스프링의 스케쥴러는 아직 생각해본적이 없었습니다.

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

위 코드는 스프링에서 스케쥴러를 등록하고 실행하는 방식 중 간단한 방법입니다. 만약 해당 태스크의 크론 스케쥴을 런타임에 변경하고 싶다면 어떻게 해야할까요?

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

# 2. ScheduledTaskRegistrar

위 코드에서처럼 `@Scheduled` 어노테이션이 붙은 `Runnable` 구현체(인자를 받지않고 아무것도 반환하지 않는)는 스프링이 
`ScheduledAnnotationBeanPostProcessor` 빈에 태스크(task)로 등록한 뒤 실행시켜 줍니다.
스프링 부트는 우리가 직접 해당 빈을 생성하지 않아도 자동으로 생성해주는 것입니다.

## 2.1 직접 태스크를 등록

```java
@EnableScheduling
@SpringBootApplication
public class TestSpringbootApplication implements SchedulingConfigurer {
  public static void main(String[] args) {
    SpringApplication.run(TestSpringbootApplication.class, args);
  }

  @Override
  public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
    CronTask task1 = new CronTask(() -> 
        System.out.println("task2: " + LocalDateTime.now()), "*/5 * * * * *");
    taskRegistrar.addCronTask(task1);
  }

  @Scheduled(cron = "*/5 * * * * *")
  public void task1() {
    System.out.println("task1: " + LocalDateTime.now());
  }
}
```

`ScheduledAnnotationBeanPostProcessor` 는 내부에서 `ScheduledTaskRegistrar` 클래스를 사용하여 태스크를 등록합니다.
이 객체에 직접 태스크를 등록하는 방법 중 쉬운것은 위 코드처럼 `SchedulingConfigurer` 인터페이스를 구현하는 것입니다.

이 인터페이스는 `@Configuration`과 `@EnableScheduling` 어노테이션이 붙은 클래스에 구현해야 합니다.
위의 예제에서는 `@SpringBootApplication` 어노테이션안에 이미 `@Configuration` 이 포함되어 있습니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/spring-scheduler-00.png)

직접 등록한 '태스크2'와 어노테이션으로 등록된 '태스크1'이 잘 작동되는 것을 확인할 수 있습니다.

## 2.2 등록되어 있는 태스크 살펴보기

```java
@RequiredArgsConstructor
@RestController
public class TestController {
  private final ScheduledAnnotationBeanPostProcessor processor;

  @GetMapping("/size")
  public ResponseEntity<?> test() {
    Set<ScheduledTask> scheduledTasks = processor.getScheduledTasks();
    System.out.println(scheduledTasks.size());
    scheduledTasks.forEach(ScheduledTask::cancel);
    
    return ResponseEntity.ok().build();
  }
}
```

간단한 컨트롤러를 만들어서 런타임시에 태스크를 수정할 수 있는지 시도해보았습니다.
우선 `ScheduledAnnotationBeanPostProcessor` 빈을 주입받아서 `getScheduledTasks` 메소드를 호출하여 등록된 태스크를 가져옵니다.
그리고 등록되어 실행중인 태스크의 갯수를 파악한 뒤 모두 정지하는 코드입니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/spring-scheduler-01.png)

서버를 시작 후 위의 경로를 요청한 결과입니다. 기대한대로 태스크는 현재 2개(태스크0, 태스크1)가 실행중이고, 모든 태스크를 정지시키는 코드가 정상적으로 처리되었습니다.
이제 해당 태스크들을 정지시키는게 아니라 태스크 내부의 속성을 변경하거나 기존의 태스크를 삭제하고 변경된 태스크를 다시 등록할 수 있는지 확인해보겠습니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/spring-scheduler-02.png)

특별한 점이 없는 클래스입니다. `future` 필드의 타입은 자바 실행자가 반환한 `Future` 타입입니다.
이 클래스는 `ScheduledFuture` 를 구성요소로 가지며 `close` 같은 메소드의 구현을 해당 퓨처 타입에 위임하고 있습니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/spring-scheduler-03.png)

다음 구성요소인 `Task` 타입입니다. 역시 특별한 점이 없는 구체 클래스입니다. 내부에 자바 `Runnable` 타입을 가지고 있습니다.
그리고 이게 어떤 도움이 될까 싶은 `toString` 오버라이딩도 있네요. 위의 2.1 항목에서 봤던 `CronTask` 는 이 `Task` 클래스를 상속하고 있습니다.

일단은 `ScheduledAnnotationBeanPostProcessor` 빈의 `getScheduledTasks` 메소드를 호출하여 가져온 태스크 객체에서는 우리가 원하는
크론 표현식의 변경과 재시작이라는 메시지를 지원하지 않는 것 같습니다. 다음으로 든 생각은 `getScheduledTasks` 메소드가 반환한 태스크 컬렉션 자체를 변경하여
사이드이펙트로 원하는 태스크의 크론 표현식 변경을 유도하는 것이었습니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/spring-scheduler-04.png)

하지만 `getScheduledTasks` 메소드가 반환한 컬렉션은 새롭게 생성된 인스턴스입니다. 따라서 반환된 컬렉션을 변경해도 원본 컬렉션에는 영향을 미치지 않습니다.

## 2.3 직접 ScheduledTaskRegistrar 빈을 등록하기

```java
@EnableScheduling
@SpringBootApplication
public class TestSpringbootApplication implements SchedulingConfigurer {
  public static void main(String[] args) {
    SpringApplication.run(TestSpringbootApplication.class, args);
  }

  @Override
  public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
    CronTask task1 = new CronTask(() -> 
        System.out.println("task2: " + LocalDateTime.now()), "*/5 * * * * *");
    taskRegistrar.addCronTask(task1);
  }

  @Bean
  public ScheduledTaskRegistrar scheduledTaskRegistrar() {
    ScheduledTaskRegistrar registrar = new ScheduledTaskRegistrar();
    CronTask task3 = new CronTask(() ->
        System.out.println("task3: " + LocalDateTime.now()), "*/5 * * * * *");
    registrar.addCronTask(task3);

    return registrar;
  }

  @Scheduled(cron = "*/5 * * * * *")
  public void task1() {
    System.out.println("task1: " + LocalDateTime.now());
  }
}
```
![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/spring-scheduler-05.png)

이번에는 `ScheduledTaskRegistrar` 빈을 직접 등록해보았습니다. 기대한대로 태스크1, 2, 3 모두 잘 작동합니다.
한가지 유의할 점은 직접 등록한 레지스터 빈과 그 빈에 등록된 태스크는 `ScheduledAnnotationBeanPostProcessor` 빈과는 별개의 컨텍스트라는 점입니다.
따라서 위의 2.2 항목의 요청 맵핑 메소드를 호출하여도 태스크1, 2 만 취소되며 태스크3은 취소되지 않습니다.
저는 크론 표현식이 변경가능한 태스크들을 이렇게 별도의 빈에 등록하고 관리하겠습니다.

# 3. 런타임에서 특정 태스크 크론 표현식 변경하기

## 3.1 ScheduledTaskRegistrar 와 CronTask 살펴보기

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/spring-scheduler-06.png)

먼저 `ScheduledTaskRegistrar` 타입(이하 레지스터)을 보겠습니다. 레지스터 타입은 `ScheduledTaskHolder` 인터페이스를 구현하고 있는 구체 클래스입니다.
`ScheduledTaskHolder` 인터페이스는 `Set<ScheduledTask> getScheduledTasks()` 추상 메소드 하나만 정의된 인터페이스 입니다.

아쉽게도 이 레지스터 타입의 다양한 행위를 정의한 인터페이스나 추상 클래스는 없어 보입니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/spring-scheduler-07.png)

이 클래스 내부에 있는 우리가 이전에 사용했던 `addCronTask()` 메소드 입니다. 오버로딩된 두 메소드 모두 `CronTask` 타입의 파라미터를 사용한다고 볼 수 있습니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/spring-scheduler-08.png)

다음은 `CronTask` 타입입니다. 이 역시 구체 클래스이며 `TriggerTask` 타입을 상속받고 있습니다.
그리고 최종적으로 `TriggerTask` 타입은 `Task` 타입을 상속받고 있습니다. 모두 구체 클래스들이며, 이 클래스들이 공통적으로 따르는 인터페이스나 추상 클래스가 없습니다.

또한 각 타입들이 구체 클래스를 상속하면서 동시에 각자 새로운 필드들을 추가하고 있으므로 `equals()` 와 `hashCode()` 메소드를 오버라이딩 하지 않았습니다.
따라서 모든 크론 태스크들은 고유한 객체로 인식되며 각 태스크의 구분하기 위한 별도의 식별자가 없습니다.

레지스터 타입의 태스크 등록 메소드들이 이러한 구체화된 타입만 인자로 받고 있으므로, 우리가 원하는 특정 태스크를 식별한 뒤 그 태스크의 크론 표현식만 변경하기 위해서는
별도의 컨테이너를 만들거나 레지스터 타입을 확장해야 할 듯 보입니다.

## 3.2 별도의 컨테이너 사용하기

```java
@EnableScheduling
@SpringBootApplication
public class TestSpringbootApplication {

  public static void main(String[] args) {
    SpringApplication.run(TestSpringbootApplication.class, args);
  }

  @Bean("cronTaskContainer")
  public Map<String, ScheduledTask> cronTaskContainer() {
    return new ConcurrentHashMap<>();
  }

  @Bean
  public ScheduledTaskRegistrar scheduledTaskRegistrar(
      @Qualifier("cronTaskContainer") Map<String, ScheduledTask> container) {
    ScheduledTaskRegistrar registrar = new ScheduledTaskRegistrar();

    ScheduledTask task1 = registrar.scheduleCronTask(new CronTask(() ->
        System.out.println("task1: " + LocalDateTime.now()), "0/5 * * * * *"));
    ScheduledTask task2 = registrar.scheduleCronTask(new CronTask(() ->
        System.out.println("task2: " + LocalDateTime.now()), "0/5 * * * * *"));

    container.put("task1", task1);
    container.put("task2", task2);

    return registrar;
  }
}
```

먼저 빈 설정 부분입니다. 예제의 간결함을 위해서 별도의 설정 클래스 없이 메인 클래스를 사용중입니다.
'cronTaskContainer' 빈을 생성해줍니다. 이 컨테이너는 태스크를 구분하기 위한 문자열 식별자와 그에 상응하는 `ScheduledTask` 객체를 매핑해두는 컨테이너입니다.
이전과 비슷하게 레지스터 빈을 직접 생성해줍니다. 

이번에는 레지스터에 태스크를 등록시 이전의 `addCronTask()` 메소드 대신 `scheduleCronTask()` 메소드를 사용합니다.
왜냐하면 `CronTask` 타입에는 현재 동작중인 태스크를 정지 시킬 수 있는 `ScheduledFuture` 인스턴스가 없기 때문입니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/spring-scheduler-09.png)

우리가 원하는 새로운 크론 표현식으로 생성한 `CronTask`(이하 크론태스크) 인스턴스를 레지스터에 등록하고 기존의 태스크는 스케쥴링을 정지시켜야 하지만 레지스터 클래스에는
그러한 기능이 없습니다. 그리고 위에서 보았듯이 크론태스크 타입은 `equals()` 와 `hashCode()` 메소드를 오버라이드 하지 않았기 때문에, 크론 표현식을
변경하고 등록한 태스크와 기존의 태스크는 완전히 별개의 객체로 인식됩니다. 

또한 이 크론태스크를 정의한 인터페이스나 추상 클래스가 없으며, 이 클래스를 상속하더라도 `expression` 맴버는 `private final` 로 선언되어 있기 때문에
상속받은 클래스에서 `expression` 맴버에 접근하거나 변경할 수 없습니다.

```java
@RestController
public class TaskController {
  private final Map<String, ScheduledTask> container;
  private final ScheduledTaskRegistrar registrar;

  public TaskController(
      @Qualifier("cronTaskContainer") Map<String, ScheduledTask> container,
      ScheduledTaskRegistrar registrar) {
    this.container = container;
    this.registrar = registrar;
  }

  @PostMapping("/task")
  public ResponseEntity<?> modifyTask(@RequestBody TaskDto taskDto) {
    String name = taskDto.getName();
    ScheduledTask findTask = container.get(name);

    if (findTask != null) {
      findTask.cancel();
      ScheduledTask updatedTask = registrar.scheduleCronTask(
          new CronTask(findTask.getTask().getRunnable(), taskDto.getCron()));
      container.put(name, updatedTask);
    }

    return ResponseEntity.ok().build();
  }

  @Getter
  public static class TaskDto {
    private String name;
    private String cron;
  }
}
```
다음은 특정 태스크의 크론 표현식을 변경한 후 재등록하고 변경전의 기존 태스크는 스케쥴링을 정지하는 컨트롤러 예제입니다.
먼저 주입받은 컨테이너에서 식별자를 통하여 원하는 태스크 인스턴스를 찾습니다. 그리고 `cancel()` 메소드를 호출하여 스케쥴링을 정지시킵니다.
다음으로 새로운 크론 표현식과 기존의 `Runnable` 인스턴스를 그대로 사용하여 새로운 크론태스크 인스턴스를 생성하고 레지스터에 등록합니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/spring-scheduler-10.png)

위 사진은 `scheduleCronTask()` 메소드입니다. 먼저 내부에서 사용중인 `unresolvedTasks` 컨테이너에서 인자로 전달된 태스크가 있다면 제거합니다.
하지만 우리가 크론식을 변경하기 위하여 이 메소드를 호출할때는 항상 새로운 크론태스크를 인자로 전달하게 됩니다.

그리고 두 번째 if 문에서 `taskScheduler` 맴버의 널 체크 후 만약 널이라면 기존의 `addCronTask()` 메소드를 호출합니다. 만약 그렇게 된다면 우리가
태스크의 크론표현식을 변경할때 마다 `cronTaskList` 와 `unresolvedTasks` 컨테이너에 불필요한 값이 계속 누적되어 메모리 누수가 발생할 수 있습니다. 
하지만 일단 우리가 레지스터 빈을 등록하고 나면 스프링이 자동으로 `taskScheduler` 맴버를 초기화하기 때문에 이런 문제는 발생하지 않습니다.

![](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/spring-scheduler-11.png)

서버가 시작 후 태스크1, 2 모두 5초 간격으로 실행중입니다. 그리고 `POST` 요청을 통하여 태스크1의 크론표현식을 매 2초간 실행되도록 변경합니다.
기대한대로 기존에 5초간 작동되던 태스크1은 2초 간격으로 변경되었고 태스크2는 그대로 5초 간격으로 실행됩니다.

## 3.3 ScheduledTaskRegistrar 확장하기

```java
public class CustomScheduledTaskRegistrar implements ScheduledTaskHolder, InitializingBean, DisposableBean {
  private final ScheduledTaskRegistrar registrar = new ScheduledTaskRegistrar();
  protected final Map<String, ScheduledTask> container = new ConcurrentHashMap<>();

  public void setTaskScheduler(TaskScheduler taskScheduler) {
    this.registrar.setTaskScheduler(taskScheduler);
  }

  public void setScheduler(@Nullable Object scheduler) {
    this.registrar.setScheduler(scheduler);
  }

  public void updateCronTask(String name, String cron) {
    ScheduledTask findTask = container.get(name);

    if (findTask != null) {
      findTask.cancel();
      ScheduledTask updatedTask =
          registrar.scheduleCronTask(new CronTask(findTask.getTask().getRunnable(), cron));
      container.put(name, updatedTask);
    }
  }

  public ScheduledTask scheduleCronTask(String name, CronTask cronTask) {
    ScheduledTask scheduledTask = registrar.scheduleCronTask(cronTask);
    container.put(name, scheduledTask);
    return scheduledTask;
  }

  @Override
  public void destroy() {
    registrar.destroy();
  }

  @Override
  public void afterPropertiesSet() {
    registrar.afterPropertiesSet();
  }

  @Override
  public Set<ScheduledTask> getScheduledTasks() {
    return registrar.getScheduledTasks();
  }
}
```

기존 레지스터의 태스크 등록 메소드들의 파라미터 타입이 전부 구체 클래스이므로, `ScheduledTaskRegistrar` 클래스를 상속하기 보다는
해당 인스턴스를 구성(composition)요소로 포함하고 대부분의 핵심로직을 위임(delegation)하는 방향으로 확장하였습니다.
그리고 `setScheduler()` 세터를 추가하여 이 클래스가 빈으로 등록될 때 스프링에서 자동으로 디폴트 스케쥴러를 주입하도록 했습니다.

기존 레지스터의 맴버들이 전부 `private`으로 선언되어 있고 별다른 훅이 없기에 로직을 변경하기엔 힘들고 3.2 항목의 로직을 단순히 캡슐화하는 방향으로 구현되었습니다.

```java
@EnableScheduling
@SpringBootApplication
public class TestSpringbootApplication {

  public static void main(String[] args) {
    SpringApplication.run(TestSpringbootApplication.class, args);
  }

  @Bean
  public CustomScheduledTaskRegistrar customScheduledTaskRegistrar() {
    CustomScheduledTaskRegistrar registrar = new CustomScheduledTaskRegistrar();
    registrar.scheduleCronTask("task1", new CronTask(() ->
        System.out.println("task1: " + LocalDateTime.now()), "0/5 * * * * *"));
    registrar.scheduleCronTask("task2", new CronTask(() ->
        System.out.println("task2: " + LocalDateTime.now()), "0/5 * * * * *"));

    return registrar;
  }
}
```
확장한 타입을 빈으로 등록하면서 태스크도 함께 등록합니다. 이번에는 태스크와 함께 식별자 역할을 하는 문자열도 함께 받습니다.

```java
@RequiredArgsConstructor
@RestController
public class TaskController {

  private final CustomScheduledTaskRegistrar registrar;

  @PostMapping("/task")
  public ResponseEntity<?> modifyTask(@RequestBody TaskDto taskDto) {
    registrar.updateCronTask(taskDto.getName(), taskDto.getCron());
    return ResponseEntity.ok().build();
  }

  @Getter
  public static class TaskDto {
    private String name;
    private String cron;
  }
}
```

크론 표현식을 변경하는 부분도 캡슐화 덕분에 많이 간소화 되었습니다. 만약 본격적으로 이 확장된 타입을 사용하게 된다면, 태스크 등록과 업데이트 메소드는 
좀 더 사용하기 편하도록 새로운 인터페이스를 정의하여 파라미터로 지정하면 좋을 것 같습니다. 그리고 그 외 특정 태스크의 취소 등 편의 메소드들도 추가하면 좋겠지요.

# 4. 마무리

스프링은 원하는 바를 다양한 방식으로 설정하고 구현할 수 있는 프레임워크입니다.
제가 원했던 '런타임시에 특정 태스크의 크론 표현식을 변경한 뒤 재시작'이란 목적을 위해서, 제가 고민한 방식 이외에도 다양한 방법이 있을 것입니다.

내용중에 틀린 부분이 있거나 동일 기능을 하는 좀 더 쉽고 효율적인 방법이 있다면 알려주시면 감사하겠습니다!
