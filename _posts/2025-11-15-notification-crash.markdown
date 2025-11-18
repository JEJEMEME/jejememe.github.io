---
layout: post
title: "[iOS] 사진 알림 화면 재귀 락 크래시 분석 보고서"
subtitle: "ErrorCase #1 · 메인 스레드에서 발생한 재귀 락 강제 종료"
date: 2025-11-15
categories: iOS Swift Combine Concurrency Debugging
author: raykim
author_url: https://github.com/raykim2414
---

# Alert Image Timeline Crash Analysis

> **이슈**: 타임라인 CTA를 누른 뒤 알림 이미지 화면을 닫을 때 메인 스레드에서 `_os_unfair_lock_recursive_abort`가 발생하며 프로세스가 종료됐다.
>
> **ErrorCase #1 · 메인 스레드에서 발생한 재귀 락 강제 종료**

## TL;DR

- 크래시는 `EXC_BREAKPOINT (SIGKILL)` + termination reason `FOUNDATION 1` 형태로 재현됐고, 메인 스레드 `AnyCancellable.cancel()` 경로에서 `_os_unfair_lock_recursive_abort`가 확인됐다.
- `Set<AnyCancellable>`와 `CancelBag` 두 컨테이너가 같은 구독을 동시에 해제하는 동안 `receive(on: DispatchQueue.main)` 콜백이 여전히 실행 중이라 Combine 내부 락에 재귀 진입했다.
- 모든 구독을 단일 컨테이너로 모으고, 해제를 다음 메인 큐 턴으로 미루며, completion 처리를 표준화하자 동일 시나리오에서 더 이상 크래시가 발생하지 않았다.

## 배경

알림 이미지/타임라인 화면은 라우팅 상태 바인딩, API 기반 미디어 페치, 좋아요·CTA 같은 사용자 액션 등 여러 Combine 파이프라인에 의존한다. 초기 구현에서는 구독 저장 전략을 두 갈래로 나눠 썼다. UI·미디어 스트림은 표준 `Set<AnyCancellable>`에 담고, 라우팅·헬퍼 스트림은 DSL 형태의 `cancelBag.collect { ... }`로 감싼 뒤 `CancelBag` 내부에 저장했다. 뷰모델이 해제되면 두 컨테이너가 거의 동시에 구독을 정리하면서도 서로 영향을 주지 않을 것이라 가정한 셈이다.

문제는 같은 구독이 두 컨테이너에 중복 저장될 수 있었고, 각각이 메인 큐에서 synchronous `cancel()`을 호출하면서 `_os_unfair_lock`에 재귀 진입이 일어났다는 점이다. 아래 코드는 당시 구조를 단순화한 예시다. 

```swift
final class NotificationViewModel {
    private let cancelBag = CancelBag()
    private var cancellables: Set<AnyCancellable> = []

    init(client: APIClient) {
        client.events
            .receive(on: DispatchQueue.main)
            .sink { completion in
                log(completion)                // 오류 로깅만 수행
            } receiveValue: { value in
                self.handle(value)             // self를 강하게 참조
            }
            .store(in: &cancellables)          // Set<AnyCancellable>에만 저장

        cancelBag.collect {
            $routingState.sink { ... }         // 이 역시 self를 강하게 참조
        }
    }

    deinit {
        cancelBag.cancelAll()                  // 내부에서 락을 잡고 전체를 취소
        // Set<AnyCancellable>는 deinit까지 유지
    }
}
```

### CancelBag이 해제를 다루는 방식

```swift
// CancelBag.swift
final class CancelBag {
    private var subscriptions = Set<AnyCancellable>()
    private let lock = NSLock()

    private func withLock<T>(_ body: () -> T) -> T {
        lock.lock()
        defer { lock.unlock() }
        return body()
    }

    func collect(@Builder _ cancellables: () -> [AnyCancellable]) {
        let newSubscriptions = cancellables()
        withLock {
            subscriptions.formUnion(newSubscriptions)
        }
    }

    func cancelAll() {
        cancelAndRemoveAll()
    }

    func cancelAndRemoveAll() {
        let currentSubscriptions: [AnyCancellable] = withLock {
            let current = subscriptions
            subscriptions.removeAll()
            return Array(current)
        }
        currentSubscriptions.forEach { $0.cancel() }   // 락 밖에서 cancel
    }
}
```

