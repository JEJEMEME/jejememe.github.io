---
layout: post
title: "[iOS] Media Viewer 자동 재생/정지 안정화 기록"
subtitle: "Error Case #2 · 썸네일 진입 시 자동재생 & 슬라이드 전환 시 정지"
date: 2025-11-16
categories: iOS Swift SwiftUI Debugging
author: raykim
author_url: https://github.com/raykim2414
---

# Media Viewer 자동 재생/정지 안정화 기록

> **Error Case #2 · 썸네일 진입 시 자동재생 & 슬라이드 전환 시 정지**

## TL;DR

- `GalleryPagingView`에서 비디오 썸네일을 탭하면 전체화면 뷰어(`FullScreenViewer`) 진입과 동시에 재생이 시작돼야 한다는 요구가 있었다.
- 기존 `GalleryPlayer`는 터치로 재생을 시작하도록 설계돼 있어, 썸네일 → 전체화면으로 넘어가도 다시 플레이 버튼을 눌러야 했다.
- 영상이 재생된 뒤 좌우로 스와이프하면 이전 영상이 계속 재생돼 소리가 겹치는 문제도 보고되었다.
- 썸네일에서 “자동 재생 대상 ID”를 넘기고, `FullScreenViewer → GalleryPlayer`가 이를 바인딩으로 공유해 **딱 한 번만** 자동재생을 허용하도록 변경했다.
- 뷰어가 슬라이드 전환을 감지하면 비활성 페이지의 플레이어를 즉시 정지하고, 슬라이드쇼/크로스페이드 로직의 인덱스 경계도 보강했다.

## 배경

홈에서 특정 사진·영상을 길게 누르면 `GalleryPagingView`가 열린다. 이 화면에서는 썸네일을 스크롤하다가 마음에 드는 항목을 전체화면으로 띄워 바로 확인할 수 있다. 하지만 동영상은 썸네일 터치 → 전체화면 전환 → 재생 버튼 터치라는 세 단계가 필요했고, 사용자 피드백으로 “내가 영상을 선택했으면 바로 재생돼야 한다”는 요구가 강하게 들어왔다.

동시에 다른 팀에서 “전체화면에서 재생 하다가 좌·우로 넘기면 이전 영상 소리가 계속 들린다”는 버그를 제보했다. 요약하면 **“첫 진입 시 자동재생, 페이지 이동 시 자동정지”**를 동시에 만족해야 했다.

## 문제 관찰

1. 썸네일을 탭해 전체화면으로 들어가면, `GalleryPlayer`는 여전히 `.Default` 상태로 플레이 버튼만 보여줬다.
2. 전체화면에서 재생 중인 상태로 다른 페이지로 스와이프하면, 이전 페이지의 `AVPlayer`가 계속 재생 상태를 유지했다.
3. 슬라이드쇼 모드에서 광고 카드가 끼어 있을 때 `currentPage - 1`이나 `currentPage + 1`이 범위를 벗어나 크래시가 발생할 가능성도 발견했다.

## 원인 분석

### 1. 자동재생을 위한 정보 전달이 없었다

`GalleryPagingView`는 전체화면 뷰어를 여는 것까지만 책임지고, 어떤 타입의 미디어인지 `FullScreenViewer`가 다시 판단했다. “처음에 내가 클릭한 게 ‘비디오’다”라는 사실이 사라졌기 때문에 `GalleryPlayer`는 항상 사용자가 직접 `loadPlayer()`를 눌러주길 기다렸다.

### 2. 플레이어가 뷰 계층의 활성 상태를 모른다

`GalleryPlayer`는 “현재 페이지가 바뀌었는지”를 알지 못했다. SwiftUI가 새 페이지를 만들면서 이전 `GalleryPlayer`도 계속 살아 있고, 내부 `AVPlayer`는 스스로 `pause()`되지 않는다.

### 3. 슬라이드쇼/크로스페이드의 인덱스 검사 누락

광고를 끼워 넣기 위해 `medias` 배열 안에 `.ad` 타입을 섞어 두었는데, 이전/다음 페이지를 찾을 때 단순히 `medias[index]`로 접근했다. `index`가 -1이 되거나 `count`와 같아지면 바로 크래시였다.

## 초기 구현이 그렇게 된 이유 (추정)

- 홈 피드와 뷰어가 `GalleryPlayer`를 공유하던 초기 구조에서는 “사용자 상호작용 이후에만 재생”이라는 단일 정책을 유지하는 편이 QA/리뷰가 간단했다. 피드에서 자동재생을 막으려다 보니 뷰어 진입 시에도 동일 정책이 그대로 적용됐다.
- SwiftUI 전환 애니메이션 중 중첩 `AVPlayer`가 동시에 재생되면 프레임 드랍이 발생해, 플레이어가 로컬 상태만 보도록 단순화했다. 대신 “현재 페이지가 무엇인지” 같은 컨텍스트는 전달하지 않아 활성 페이지 구분이 불가능했다.
- 슬라이드쇼/크로스페이드 모듈은 광고 카드가 거의 없던 시기에 작성되어 “정규 미디어만 있다”는 가정을 유지했고, 광고가 연속으로 등장하는 케이스를 대비하지 못했다.

## 해결 과정

### 1. 썸네일 → 뷰어로 “자동재생 대상” 전달

```swift
// GalleryPagingView.swift
@State private var autoPlayTargetMediaID: Int? = nil

thumbView(media).onTapGesture {
    if media.isVideo { autoPlayTargetMediaID = media.idx }
    viewModel.fullDetailSheet = SheetDetail(type: .ImageViewer)
}

.fullScreenCover(item: $viewModel.fullDetailSheet) {
    FullScreenViewer(
        …,
        initialPage: page,
        autoPlayMediaID: autoPlayTargetMediaID,
        medias: $viewModel.medias,
        hasReachedEnd: $viewModel.hasReachedEnd,
        onDismiss: { lastIndex, _ in
            page = lastIndex
            autoPlayTargetMediaID = nil
        }
    )
}
```

