---
title: "언리얼 엔진 게임 성능 최적화: GC(Garbage Collection) 정복하기"
date: 2025-12-24 11:00:00 +0900
categories: [Unreal Engine, Optimization]
tags: [Performance, GC, C++, Blueprint]
---

게임 개발, 특히 언리얼 엔진을 사용한 프로젝트에서 **프레임 드랍(Frame Drop)**이나 **히치(Hitch)** 현상은 사용자 경험을 해치는 주범입니다. 이 중 상당수는 메모리 관리, 즉 **가비지 컬렉션(Garbage Collection, GC)**이 실행될 때 발생합니다.

오늘은 이 GC로 인한 성능 저하를 막고, 게임을 부드럽게 만들기 위한 두 가지 핵심 전략인 **Incremental GC**와 **Manual GC**를 활용한 최적화 방법을 알아보겠습니다.

## 1. 성능 저하의 원인: "Stop-the-World"
언리얼 엔진의 기본 GC는 실행되는 순간 게임 로직을 멈추고 메모리를 정리합니다. 오브젝트가 수만 개 이상인 오픈월드 게임에서는 이 과정이 100ms 이상 걸리기도 하는데, 이는 60fps 기준(16ms)에서 약 6프레임 이상을 건너뛰게 만들어 화면이 툭 끊기는 느낌을 줍니다.

## 2. 전략 A: 작업을 쪼개서 부드럽게 (Incremental GC)
가장 추천하는 첫 번째 방법은 GC 작업을 여러 프레임에 분산시키는 것입니다.

### 왜 성능에 좋은가?
한 번에 30ms가 걸릴 작업을 10프레임 동안 3ms씩 나누어 처리한다면, 프레임 시간에는 거의 영향을 주지 않습니다.

### 적용 방법
1. **Project Settings > Engine > Garbage Collection** 이동
2. **Incremental Gather Unreachable Objects**: `True` 체크
3. **Time Limit**: `0.002` (2ms) 설정
   - *팁*: 2ms는 60fps 게임에서 안전한 수치입니다. 만약 30fps 타겟이라면 조금 더 늘려도 됩니다.

이 설정만으로도 전투 중이나 필드 이동 시 발생하는 미세한 끊김을 대폭 줄일 수 있습니다.

## 3. 전략 B: 타이밍을 지배하라 (Manual GC)
두 번째 전략은 GC 자동 실행을 끄고, **우리가 원하는 타이밍**에만 실행하는 것입니다.

### 왜 성능에 좋은가?
전투 중이나 보스 레이드 같이 중요한 순간에는 아예 GC를 돌리지 않음으로써 100% 성능을 보장합니다. 대신 로딩 화면, 인벤토리 열기, 컷신 시작 등 "화면이 멈춰도 되는 순간"에 몰아서 처리합니다.

### 적용 방법

**1단계: 자동 GC 비활성화**
프로젝트 세팅에서 `Time Between Purging Pending Kill Objects`를 매우 큰 값(예: `999999`)으로 설정하여 엔진이 스스로 GC를 하지 못하게 막습니다.

**2단계: 적절한 타이밍에 호출 (C++ 예시)**
```cpp
#include "Engine/Engine.h"

void UMyGameInstance::OnEnterLoadingScreen()
{
    // 로딩 화면 진입 시 강제 GC 수행
    // true 파라미터는 Full Purge를 의미합니다.
    if (GEngine)
    {
        GEngine->ForceGarbageCollection(true);
    }
}
```

**3단계: 블루프린트 활용**
블루프린트에서는 `Collect Garbage` 노드를 호출하면 됩니다. 레벨 스트리밍이 완료된 직후나 UI가 화면을 가릴 때 호출하는 것이 좋습니다.

## 4. 궁극의 최적화: 하이브리드 패턴
실무에서는 이 두 가지를 섞어서 사용하는 것이 가장 효과적입니다.

*   **평상시**: **Incremental GC**를 켜두어(Time Limit 2~3ms) 메모리가 위험 수위까지 차오르는 것을 방지합니다.
*   **전환 시점**: 레벨 이동이나 미션 클리어 시점에는 **Manual GC**(`ForceGarbageCollection`)를 한 번 호출하여 메모리를 완전히 초기화합니다.

## 요약
1. **끊김이 싫다면**: `Incremental GC`를 켜세요.
2. **완벽한 제어가 필요하다면**: 자동 GC 타이머를 끄고 직접 호출하세요.
3. **메모리와 성능 모두 잡으려면**: 두 가지를 혼합하여 상황에 맞게 대처하세요.

GC 최적화는 단순히 설정을 켜는 것뿐만 아니라, 우리 게임의 흐름(Flow)을 파악하고 적절한 타이밍을 설계하는 것이 중요합니다.
