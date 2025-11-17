---
layout: post
title: "[iOS] MediaPaging 좌우 스와이프 튐 현상 기록"
subtitle: "Error Case #4 · Timeline Sync Race & Pager State Drift"
date: 2025-11-17
categories: iOS Swift SwiftUI Debugging
author: raykim
author_url: https://github.com/raykim2414
---

# MediaPaging 좌우 스와이프 튐 현상 기록

> **Error Case #4 · Timeline Sync Race & Pager State Drift**

## TL;DR

- DetailDayView에서 `MediaPagingView`로 진입하자마자 백그라운드 정렬 응답이 오면 `page` 상태를 PID 위치로 즉시 덮어썼다.
- 사용자가 로딩 전 좌우 스와이프나 버튼을 입력하면 응답 도착 시 `page`가 다시 되돌아가 화면이 다른 사진으로 튀었다.
- “사용자 입력”과 “프로그램틱 전환”을 같은 상태 경로로 처리해 레이스가 발생했다.
- 초기 PID 포커싱은 한 번만 허용하고, 강제 전환/사용자 전환을 플래그로 분리해 이후 이벤트를 차단했다.
- **재현 한 줄 요약**: DetailDayView에서 넘어온 직후, 로딩이 끝나기 전에 좌우 스와이프(또는 내비 버튼)를 연타하면 Timeline 응답과 충돌한다.

## 배경

- 광고 삽입으로 인해 Pager의 실제 페이지(`page`)와 미디어 인덱스(`mediaIndex`)를 따로 관리한다.
- 타임라인 알림으로 진입하면 전달받은 `pid`를 포커싱해야 한다.
- 데이터 로딩 완료 신호를 받으면 `self.page`를 즉시 PID 위치로 덮어쓰도록 구현되어 있었고, 사용자 상호작용 여부를 고려하지 않았다.

## 원인 분석

### 1. 프로그램틱/사용자 전환 구분 부재

```
.onReceive(viewModel.isTimelineMediaLoaded) { loaded in
    if loaded {
        page = pidIndex
    }
}
```

- 로딩 완료 시마다 PID 위치로 무조건 이동했다.
- 이미 사용자가 페이지를 옮겼는지, 초기 전환이 끝났는지에 대한 체크가 없었다.

### 2. `page` 변경 출처 추적 실패

```
.onChange(of: page) { newValue in
    viewModel.changePage(page: newValue, updateCount: true)
    mediaIndex = newValue - viewModel.getPreviousAdCount(in: newValue)
}
```

- 모든 경로에서 동일한 `.onChange`가 실행돼 Timeline 초기화와 실제 스와이프를 구분하지 못했다.
- “사용자 입력 → PID 덮어쓰기 → `.onChange` → 다시 사용자 입력” 루프가 반복됐다.

### 3. 수동 내비·뷰어 복귀에도 플래그 부재

- 좌/우 버튼, 전체 화면 뷰어 exit 모두 `page`를 직접 조정하지만 상호작용 여부를 기록하지 않았다.
- 버튼 입력이 느린 기기에서는 서버 응답이 먼저 도착해 버튼 입력이 무시되는 것처럼 보였다.

## 해결 과정

### 1. 상태 플래그 도입

```
@State private var hasUserInteractedWithPager = false
@State private var isProgrammaticPageChange = false
@State private var hasAppliedTimelineInitialPage = false
```

- 사용자 입력 여부, 코드가 강제로 바꿨는지, Timeline 초기 이동을 이미 처리했는지를 개별 플래그로 유지했다.
- 모든 플래그는 `@State` 범위에만 존재하며 외부로 노출되지 않는다.

| 상태 플래그 | true 의미 | false 의미 | 리셋 타이밍 |
|-------------|-----------|------------|-------------|
| `hasUserInteractedWithPager` | 사용자가 스와이프/버튼 등으로 직접 페이지를 바꿈 | 아직 프로그램틱 이동만 발생 | 사용자 입력 직후 set, 뷰 초기화 시 초기화 |
| `isProgrammaticPageChange` | 다음 `.onChange`는 코드가 강제한 이동 | 사용자 입력에서 발생한 이동 | `.onChange`에서 프로그램틱 이동 처리 후 false |
| `hasAppliedTimelineInitialPage` | Timeline PID 포커싱을 이미 한 번 실행 | 아직 PID 이동을 수행하지 않음 | PID 이동 직후 true, 뷰 재생성 시 초기화 |

