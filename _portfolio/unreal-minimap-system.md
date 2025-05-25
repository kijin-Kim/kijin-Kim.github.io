---
title: "언리얼 미니맵 및 안개 시스템"
excerpt: "언리얼 엔진으로 구현한 미니맵 시스템입니다."
toc: true
toc_sticky: true
use_math: true
comments: true
header:
  teaser: assets/images/targeting-system-teaser.gif
---

## 1. 서론 및 시스템 개요

본 포스팅은 언리얼 엔진 5에서 미니맵 및 안개(Fog of War) 시스템을 구현하고 기술적 문제를 해결한 과정을 공유하고자 합니다.

본 시스템은 다음과 같은 핵심 기능들을 구현하였습니다.
* 자신 및 팀원의 시야 범위가 미니맵에 나타나게 하는 기능.
* 자신 및 팀원의 시야 범위 안에 적군 플레이어가 들어오면 미니맵에 표시하는 기능.

TODO: 프리뷰 GIF 또는 영상

## 2. 시스템 아키텍처 및 핵심 구현
### 2.1 데이터 정의 및 초기화
#### 2.1.1 시야각 렌더 타겟과 정보 렌더 타겟 정의
`GunnerGameState.h`에 정의된 `FGunnerFogOfWarRenderTargets`는 시야(`VisionConeRenderTarget`)와 정보(`InformationRenderTarget`) 렌더 타겟을 함께 관리합니다.
```cpp
USTRUCT(BlueprintType)
struct FGunnerFogOfWarRenderTargets
{
    GENERATED_BODY()
    /*캐릭터의 시야 폴리곤 렌더링을 위한 렌더 타겟*/
    UPROPERTY(BlueprintReadOnly)
    TObjectPtr<UCanvasRenderTarget2D> VisionConeRenderTarget;
    /*캐릭터의 초상화 렌더링을 위한 렌더 타겟*/
    UPROPERTY(BlueprintReadOnly)
    TObjectPtr<UCanvasRenderTarget2D> InformationRenderTarget;
};
```

#### 2.1.2 플레이어별 렌더 타겟 생성 및 관리
`AGunnerGameState`는 `PlayerFogOfWarRenderTargets` TMap을 사용해 각 플레이어 ID에 해당하는 `FGunnerFogOfWarRenderTargets` 인스턴스를 저장합니다. 이 함수는 렌더 타겟이 없으면 새로 생성하고, 있으면 반환하여 각 플레이어의 시야 및 정보가 독립적인 렌더 타겟에 그려지도록 합니다.
```cpp
 const FGunnerFogOfWarRenderTargets& AGunnerGameState::FindOrAddPlayerFogOfWarRenderTargets(int32 PlayerID)
{
    if (!PlayerFogOfWarRenderTargets.Contains(PlayerID))
    {
        FGunnerFogOfWarRenderTargets NewRenderTargets;
        NewRenderTargets.VisionConeRenderTarget = UCanvasRenderTarget2D::CreateCanvasRenderTarget2D(
            GetWorld(), UCanvasRenderTarget2D::StaticClass(), 1024, 1024);
        NewRenderTargets.InformationRenderTarget = UCanvasRenderTarget2D::CreateCanvasRenderTarget2D(
            GetWorld(), UCanvasRenderTarget2D::StaticClass(), 1024, 1024);
        PlayerFogOfWarRenderTargets.Add(PlayerID, NewRenderTargets);
    }
    return PlayerFogOfWarRenderTargets[PlayerID];
}
```

#### 2.1.3 초기화 및 업데이트

- TODO: BP_GunnerCharacter 마다 있는 사진

