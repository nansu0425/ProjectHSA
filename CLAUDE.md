# CLAUDE.md — HSA 프로젝트 작업 지침

이 문서는 Claude Code가 **HSA(Heroes Survival Arena)** 코드 작업 시 따라야 할 프로젝트 규약이다.
프로젝트 개요·게임 소개·core loop는 [README](README.md), 단계별 구현 순서는 [ROADMAP](Documentation/ROADMAP.md)을 본다.

> 사용자 응대 언어, git commit 승인 게이트 등 상시 개인 규칙은 글로벌 `~/.claude/CLAUDE.md`에 있다. 여기엔 중복 기재하지 않는다.

## 프로젝트 개요

- **UE 5.7 / C++**, 단일 runtime module `HSA`, 타겟 플랫폼 PC/Windows.
- **목적**: UE5 Gameplay Systems 프로그래밍 학습. 아트가 아니라 gameplay 프로그래밍이 본질.
- **현재 단계**: ROADMAP 기준 **Phase 0(프로젝트 기반)** 진행 중 — 아직 기본 boilerplate만 존재하고 gameplay 코드는 없다.
- 작업을 시작하기 전 ROADMAP에서 **현재 Phase의 범위**를 확인하고, 그 범위를 벗어나는 선행 구현을 임의로 하지 않는다.

## 빌드 / 검증 정책 (중요)

- **Claude은 C++ 코드 작성·수정까지만 한다.** 빌드/컴파일/Live Coding/에디터 작업은 **사용자가 수행**한다.
- 코드 변경 후 "컴파일이 통과한다"고 단정하지 않는다. 빌드는 사용자 몫이며, 사용자가 전달하는 빌드 결과(에러)를 받아 다음 작업을 이어간다.
- 빌드 명령은 *참고용*이며, **engine 설치 경로는 추측해 적지 않는다**(사용자가 알려주면 그때 보강). 일반형:
  - `"<UE_ROOT>/Engine/Build/BatchFiles/Build.bat" HSAEditor Win64 Development -project="<프로젝트 경로>/HSA.uproject"`
  - 경로는 사용자 환경에 맞게 사용자가 채운다.

## 편집 금지 / 생성 산출물

- 절대 편집하지 않는다: `Intermediate/`, `Binaries/`, `Saved/`, `DerivedDataCache/`, `.vs/`.
- `*.generated.h` 등 UHT 생성물은 직접 수정하지 않는다 — 헤더의 `UCLASS/UPROPERTY/GENERATED_BODY`를 고치면 빌드 시 재생성된다.
- `.uasset` / `.umap`은 binary라 텍스트로 편집할 수 없다. 내용 변경은 **에디터 작업**으로 사용자에게 위임한다.

## C++ ↔ Blueprint / 에디터 경계

- **Claude이 하는 것**: `UCLASS`/`USTRUCT`/`UENUM`/`UFUNCTION` 기반 C++ 클래스와 로직, `UPROPERTY(EditAnywhere/BlueprintReadWrite)`로 데이터 노출.
- **에디터에서(사용자) 하는 것**: Blueprint 파생, `InputMappingContext`/`InputAction` asset, `DataAsset` 인스턴스, Animation Blueprint, 레벨 배치, 플러그인 활성화.
- 따라서 **asset 참조(경로/GUID)를 추측해 하드코딩하지 않는다.** C++에선 `UPROPERTY`로 노출해 에디터에서 지정하게 하고, 직접 로드가 필요하면 soft reference(`TSoftObjectPtr`/`TSoftClassPtr`)를 우선한다.

## 코드 컨벤션

- **네이밍**: UE 접두사 규약을 따른다 — `A`(Actor), `U`(UObject), `F`(struct), `E`(enum), `I`(interface), `b`(bool). 그 위에 프로젝트 접두사 `HSA`를 붙인다.
  - 예: `AHSACharacter`, `AHSAPlayerController`, `UHSAAttributeSet`, `AHSAGameMode`, `AHSAGameState`, `AHSAEnemy`, `UHSAHeroData`. (ROADMAP에 등장하는 이름과 일치시킨다.)
- **폴더 구조**(Phase 0에서 확립): `Source/HSA/Public`과 `Source/HSA/Private`를 분리하고, 그 아래 **시스템별 디렉터리**를 둔다(예: `Characters/`, `Abilities/`, `Heroes/`, `Enemies/`, `Wave/`, `UI/`). 새 클래스는 이 구조를 따른다.
  - public 헤더(`.h`)는 `Public/`, 구현(`.cpp`)은 `Private/`에 둔다.
- **include 위생**: `#include`는 최소화하고, 헤더에서는 전방선언(forward declaration)을 우선한다.
- **UE 관용구**: raw `new`/STL 컨테이너 대신 UE 타입을 쓴다(`TArray`/`TMap`/`TSet`, `UPROPERTY` 포인터는 `TObjectPtr`). Epic C++ 코딩 표준(중괄호는 새 줄)을 따른다.

## 설계 원칙 (ROADMAP 3원칙 — 상시 guardrail)

모든 작업에서 아래를 지킨다. 배경·근거는 [ROADMAP](Documentation/ROADMAP.md)의 "핵심 원칙" 참고.

1. **GAS 우선** — 전투·능력·버프·데미지 등 전투/능력 관련 시스템은 GAS로 구현한다(ability / effect / attribute / gameplay tag).
2. **"멀티 전환 가능한 싱글플레이"** — 처음엔 standalone(싱글)으로 만들되 멀티 전환을 의식해 설계한다:
   - `HasAuthority()` 분기를 가정한 구조로 작성한다.
   - 클라이언트 **입력 처리**와 서버 **권위 로직(authoritative logic)** 을 분리한다(입력은 `PlayerController`에서 받아 위임).
   - 게임 상태는 `GameState`에, 플레이어 상태는 `PlayerState`에 둔다.
   - GAS의 replication / prediction을 나중에 켤 수 있게 설계한다.
3. **ASC 부착 위치** — player의 `UAbilitySystemComponent`는 **`PlayerState`** 에 둔다(멀티 정석: respawn·관전 대비). pawn 수명과 함께 사라져도 되는 적은 pawn에 둬도 무방하다.
4. **Asset은 placeholder/무료** — 직접 제작은 최소화한다(기본 도형, Epic/Fab 무료 에셋).

## 현재 의존성 상태 (작업 시 주의)

- 현재 의존성([Source/HSA/HSA.Build.cs](Source/HSA/HSA.Build.cs)): `Core, CoreUObject, Engine, InputCore, EnhancedInput`.
- 활성 플러그인([HSA.uproject](HSA.uproject)): `ModelingToolsEditorMode`만. **`GameplayAbilities`(GAS) 플러그인/모듈은 아직 미활성**이다.
- GAS 작업(Phase 2~)에 진입하려면 선행으로:
  - (a) [HSA.uproject](HSA.uproject) plugins에 `GameplayAbilities` 추가,
  - (b) [HSA.Build.cs](Source/HSA/HSA.Build.cs)에 `GameplayAbilities`, `GameplayTags`, `GameplayTasks` 추가.
- `.uproject` / `Build.cs` 수정은 빌드에 미치는 영향이 크므로, **변경 전 사용자에게 알리고 진행**한다.
