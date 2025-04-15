# Beyond the basics of structured concurrency

키워드: Concurrency

# Task Hierarchy

구조화된 동시성을 사용하면 concurrency코드를 추론할 수 있음

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image.png)

구조화된 작업은 async let과 taskGroup으로 만들어지지만 비구조화된 작업은 Task와 Task.detached로 만들어짐

구조화된 작업은 로컬 변수처럼 작업이 선언된 스코프에서 끝까지 살아남음

그리고 스코프 밖으로 나가면 자동으로 취소 → 작업의 수명을 분명히 알 수 있다

애플: 가능한 한 구조화된 작업을 사용해라!

예시로 요리를 해볼거다!

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%201.png)

수프를 조리하려면 여러 단계를 거쳐야 함

요리사들이 재료를 썰고 닭고기를 재우고 국물을 끓인 다음 조리해야 수프를 완성함

어떤건 동시에 가능하지만 어떤건 특정한 순서에 따라 진행해야만 함

makeSoup함수에 집중해보자

```swift
func makeSoup(order: Order) async throws -> Soup {
    let boilingPot = Task { try await stove.boilBroth() }
    let choppedIngredients = Task { try await chopIngredients(order.ingredients) }
    let meat = Task { await marinate(meat: .chicken) }
    let soup = await Soup(meat: meat.value, ingredients: choppedIngredients.value)
    return await stove.cook(pot: boilingPot.value, soup: soup, duration: .minutes(10))
}
```

구조화되지 않은 작업을 만들어서 동시성을 부여하고 기다릴 수 도 있다

이건 swift에서 권장하는 동시성 이용법이 아니다

```swift
func makeSoup(order: Order) async throws -> Soup {
    async let pot = stove.boilBroth()
    async let choppedIngredients = chopIngredients(order.ingredients)
    async let meat = marinate(meat: .chicken)
    let soup = try await Soup(meat: meat, ingredients: choppedIngredients)
    return try await stove.cook(pot: pot, soup: soup, duration: .minutes(10))
}
```

만들어야 할 child task 개수를 알고있으므로 편리한 async let문을 사용하면 됨

이 task들은 부모작업과 구조화된 관계를 형성함

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%202.png)

makeSoup에는 재료 썰기 닭고기 재우기, 국물 끓이기로  child task가 셋임

chopIngredients는 작업 그룹을 사용해 각 재료에 해당하는 child task를 만듬

재료가 세 가지라면 자식도 세 개.

부모 자식 계층 구조에서 Task tree가 만들어짐

# Task Cancellation

작업 취소는 앱에 작업 결과가 더는 필요하지 않으므로 작업을 중지하고 부분적인 결과만을 반환하거나 오류를 발생시켜야 한다는 신호를 보낼 때 쓰임

예시에서는 손님이 떠나거나 주문을 바꾸거나 폐점 시간이 됐다면 수프 만들기를 중단

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%203.png)

구조화된 작업은 스코프를 벗어나면 암묵적으로 취소

작업 그룹에 cancelAll을 호출해 활성화된 자식 작업과 앞으로 생길 자식 작업을 모두 취소할 수도 있음

하지만 비구조화된 작업은 cancel 함수를 이용해 명시적으로 취소됨

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%204.png)

부모 작업을 취소하면 자식 작업도 모두 취소

취소는 협동적이므로 자식 작업이 곧바로 중지되지는 않음

단지 해당 작업에 isCancelled 플래그가 생길 뿐임

실제 취소는 사실 코드 안에서 이뤄짐

```swift
func makeSoup(order: Order) async throws -> Soup {
    async let pot = stove.boilBroth()

    guard !Task.isCancelled else {
        throw SoupCancellationError()
    }

    async let choppedIngredients = chopIngredients(order.ingredients)
    async let meat = marinate(meat: .chicken)
    let soup = try await Soup(meat: meat, ingredients: choppedIngredients)
    return try await stove.cook(pot: pot, soup: soup, duration: .minutes(10))
}
```

취소는 경쟁이라서 검사 전에 작업을 취소하면 makeSoup는 SoupCancellationError를 발생시키고 

가드가 실행된 후에 작업을 취소하면 프로그램은 계속해서 수프를 조리함

부분적인 결과를 반환하는 대신 취소 오류를 발생시키려면 Task.checkCancellation을 호출하면 됨

그러면 작업이 취소됐을 때 이게 CancellationError를 발생시킴

**비싼 작업을 시작하기 전에 작업 취소 상태를 검사하는 것은 중요**