각 캐릭터마다 `UGunnerFogOfWarComponent`를 추가하고 적절한 초기화 시점에 `FindOrAddPlayerFogOfWarRenderTargets`함수를 호출하여, 적절한 렌더 타겟을 얻고, 해당 렌더 타겟의 렌더링 델리게이트에 시야 콘과 정보를 그리는 함수를 등록했습니다.`VisionConeRenderTarget`은 `DrawVision`을 `InformationRenderTarget`은 `DrawInformation`을 호출합니다.
```cpp
void UGunnerFogOfWarComponent::SetupFogOfWar(APlayerState* PlayerState)
{
    // ... (생략)
    AGunnerGameState* GameState = GetWorld()->GetGameStateChecked<AGunnerGameState>();
    const FGunnerFogOfWarRenderTargets& RenderTargets = GameState->FindOrAddPlayerFogOfWarRenderTargets(TeamAgentInterface->GetGenericTeamId());
    RenderTargets.VisionConeRenderTarget->OnCanvasRenderTargetUpdate.AddUniqueDynamic(this, &UGunnerFogOfWarComponent::DrawVision);
    RenderTargets.InformationRenderTarget->OnCanvasRenderTargetUpdate.AddUniqueDynamic(this, &UGunnerFogOfWarComponent::DrawInformation);
    // ... (생략)
}
```
`AGunnerGameState`는 매 프레임 마다 로컬에 존재하는 모든 렌더 타겟에 대하여 `UpdateResource`함수를 호출합니다. 해당 함수는 초기화 시점에 등록한 `OnCanvasRenderTargetUpdate` 델리게이트의 호출을 유발합니다.

```cpp
void AGunnerGameState::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);
	for (auto& [PlayerID, RenderTargets] : PlayerFogOfWarRenderTargets)
	{
		if (RenderTargets.VisionConeRenderTarget)
		{
			RenderTargets.VisionConeRenderTarget->UpdateResource();
		}

		if (RenderTargets.InformationRenderTarget)
		{
			RenderTargets.InformationRenderTarget->UpdateResource();
		}
	}
}
```
#### 2.1.4 정적 시야 오클루전 지오메트리 데이터 준비
`GunnerMiniMapData.h`는 맵 지오메트리 데이터 구조화를 위한 `USTRUCT`들을 정의합니다.
* `FGunnerGeometryVertex`: 맵 정점(X, Y 좌표) 정의.
    
* `FGunnerGeometryLine`: 두 정점(`Start`, `End` 인덱스)을 연결하는 선분 정의.
    
* `FGunnerGeometryGroup`: 정점, 선분, Z 높이(`ZHeight`)를 포함하는 논리적 맵 지오메트리 그룹 정의. `ZHeight`는 시야 차단 구현에 사용됩니다.


### 2.2 시야 및 정보 렌더링
#### 2.2.1 시야 폴리곤 렌더링
매 프레임마다 `DrawVision`함수가 호출되면, 렌더 타겟에 플레이어 시야 영역을 `VisionConeRenderTarget`에 그립니다.
*   **Raycasting 및 선분 교차 알고리즘**: 플레이어 눈 위치(`PlayerEyeLocation`)와 시야 각도(`VisionAngleDeg`)를 기반으로 Ray를 발사하고, `UGunnerMapGeometryData`의 선분(`EdgeSegments`)과 교차하는지 검사하여 시야 폴리곤 경계를 형성합니다.
    
    *   `LineSegmentIntersection` 함수는 두 2D 선분(A-B, C-D)의 교차 여부와 교차점을 계산합니다.
        
    ```cpp
        static bool LineSegmentIntersection(
            const FVector2D& A, const FVector2D& B,
            const FVector2D& C, const FVector2D& D,
            FVector2D& OutIntersection)
        {
            // ... (교차점 계산 로직)
        }
    ```    
    
*   **`UGunnerMapGeometryData`를 통한 맵 데이터 주입**: `GeometryAsset` 프로퍼티를 통해 에디터에서 생성된 맵 지오메트리 데이터를 주입받아 시야 계산에 활용하며, 플레이어의 Z 높이와 맵 지오메트리 Z 높이 차이를 고려하여 시야 차단을 적용합니다.

*   **RayCount 조절**: `DrawVision`에서 발사하는 Ray 개수(`RayCount`)를 적절히 조절하여 계산량을 최적화했습니다. 저희는 64개의 Ray를 사용하여 시각적 품질과 성능의 균형을 찾았습니다.

*   **Z 높이 기반 필터링**: `DrawVision`에서 시야를 차단하는 `EdgeSegments` 검사 시, 플레이어 Z 높이와 Edge ZHeight 차이가 일정 임계값(125.0f)을 초과하는 Edge는 계산에서 제외하여 불필요한 연산을 줄였습니다.
    

