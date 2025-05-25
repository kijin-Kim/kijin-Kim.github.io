---
title: "언리얼 미니맵 및 안개 시스템"
excerpt: "언리얼 엔진으로 구현한 미니맵 시스템입니다."
toc: true
toc_sticky: true
use_math: true
header:
  teaser: assets/images/targeting-system-teaser.gif
---

## 1. 서론 및 시스템 개요

본 포스팅은 언리얼 엔진 5에서 미니맵 및 안개(Fog of War) 시스템을 구현하고 기술적 문제를 해결한 과정을 공유하고자 합니다.

본 시스템은 다음과 같은 핵심 기능들을 구현하였습니다.
* 자신 및 팀원의 시야 범위가 미니맵에 나타나게 하는 기능.
* 자신 및 팀원의 시야 범위 안에 적군 플레이어가 들어오면 미니맵에 표시하는 기능.

## 2. 시스템 아키텍처 및 핵심 구현

효율적이고 확장 가능한 미니맵 및 안개 시스템을 위해, 기능별 역할을 분리한 컴포넌트 기반 아키텍처를 설계하고 구현했습니다.

### 게임 상태 및 렌더 타겟 관리

`AGunnerGameState`는 게임 상태를 관리하며, 미니맵 및 안개 시스템의 핵심 데이터를 중앙 관리합니다.

### 구조체를 통한 렌더 타겟 캡슐화

`GunnerGameState.h`에 정의된 `FGunnerFogOfWarRenderTargets`는 시야(`VisionConeRenderTarget`)와 정보(`InformationRenderTarget`) 렌더 타겟을 함께 관리합니다.
```cpp
USTRUCT(BlueprintType)
struct FGunnerFogOfWarRenderTargets
{
    GENERATED_BODY()

    UPROPERTY(BlueprintReadOnly)
    TObjectPtr<UCanvasRenderTarget2D> VisionConeRenderTarget;

    UPROPERTY(BlueprintReadOnly)
    TObjectPtr<UCanvasRenderTarget2D> InformationRenderTarget;
};
```
    

### 플레이어별 렌더 타겟 생성 및 관리

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
    

### 시야 및 정보 렌더링

`UGunnerFogOfWarComponent`는 `UActorComponent`로서 플레이어 캐릭터에 부착되어 시야 계산 및 미니맵 정보 렌더링을 담당합니다.

### 플레이어 팀 변경 시 델리게이트 등록/해제

이 함수는 플레이어의 `PlayerState`를 받아, 팀 변경 시 이전 팀의 렌더 타겟 델리게이트를 해제하고 새로운 팀의 렌더 타겟 델리게이트를 등록하여 시야/정보가 올바른 렌더 타겟에 그려지도록 관리합니다.

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

### 맵 지오메트리 기반 시야 폴리곤 렌더링

이 함수는 `UCanvasRenderTarget2D` 업데이트 델리게이트를 통해 호출되며, 플레이어 시야 영역을 `VisionConeRenderTarget`에 그립니다.

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
    

### `DrawInformation`: 플레이어 아이콘 (자신, 아군, 적군) 렌더링

이 함수는 `InformationRenderTarget`에 플레이어 자신, 아군, 시야 내 적군 캐릭터 아이콘을 그립니다.

*   **`DrawPlayerIcon`**: 플레이어 아이콘을 그리는 헬퍼 함수로, 위치, 방향, 팀에 따라 다른 머티리얼을 사용합니다.
    
*   **`LineTraceSingleByChannel`을 이용한 시야 내 적군 감지**: `UGameplayStatics::GetAllActorsOfClass`로 모든 캐릭터를 가져온 후, 플레이어 눈 위치에서 각 캐릭터까지 Line Trace를 수행합니다. 아군은 무시하고, 시야 각도(71.0f) 내의 캐릭터만 고려하여 성능을 최적화합니다. 히트된 액터가 대상 캐릭터와 동일한 경우에만 아이콘을 그립니다.
    

### 데이터 모델: 맵 지오메트리 구조화

`GunnerMiniMapData.h`는 맵 지오메트리 데이터 구조화를 위한 `USTRUCT`들을 정의합니다.

*   `FGunnerGeometryVertex`: 맵 정점(X, Y 좌표) 정의.
    
*   `FGunnerGeometryLine`: 두 정점(`Start`, `End` 인덱스)을 연결하는 선분 정의.
    
*   `FGunnerGeometryGroup`: 정점, 선분, Z 높이(`ZHeight`)를 포함하는 논리적 맵 지오메트리 그룹 정의. `ZHeight`는 시야 차단 구현에 사용됩니다.
    

