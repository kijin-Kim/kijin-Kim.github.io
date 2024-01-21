---
title: "언리얼 엔진 타겟팅 시스템"
excerpt: "언리얼 엔진으로 구현한 타겟팅 시스템입니다."
order: 1
toc: true
toc_sticky: true
use_math: true
header:
  teaser: assets/images/targeting-system-teaser.gif
---

<a href="https://github.com/kijin-Kim/Scarlet/blob/master/Source/Scarlet/Core/TargetingComponent.cpp" rel="nofollow noopener noreferrer"><i class="fab fa-fw fa-github fa-3x" aria-hidden="true"></i></a>



## 1. 서론
이 문서는 언리얼 엔진으로 구현한 타겟팅 시스템에 대한 기술적인 내용을 담고있습니다.

{% include video id="c_RUybooJAA?si=fnNBzhcZIb-7uy23" provider="youtube" %}


### 2. 알고리즘

#### 2.1 타겟팅 알고리즘
1. **벡터 계산:**
   $$ \vec{V}_{\text{target}} = \vec{P}_{\text{target}} - \vec{P}_{\text{player}} $$
   여기서 $$ \vec{V}_{\text{target}} $$은 대상까지의 벡터, $$ \vec{P}_{\text{target}} $$은 대상의 위치, $$ \vec{P}_{\text{player}} $$는 플레이어의 위치를 나타냅니다.

2. **내적을 통한 각도 측정:**
   $$ \cos(\theta) = \frac{\vec{V}_{\text{target}} \cdot \vec{V}_{\text{camera}}}{\|\vec{V}_{\text{target}}\| \|\vec{V}_{\text{camera}}\|} $$ <br/>
   여기서 $$ \vec{V}_{\text{camera}} $$는 카메라의 전방 벡터를 나타냅니다.

3. **최댓값 비교 및 Sphere Trace 실행:**
   $$ \text{If } \cos(\theta) > \cos(\theta)_{\text{max}}, \text{ then execute Sphere Trace} $$

4. **타겟팅 대상의 결정:**
   Sphere Trace가 대상과 충돌할 경우, 최대 $$ \cos(\theta) $$ 값을 현재의 값으로 갱신합니다.

5. **반복 과정:**
   이 과정을 모든 대상에 대해 반복합니다.

```cpp
float MaxDot = -1.0f;
for (AActor* Candidate : TargetCandidates)
{
	const FVector ToCandidate = UKismetMathLibrary::GetDirectionUnitVector(OwnerCamera->GetComponentLocation(), Candidate->GetActorLocation());
	const float Dot = OwnerCamera->GetForwardVector().Dot(ToCandidate);
	if (Dot > MaxDot)
	{
		FHitResult HitResult;
		const TArray<AActor*> ActorsToIgnore;
		UKismetSystemLibrary::SphereTraceSingle(this, OwnerCamera->GetComponentLocation(), Candidate->GetActorLocation(), 50.0f,
		                                        TraceTypeQuery1,
		                                        false, ActorsToIgnore, EDrawDebugTrace::None, HitResult, true);


		if (HitResult.bBlockingHit && HitResult.GetActor() == Candidate)
		{
			TargetedActor = Candidate;
			MaxDot = Dot;
		}
	}
}
```

#### 2.2 타겟팅 전환 알고리즘
1. **수평 벡터 계산:**
   $$ \vec{V}_{\text{target_flat}} = \vec{V}_{\text{target}} - (\vec{V}_{\text{target}} \cdot \hat{k}) \hat{k} $$
   여기서 $$ \hat{k} $$는 z축 단위 벡터입니다.

2. **카메라 벡터의 수평화:**
   $$ \vec{V}_{\text{camera_flat}} = \vec{V}_{\text{camera}} - (\vec{V}_{\text{camera}} \cdot \hat{k}) \hat{k} $$

3. **외적을 통한 방향 판별:**
   $$ \vec{V}_{\text{cross}} = \vec{V}_{\text{target_flat}} \times \vec{V}_{\text{camera_flat}} $$
   이 외적의 z-성분의 부호를 통해 대상이 카메라 기준으로 왼쪽 또는 오른쪽에 위치하는지를 판별합니다.

4. **타겟팅 전환의 수행:**
   타겟팅 알고리즘의 3, 4, 5번 과정을 동일하게 수행합니다.

```cpp
AActor* NewTargetedActor = nullptr;
FVector CameraForwardProj = OwnerCamera->GetForwardVector();
CameraForwardProj.Z = 0.0f;
float MaxDot = -1.0f;
for (AActor* Candidate : TargetCandidates)
{
	FVector CandidateLocation = Candidate->GetActorLocation();
	FVector ToCandidate = UKismetMathLibrary::GetDirectionUnitVector(OwnerCamera->GetComponentLocation(), CandidateLocation);
	ToCandidate.Z = 0.0f;
	if (Candidate != TargetedActor)
	{
		const FVector Crossed = CameraForwardProj.Cross(ToCandidate);
		if (bToRight ? Crossed.Z > 0 : Crossed.Z < 0)
		{
			const float Dot = CameraForwardProj.Dot(ToCandidate);
			if (Dot > MaxDot)
			{
				FHitResult HitResult;
				TArray<AActor*> ActorsToIgnore;
				ActorsToIgnore.Add(TargetedActor);
				UKismetSystemLibrary::SphereTraceSingle(this, OwnerCamera->GetComponentLocation(), Candidate->GetActorLocation(), 50.0f,
				                                        TraceTypeQuery1,
				                                        false, ActorsToIgnore, EDrawDebugTrace::None, HitResult, true);
				if (HitResult.bBlockingHit && HitResult.GetActor() == Candidate)
				{
					NewTargetedActor = Candidate;
					MaxDot = Dot;
				}
			}
		}
	}
}

if (NewTargetedActor)
{
	TargetedActor = NewTargetedActor;
}
```






<!-- ## 2. 타겟팅 시스템 개요


## 3. 알고리즘
### 3.1 타겟팅 알고리즘


## 4. 구현
### 4.1 내적을 이용한 타겟 선정
- 타겟 벡터 계산 방법
- 내적값 비교를 통한 타겟 선정
### 4.2 Sphere Trace를 활용한 타겟 락온
- 각도 계산 및 Trace 실행
### 4.3 락온 전환 구현
- 카메라 벡터 조정
- 외적을 이용한 각도 부호 계산
- 플레이어 입력에 따른 타겟 선택 로직 -->