#### 2.2.2 플레이어 아이콘 (자신, 아군, 적군) 렌더링
매 프레임마다 `DrawInformation`함수가 호출되면, `InformationRenderTarget`에 플레이어 자신, 아군, 시야 내 적군 캐릭터 아이콘을 그립니다.

*   **`DrawPlayerIcon`**: 플레이어 아이콘을 그리는 헬퍼 함수로, 위치, 방향, 팀에 따라 다른 머티리얼을 사용합니다.
    
*   **`LineTraceSingleByChannel`을 이용한 시야 내 적군 감지**: `UGameplayStatics::GetAllActorsOfClass`로 모든 캐릭터를 가져온 후, 플레이어 눈 위치에서 각 캐릭터까지 Line Trace를 수행합니다. 아군은 무시하고, 시야 각도(71.0f) 내의 캐릭터만 고려하여 성능을 최적화합니다. 히트된 액터가 대상 캐릭터와 동일한 경우에만 아이콘을 그립니다.

*   **불필요한 트레이스 방지**: `DrawInformation`에서 적군 감지 시, 플레이어 시야 각도(71.0f) 내 액터만 Line Trace를 수행하도록 필터링 로직을 추가했습니다. `CollisionParams.AddIgnoredActors`로 아군 액터와의 불필요한 충돌 검사를 건너뛰어 연산량을 줄였습니다.


### 2.3 UI 연동: 위젯 블루프린트 및 머티리얼

C++ 컴포넌트에서 효율적으로 처리된 미니맵 및 안개 시스템 로직의 결과를 UI에 효과적으로 표시하기 위해, 언리얼 엔진의 UMG(Unreal Motion Graphics) 시스템과 연동했습니다. `WBP_Minimap` 위젯 블루프린트는 C++ 렌더 타겟을 미니맵 UI에 표시하며, 플레이어 움직임에 따라 아이콘을 동적으로 업데이트하는 역할을 담당합니다.

### 머터리얼 구현
TODO: 머터리얼 설명
### `WBP_Minimap`의 핵심 컴포넌트 및 역할
TODO: 미니맵 UI 설명
`WBP_Minimap` 위젯은 효율적인 미니맵 표시를 위해 주로 `MinimapImage`와 `MinimapInformationOverlay` 컴포넌트를 활용합니다.

*   `MinimapImage` (`UImage`): 미니맵 배경이며, `UGunnerFogOfWarComponent`에서 실시간으로 그려지는 `VisionConeRenderTarget`와 `InformationRenderTarget`를 배경으로 표시합니다. 초기화 시 동적 머티리얼 인스턴스(MID)를 브러시로 설정하여 렌더 타겟 내용을 받아옵니다.

    

### `MinimapDynamicMaterialInstance` 생성 및 `MinimapImage`에 적용

`WBP_Minimap`의 PreConstruct 또는 Construct 이벤트(블루프린트 기준)에서 `MinimapMaterial`을 기반으로 `MinimapDynamicMaterialInstance`를 생성합니다. 이 MID는 런타임에 파라미터를 변경할 수 있으며, `UGunnerGameState`가 관리하는 `UCanvasRenderTarget2D`를 텍스처 파라미터로 받아 미니맵에 표시합니다.

    // WBP_Minimap 블루프린트의 Event Graph (PreConstruct 또는 Construct)
    // MinimapMaterial로부터 동적 머티리얼 인스턴스를 생성하고, MinimapImage의 브러시로 설정
    // K2Node_CallMaterialParameterCollectionFunction_0: CreateDynamicMaterialInstance (동적 머티리얼 인스턴스 생성)
    // K2Node_VariableSet_1: MinimapDynamicMaterialInstance 변수에 할당
    // K2Node_CallFunction_1: SetBrushFromMaterial (MinimapImage에 적용)
    
TODO: 최종 GIF 또는 영상


## 3. 향후 개선 과제

* **다른 오브젝트 표시**: 현재 플레이어 캐릭터 아이콘 위주이지만, 추후 게임 내 기능 확장에 따라 스킬, 구조물, 핑 등 다른 오브젝트를 추가 표시할 계획입니다.

## 4. 마치며
긴 글 읽어주셔서 감사합니다. 궁금한 점이나 피드백은 언제든지 환영합니다.