`autoPlayTargetMediaID`는 썸네일을 탭할 때만 값이 들어가고, 전체화면 뷰어에서 돌아오면 nil로 초기화된다.

### 2. `FullScreenViewer`가 현재 페이지와 자동재생 ID를 공유

```swift
@State private var pendingAutoPlayMediaID: Int?
@State private var activeMediaID: Int?

GalleryPlayer(
    showControls: $viewModel.isShowContents,
    currentPage: $page,
    showThumbnail: true,
    mediaModel: media,
    autoPlay: pendingAutoPlayMediaID == media.idx,
    mediaID: media.idx,
    activeMediaID: $activeMediaID
)
.onAppear {
    if pendingAutoPlayMediaID == media.idx {
        pendingAutoPlayMediaID = nil
    }
}
```

- `activeMediaID`는 페이지가 바뀔 때마다 갱신된다.
- 뷰어가 사라질 때 슬라이드쇼 타이머를 `cancelTimer()`로 정리해, 다시 들어오더라도 남은 구독이 없도록 했다.

### 3. `GalleryPlayer`가 활성 상태와 자동재생을 관리

```swift
struct GalleryPlayer<BottomView: View>: View {
    let autoPlay: Bool
    let mediaID: Int?
    @Binding var activeMediaID: Int?
    @State private var didTriggerAutoPlay = false

    private var isCurrentlyActive: Bool {
        guard let mediaID else { return false }
        return mediaID == activeMediaID
    }

    var body: some View {
        …
        .onAppear {
            triggerAutoPlayIfNeeded()
            handleActiveStateChange(isActive: isCurrentlyActive)
        }
        .onChange(of: activeMediaID) { _ in
            handleActiveStateChange(isActive: isCurrentlyActive)
            triggerAutoPlayIfNeeded()
        }
    }

    func resetPlayer() {
        player?.pause()
        playerStatus = .Default
        playerItemObserver = PlayerItemObserver()
        player = nil
        videoPlaying = false
        serverLoad = false
        // didTriggerAutoPlay는 초기화하지 않는다.
    }

    private func triggerAutoPlayIfNeeded() {
        guard autoPlay, !didTriggerAutoPlay, isCurrentlyActive else { return }
        didTriggerAutoPlay = true
        if playerStatus == .Default {
            loadPlayer()
        }
    }

    private func handleActiveStateChange(isActive: Bool) {
        if !isActive {
            player?.pause()
            videoPlaying = false
        }
    }
}
```

- 최초 진입한 페이지에서만 `autoPlay`가 true이고, `isCurrentlyActive`도 true이기 때문에 `loadPlayer()`가 자동 호출된다.
- 이후 페이지를 옮겼다가 다시 돌아오면 `didTriggerAutoPlay`가 true로 남아 있어 자동재생이 재발동하지 않는다.
- 활성 페이지가 바뀌면 `handleActiveStateChange`가 즉시 `pause()`를 호출해 오디오가 겹치지 않는다.

### 4. 슬라이드쇼 및 크로스페이드의 경계 보강

```swift
var prevPageIndex = page - 1
while prevPageIndex >= 0 && (medias[safe: prevPageIndex]?.presentType == .ad) {
    prevPageIndex -= 1
}

var nextPageIndex = currentPage + 1
while nextPageIndex < medias.count && (medias[safe: nextPageIndex]?.presentType == .ad) {
    nextPageIndex += 1
}
```

광고 카드가 연속으로 이어져 있어도 배열 경계를 벗어나지 않으며, 더 이상 이전 카드가 없으면 슬라이드쇼를 종료한다.

## 결과

- 비디오 썸네일을 탭하면 전체화면으로 전환되자마자 재생이 시작된다.
- 재생 중 좌우 스와이프로 다른 페이지로 이동하면 기존 플레이어가 즉시 일시정지되어 소리가 겹치지 않는다.
- 같은 영상으로 되돌아와도 자동재생이 재발동하지 않아, 사용자가 직접 재생 여부를 제어할 수 있다.
- 슬라이드쇼와 크로스페이드 모드에서 광고 카드 사이로 빠르게 이동해도 크래시가 발생하지 않는다.

## 얻은 교훈

1. **상태 전달을 명시적으로**: “어떤 썸네일을 탭했는가” 같은 컨텍스트는 가장 상위 뷰가 알고 있으므로, 하위 뷰로 명확하게 내려줘야 한다.
2. **재생 컴포넌트는 활성 페이지를 알아야 한다**: SwiftUI에서 같은 View가 여러 개 생성될 수 있으므로, “현재 화면에 보이는가?”를 외부에서 알려줘야 오디오/비디오를 안전하게 제어할 수 있다.
3. **once-only 플래그는 의도적으로 초기화하지 않는다**: 자동재생처럼 “첫 진입에서만” 실행되어야 하는 로직은, 한 번 발동 후에는 명시적으로 플래그를 유지해야 재진입 시 부작용이 없다.
4. **슬라이드쇼/무한 스크롤에는 경계가 필수**: 광고·로딩 플레이스홀더처럼 일반 미디어가 아닌 항목이 섞일 때는 `subscript(safe:)` 같은 안전 접근을 기본값으로 생각하자.

이제 Media Viewer는 **“썸네일을 탭하면 즉시 재생, 페이지를 넘기면 조용히 정지”**라는 사용자의 직관적인 기대를 충족한다. 추가 트러블이 없다면 이 설계를 다른 뷰어에도 확장할 계획이다.

