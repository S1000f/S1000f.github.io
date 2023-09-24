---
title: Kotlin DSL
published: false
tags: kotlin
---

유명한 코틀린 입문서인 'Kotlin IN ACTION (오현석 옮김,에이콘,2017)' 에서는 코틀린으로 DSL을 만드는 방법을 책의 한 장을 할애하여 설명하고 있습니다.
그리고 최근 Gradle 에서도 코틀린 DSL을 사용한 스크립트 작성도 지원하고 있습니다.

마침 '형식언어와 오토마타'를 공부하고 있어서 연습삼아 코틀린 DSL 스타일로 간단한 DFA를 만들어 보았습니다.

---

# 1. 구현

```kotlin
typealias Symbol = Char
typealias Alphabet = Set<Symbol>
```

먼저 '심벌(symbol)'과 '알파벳(alphabet)'을 정의합니다. 문자열은 코틀린에 내장되어 있는 `String`타입을 그대로 사용하겠습니다.
심벌은 문자열을 구성하는 문자 하나를 의미하므로, 기본타입인 `Char`타입에 별칭을 붙여 사용하겠습니다. 알파벳은 심벌들의 유한집합을 의미하므로 역시
`Set<Symbol>` 타입에 별칭을 붙여 사용하겠습니다.

```kotlin
typealias FiniteAccepter = (input: String) -> Boolean
```
유한 인식기는 문자열을 입력으로 받아서 이를 승인(accept)하거나 거부(reject)하는 기능을 합니다.
이를 표현하기 위해서 간단한 타입을 하나 정의했습니다. 코틀린은 함수 타입을 정의할때 가독성을 위해서 파라미터에 이름을 붙일 수 있습니다.

```kotlin
interface State {
    val isFinal: Boolean
    val isStart: Boolean
    val label: String
}

data class StateImpl(
    override val label: String,
    override val isFinal: Boolean,
    override val isStart: Boolean,
) : State

fun stateOf(
    label: String = "",
    isFinal: Boolean = false,
    isStart: Boolean = false
): State = StateImpl(label, isFinal, isStart)
```

유한 인식기는 유한개의 상태(state)를 가지고 있습니다. 이 상태를 표현하는 인터페이스를 정의했습니다.
이 내부 상태들은 중 하나는 초기상태(initial state)이고, 여러 개의 승인상태(final state)를 가질 수 있습니다.
그리고 각 상태들을 쉽게 구분하기 위해 라벨(label)을 부여할 수 도 있습니다.

---

```kotlin
typealias TransitionFunction = (state: State, symbol: Symbol) -> State
```

심벌과 상태가 정의되었으므로 이제 '전이함수(transition function)'를 정의합니다. 유한 인식기는 초기상태에서 시작하며, 입력 문자열의 가장 왼쪽의 심벌부터
읽습니다. 읽은 심벌과 현재 상태로 부터 다음 상태로의 전이를 하게되는데 이때 사용되는것이 전이함수입니다.
위의 함수는 결정적 유한 인식기에서만 사용할 수 있습니다.

```kotlin
fun TransitionFunction.accepter(
    startState: State
): FiniteAccpeter = { input ->
    var currentState = startState
    for (symbol in input.toList()) {
        currentState = this(currentState, symbol)
    }
    currentState.isFinal
}
```

전이함수를 정의했으므로 이제 유한인식기를 구현할 수 있습니다. `TransitionFunction` 타입의 확장함수를 정의하여 유한인식기 타입을 반환하도록 했습니다.
유한 인식기는 내부에 상태를 가지고 있으며, 입력 문자열을 처리함에 따라 '형상(configuration)'이 계속 변화하는 객체입니다.
따라서 위와 같이 클로저보다는 내부 상태를 변화시키는 전통적인 클래스가 더 어울릴것 같지만, 그냥 귀찮아서 일단 이대로 사용하겠습니다.

```kotlin
@Test
fun accepterTest() {
    // given
    val q0 = stateOf("q0", isStart = true)
    val q1 = stateOf("q1", isFinal = true)
    val q2 = stateOf("q2")

    val a: Symbol = 'a'
    val b: Symbol = 'b'

    // when
    val f: TransitionFunction = { state, symbol ->
        when (state) {
            q0 -> when (symbol) {
                a -> q0
                b -> q1
                else -> q0
            }
            q1 -> when (symbol) {
                a, b -> q2
                else -> q1
            }
            q2 -> q2
            else -> q0
        }
    }

    val accepter = f.accepter(q0)

    // then
    assertTrue(accepter("b"))
    assertTrue(accepter("ab"))
    assertTrue(accepter("aab"))

    assertFalse(accepter("a"))
    assertFalse(accepter("aa"))
    assertFalse(accepter("bb"))
    assertFalse(accepter("ba"))
    assertFalse(accepter("aba"))
    assertFalse(accepter("abb"))
}
```

테스트를 해보니 예상대로 동작합니다. 위 인식기의 상태전이도는 아래와 같습니다.

![dfa](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/dfa.jpg)