### `UGunnerMapGeometryData`를 통한 데이터 에셋 관리

`UGunnerMapGeometryData`는 `UDataAsset`을 상속받아 `FGunnerGeometryGroup` 배열을 포함합니다. 이를 통해 레벨 디자이너/아티스트가 에디터에서 맵 지오메트리 데이터를 에셋으로 관리하고, `UGunnerFogOfWarComponent`에서 참조하여 시야 계산에 활용합니다.

### UI 연동: 위젯 블루프린트 및 머티리얼

C++ 컴포넌트에서 효율적으로 처리된 미니맵 및 안개 시스템 로직의 결과를 UI에 효과적으로 표시하기 위해, 언리얼 엔진의 UMG(Unreal Motion Graphics) 시스템과 연동했습니다. `WBP_Minimap` 위젯 블루프린트는 C++ 렌더 타겟을 미니맵 UI에 표시하며, 플레이어 움직임에 따라 아이콘을 동적으로 업데이트하는 역할을 담당합니다.

### `WBP_Minimap`의 핵심 컴포넌트 및 역할

`WBP_Minimap` 위젯은 효율적인 미니맵 표시를 위해 주로 `MinimapImage`와 `MinimapInformationOverlay` 컴포넌트를 활용합니다.

*   `MinimapImage` (`UImage`): 미니맵 배경이며, `UGunnerFogOfWarComponent`에서 실시간으로 그려지는 `VisionConeRenderTarget`와 `InformationRenderTarget`를 배경으로 표시합니다. 초기화 시 동적 머티리얼 인스턴스(MID)를 브러시로 설정하여 렌더 타겟 내용을 받아옵니다.
    
*   `MinimapInformationOverlay` (`UOverlay`): 플레이어 캐릭터 아이콘 등 동적 오브젝트 아이콘을 추가/관리하는 데 사용됩니다. `UGunnerFogOfWarComponent` 계산 결과에 따라 아이콘이 업데이트됩니다.
    

### `MinimapDynamicMaterialInstance` 생성 및 `MinimapImage`에 적용

`WBP_Minimap`의 PreConstruct 또는 Construct 이벤트(블루프린트 기준)에서 `MinimapMaterial`을 기반으로 `MinimapDynamicMaterialInstance`를 생성합니다. 이 MID는 런타임에 파라미터를 변경할 수 있으며, `UGunnerGameState`가 관리하는 `UCanvasRenderTarget2D`를 텍스처 파라미터로 받아 미니맵에 표시합니다.

    // WBP_Minimap 블루프린트의 Event Graph (PreConstruct 또는 Construct)
    // MinimapMaterial로부터 동적 머티리얼 인스턴스를 생성하고, MinimapImage의 브러시로 설정
    // K2Node_CallMaterialParameterCollectionFunction_0: CreateDynamicMaterialInstance (동적 머티리얼 인스턴스 생성)
    // K2Node_VariableSet_1: MinimapDynamicMaterialInstance 변수에 할당
    // K2Node_CallFunction_1: SetBrushFromMaterial (MinimapImage에 적용)
    

### Tick 이벤트에서 플레이어 위치/회전을 머티리얼 파라미터로 전달하여 미니맵 아이콘 업데이트

`WBP_Minimap`의 Tick 이벤트는 매 프레임 호출되어 미니맵 동적 요소를 업데이트합니다. 여기서는 플레이어 캐릭터의 월드 위치와 회전 정보를 가져와 `MinimapDynamicMaterialInstance`의 머티리얼 파라미터로 전달합니다.

*   **플레이어 위치 및 회전 정보 획득**: `GetOwningPlayerPawn` 함수로 현재 위젯을 소유한 플레이어 폰을 가져와 액터 위치(`K2_GetActorLocation`)와 컨트롤 회전(`GetControlRotation`)을 얻습니다.
    
