---
title: "[Unreal Engine] GC(Garbage Collection) 최적화: 수동 제어와 Incremental GC"
date: 2025-12-24 00:00:00 +0900
categories: [Unreal Engine, Performance]
tags: [GC, Optimization, C++, Blueprint]
---

언리얼 엔진으로 게임을 개발하다 보면 주기적으로 발생하는 **GC(Garbage Collection)** 때문에 프레임 드랍(Hitch)을 경험하곤 합니다. 특히 오픈월드나 액션 게임처럼 퍼포먼스가 중요한 프로젝트에서는 이 현상이 치명적일 수 있습니다.

이번 포스트에서는 GC 타이밍을 직접 제어하는 방법과, 프레임을 부드럽게 유지해주는 **Incremental GC**에 대해 정리해보겠습니다.

## 1. 주기적 GC 비활성화하고 원할 때만 실행하기 (Manual GC)

기본적으로 언리얼 엔진은 일정 주기(기본 60초)마다 GC를 수행합니다. 이를 비활성화하고, 로딩 화면 등 게임 흐름에 방해가 되지 않는 순간에만 수동으로 GC를 실행할 수 있습니다.

### 설정 방법 (Project Settings)
1. **Project Settings** > **Engine** > **Garbage Collection**으로 이동합니다.
2. **Time Between Purging Pending Kill Objects** 값을 매우 크게 설정합니다. (예: `999999`)
   - 이렇게 하면 사실상 자동 GC 타이머가 작동하지 않게 됩니다.

또는 `DefaultEngine.ini` 파일에서 직접 수정도 가능합니다:
```ini
[/Script/Engine.GarbageCollectionSettings]
gc.TimeBetweenPurgingPendingKillObjects=999999
```

### 수동 호출 방법
이제 자동 GC가 돌지 않으므로, 메모리 누수를 막기 위해 적절한 시점에 직접 호출해줘야 합니다.

* **C++**:
  ```cpp
  // true를 넘기면 전체 GC(Full Purge)를 강제로 수행합니다.
  GEngine->ForceGarbageCollection(true);
  ```
* **Blueprint**: `Collect Garbage` 노드를 사용합니다.

### 주의사항
이 방식은 GC 순간에 프레임이 튀는 것을 막을 수 있지만, 오랫동안 GC를 호출하지 않으면 메모리 사용량이 계속 증가합니다. 따라서 **로딩 화면, 레벨 이동, 인벤토리 진입** 등 멈춤 현상이 용인되는 시점에 반드시 호출해야 합니다.

---

## 2. 끊김 없는 부드러운 처리: Incremental GC

**Incremental GC (점진적 가비지 컬렉션)**는 GC 작업을 한 프레임에 몰아서 처리하지 않고, **여러 프레임에 조금씩 나누어서 처리**하는 방식입니다.

### 작동 원리
기존 GC(Full GC)가 '대청소'라면, Incremental GC는 '매일 조금씩 하는 청소'와 같습니다.
* **Reachability Analysis 분산**: GC의 가장 무거운 작업인 '도달 가능성 분석'을 시간 제한(Time Budget)을 두고 수행합니다.
* **Time Slicing**: 예를 들어 이번 프레임에 2ms만 사용하고, 남은 작업은 다음 프레임으로 미룹니다. 결과적으로 플레이어는 렉을 느끼지 못합니다.

### 설정 방법
1. **Project Settings** > **Engine** > **Garbage Collection**으로 이동합니다.
2. 다음 옵션들을 확인합니다:
   - **Allow Parallel GC**: 활성화 (필수)
   - **Incremental Gather Unreachable Objects**: 활성화
   - **Time Limit**: GC가 한 프레임당 사용할 최대 시간(ms). 보통 `0.002` (2ms) 정도가 적당합니다.

### 장단점 비교

| 특징 | Full GC (기존/수동) | Incremental GC (점진적) |
| :--- | :--- | :--- |
| **프레임 영향** | 실행 순간 큰 **히치(Hitch)** 발생 | 렉이 거의 없거나 매우 균일함 |
| **메모리 효율** | 청소 후 즉시 메모리 확보 | 청소 주기가 길어져 메모리 점유 시간이 긺 |
| **CPU 비용** | 한 번에 처리하므로 총 비용은 낮음 | 관리 오버헤드로 인해 **총 CPU 비용은 약간 높음** |

---

## 3. 결론: 하이브리드 전략 추천

가장 이상적인 방법은 두 가지를 적절히 섞어서 사용하는 것입니다.

1. **평상시 (인게임)**: **Incremental GC**를 켜두어(2ms 제한) 메모리가 무한정 늘어나는 것을 막으면서 부드러운 프레임 유지.
2. **특정 시점 (로딩/컷신)**: 지역 이동이나 로딩 화면 등 화면이 전환되는 순간에는 `ForceGarbageCollection(true)`을 호출하여 메모리를 깔끔하게 정리.

이 전략을 통해 쾌적한 게임 플레이 환경과 안정적인 메모리 관리를 동시에 잡을 수 있습니다.
