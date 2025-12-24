---
title: "각속도 기반 호밍 구현 (ProjectileMovementComponent, Unreal C++)"
date: 2025-12-24 00:00:00 +0900
categories: [Unreal Engine, C++]
tags: [Homing, Math, ProjectileMovement]
---

이 문서는 Unreal Engine의 `UProjectileMovementComponent`를 확장/활용하여 **주어진 각속도(ω, deg/s)** 만큼만 타겟 방향으로 발사체를 회전시키는 방식으로 호밍을 구현하는 방법을 설명합니다.

---

## 개념 요약

- 기존 `HomingAccelerationMagnitude` 대신, **각속도 제한**을 사용.
- 매 Tick 마다:
  1. 현재 진행 방향(F) = Normalize(Velocity)
  2. 타겟 방향(T) = Normalize(TargetPos - MyPos)
  3. 각도 Δθ = acos(clamp(dot(F, T), -1, 1))
  4. 허용 회전각 θ_max = radians(ω_deg) × Δt
  5. 실제 회전각 θ = min(Δθ, θ_max)
  6. 회전축 = normalize(cross(F, T)) (180° 특이 케이스는 임의 직교축 사용)
  7. 쿼터니언 회전 적용 → 새 Velocity 방향

---

## 코드 예시

```cpp
// MyProjectile.h
UCLASS()
class AMyProjectile : public AActor
{
    GENERATED_BODY()
public:
    virtual void Tick(float DeltaSeconds) override;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Homing")
    float MaxAngularSpeedDeg = 180.f;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Homing")
    TWeakObjectPtr<USceneComponent> HomingTarget;

    UPROPERTY(EditAnywhere, BlueprintReadWrite, Category="Homing")
    bool bKeepSpeedMagnitude = true;

protected:
    virtual void BeginPlay() override;
    UPROPERTY(VisibleAnywhere) UProjectileMovementComponent* ProjectileMovement = nullptr;

private:
    void UpdateAngularHoming(float DeltaTime);
};
```

```cpp
// MyProjectile.cpp
void AMyProjectile::Tick(float DeltaSeconds)
{
    Super::Tick(DeltaSeconds);
    UpdateAngularHoming(DeltaSeconds);
}

void AMyProjectile::UpdateAngularHoming(float DeltaTime)
{
    if (!ProjectileMovement || !HomingTarget.IsValid()) return;

    FVector MyLoc = GetActorLocation();
    FVector TargetLoc = HomingTarget->GetComponentLocation();

    FVector ToTarget = TargetLoc - MyLoc;
    if (!ToTarget.Normalize()) return;

    FVector Vel = ProjectileMovement->Velocity;
    float Speed = Vel.Length();
    if (Speed <= KINDA_SMALL_NUMBER) return;

    FVector Fwd = Vel / Speed;

    float Dot = FMath::Clamp(FVector::DotProduct(Fwd, ToTarget), -1.f, 1.f);
    float Angle = FMath::Acos(Dot);
    if (Angle < KINDA_SMALL_NUMBER) return;

    FVector Axis = FVector::CrossProduct(Fwd, ToTarget);
    if (!Axis.Normalize())
    {
        Axis = FVector::Orthogonal(Fwd).GetSafeNormal();
    }

    float MaxStep = FMath::DegreesToRadians(MaxAngularSpeedDeg) * DeltaTime;
    float Step = FMath::Min(Angle, MaxStep);

    FQuat dq(Axis, Step);
    FVector NewFwd = dq.RotateVector(Fwd).GetSafeNormal();

    float NewSpeed = bKeepSpeedMagnitude ? Speed : ProjectileMovement->Velocity.Length();
    ProjectileMovement->Velocity = NewFwd * NewSpeed;
}
```

---

## 확장 아이디어

- **2D 전용 호밍**: 수직축(Z) 기준 평면 투영 후 동일 계산
- **리드 조준**: 타겟 속도를 고려한 예측 위치를 목표로 지정
- **PID 제어**: 단순 clamp 대신 각속도 제어를 PID 방식으로 개선 가능
- **서브스텝**: 빠른 탄체는 Δt를 쪼개 여러 번 회전 → 부드러운 궤적

---

## 디버그 시각화

```cpp
DrawDebugLine(GetWorld(), MyLoc, MyLoc + Fwd * 300.f, FColor::Green, false, 0.f, 0, 1.f);
DrawDebugLine(GetWorld(), MyLoc, MyLoc + ToTarget * 300.f, FColor::Red, false, 0.f, 0, 1.f);
```

---

## 요약

- `HomingAccelerationMagnitude` 없이 각속도로 제어 가능
- 발사체 진행 방향을 매 프레임 제한된 각도만큼 타겟으로 보정
- 더 자연스러운 곡선 호밍 구현 가능