*   **좌표 변환 및 머티리얼 파라미터 전달**:
    
    *   월드 좌표계 플레이어 위치(X, Y)는 미니맵 텍스처의 0~1 UV 좌표로 변환되어 `NormalizedPosition VectorParameter`로 `MinimapDynamicMaterialInstance`에 전달됩니다.
        
    *   플레이어의 Yaw 회전 값은 미니맵 아이콘 방향을 결정하며, `Rotation ScalarParameter`로 머티리얼에 전달됩니다.
        

    // WBP_Minimap 블루프린트의 Event Graph (Tick 이벤트)
    // 플레이어 폰의 위치를 가져와 MinimapDynamicMaterialInstance의 NormalizedPosition 파라미터로 설정
    // K2Node_CallFunction_48: GetOwningPlayerPawn
    // K2Node_CallFunction_47: K2_GetActorLocation
    // K2Node_PromotableOperator_21: 위치 조정 (예: 맵 원점 기준)
    // K2Node_PromotableOperator_19: 스케일 조정
    // K2Node_PromotableOperator_20: 중앙 정렬
    // K2Node_PromotableOperator_18: 최종 위치 조정
    // K2Node_CallFunction_56: SetVectorParameterValue (NormalizedPosition)
    
    // 플레이어 컨트롤 회전(Yaw)을 가져와 MinimapDynamicMaterialInstance의 Rotation 파라미터로 설정
    // K2Node_CallFunction_53: GetOwningPlayer
    // K2Node_CallFunction_54: GetControlRotation (Yaw 값 추출)
    // K2Node_PromotableOperator_22: 회전 값 조정 (예: 360도 보정)
    // K2Node_CallFunction_59: SetScalarParameterValue (Rotation)
    

이러한 UI 연동 방식을 통해 C++에서 계산된 시야 및 정보 데이터가 실시간으로 미니맵 위젯에 반영되며, 플레이어는 직관적으로 전장 상황을 파악할 수 있습니다. 다음 섹션에서는 이 시스템 구현 시 겪었던 주요 도전 과제와 해결 방안을 설명합니다.

## 3. 최적화: Raycasting 및 Line Tracing의 성능 부하


**도전 과제**: `UGunnerFogOfWarComponent::DrawVision`에서 시야 계산을 위해 맵 지오메트리 데이터에 수많은 Raycasting과 Line Tracing을 수행해야 했습니다. 특히 매 프레임 연산 시 게임 성능에 심각한 병목 현상을 초래할 수 있었습니다. `DrawInformation`에서 시야 내 적군 감지 시 `LineTraceSingleByChannel` 또한 잠재적 성능 저하 요인이었습니다.

**해결 방안**:

*   **RayCount 조절**: `DrawVision`에서 발사하는 Ray 개수(`RayCount`)를 적절히 조절하여 계산량을 최적화했습니다. 저희는 64개의 Ray를 사용하여 시각적 품질과 성능의 균형을 찾았습니다.
    
*   **불필요한 트레이스 방지**: `DrawInformation`에서 적군 감지 시, 플레이어 시야 각도(71.0f) 내 액터만 Line Trace를 수행하도록 필터링 로직을 추가했습니다. `CollisionParams.AddIgnoredActors`로 아군 액터와의 불필요한 충돌 검사를 건너뛰어 연산량을 줄였습니다.
    
*   **Z 높이 기반 필터링**: `DrawVision`에서 시야를 차단하는 `EdgeSegments` 검사 시, 플레이어 Z 높이와 Edge ZHeight 차이가 일정 임계값(125.0f)을 초과하는 Edge는 계산에서 제외하여 불필요한 연산을 줄였습니다.
    

## 4. 향후 개선 방향

현재 시스템은 기본적인 미니맵 및 안개 기능을 안정적으로 제공하지만, 더 풍부한 게임 플레이 경험을 위해 다음과 같은 개선/확장 방향을 고려할 수 있습니다.

*   **다른 오브젝트 표시**: 현재 플레이어 캐릭터 아이콘 위주이지만, 추후 게임 내 기능 확장에 따라 스킬, 구조물, 핑 등 다른 오브젝트를 추가 표시할 계획입니다.
    
*   **미니맵 UI/UX 개선**: 확대/축소 기능, 맵 마커 커스터마이징, 다양한 맵 스타일 지원 등을 통해 사용자 편의성을 높일 수 있습니다.
    
*   **안개 시스템 시각 효과 강화**: 안개가 걷히는 효과, 어두운 영역의 표현 방식 등을 개선하여 시각적인 몰입도를 향상시킬 수 있습니다.
    

## 5. 마치며

이번 미니맵 및 안개 시스템 구현은 언리얼 엔진의 렌더 타겟, C++ 드로잉 API, UMG 위젯 시스템에 대한 깊이 있는 이해를 요구하는 도전적인 프로젝트였습니다. 이 과정에서 얻은 문제 해결 경험과 기술 노하우, 특히 `UCanvasRenderTarget2D` 활용 및 최적화 방안은 향후 게임 개발에 큰 자산이 될 것입니다.

이 글이 유사 시스템 구현을 목표하는 개발자분들께 영감을 주거나 도움이 되기를 바랍니다. 궁금한 점이나 피드백은 언제든지 환영합니다. 감사합니다.