취소 검사는 동기식이므로 취소에 반응해야 하는 모든 함수는 동기식이든 비동기식이든 진행 전에 작업 취소 상태를 검사해야 함

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%205.png)

 isCancelled나 checkCancellation으로 폴링하는 것은 작업이 실행 중일 때는 유용하지만 때로는 

AsyncSequence를 구현할 때처럼 작업이 일시 중단되어 실행되는 코드가 없을 때 취소에 반응해야 하는 상황도 있음

withTaskCancellationHandler는 이런 경우에 유용

이런함수를 도입햇다고해보자

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%206.png)

주문이 들어오는대로 수프를 만든다

비동기적인 for 루프에서 makeSoup 실행전에 새주문을 받으면

위에서 정의한대로 취소를 처리하여 오류를 발생시킴

아래에는 다른시나리오임

작업이 실행되고 있지않아서 취소이벤트를 명시적으로 폴링할 수 없음

그대신 취소 핸들러를 사용해서 취소 이벤트를 탐지해서 비동기적 for루프에서 빠져나와야함

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%207.png)

```swift
public func next() async -> Order? {
    return await withTaskCancellationHandler {
        let result = await kitchen.generateOrder()
        guard state.isRunning else {
            return nil
        }
        return result
    } onCancel: {
        state.cancel()
    }
}
```

AsyncSequence는 많은 경우 상태 기계로 구현되는데 상태 기계는 시퀀스 실행을 멈추는 데 쓰임

 isRunning이 true일 때 시퀀스는 계속해서 주문을 발생시켜야 합니다 작업이 취소되면 시퀀스가 끝났으니 정지해야 한다고 알려야 하죠 그러려면 시퀀스 상태 기계에서 cancel 함수를 동기적으로 호출하면 됩니다 

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%208.png)

액터는 캡슐화된 상태를 보호하는 데는 좋지만 우리는 상태 기계에서 개별 프로퍼티를 수정하고 읽고자 하므로 액터는 그다지 적합한 도구가 아닙니다 

게다가 **액터에 실행되는 연산의 순서를 보장할 수 없으므로** 취소가 먼저 실행될지도 확실히 알 수 없습니다 다른 무언가가 필요합니다

Swift Atomics 패키지에 있는 아토믹을 쓰기로 했지만 디스패치 큐나 Lock을 써도 됨

---

## 아토믹이란?

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%209.png)

사용예시)

- 카운터 값이 **정확하게 증가**하긴 해야함 → 원자성은 중요
- 누가 먼저 증가했는지는 **중요하지 않음**.
- 이런 상황에서는 `relaxed`를 쓰는 것이 가장 빠르고 효율적
- 원자성보장, 순서보장x, 성능빠름

---

이런 메커니즘을 사용하면 공유 상태를 동기화하고 경쟁 상태를 피하면서도 실행 중인 상태 기계를 취소할 수 있음

취소 핸들러에 비구조화된 작업을 도입하지 않아도됨

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%2010.png)

작업 트리는 작업 취소 정보를 자동으로 전파함

우리는 취소 토큰과 동기화를 신경 쓸 필요 없이 Swift 런타임이 알아서 안전하게 처리하도록 두면 됨

이것만 기억하기 → 취소해도 작업 실행이 멈추지는 않음

단지 작업이 취소됐으니 최대한 빨리 작업을 끝내라는 신호를 작업에 보낼 뿐

취소 여부를 검사하는 건 코드의 몫

# Task Priority

우선순위는 해당 작업이 얼마나 긴급한지를 시스템에 알리는 방법

버튼을 누를 때 반응하는 것 같은 특정한 작업은 즉각 실행하지 않으면 앱이 멈춘 것처럼 보이게 됨

반면 서버에서 콘텐츠를 프리패칭하는 것 같은 작업은 누구의 눈에도 띄지 않게 백그라운드에서 실행할 수 있음

우선순위 역전이란?

우선순위 역전은 우선순위가 높은 작업이 우선순위가 낮은 작업의 결과를 기다릴 때 발생합니다 

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%2011.png)

본적으로 자식 작업은 부모에게 우선순위를 상속받으므로 makeSoup가 실행하는 작업의 우선순위가 중간이라고 가정한다면 실행되는 자식 작업의 우선순위도 모두 중간

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%2012.png)

