### 왜 참조 타입은 기본적으로 Sendable이 채택되지 않는가?

기본적으로 참조 타입은 힙 영역에 위치해 있기 때문이다. 힙 영역은 두 개 이상의 실행 흐름이 동시에 접근할 수 있기 때문에 Data Race 문제가 발생한다. 그렇기 때문에 기본적으로 Sendable 할 수 없다.

!image.png

---

### Sendable 타입도 Data Race가 발생할 수 있는가?

클로저의 캡처로 인해서 Data Race가 발생할 수 있다.

```swift
func call() {
    var num1: Int = 10
    
    Task { // ⛔️
        num1 += 10
    }
    
    num1 += 10
}
```

클로저의 캡처는 기본적으로 스택내에 있는 변수 주소를 캡처한다. 즉, `num1` 의 변수 주소를 캡처한다는 뜻이다.

그렇기 때문에 `num1` 의 변수에 접근할 수 있는 실행 흐름은 두 개 이상이 된다. 그렇기 때문에 Data Race가 발생할 수 있다.

만약 아래와 같이 캡처리스트를 사용하면 `num1` 의 변수 주소가 아닌 값 자체를 캡처하기 때문에 Data Race는 발생하지 않는다.

```swift
func call() {
    var num1: Int = 10
    
    Task { [num1] in // ✅
        var num2 = num1
        num2 += 10
    }
    
    num1 += 10
}
```

---

### Sendable 함수 타입은 왜 지역 변수를 캡처할 수 없는가?

`@Sendable` 클로저는 여러 번 동시에 호출될 수 있으므로, 클로저가 캡처하는 모든 값 역시 동시적으로 접근해도 안전해야 하기 때문이다.

```swift
func callConcurrently(
    _ closure: @escaping @Sendable () -> Void
) {
    Task {
        await withDiscardingTaskGroup { group in
            for _ in 0..<10 {
                group.addTask {
                    closure()
                }
            }
        }
    }
}

func capture() {
    var result = 0
    result += 1

    callConcurrently {
        print(result)
    }
}
```

위 코드에서 10개의 Task가 클로저를 동시 실행할 수 있다. 해당 경우에 `result` 에 동시적으로 접근할 수가 있다. 

 **중요한 사실은**`result` **가 Sendable이라는 사실만으로 가변 변수의 캡처까지 안전해지는 것은 아니라는 점이다.**


> ❓ **그렇다면 액터는 가능한가? 액터는 직렬 실행자 때문에 동시적으로 접근할 수 없잖아.**


액터 또한 가변 변수라면 캡처할 수 없다. 핵심은 Actor가 보호하는 메모리와 `var` 변수가 저장되는 메모리는 서로 다르다는 점이다.

```swift
actor Bank {
    let id: Int

    init(id: Int) {
        self.id = id
    }

    func deposit() {
        print("Deposit to bank \(id)")
    }
}

func example() {
    var bank = Bank(id: 1)

    let closure: @Sendable () async -> Void = {
        await bank.deposit() // ❌
    }

    Task.detached {
        await closure()
    }

    bank = Bank(id: 2)
}
```

다음과 같이 `bank` 가 저장된 지역 변수의 메모리는 여전히 동시적으로 접근이 가능하기 때문에 허용하지 않는다.