- `collect`는 `@resultBuilder`를 사용해 DSL처럼 여러 구독을 선언적으로 모은 뒤, 내부 `Set`에 저장한다.
- `cancelAndRemoveAll`은 락을 잡고 현재 구독을 복사한 뒤, 락을 풀고 순차적으로 `cancel()`을 호출한다. 즉, “모든 구독을 한 번에 정리해 주는 상자” 역할을 한다.

위 구현은 안전한 편이지만, 아래처럼 락을 잡은 채로 `cancel()`을 직접 호출하는 변형은 재귀 락을 매우 쉽게 유발한다.

```swift
final class NaiveCancelBag {
    private var cancellables: [AnyCancellable] = []
    private let lock = NSLock()

    func collect(_ builder: () -> AnyCancellable) {
        lock.lock()
        cancellables.append(builder())
        lock.unlock()
    }

    func cancelAll() {
        lock.lock()
        cancellables.forEach { $0.cancel() }   // cancel 안에서 같은 락을 다시 잡을 수 있다
        lock.unlock()
    }
}
```

`cancel()` 내부가 동일한 `NSLock` 혹은 `os_unfair_lock`을 잡으면, 락을 쥔 채로 다시 같은 락을 요구하는 “재귀 진입”이 된다. 이번 이슈는 이러한 구조가 Combine 내부 구현과 맞물리며 밖으로 드러난 사례다.

### `cancelBag.collect { ... }`가 의미하는 것

```swift
cancelBag.collect {
    $routingState.sink { state in
        // state 변화에 따라 UI 라우팅
    }
    viewEventPublisher.sink { event in
        handle(event)
    }
}
```

- `collect` 블록 안에서 만든 모든 `sink`가 `CancelBag` 내부 `Set`에 저장된다.
- 각각의 `sink`는 다시 뷰모델을 강하게 붙잡고 있으므로, 컨테이너가 해제되기 전까지 뷰모델도 함께 살아 있게 된다.

## Root Cause

1. 타임라인 CTA 액션이 여전히 `receive(on: DispatchQueue.main)` 파이프라인을 따라 흐르는 동안 뷰모델이 해제되었다.
2. `deinit`이 즉시 `cancelBag.cancelAll()`을 호출해 같은 큐에서 모든 구독을 취소하려 했다.
3. 해당 구독 중 일부는 `Set<AnyCancellable>`에도 저장돼 있었고, `cancel()` 과정에서 Combine이 추가 정리 작업을 다시 메인 큐에 예약했다.
4. 예약된 정리가 끝나기 전에 `Set<AnyCancellable>` 역시 소멸되며 동일한 구독에 대해 두 번째 `cancel()`이 호출되었다.
5. `os_unfair_lock`은 재진입을 허용하지 않으므로 Combine 내부 락이 두 번째 진입을 감지하고 `_os_unfair_lock_recursive_abort`를 발생시켰다.

즉, 중복 저장된 구독을 두 컨테이너가 같은 큐에서 동기 취소하면서 `receive(on:)` 콜백과 버무려져 메인 스레드 락 재진입이 일어난 셈이다.

## 문제 관찰

TestFlight 사용자에게서 “좋아요 버튼을 누르자마자 앱이 꺼졌다”는 제보가 들어왔다. 크래시 리포트에는 `EXC_BREAKPOINT (SIGTRAP)`와 함께 `FOUNDATION 1` 이유 코드가 기록됐고, 스택 최상단에는 `_os_unfair_lock_recursive_abort`가 있었다. 메인 스레드에서 좋아요 응답을 처리하는 Combine 스트림이 해제되는 동안 동일한 경로로 다시 진입한 것이다.

뷰모델 해제 시점에 `receive(on:)`가 메인 스레드에 예약해 둔 작업이 남아 있으면, 구독 해제 컨테이너와 `AnyCancellable`이 서로 락을 잡고 재진입하게 된다. 이때 `os_unfair_lock`은 재귀 호출을 허용하지 않으므로, 시스템이 즉시 프로세스를 중단한다.