- 우선순위가 더 높은 작업의 결과를 기다리면 작업 트리의 자식 작업 모두 우선순위가 상승합니다 
주의할 점은 작업 그룹의 다음 결과를 기다리면 그룹 내 모든 자식 작업의 우선순위가 상승
→ 어떤 작업이 다음으로 완료될 가능성이 가장 높은지 모름
- 동시성 런타임은 우선순위 큐로 작업 스케줄을 잡으므로 우선순위가 높은 작업은 우선순위가 낮은 작업보다 먼저 선택되어 실행 
작업의 우선순위가 높아지면 남은 수명 내내 높게 유지되며 우선순위 상승을 되돌릴 방법은 없음

# Task Group Patterns

요리시에 도마를 놓을 수 있는 자리는 한정되어 있다

너무 많은 재료를 동시에 썬다면 다른 작업을 할 공간이 모자라니 동시에 써는 재료의 개수를 제한해보자

```swift
withTaskGroup(of: Something.self) { group in
    for _ in 0..<maxConcurrentTasks { // << 동시적 작업이 너무 많이 생성되지 않도록 최대 개수만큼만 작업을 생성
        group.addTask { }
    }
    while let <partial result> = await group.next() { // << 최대 개수만큼 작업이 실행되면 그중 하나가 끝나기를 기다립니다
    // 작업 하나가 끝났는데 중지 조건을 충족하지 않는다면 
        if !shouldStop { 
            group.addTask { } // << 새로운 작업을 생성하여 계속 진도를 나갑니다 이러면 그룹에 있는 동시적 작업의 개수가 제한됨
        }
    }
}
```

다른 예시를 보자

```swift
func run() async throws {
    try await withThrowingTaskGroup(of: Void.self) { group in
    // 요리사들은 각자의 작업을 하며 교대 근무를 시작합니다
        for cook in staff.keys {
            group.addTask { try await cook.handleShift() }
        }

    // 요리사들이 일할 때 우리는 타이머를 맞추고 
        group.addTask {
            // keep the restaurant going until closing time
            try await Task.sleep(for: shiftDuration)
        }

    // 타이머가 다 돌아가면 진행 중인 근무를 모두 취소합니다 
        try await group.next()
        // cancel all ongoing shifts
        group.cancelAll()
    }
}
```

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%2013.png)

withDiscardingTaskGroup API가 새로 생김

폐기 작업 그룹은 완료된 자식 작업의 결과를 유지하지 않음

작업에서 사용한 리소스는 작업이 끝나자마자 자유롭게 쓸 수 있음

위의 요리사 run()코드를 아래와같이 바꿀수있음 

```swift
func run() async throws {
    try await withThrowingDiscardingTaskGroup { group in
        for cook in staff.keys {
            group.addTask { try await cook.handleShift() }
        }

        group.addTask { // keep the restaurant going until closing time
            try await Task.sleep(for: shiftDuration)
            throw TimeToCloseError()
        }
    }
}

/* before
func run() async throws {
    try await withThrowingTaskGroup(of: Void.self) { group in
        for cook in staff.keys {
            group.addTask { try await cook.handleShift() }
        }

        group.addTask {
            // keep the restaurant going until closing time
            try await Task.sleep(for: shiftDuration)
        }

    // 타이머가 다 돌아가면 진행 중인 근무를 모두 취소합니다 
        try await group.next()
        // cancel all ongoing shifts
        group.cancelAll()
    }
}
*/
```

discardTaskGroup은 자기 자식을 자동으로 정리하므로 명시적으로 그룹을 취소하고 정리할 필요가 없음

형제 자동 취소 기능도 있어서 자식 작업 중 하나가 오류를 발생시키면 나머지 작업도 모두 자동으로 취소됨

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%2014.png)

새로운 discardTaskGroup은 작업 하나가 끝날 때마다 리소스를 방출함

결과를 수집해야 하는 일반 작업 그룹과는 다름

그 덕에 아무것도 반환할 필요 없는 작업이 많을 때 메모리 소비를 줄일 수 있음

예를 들면 계속 이어지는 요청을 처리할 때 작업 그룹이 값을 반환하게 하면서도 동시적 작업의 개수를 제한하고 싶을 때
앞서 다룬 일반적 패턴을 이용하면 한 작업이 끝나야 다른 작업이 시작하므로 작업이 폭발적으로 늘지 않음

# Task-Local Values

TaskLocal Value이란 주어진 작업, 더 정확히 말하면 작업 계층 구조와 연결된 데이터

값은 전역 변수와 비슷하지만  TaskLocal값에 바인딩된 값은 현재의 작업 계층 구조에서만 나올 수 있음

TaskLocal 값은 정적 프로퍼티로 선언되는데 이때 TaskLocal 프로퍼티 래퍼를 씀

