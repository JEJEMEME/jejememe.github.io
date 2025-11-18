---
layout: post
title: "[iOS] PhotoBook Preview 메모리 안정화 기록"
subtitle: "Error Case #5 · OOM, Alpha Flush, 그리고 렌더링 파이프라인 재편"
date: 2025-11-18
categories: iOS Swift SwiftUI Debugging
author: raykim
author_url: https://github.com/raykim2414
---

# PhotoBook Preview 메모리 안정화 기록

> **Error Case #5 · OOM, Alpha Flush, 그리고 렌더링 파이프라인 재편**

## TL;DR

- 주문 버튼을 누르는 순간 4K급 페이지들을 한꺼번에 렌더·JPEG화하면서 메모리가 3GB까지 치솟아 앱이 즉사했다.
- `renderedOpaque()`를 단계마다 반복 호출해 같은 이미지를 세 번씩 다시 그리다 보니 CPU/메모리 낭비가 컸고, JPEG 결과도 매번 달라졌다.
- 모든 렌더링을 배치 단위(`3장씩`)로 쪼개고, 렌더 직후 `malloc_zone_pressure_relief`를 호출해 힙을 비웠다.
- 알파 제거는 최종 저장 단계(`saveGoods`)에서만 한 번 수행하도록 통합해 결과물을 메인과 동일하게 맞췄다.

## 시나리오 & 재현 조건

- **환경**: iOS 18.x, SwiftUI + UIKit 혼합 레이어, 실기기(iPhone 15 Pro, iPad 10세대).
- **데이터 세트**: 80~100페이지 프로젝트, 각 페이지는 2500×1815px 수준의 최종 캔버스를 생성함.
- **재현 절차**
  1. `PBPreview`에서 전체 페이지를 빠르게 넘긴다(프리뷰 캐시가 가득 찬 상태).
  2. “제작 주문”을 누른 뒤 Instruments → Memory Graph로 관찰한다.
  3. `[PB] pb_N_page success` 로그가 연속으로 찍히다가 곧바로 OOM.
- **관찰**: `resident size`가 2.8~3.0GB에서 정점, error log에는 `writeImageAtIndex:1086 ... AlphaPremulLast`가 반복적으로 나타난다.

## 문제 관찰

1. 렌더 루프는 **페이지 수만큼 동기 실행**하며, 한 장 완성될 때까지 UI 스레드를 점유했다.
2. 페이지 한 장당 `ImageRenderer → renderedOpaque → saveGoods(renderedOpaque)`의 삼중 렌더를 거치면서 메모리가 세 배로 불었다.
3. `MemoryPressureHandler` 같은 방어 코드가 없어서 ARC가 참조를 해제해도 시스템이 즉시 메모리를 회수하지 못했다.
4. 빈 페이지/에필로그 같은 보조 섹션도 동일 방식으로 처리되어 누수 구간이 한 곳이 아니었다.

## 원인 분석

### 1. 중복 렌더링

```swift
// before
let sanitized = generatedImage.renderedOpaque(.white)       // 1차
let merged = merge(imgA: left, imgB: right)?.renderedOpaque // 2차
let flattened = image.renderedOpaque(.white)                // 3차
```

- 같은 비트맵을 매 단계마다 다시 그려서 CPU/GPU가 반복적으로 소모되었다.
- JPEG 압축 전에 픽셀이 여러 번 변하면서 파일 크기와 색상이 조금씩 달랐다.

### 2. 렌더링 배치 부재

- `snapShot()`는 커버→프롤로그→모든 페이지→에필로그를 **연속 for-loop**로 돌렸다.
- SwiftUI `ImageRenderer`는 내부적으로 CoreAnimation 리소스를 잡는데, `autoreleasepool` 없이 수십 번 생성하니 누적되었다.

### 3. 알파 경고

- 투명 픽셀이 남은 채 `jpegData()`를 만들면 CoreGraphics가 `AlphaPremulLast` 경고를 내고, 디코딩 시 메모리가 2배 필요하다.
- 각 단계에서 흰 배경을 깔아도, 중복 렌더링 탓에 여전히 경고가 남았다(한 번 더 flatten할 때 alpha가 되살아남).

## 해결 과정

### 1. 렌더링 파이프라인 재편

```swift
func snapShot() async throws {
    try await processPagesInBatches(batchSize: 3)
    if odd { try await processEmptyPage() }
    try await processEpilogue()
}
```

- 페이지를 3장씩 처리하도록 `processPagesInBatches`를 추가했다.
- 배치가 끝날 때마다 `MemoryPressureHandler.relieve()`와 `Task.yield()`를 호출해 힙을 강제로 비웠다.
- 빈 페이지, 에필로그에도 동일한 패턴을 적용했다.

### 2. `View.makeImage` 내부에 `autoreleasepool` 배치

```swift
public func makeImage(...) async -> UIImage {
    return autoreleasepool {
        let renderer = ImageRenderer(content: content)
        renderer.isOpaque = true
        return renderer.uiImage ?? UIImage()
    }
}
```

- 각 `ImageRenderer`가 scope를 벗어나자마자 해제되도록 강제했다.
- SwiftUI 뷰를 고해상도로 여러 번 렌더해도 메모리가 누적되지 않는다.

### 3. 알파 제거 위치 단일화

```swift
func saveGoods(...) -> Result<Bool, Error> {
    let flattened = image.renderedOpaque(backgroundColor: .white)
    guard let jpeg = flattened.jpegData(compressionQuality: 1.0) else { ... }
    try data.write(to: url)
}
```

- `loadPageImage`, `mergeAndSave`, `processImageAndSave`에서는 더 이상 `renderedOpaque`를 호출하지 않는다.
- 최종 저장 시 한 번만 flattened 하므로 JPEG 결과가 안정적이고, `AlphaPremulLast` 경고도 사라졌다.

### 4. 메모리 압박 완화 헬퍼

```swift
enum MemoryPressureHandler {
    static func relieve() {
        #if canImport(Darwin)
        malloc_zone_pressure_relief(nil, 0)
        #endif
    }
}
```

- 렌더링/병합/저장 직후마다 호출해 arc가 풀어둔 메모리를 커널이 바로 회수하도록 했다.


## 얻은 교훈

1. **렌더링은 반드시 배치 단위**로 나눠야 한다. “for-loop 한 번으로 100장을 렌더”하는 패턴은 언제든지 OOM을 낳는다.
2. **알파 제거 시점은 단일화**할 것. flatten을 여러 번 하면 성능이 떨어지고, JPEG 일관성도 깨진다.
3. **SwiftUI Renderer는 autoreleasepool로 감싸라.** 렌더러가 CG 리소스를 잡고 있어서 스코프를 벗어나도 바로 해제되지 않는다.
4. **MemoryPressureHandler 같은 안전장치는 필수**다. ARC가 참조를 끊었다고 해서 커널이 즉시 메모리를 돌려주는 건 아니다.

이번 조치로 PhotoBook 제작 플로우는 “100페이지 × 4K 렌더” 시나리오에서도 안정적으로 동작한다. 다음 단계는 ZIP 압축/업로드를 백그라운드 큐로 완전히 옮기는 것이다.


