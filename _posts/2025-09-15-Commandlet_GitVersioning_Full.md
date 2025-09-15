---
title: Commandlet을 이용한 Git 브랜치/커밋 기반 빌드 버전 관리
date: 2025-09-15 00:03:20 +0900
categories: [UnrealEngine]
tags: [programming, unreal engine]
---


# Commandlet을 이용한 Git 브랜치/커밋 기반 빌드 버전 관리

이 문서는 Unreal Engine의 **Commandlet**을 이용하여 Git 저장소의 브랜치 이름과 커밋 해시를 추출하고, 이를 조합하여 빌드 버전을 생성 및 저장하는 방법과 전체 C++ 코드 예시를 제공합니다.

---

## 1. 개요
- Git 저장소 정보(브랜치 이름, 커밋 해시)를 읽어와 고유한 빌드 버전 문자열을 생성
- Commandlet 실행 시 `DefaultGame.ini` 및 `Saved/Config` 내 `Game.ini`에 버전 기록
- CI/CD 파이프라인에서 자동 실행 → 빌드 시점에 버전 자동 업데이트

---

## 2. 브랜치 이름과 커밋 해시 추출 로직

### `.git/HEAD` 직접 읽기
```cpp
FString HeadPath = FPaths::Combine(FPaths::ProjectDir(), TEXT(".git/HEAD"));
FString HeadContent;
FFileHelper::LoadFileToString(HeadContent, *HeadPath);
HeadContent = HeadContent.TrimStartAndEnd();

FString BranchName;
FString CommitHash;

if (HeadContent.StartsWith(TEXT("ref:")))
{
    // 브랜치 참조 경로
    FString RefPath = HeadContent.RightChop(5); // ex) refs/heads/release/multiraid
    BranchName = RefPath;
    BranchName.ReplaceInline(TEXT("refs/heads/"), TEXT(""));
    BranchName.ReplaceInline(TEXT("/"), TEXT(".")); // "/" → "."

    // 커밋 해시 가져오기
    FString RefFile = FPaths::Combine(FPaths::ProjectDir(), TEXT(".git"), RefPath);
    FFileHelper::LoadFileToString(CommitHash, *RefFile);
}
else
{
    // Detached HEAD
    BranchName = TEXT("detached");
    CommitHash = HeadContent;
}

CommitHash = CommitHash.TrimStartAndEnd();
FString ShortHash = CommitHash.Left(7);
```

---

## 3. 버전 문자열 조합
```cpp
FString VersionString = FString::Printf(TEXT("%s-%s"), *BranchName, *ShortHash);
// 결과 예: "release.multiraid-a1b2c3d"
```

확장 예:
```cpp
// YY.MM.DD-순번-브랜치-해시
// "25.9.15-1234-release.multiraid-a1b2c3d"
```

---

## 4. Commandlet 전체 코드

### 헤더
```cpp
#pragma once
#include "Commandlets/Commandlet.h"
#include "UpdateVersionCommandlet.generated.h"

UCLASS()
class UUpdateVersionCommandlet : public UCommandlet
{
    GENERATED_BODY()
public:
    UUpdateVersionCommandlet(const FObjectInitializer& ObjectInitializer);
    virtual int32 Main(const FString& Params) override;
};
```

### 구현
```cpp
#include "UpdateVersionCommandlet.h"
#include "Misc/ConfigCacheIni.h"
#include "Misc/Paths.h"
#include "Misc/FileHelper.h"

UUpdateVersionCommandlet::UUpdateVersionCommandlet(const FObjectInitializer& ObjectInitializer)
    : Super(ObjectInitializer)
{
    IsClient = false;
    IsEditor = true;
    LogToConsole = true;
}

int32 UUpdateVersionCommandlet::Main(const FString& Params)
{
    // Git HEAD 읽기
    FString HeadPath = FPaths::Combine(FPaths::ProjectDir(), TEXT(".git/HEAD"));
    FString HeadContent;
    FFileHelper::LoadFileToString(HeadContent, *HeadPath);
    HeadContent = HeadContent.TrimStartAndEnd();

    FString BranchName;
    FString CommitHash;

    if (HeadContent.StartsWith(TEXT("ref:")))
    {
        FString RefPath = HeadContent.RightChop(5);
        BranchName = RefPath;
        BranchName.ReplaceInline(TEXT("refs/heads/"), TEXT(""));
        BranchName.ReplaceInline(TEXT("/"), TEXT("."));

        FString RefFile = FPaths::Combine(FPaths::ProjectDir(), TEXT(".git"), RefPath);
        FFileHelper::LoadFileToString(CommitHash, *RefFile);
    }
    else
    {
        BranchName = TEXT("detached");
        CommitHash = HeadContent;
    }

    CommitHash = CommitHash.TrimStartAndEnd();
    FString ShortHash = CommitHash.Left(7);

    FString VersionString = FString::Printf(TEXT("%s-%s"), *BranchName, *ShortHash);

    // Config 갱신
    const TCHAR* Section = TEXT("/Script/EngineSettings.GeneralProjectSettings");
    const TCHAR* Key = TEXT("ProjectVersion");

    // Saved Config 업데이트
    GConfig->SetString(Section, Key, *VersionString, GGameIni);
    GConfig->Flush(false, GGameIni);

    // DefaultGame.ini 업데이트 (패키징 반영)
    FString DefaultGameIni = FPaths::Combine(FPaths::ProjectConfigDir(), TEXT("DefaultGame.ini"));
    FConfigFile DefaultCfg;
    DefaultCfg.Read(DefaultGameIni);
    DefaultCfg.SetString(Section, Key, *VersionString);
    DefaultCfg.Write(DefaultGameIni);

    UE_LOG(LogTemp, Display, TEXT("ProjectVersion updated to %s"), *VersionString);
    return 0;
}
```

---

## 5. 실행 방법
```bash
UnrealEditor-Cmd.exe MyProject.uproject -run=UpdateVersion
```

실행 후 `DefaultGame.ini`:
```ini
[/Script/EngineSettings.GeneralProjectSettings]
ProjectVersion=release.multiraid-a1b2c3d
```

---

## 6. 활용 시나리오
- **CI/CD**: 빌드 직전 Commandlet 실행 → 버전 자동 업데이트
- **런타임**: `FApp::GetProjectVersion()` 호출 → HUD, 로딩 화면, 로그에 빌드 정보 출력
- **버전 관리**: 브랜치 + 해시 + 날짜 + 순번 조합으로 고유 빌드 식별자 생성
