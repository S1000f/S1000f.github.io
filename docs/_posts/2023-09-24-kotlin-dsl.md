---
title: Kotlin DSL
published: true
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

테스트를 해보니 예상대로 동작합니다. 위 인식기의 상태전이도는 아래와 같습니다. 정규표현은 `a*b` 입니다.

![dfa](https://raw.githubusercontent.com/S1000f/S1000f.github.io/master/docs/images/dfa.jpg)

DSL를 적용하고 싶은 부분은 전이함수 생성 부분입니다. `when`의 매칭을 사용하여 직관적으로 의미를 파악할 수 는 있지만 불필요한 `else`가 많이 보입니다.
그리고 `when(state)`와 `when(symbol)`에 이름을 부여하여 좀 더 의미를 명확히 할 수 있을것 같습니다.

---

# 2. DSL 만들기

DSL(Domain-Specific Language)은 도메인 특화 언어로써, 특정 도메인에 대한 표현력과 가독성을 높이고 좀 더 선언적인 코드 작성이 가능하도록 도와줍니다.
그러니 제가 만드는 매우 간단한 이 DSL의 도메인은 DFA 라고 볼 수 있겠습니다.

```kotlin
typealias TransitionBuilder = MutableSet<Pair<Symbol, State>>

fun TransitionBuilder.transit(s: Symbol, q: State): TransitionBuilder =
    apply { add(s to q) }

typealias TransitionTable = MutableMap<State, TransitionBuilder>
```

먼저 `TransitionBuilder`는 기존 방식에서 `when(symbol) {...}` 부분을 대체합니다. 
심벌과 상태의 쌍은 `Pair<Symbol, State>` 타입을 사용하겠습니다. 그리고 여러개의 쌍을 저장해야 하므로 최종적으로 `Set<Pair<Symbol, State>>` 타입이 됩니다.
이 타입은 인터페이스를 정의하거나 혹은 클래스를 사용하여 `Set` 내장객체에게 핵심 기능을 위임하도록 직접 구현할 수 도 있으나 지금은 그 외의 역할은 없으므로 타입별칭으로 대체했습니다.

`TransitionBuilder`는 `transit` 확장함수를 정의하여 쌍을 추가할 수 있도록 했습니다.

다음으로 `TransitionTable`은 '상태전이표'로써, 기존 방식에서 `when(state) {...}` 부분을 대체합니다.
상태와 `TransitionBuilder`의 집합을 매핑해야 하므로 `Map<State, TransitionBuilder>` 타입이 됩니다.

여기까지만 작성할 경우, 전이함수를 생성하기 위해서 코틀린의 기본 함수들인 `mapOf, setOf`등을 사용해야 합니다.
여기에 직접 이름을 붙이기 위해서 수신객체 지정 람다를 사용하겠습니다.

---

```kotlin
inline fun TransitionTable.state(q: State, action: TransitionBuilder.() -> Unit): TransitionTable {
    this[q] = mutableSetOf()
    this[q]?.action()
    return this
}

inline fun transition(action: TransitionTable.() -> Unit): TransitionTable {
    val map: TransitionTable = mutableMapOf()
    return map.apply(action)
}
```

`TransitionTable.state` 확장함수는 상태값 하나와 `TransitionBuilder.() -> Unit` 수신객체 지정 람다를 받습니다.
수신객체 지정 람다의 작동은 확장함수와 유사합니다.
파라미터 `action`의 람다는 `TransitionBuilder`를 수신객체로 받으며, 이를 통해 `TransitionBuilder.transit` 함수를 호출할 수 있습니다.

이는 결국 `TransitionBuilder().apply { this.trasit(); this.transit(); ... }` 와 같은 기능을 합니다.

다음으로 `transition` 확장함수는 `TransitionTable.() -> Unit` 수신객체 지정 람다를 받습니다.
위와 유사하게 `TransitionTable`을 수신객체로 받으며, 이를 통해 `TransitionTable.state` 함수를 호출할 수 있고 이를 통해 누적된 변화를 반환합니다.

```kotlin
fun TransitionTable.toTransitionFunction(): TransitionFunction =
    { state, symbol ->
        this[state]?.find { (s, _) -> s == symbol }?.second ?: state
    }

val TransitionTable.accepter: FiniteAccpeter
    get() = get@{ input ->
        val start: State = this.keys.find { it.isStart } ?: return@get false
        this.toTransitionFunction().accepter(start)(input)
    }
```

마지막으로 이미 작성한 기능을 사용하여 `TransitionTable`도 `FiniteAccepter`함수를 반환할 수 있도록 했습니다.

---

# 3. 사용해보기

```kotlin
@Test
fun usingDslTest() {
    // given
    val q0 = stateOf("q0", isStart = true)
    val q1 = stateOf("q1", isFinal = true)
    val q2 = stateOf("q2")

    val a: Symbol = 'a'
    val b: Symbol = 'b'

    // when
    val f = transition {
        state(q0) {
            transit(a, q0)
            transit(b, q1)
        }
        state(q1) {
            transit(a, q2)
            transit(b, q2)
        }
    }

    val accepter = f.accepter

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

DSL을 사용하여 처음의 테스트를 똑같이 작성해보았습니다. 전이함수 작성 부분만 달라졌습니다.
이전과 비교하여 코드 작성이 간소화되고 의미의 전달이 좀 더 명확해진 것 같습니까?
```kotlin
state(q0) {
    transit(a, q0)
    transit(b, q1)
}
```
이 부분을 '상태 q0은 전이한다 심벌 a 인 경우 q0로, 심벌 b 인 경우 q1로'라고 읽을 수 있습니다. 사람에 따라선 이전 방식이 좀 더 가독성이 좋다고 느낄 수도 있습니다.
하지만 이렇게 DSL을 사용하면 에디터의 자동완성이나 KDoc 등의 도움을 받을 수도 있습니다.
그리고 저는 단순히 예제를 위한 구현이었지만, 만약 규모가 큰 툴이나 라이브러리를 작성해야 한다면 DSL을 사용하는것이 API 사용자들에게 훨씬 편리할 것 같습니다.
코틀린의 'Exposed' 라이브러리 처럼 말이죠.

이만 포스트를 마치겠습니다.
내용 중에 틀린 부분이 있거나 내용이 빠진게 있다면 댓글로 알려주시면 감사하겠습니다!

