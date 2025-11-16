---
layout: post
title: "[iOS] 사진 알림 화면 재귀 락 크래시 분석 보고서"
subtitle: "ErrorCase #1 · 메인 스레드에서 발생한 재귀 락 강제 종료"
date: 2025-11-15
categories: iOS Swift Combine Concurrency Debugging
author: raykim
author_url: https://github.com/raykim2414
---

# 사진 알림 화면 재귀 락 크래시 분석 보고서

> **ErrorCase #1 · 메인 스레드에서 발생한 재귀 락 강제 종료**

## TL;DR

- 좋아요 버튼을 누르는 순간, 드물게 `EXC_BREAKPOINT (SIGTRAP)`이 발생하고 `FOUNDATION 1` 코드로 앱이 종료됐다.
- 커스텀 구독 해제 컨테이너와 `AnyCancellable`이 동일한 `_os_unfair_lock`을 재귀적으로 잡으면서 시스템이 안전장치로 프로세스를 종료했다.
- 구독을 약한 참조로 바꾸고, 해제 순서를 명시하며, completion 처리를 통합한 뒤 문제를 재현하지 못했다.

## 배경

사진과 영상을 빠르게 확인하는 알림형 화면이 있다. 이 화면은 여러 API 응답을 Combine 스트림으로 받아 UI를 갱신한다. 우리는 여러 구독을 한 번에 정리하기 위해 커스텀 “구독 해제 컨테이너(CancelBag)”를 만들어 두었고, 화면이 내려갈 때 이 컨테이너가 모든 구독을 자동으로 해제해 줄 것이라고 가정했다.

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

## 해결 방법

### 1. 구독에서 뷰모델 참조를 약하게 유지

```swift
client.events
    .receive(on: DispatchQueue.main)
    .sink { [weak self] completion in
        self?.log(completion)
    } receiveValue: { [weak self] value in
        guard let self else { return }
        self.handle(value)
    }
    .store(in: &cancellables)
```

뷰모델이 해제되는 도중에도 콜백이 들어오면 조용히 무시되므로, 런루프 작업과 소멸 시점이 충돌하지 않는다. 앞서 든 회전문 비유로 보면, 먼저 들어간 사람이 완전히 빠져나가기 전에 뒤따라오는 사람을 문 밖에서 기다리게 하는 셈이다.

### 2. 해제 순서 정의

```swift
deinit {
    cancellables.forEach { $0.cancel() }
    cancellables.removeAll()
    cancelBag.cancelAll()
}
```

먼저 `AnyCancellable`을 정리해 Combine 내부 락을 해제한 뒤, 마지막에 구독 해제 컨테이너를 닫도록 순서를 고정했다. 이제 동시에 락을 잡는 주체가 존재하지 않는다. 회전문 안에 한 명만 존재하도록 순번을 강제해, 같은 칸으로 두 명이 동시에 진입할 가능성을 제거한 것이다.

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

모든 구독에서 동일한 completion 경로를 사용하면, 뷰모델이 중간에 사라져도 로그와 정리 로직이 일관되게 실행된다. 즉, 회전문을 빠져나가는 동선이 하나뿐이어서, 누가 먼저 나가든 같은 절차를 거치며 문이 다시 잠기지 않는다.

### 조치별로 재귀 락이 차단되는 지점

- **약한 참조**: Step 2에서 예약된 cleanup 작업이 실행될 때 이미 `self`가 해제돼 있다면, 콜백이 조용히 return 하므로 Step 3~4로 진행하지 않는다.
- **해제 순서**: Step 1에서 `Set<AnyCancellable>`를 먼저 비워 Step 4의 “두 번째 cancel 호출” 자체를 제거한다.
- **공통 completion 경로**: 모든 구독이 동일한 종료 루틴을 거치므로, Step 2에서 어떤 구독이 먼저 끝나더라도 cleanup가 중복 호출되지 않는다.

## 초기 구현이 그렇게 된 이유 (추정)

- **구현 속도 우선**: 당시 화면이 내부 QA 용도라 “모든 구독을 컨테이너에 넣고 한 번에 취소한다”는 단순 패턴을 택했다.

## 얻은 교훈

1. **비동기 해제 순서는 명시하자**: “언젠가 컨테이너가 취소해주겠지”라는 추상적인 가정은 교착 상태로 이어질 수 있다.
2. **약한 참조를 기본값으로 생각하자**: UI 오브젝트를 다루는 Combine 구독은 기본적으로 `[weak self]`를 붙이고, 정말 필요한 경우에만 강한 참조를 허용해야 한다.


