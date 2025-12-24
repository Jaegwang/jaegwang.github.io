---
title: "Git Bundle로 서브모듈까지 완벽하게 백업하고 복원하는 방법"
date: 2025-12-24 00:00:00 +0900
categories: [Git, DevOps]
tags: [Git, Backup, Submodule, Bundle]
---

# 🧩 Git Bundle로 서브모듈까지 완벽하게 백업하고 복원하는 방법

Git 저장소를 안전하게 백업하려면 단순히 폴더를 압축하는 대신,  
**Git이 공식적으로 제공하는 백업 기능인 `git bundle`을 사용하는 것이 가장 안전하고 깔끔한 방법**입니다.

하지만 많은 사람들이 실수하는 부분이 하나 있습니다:

> **git bundle은 기본적으로 서브모듈(Submodule)을 자동으로 포함하지 않는다.**

이 글에서는 **서브모듈까지 포함하여 완전한 Git 백업을 만드는 올바른 방법**과  
**복원 절차**를 정리합니다.

---

## 📌 1. git bundle이란?

`git bundle`은 Git 저장소 전체를 하나의 파일로 패키징하는 기능입니다.

포함되는 것:

- 모든 커밋
- 모든 브랜치
- 모든 태그
- Git 메타데이터 전체

패키징 예:

```bash
git bundle create backup.bundle --all
```

복원 예:

```bash
git clone backup.bundle myproject
```

---

## 📌 2. git bundle이 서브모듈을 포함하지 않는 이유

Git Submodule은 **루트 저장소가 서브모듈의 소스 코드를 직접 포함하는 구조가 아닙니다.**  
커밋 해시(pointer)만 포함하고 있으며, 실제 서브모듈의 Git 저장소는 **독립된 repo**입니다.

그래서 git bundle은 다음을 절대 하지 않습니다:

- 서브모듈 저장소까지 자동 백업 ❌
- 서브모듈 내용을 main.bundle에 포함 ❌

따라서 **서브모듈도 별도의 bundle로 각각 백업해야 합니다.**

---

## 📌 3. 전체 백업을 만드는 정석 방법

### ✔ 3-1. 루트 저장소 bundle 생성

```bash
git bundle create main.bundle --all
```

### ✔ 3-2. 서브모듈도 각각 bundle 생성

서브모듈은 수동으로 만들어도 되지만, Git은 이를 자동화하는 명령을 제공합니다.

---

## 📌 4. 서브모듈을 자동으로 bundle 백업하는 방법

Git은 `git submodule foreach` 명령을 제공하여  
**각 서브모듈을 순회하며 작업을 자동으로 수행**할 수 있습니다.

### ✅ 서브모듈 각각을 bundle로 생성하는 명령

```bash
git submodule foreach 'git bundle create $(basename $sm_path).bundle --all'
```

### 실행 결과 예시

프로젝트 구조가 아래와 같다면:

```
.
├── SubmoduleA/
├── SubmoduleB/
```

실행 후 생성되는 bundle 파일들:

```
main.bundle
SubmoduleA.bundle
SubmoduleB.bundle
```

이 파일들만 보관하면 **프로젝트 전체 + 모든 서브모듈의 완전한 Git 히스토리**가 보존됩니다.

---

# 📌 5. git bundle로 전체 복원하는 방법

### ✔ 5-1. 루트 저장소 복원

```bash
git clone main.bundle myproject
cd myproject
```

### ✔ 5-2. 서브모듈 초기화

```bash
git submodule init
git submodule update --init
```

### ✔ 5-3. 각 서브모듈 폴더에서 bundle을 fetch 후 checkout

```bash
cd SubmoduleA
git fetch ../../SubmoduleA.bundle
git checkout FETCH_HEAD
```

---

# 📌 6. 전체 흐름 요약

## 🔹 백업

```bash
git bundle create main.bundle --all
git submodule foreach 'git bundle create $(basename $sm_path).bundle --all'
```

## 🔹 복원

```bash
git clone main.bundle myproject
cd myproject

git submodule init
git submodule update --init

cd SubmoduleX
git fetch ../../SubmoduleX.bundle
git checkout FETCH_HEAD
```

---

# 🎯 결론

`git bundle`은 **파일 1개로 Git 저장소 전체를 보관할 수 있는 공식 백업 방식**이며,  
서브모듈까지 백업하려면 **서브모듈도 각각 bundle로 생성**해야 합니다.

특히 아래 명령은 서브모듈을 자동으로 순회하며  
“서브모듈 bundle 백업 파일”을 생성하는 가장 깔끔한 방법입니다:

```bash
git submodule foreach 'git bundle create $(basename $sm_path).bundle --all'
```

이 구조를 사용하면:

- 서버 없이도 Git 저장소 전체를 완전하게 보관 가능  
- 외장 HDD/NAS/클라우드에 아카이브 보관 가능  
- 어떤 환경에서든 쉽게 복구 가능  
