---
layout: post
title: "[iOS] 상세 썸네일 파이프라인 안정화 기록"
subtitle: "PhotoManager in-flight 안전화 · Hosting 재사용 보강"
date: 2025-11-22
categories: iOS Swift SwiftUI Debugging
author: raykim
author_url: https://github.com/raykim2414
---

# 상세 썸네일 파이프라인 안정화 기록

> **PhotoManager in-flight 안전화 · Hosting 재사용 보강**

## TL;DR

- 동일 asset에 대한 중복 썸네일 요청이 서로 completion을 덮어써 사라지는 레이스가 존재했다. in-flight 레지스트리를 한 번의 sync 블록에서 처리하고 관찰자를 참조 타입으로 관리해 손실을 막았다.
- `PHCachingImageManager` 관련 취소·stopCaching 호출을 직렬 큐 밖으로 빼서 빠른 스크롤 시 대기열을 비우고, 외부에서 CPU를 쓰는 작업은 가능한 한 동기화 범위 밖으로 이동했다.
- SwiftUI 셀 재사용 시 `UIHostingController`가 nil 이후 다시 superview에 붙지 않는 문제가 있어, 재부착 로직과 함께 host controller 자체를 lifecycle에 맞춰 정리했다.
- 캐시 키를 픽셀 단위까지 세분화하고 cancel 플래그만 큐 안에서 계산하도록 바꿔, 체감 성능 변화 없이 내부 안정성과 가시성만 올렸다.

## 시나리오 & 배경

- **환경**: 특정 상세 화면은 스크롤이 빠른 썸네일 그리드와 풀스크린 뷰가 혼재한 구조라 SwiftUI 셀 안에서 UIKit 기반 이미지 파이프라인을 재사용해야 했다. 캐시 레이어는 Photos 프레임워크와 네이티브 캐시를 혼용한다.
- **증상**:
  - 동일 셀/asset이 짧은 시간에 여러 번 요청되면 콜백 관찰자가 덮여 completion이 오지 않아 빈 셀로 남았다.
  - 스크롤 취소가 연속으로 들어오면 `inflightQueue.sync` 안에서 `cancelImageRequest`가 실행되어 다른 요청들이 필요한 만큼 parallelism을 확보하지 못했다.
  - SwiftUI 재사용 주기에서 host view를 nil로 비운 직후 다시 채울 때, `UIHostingController`의 view가 superview에 붙지 않아 셀이 완전히 비어 보이는 케이스가 있었다.

위 세 가지 증상이 동시에 나타나면 “랜덤하게 셀이 비어 있다”라는 인상만 남기 때문에, 각각을 분리해서 재현·고립시키고 한 번에 하나씩 안정화했다.

## 1. In-flight 요청 관리 원자화

- 중복 요청을 한 번의 동기 블록에서 판단하도록 바꾸고, 관찰자 컨테이너를 참조 타입으로 유지했다.
- Photos 요청 ID가 확정되기 전에는 임시 토큰을 사용해 취소를 지연시켰고, degraded 콜백에서는 inflight 항목을 그대로 둬 completion이 누락되지 않도록 했다.
- 결과적으로 동일 asset의 반복 요청이 들어오더라도 completion이 서로 덮어쓰지 않아 빈 셀이 남지 않는다.

## 2. 취소·캐싱 정리: 락 범위 최소화

- 동기 큐에서는 “취소가 필요한가?” 같은 결론만 구하고, `cancelImageRequest`/`stopCachingImages`처럼 무거운 호출은 큐 밖에서 실행하도록 재구성했다.
- 취소 여부를 boolean 플래그로 반환하는 헬퍼를 두어, 빠른 스크롤 상황에서도 inflight 큐가 다른 요청을 오래 막지 않게 했다.
- 이 구조를 적용한 뒤 Thread Sanitizer에서 보이던 경합 경고가 사라지고, 스크롤 중 hitch가 체감상 줄었다.

## 3. 캐시 키 정밀도 향상

- `targetSize`를 반올림 없이 pixel 단위 Int로 변환해 캐시 키를 구성했다. scale 조합이 미묘하게 다른 요청이 같은 키를 공유하면서 이미지가 당겨 보이거나 흐릿하게 보이는 현상을 막는다.
- 키 생성 함수를 별도 유틸로 빼 두면, LRU 캐시/Photos 프리히트/디스크 캐시 모두 동일한 계산식을 공유할 수 있어 추적이 쉽다.

## 4. Hosting 재사용 보강

- SwiftUI 뷰를 UIKit 셀에 얹을 때는 host view만 nil로 만들지 말고 controller 자체 lifecycle을 함께 정리해야 한다.
- 재사용 시에는 rootView를 교체한 뒤, host view가 superview에 부착돼 있는지 다시 검증하는 루틴을 넣었다.
- prepareForReuse에서 controller와 view를 동시에 정리하자, “nil로 만들었는데 다시 붙지 않는” 공백 현상이 사라졌다.

## 5. 현업에서 써먹을 수 있는 체크리스트

- **In-flight map을 참조 타입으로 유지**: 관찰자가 여러 곳에서 추가/제거되면 value type은 위험해진다.
- **lock 안에서는 결정만, 실행은 밖에서**: Photos, SDWebImage, 네트워크 취소 등 무거운 호출은 큐 밖으로 옮긴다.
- **캐시 키는 픽셀 기준으로**: 논리 단위 대신 실제 렌더 사이즈를 기반으로 키를 만들면 동일 asset이라도 필요한 품질을 유지할 수 있다.
- **HostingController 재사용 규약 만들기**: `rootView` 갱신, superview 재부착, prepareForReuse 루틴을 하나의 헬퍼로 정리한다.

## 기대 효과 & 앞으로

- 동일 asset 다중 요청/취소가 겹쳐도 completion 누락이 사라져 썸네일 로딩이 안정화됐다.
- inflight 큐 충돌이 줄어들어 빠른 스크롤에서도 hitching 없이 이미지가 들어온다.
- 셀 재사용 시 뷰 중첩/소실 이슈가 정리되어 QA에서 보고되던 “랜덤 빈칸”이 재발하지 않았다.

아직 `requestId`가 `.invalid`인 아주 짧은 구간에서 들어오는 취소는 무시되지만, 화면 표시에는 영향이 없다. 더 정밀한 취소를 원하면 request ID가 확정된 시점 이후에 토큰을 노출하거나, invalid 상태에서는 cancel을 바로 스킵하도록 guard를 추가하면 된다. 구현 난이도에 비해 체감차는 크지 않으니, 팀 상황에 맞게 선택하면 된다.

최종 교차 검증은 해당 상세 화면을 한 번 빌드해 빠르게 스크롤하고, Photos 권한 변동·메모리 경고 상황을 번갈아가며 확인하는 것으로 충분했다. 다시 비슷한 증상이 생기면, inflight map dump와 hosting helper 로그를 바로 확인할 수 있도록 디버그 플래그를 유지해 두는 것을 추천한다.