### 2. 타임라인 응답 가드

```
.onReceive(viewModel.isTimelineMediaLoaded) { loaded in
    guard loaded,
          let pid = viewModel.pid,
          viewModel.timelineID != nil,
          !hasUserInteractedWithPager,
          !hasAppliedTimelineInitialPage,
          !viewModel.medias.isEmpty
    else { return }

    let targetIndex = viewModel.medias.firstIndex { $0.id == pid } ?? 0
    hasAppliedTimelineInitialPage = true
    isProgrammaticPageChange = true
    page = targetIndex
}
```

- 사용자 입력 이후에는 더 이상 PID 강제 이동을 허용하지 않는다.
- PID를 찾지 못하면 0번으로 폴백해 비어있는 배열에서도 안전하다.

### 3. `onChange`에서 출처 분기

```
.onChange(of: page) { newValue in
    viewModel.changePage(page: newValue, updateCount: true)
    mediaIndex = newValue - viewModel.getPreviousAdCount(in: newValue)

    if isProgrammaticPageChange {
        isProgrammaticPageChange = false
    } else {
        hasUserInteractedWithPager = true
    }
}
```

- 코드가 강제로 바꾼 경우에는 플래그만 리셋하고, 나머지는 사용자 상호작용으로 기록한다.
- 이후 Timeline 응답이 도착해도 guard 조건에서 걸러진다.

### 4. 수동 내비/뷰어 복귀 보완

```
Button {
    hasUserInteractedWithPager = true
    page -= 1
}

Button {
    hasUserInteractedWithPager = true
    page += 1
}

onDismiss: { lastIndex, _ in
    isProgrammaticPageChange = true
    page = lastIndex
}
```

- 버튼 입력 즉시 사용자 플래그를 세워 Timeline 응답이 더 이상 개입하지 못하게 했다.
- 전체 화면 뷰어에서 닫을 때는 프로그램틱 전환으로 표시해 `.onChange`에서 불필요한 사용자 이벤트로 기록되지 않게 했다.

## 결과

- 타임라인 진입 직후 스와이프/버튼 입력을 반복해도 외부 응답이 `page`를 다시 덮어쓰지 않는다.
- PID를 찾지 못해도 0번으로 안전하게 폴백하여 크래시나 빈 화면 없이 진행된다.
- 수동 버튼 및 뷰어 복귀 시에도 플래그가 정확히 동작해 다시 튀는 현상이 재발하지 않았다.
- QA 재현 영상과 로그 모두에서 “스와이프 중 갑자기 다른 사진으로 이동” 현상이 사라졌다.

## 보안 점검

- 새로 추가된 상태 플래그는 로컬 메모리에서만 관리되며, 네트워크·로그로 전파되지 않는다.
- PID와 타임라인 ID는 기존과 동일하게 서버 응답 내에서만 사용되고, 로그에는 난독화된 식별자만 남는다.
- 광고, 결제, 민감 데이터 경로에는 변화가 없으며 외부 SDK 호출이나 디버그 정보 노출도 발생하지 않는다.

## 얻은 교훈

1. **사용자 입력과 프로그램틱 입력을 명시적으로 분리**해야 SwiftUI 상태 레이스를 예방할 수 있다.
2. **비동기 응답으로 초기 상태를 덮어쓸 때는 “이미 처리했는지”를 기록**해야 불필요한 반복 전환을 막을 수 있다.
3. **Pager와 같이 파생 상태가 많은 뷰는 라운드트립 경로(내비 버튼, 뷰어, PID 복귀)를 모두 커버**해야 한다.
4. **로그와 상태 변수는 재현에 필요한 최소한의 정보만 남겨야** 하며 보안상 불필요한 세부 사항은 제거하는 편이 안전하다.

이번 패치로 MediaPaging은 “사용자가 상호작용 중일 때 외부 이벤트가 페이지를 강제로 바꾸지 않는다”는 전제를 다시 충족한다. 다음 단계는 Pager 컴포넌트 레벨에서 사용자/프로그램틱 전환 이벤트를 더 정교하게 분리해 뷰모델 단위 플래그 의존도를 줄이는 것이다.

