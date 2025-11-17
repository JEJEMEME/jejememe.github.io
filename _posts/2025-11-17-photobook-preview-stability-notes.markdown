---
layout: post
title: "[iOS] PhotoBook Preview 메모리 안정화 기록"
subtitle: "Error Case #3 · OOM, Cache Thrash, 그리고 슬라이더 무력화"
date: 2025-11-17
categories: iOS Swift SwiftUI Debugging
author: raykim
author_url: https://github.com/raykim2414
---

# PhotoBook Preview 메모리 안정화 기록

> **Error Case #3 · OOM, Cache Thrash, 그리고 슬라이더 무력화**

## TL;DR

- 동기식 전체 페이지 렌더 + 압축이 돌아가는 동안 공용 `imageCache`가 수백 MB까지 팽창해 OOM을 촉발했다.
- 프리뷰/제작 모드가 같은 캐시를 공유하면서 “렌더 중 다시 캐시에 적재” 패턴이 무한 반복됐다.
- 주문 플로우 진입 시 캐시를 명시적으로 비우고, 제작 단계는 디스크 직독 경로로 분리해 캐시 thrash를 차단했다.
- 슬라이더는 비활성화된 채 장식만 담당하고 있어 디버깅 중 UX 시나리오 자체가 실패했다. 상호작용을 복원했다.

## 시나리오 & 재현 조건

- **환경**: iOS 18.x, SwiftUI + UIKit 혼합 뷰, 4GB RAM급 디바이스(아이패드 9세대/아이폰 15 등)에서 재현.
- **데이터 세트**: 100페이지 프로젝트, 각 페이지가 12MP 내외 JPEG을 참조.
- **재현 절차**
  1. `PBPreview`에서 페이지를 전부 스크롤해 썸네일/페이지/커버 이미지를 캐시에 올린다.
  2. 바로 “제작 주문” 버튼을 눌러 동기 렌더 + ZIP 업로드 루틴을 시작한다.
  3. Instruments → Memory Graph / Allocations에서 `PBMakeViewModel.imageCache` 객체가 빠르게 수백 MB를 점유하고, 2~3회 GC 사이클 후 프로세스가 종료되는 것을 확인한다.
- **관찰 지표**: resident size 1.7~1.8GB, dirty memory 급증, OOM 로그에는 메모리 워닝 없이 jetsam reason `0x8badf00d`가 기록됐다.

## 배경: 왜 터졌나

- 사진 선택 → 페이지 편집 → 옵션 선택 → `PBPreview` 확인 → 주문이 기본 플로우다.
- 프리뷰 상태에서 이미 썸네일/페이지/커버 이미지가 메모리에 올라와 있다.
- 주문 단계에서 다시 모든 페이지를 풀 해상도로 그리는 순간 여유 메모리가 몇 MB만 남고, QA가 즐겨 쓰는 **100장 프로젝트**에서는 거의 즉사 조건이 됐다.

## 문제 관찰

- 주문 버튼을 누르면 `[PB] pb_XX_page success`만 찍히다가 경고 없이 곧바로 OOM 크래시가 발생했다.
- Instruments에서는 `PBMakeViewModel.imageCache`에 풀 해상도 이미지가 계속 쌓이고, 스냅샷 루프도 동일 이미지를 다시 캐시에 올려 메모리가 두 배로 뛰었다.
- 프리뷰 UI가 계속 남아 있어 SwiftUI body가 반복 계산되고, `loadImageSynchronously` 호출 때마다 캐시가 중복으로 채워졌다.
- 상단 슬라이더는 장식용이라 QA가 "드래그가 안 된다"고 버그를 열었다.

## 원인 분석

### 1. 캐시 재적재 루프

- `loadImageSynchronously`는 조건 없이 `cache(image:for:)`를 호출했다.
- 주문 버튼 → `viewModel.isMake = true` → SwiftUI body 리빌드 → 페이지별 이미지가 다시 로딩되며 그때마다 캐시에 새 항목이 추가됐다.
- 스냅샷 루프도 같은 이미지를 다시 캐시에 넣어 LRU가 끊임없이 축출/재적재를 반복하며 쓰레싱이 발생했다.

```swift
// PBMakeViewModel.swift
func loadImageSynchronously(url: String) -> UIImage {
    guard !url.isEmpty else { return UIImage() }
    let image = readImageFromDisk(url: url)
    cache(image: image, for: url)        // 항상 캐시에 적재
    return image
}

// PageSpreadView.swift
Image(uiImage: viewModel.loadImageSynchronously(url: page.localID))
    .resizable()
    .aspectRatio(contentMode: .fit)
```

- SwiftUI가 body를 다시 만들 때마다 위 `Image` 구문이 다시 실행되어 캐시 적재가 중복 발생했다.

### 2. 캐시 클리어 타이밍 부재

- 프리뷰 → 주문 사이에 캐시를 정리할 훅이 전혀 없었다.
- 로딩 뷰를 띄워도 실제 이미지 객체는 남아 있어 스냅샷 루프 직전에 메모리를 해제할 시간이 없었다.

```swift
// PBPreview.swift (이전)
Button("제작 주문") {
    viewModel.isMake = true          // 상태만 바꾸고
    makePhotobook()                  // 바로 스냅샷 루프 진입
}
```