TaskLocal은 옵셔널로 설정하는 게 좋음

값 집합이 없는 작업은 기본값을 반환해야 하는데 기본값은 nil 옵셔널로 손쉽게 나타낼 수 있음

바인딩되지 않은 TaskLocal은 자기의 기본값을 담고 있습니다

TaskLocal 값은 명시적으로 지정될 수 없고 특정한 스코프에 바인딩되어야 함

바인딩은 스코프가 지속되는 동안 유지되며 스코프가 끝나면 원래 값으로 되돌아감

예시로 살펴보자

TaskLocal인 cook에 sakura 이름을 바인딩함

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%2015.png)

자식들은 작업 로컬 스토리지에 어떤 값도 저장하지 않음

작업 로컬 변수에 바인딩된 값을 찾으려면 그 값을 가진 작업을 찾을 때까지 부모 각각을 반복해서 탐색해야 함

바인딩된 값을 가진 작업을 찾으면 작업 로컬은 그 값을 취함

부모 작업이 없는 뿌리에 도달하면 작업 로컬은 바인딩되지 않았으며 우린 원래의 기본값을 찾게 됨

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%2016.png)

Swift 런타임은 이런 쿼리를 더 빨리 실행하도록 최적화했다

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%2017.png)

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%2018.png)

트리를 탐색하는 대신 **우리가 찾는 키가 있는 작업을 직접 참조**하면 됨

작업 트리의 반복적인 성질은 이전 값을 잃지 않으면서 값을 섀도잉하는 데 유용

## 로깅 - (SwiftLog 라이브러리) 서버사이드 로그 관련…

SwiftLog는 로깅 API 패키지로 다양한 지원 구현을 갖추었으며 이걸 사용하면 서버를 변경하지 않고도 필요에 맞는 로깅 백엔드를 드롭인할 수 있음

MetadataProvider는 SwiftLog 1.5에 생긴 새 API

MetadataProvider를 구현하면 로깅 로직을 쉽게 추상화하여 관련 값에 대한 정보를 확실히 일관되게 내보낼 수 있음

MetadataProvider는 딕셔너리와 비슷한 구조를 사용

로깅되는 값에 이름을 매핑함

multiplex 함수를 이용해서 여러 MetadataProvider를 결합해 하나의 객체로만듬

```swift
let orderMetadataProvider = Logger.MetadataProvider {
    var metadata: Logger.Metadata = [:]
    if let orderID = Kitchen.orderID {
        metadata["orderID"] = "\(orderID)"
    }
    return metadata
}

let chefMetadataProvider = Logger.MetadataProvider {
    var metadata: Logger.Metadata = [:]
    if let chef = Kitchen.chef {
        metadata["chef"] = "\(chef)"
    }
    return metadata
}

let metadataProvider = Logger.MetadataProvider.multiplex([orderMetadataProvider,
                                                          chefMetadataProvider])

LoggingSystem.bootstrap(StreamLogHandler.standardOutput, metadataProvider: metadataProvider)

let logger = Logger(label: "KitchenService")
```

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%2019.png)

MetadataProvider값은 로그에 명확히 나열되므로 수프 조리 과정에서 주문을 추적하기가 더 쉬워집니다

![image.png](Beyond%20the%20basics%20of%20structured%20concurrency/image%2020.png)

- 작업 로컬 값을 사용하면 작업 계층 구조에 정보 첨부가능
- 분리된 작업을 제외한 모든 작업은 현재의 작업에서 작업 로컬 값을 상속받음
- 작업들은 주어진 스코프 안에서 특정한 작업 트리에 바인딩되어 저수준 빌딩 블록을 제공하는데 이것을 사용해서 작업 계층 구조를 통해 추가 콘텍스트 정보를 전파할 수 있음

# Task Traces - 분산 서버 디버깅 툴 (Skip)

새로 나온 Swift 분산 추적 패키지를 소개합니다 

작업 트리는 단일 작업 계층 구조에서 자식 작업을 관리하기에 좋습니다 

분산 추적을 이용하면 여러 시스템에서 작업 트리의 이점을 활용하여 성능 특성과 작업 관계를 파악할 수 있습니다

Swift 분산 추적 패키지는 OpenTelemetry 프로토콜 구현을 포함하므로 Zipkin이나 Jaeger 같은 기존의 추적 솔루션도 훌륭하게 작동합니다

Swift 분산 추적의 목표는 Xcode Instruments의 불투명한 '응답 대기 중'에 서버에 일어나는 일에 대한 자세한 정보를 채워 넣는 것입니다
