### 메인 액터에서 Task를 사용하면 메인 쓰레드에서 동작하는가?

맞다. Task는 실행 액터를 상속받기 때문이다.

```swift
@MainActor
func execute() {
    Task {
        print(isMainThread())
        await asyncFunc()
        print(isMainThread())
    }
}

func asyncFunc() async {
    print("AsyncFunc")
}

// true
// AsyncFunc
// true
```

하지만 `detached` 를 사용했을 경우에는 역시 상속받지 않는다.

```swift
@MainActor
func execute() {
    Task.detached {
        print(isMainThread())
        await asyncFunc()
        print(isMainThread())
    }
}

// false
// AsyncFunc
// false
```

---

### weak self 는 필수인가?

그렇지 않다. 일반적인 경우에 작업이 끝나는 즉시 내부 클로저가 소멸하기 때문에 클로저 내부에서 캡처된 참조도 해제된다. 하지만 `self` 가 사라진 경우에 작업을 취소하고 중지하고 싶다면 다음과 같이 사용할 수는 있다.

```swift
func start() {
    task = Task { [weak self] in
        for i in 0..<100 {
            guard let self else { return }
            try? await Task.sleep(for: .seconds(1))
            print("✅ 작업 실행: \(i)")
        }
    }
}
```