조금 더 풀어서 설명하면, 한 줄로 된 회전문을 생각하면 이해하기 쉽다. 회전문 한 칸을 누군가가 잡은 채 아직 빠져나가지 않았는데, 뒤에 있던 다른 사람이 같은 칸으로 억지로 들어오려 하면 문이 멈추면서 경보가 울린다. 여기서 회전문은 `os_unfair_lock`, 먼저 들어온 사람은 구독 해제 컨테이너, 뒤따라 억지로 들어오려 한 사람은 `AnyCancellable`이다. 둘 다 같은 칸(락)을 잡으려다 시스템이 위험하다고 판단하고 문 전체를 잠가 버린 것이 이번 크래시다.

### 재귀 락이 실제로 발생한 순서

1. `NotificationViewModel.deinit` → `cancelBag.cancelAll()` 호출. 컨테이너가 락을 잡고 내부 구독을 순회하기 시작한다.
2. 컨테이너 안에 있던 어떤 `sink`가 `cancel()`되면서 Combine의 `ReceiveOn` 연산자가 cleanup 코드를 메인 큐에 예약한다.
3. 예약된 cleanup 작업이 즉시 실행되며, 동일한 `AnyCancellable` 인스턴스가 `Set<AnyCancellable>`에서도 해제된다.
4. `Set<AnyCancellable>`이 deinit 단계에서 다시 `cancel()`을 호출해 Combine 내부 락을 재차 잡으려 한다.
5. 앞선 1단계에서 이미 락이 잡혀 있으므로 `_os_unfair_lock_recursive_abort`가 트리거되고, 시스템이 프로세스를 종료한다.

### 실제 스택 트레이스 (간략화)

```
Thread 0 Crashed:
0   libsystem_platform.dylib      _os_unfair_lock_recursive_abort
1   libsystem_platform.dylib      _os_unfair_lock_lock_slow
2   Combine                       AnyCancellable.cancel()
3   SSCK3                         CancelBag.cancelAll()
4   SSCK3                         NotificationViewModel.deinit
5   libswiftCore.dylib            swift_deallocClassInstance
```

### 왜 구독이 두 컬렉션에 나뉘어 있었나?

- UI 이벤트를 다루는 “주요 스트림”은 `Set<AnyCancellable>`에 넣어 표준 Combine 패턴대로 관리했다.
- 반면 라우팅이나 일시적인 상태 바인딩은 DSL 형태(`cancelBag.collect`)로 묶어두는 것이 편해 보여 별도의 컨테이너에 넣었다.
- 결과적으로 동일한 구독이 두 컬렉션에 중복 저장되는 경우가 생겼고, “컨테이너가 지워줄 것”이라는 가정 때문에 이 중복을 정리하지 않은 채 출시되었다.

## 원인 추적 과정

1. **락 사용 지점 재검토**  
   화면 코드 자체에서는 별도의 락을 사용하지 않았고, Combine 내부가 잡는 `os_unfair_lock`만 남아 있었다.

2. **구독 해제 타이밍 점검**  
   뷰모델 `deinit` 시점에 구독 해제 컨테이너가 먼저 모든 구독을 끊고, 이후 `Set<AnyCancellable>`이 자연스럽게 해제되도록 두었다. 두 경로가 서로 다른 컬렉션을 관리하지만, 결국 동일한 Combine 구독을 동시에 취소한다는 사실을 간과했다.

3. **스택 재구성**  
   `ReceiveOn` → `Sink` → `AnyCancellable.deinit` 순서가 두 번 반복되면서 `_os_unfair_lock_recursive_abort`가 호출되었고, 메인 스레드 런루프가 같은 블록을 재실행하는 형태였음을 확인했다.

## Fix Summary

1. **구독 저장소 단일화**: `AlertImageView.ViewModel` 내부 모든 구독을 `CancelBag` 한 곳에만 모아 어느 컨테이너가 언제 해제하는지 모호하지 않게 했다.
2. **지연 취소 헬퍼 추가**: `cancelDeferred(on:)`를 도입해 락을 잡은 채 바로 `cancel()`하지 않고, 다음 메인 큐 턴에서 안전하게 구독을 끊도록 했다.
3. **완료 처리·약한 캡처 표준화**: 모든 `sink`가 동일 completion 함수와 `[weak self]` 패턴을 사용하도록 정리해, 늦게 도착한 콜백이 중복 정리를 유발하지 않게 했다.

아래는 각 조치에 대한 구현 세부다.