- `imageCache`/`cacheOrder`를 초기화하는 호출이 없어, 주문 진입 직후에도 수백 MB가 그대로 남아 있었다.

### 3. 슬라이더 비활성화

- `SwiftUISlider.makeUIView`에서 `slider.isUserInteractionEnabled = false`로 고정되어 있었다.
- 값 표시만 가능한 상태라 사용자가 직접 페이지를 움직일 수 없었다.

```swift
func makeUIView(context: Context) -> UISlider {
    let slider = UISlider()
    slider.minimumValue = 0
    slider.maximumValue = Float(totalPageCount)
    slider.isUserInteractionEnabled = false   // 장식 전용
    return slider
}
```

- QA가 페이지 점프를 요청해도 UI 자체가 반응하지 않아 버그로 분류되었다.

## 해결 과정

### 1. `loadImageSynchronously`에 캐시 옵션 추가

```swift
@discardableResult
func loadImageSynchronously(url: String, cacheResult: Bool = true) -> UIImage {
    guard !url.isEmpty else { return UIImage() }
    let image = readImageFromDisk(url: url)
    if cacheResult {
        cache(image: image, for: url)
    }
    return image
}
```

- 기본은 기존과 동일하게 캐시 적재.
- 스냅샷 루프 등 캐시가 필요 없는 곳에서는 `cacheResult: false`를 넘겨 디스크에서만 읽게 했다.

### 2. 프리뷰/제작 모드별 이미지 경로 분리

```swift
private func resolvedImage(localID: String?) -> UIImage {
    guard let id = localID, !id.isEmpty else { return UIImage() }
    if viewModel.isMake {
        return viewModel.loadImageSynchronously(url: id, cacheResult: false)
    } else {
        return viewModel.loadEditImage(url: id)
    }
}
```

- 프리뷰에서는 캐시를 그대로 활용해 스크롤 성능을 유지한다.
- 제작 모드(`isMake = true`)에서는 디스크 직독만 수행해 캐시를 건드리지 않는다.

### 3. 주문 직전 캐시 비우기 + 지연 호출

```swift
Button("제작 주문") {
    viewModel.isMake = true
    viewModel.clearImageCache()
    DispatchQueue.main.asyncAfter(deadline: .now() + 1) {
        makePhotobook()
    }
}
```

- `clearImageCache()`가 `imageCache`와 LRU 순서를 동시에 초기화한다.
- 1초 지연으로 ARC가 기존 이미지를 정리할 시간을 확보한 뒤 스냅샷을 돌렸다.

### 4. 슬라이더 상호작용 복원

```swift
slider.isUserInteractionEnabled = true
slider.addTarget(context.coordinator,
                 action: #selector(Coordinator.valueChanged(_:)),
                 for: .valueChanged)

@objc func valueChanged(_ sender: UISlider) {
    let rounded = Double(sender.value.rounded())
    let clamped = min(max(rounded, Double(sender.minimumValue)), Double(sender.maximumValue))
    if sender.value != Float(clamped) {
        sender.value = Float(clamped)
    }
    self.value.wrappedValue = clamped
}
```

- 페이지 단위로만 움직이도록 반올림 후 min/max로 범위를 제어했다.
- 값이 바뀌면 `TabView`가 즉시 해당 페이지로 이동한다.

## 결과

- 주문 버튼 클릭 시 메모리 피크가 20~30% 감소했다.
- 스냅샷 루프가 캐시를 다시 채우지 않아 렌더링 종료 후에도 캐시 상태가 안정적으로 유지된다.
- 슬라이더 드래그만으로 페이지를 이동할 수 있어 QA 피드백이 종료됐다.
- 100장 프로젝트에서도 OOM 리포트가 재현되지 않는다.

## 보안 점검

- 로그는 `[PB] pb_XX_page` 같은 추상화된 형식만 남기며 사용자 식별자를 기록하지 않는다.
- 이미지 로딩·캐시는 앱 샌드박스 내 로컬 식별자만 사용하고 외부 네트워크/SDK에 공유하지 않는다.
- 이번 수정 어디에서도 토큰, 비밀키, 고객 데이터는 노출되지 않는다.

## 얻은 교훈

1. **캐시는 용도별로 갈라야** 한다. 프리뷰 JPEG와 제작용 원본을 같은 딕셔너리에 넣으면 관리가 꼬인다.
2. **스냅샷 루프는 기본적으로 메모리 절약 모드**로 돌려야 한다. 캐시 미사용 옵션, `clearImageCache()`, `autoreleasepool` 세트가 기본 장비다.
3. **UI에서 보이는 컨트롤은 다 동작해야 한다.** 장식 슬라이더를 둘 바엔 `ProgressView`가 낫고, 슬라이더를 남길 거라면 끝까지 상호작용을 살려야 한다.
4. SwiftUI 상태(`isMake`)는 body 재빌드를 야기한다. 상태 전환 시 부수효과(캐시 삭제, 디스크 I/O)를 명확히 묶지 않으면 성능 문제가 연쇄적으로 터진다.

이번 업데이트로 PhotoBook 미리보기는 "100장 풀 프로젝트도 OOM 없이 주문까지 완료"라는 기준을 다시 충족한다. 다음 목표는 페이지 렌더링을 백그라운드 큐로 분리해 UI 스레드를 더 가볍게 만드는 것이다.