## 해결 방법

### 1. 단일 CancelBag + 약한 캡처 기본화

```swift
final class AlertImageViewModel {
    private let cancelBag = CancelBag()

    init(client: APIClient, router: Router) {
        client.events
            .receive(on: DispatchQueue.main)
            .sink { [weak self] completion in
                self?.handle(completion)
            } receiveValue: { [weak self] value in
                self?.handle(value)
            }
            .store(in: cancelBag)

        cancelBag.collect {
            router.timelineCTA
                .sink { [weak self] action in
                    self?.route(action)
                }
        }
    }
}
```

`Set<AnyCancellable>`를 완전히 제거하고 `CancelBag` 한 곳에만 구독을 저장한다. 모든 `sink`가 `[weak self]`로 캡처하므로, 뷰모델 해제 도중 들어온 콜백은 자연스럽게 무시되고 동일 구독이 다른 컨테이너로 흘러드는 일도 없다.

### 2. `cancelDeferred(on:)` 헬퍼 도입

```swift
extension CancelBag {
    func cancelDeferred(on queue: DispatchQueue = .main) {
        let snapshot: [AnyCancellable] = withLock {
            let current = subscriptions
            subscriptions.removeAll()
            return Array(current)
        }

        queue.async {
            snapshot.forEach { $0.cancel() }
        }
    }
}

deinit {
    cancelBag.cancelDeferred()
}
```

락을 잡은 상태에서는 구독 목록만 복사하고, 실제 `cancel()` 호출은 다음 메인 큐 턴으로 미룬다. `receive(on:)`이 아직 같은 런루프에서 실행 중이더라도 콜백이 끝날 시간을 벌 수 있어, `_os_unfair_lock`이 재귀로 잠기는 상황을 피할 수 있다.

### 3. 완료 처리 분리

```swift
private func handle(completion: Subscribers.Completion<Error>) {
    switch completion {
    case .finished:
        logSuccess()
    case .failure(let error):
        logFailure(error)
    }
}
```

모든 구독이 동일한 completion 함수와 `[weak self]` 패턴을 사용하면, 늦게 들어온 이벤트라도 같은 정리 루틴을 거쳐 idempotent하게 종료된다. 회전문을 빠져나가는 동선이 하나뿐이어서, 누가 먼저 나가든 같은 절차를 거치며 문이 다시 잠기지 않는다.

### 조치별로 재귀 락이 차단되는 지점

- **단일 CancelBag + 약한 캡처**: 뷰모델이 해제되면 추가 콜백이 바로 drop되어 중복 취소 경로가 열리지 않는다.
- **cancelDeferred(on:)**: 락을 잡은 채 바로 `cancel()`하지 않아 `receive(on:)` 콜백이 동일 락에 재진입할 기회가 없다.
- **공통 completion 경로**: 어떤 구독이 먼저 종료되더라도 동일한 루틴을 타므로 cleanup가 중복 실행돼도 부작용이 없다.

## Verification

- “알림 이미지 진입 → 타임라인 CTA 탭 → 화면 닫기” 플로우를 iOS 17/18 시뮬레이터와 실제 QA 기기에서 수십 회 반복했지만 `_os_unfair_lock_recursive_abort`가 재현되지 않았다.
- 동일 빌드 Cohort의 크래시 로그를 다시 수집한 결과, 패치 이후에는 `FOUNDATION 1`/`EXC_BREAKPOINT` 레코드가 더 이상 보고되지 않았다.

## 초기 구현이 그렇게 된 이유 (추정)

- **구현 속도 우선**: 당시 화면이 내부 QA 용도라 “모든 구독을 컨테이너에 넣고 한 번에 취소한다”는 단순 패턴을 택했다.

## Takeaways

1. **뷰모델당 취소 컨테이너는 하나로 통일**해 어느 경로에서 구독이 해제되는지 항상 추적 가능하게 유지하자.
2. **`receive(on:)`과 해제가 같은 큐에서 맞부딪칠 수 있으면 취소를 다음 큐 턴에 위임**해 런루프가 락을 풀 시간을 벌어주자.
3. **라우팅/상태 바인딩 구독을 외부로 중복 노출하지 말고 캡슐화**해, 동일 구독이 여러 소유자에게 흩어지는 상황을 원천 차단하자.


