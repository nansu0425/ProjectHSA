# Phase 1 작업 계획 — 플레이어 캐릭터 기반

이 문서는 [ROADMAP](ROADMAP.md)의 **Phase 1(플레이어 캐릭터 기반)** 을 실행 가능한 체크리스트로 푼 것이다.
전체 단계 흐름·원칙은 ROADMAP을, 작업 규약(빌드 정책·편집 금지·컨벤션)은 [CLAUDE.md](../CLAUDE.md), C++ 스타일은 [CODING-STANDARD](CODING-STANDARD.md)를 본다.

> **담당 표기**: 각 작업에 `[Claude]`(코드/설정 파일 작성·수정) / `[사용자]`(빌드·에디터·에셋 작업)를 명시한다.
> Claude은 C++/설정 파일 작성까지만 하고, **빌드·컴파일·에디터 작업은 사용자가 수행**한다(CLAUDE.md 빌드 정책).

---

## 1. 목표

조작 가능한 third-person 캐릭터를 만든다.
완료 시 → 맵에서 **이동/시점 조작/점프** 가 되고, locomotion 애니메이션이 재생되는 캐릭터가 있는 상태.

---

## 2. 전제 (착수 전 이미 갖춰진 것)

Phase 1에서 **건드리지 않는 불변 사실**이다. 작업 **진행 상태**는 여기가 아니라
**§3 체크리스트가 유일한 출처(single source of truth)** 다.

| 전제 | 근거 |
|------|------|
| UE 5.7 단일 `HSA` runtime 모듈, `Public`/`Private` 구조 | [HSA.uproject](../HSA.uproject), [Public/HSA.h](../Source/HSA/Public/HSA.h) |
| `EnhancedInput` 의존성 + 기본 입력 클래스 설정 | [HSA.Build.cs](../Source/HSA/HSA.Build.cs), [DefaultInput.ini](../Config/DefaultInput.ini) (`DefaultPlayerInputClass`/`DefaultInputComponentClass`=EnhancedInput) |
| 테스트 레벨 `L_TestArena` + 기본 맵 지정 | [DefaultEngine.ini](../Config/DefaultEngine.ini) (`GameDefaultMap`/`EditorStartupMap`) |
| GAS 의존성·플러그인 활성화 (Phase 1에선 사용 안 함, Phase 2부터) | [HSA.Build.cs](../Source/HSA/HSA.Build.cs), [HSA.uproject](../HSA.uproject) |

> `InputMappingContext`/`InputAction` **에셋**, 캐릭터 메시/애니는 아직 없다 → Phase 1에서 만든다.
> GameMode는 아직 없어 `GlobalDefaultGameMode`가 미지정이다 → Phase 1에서 `AHSAGameMode`를 둔다.

---

## 3. 작업 체크리스트 (순서대로)

### 작업 1 — 시스템별 디렉터리 생성

- [ ] `[Claude]` `Source/HSA/{Public,Private}/Characters` 생성(`AHSACharacter`).
- [ ] `[Claude]` `Source/HSA/{Public,Private}/Player` 생성(`AHSAPlayerController`).
- [ ] `[Claude]` `Source/HSA/{Public,Private}/Core` 생성(`AHSAGameMode`).

> Phase 0 규약대로 시스템별 디렉터리는 **해당 클래스가 처음 생기는 Phase에서** 만든다(빈 폴더는 git에 추적되지 않으므로 클래스 파일과 함께 생성).

### 작업 2 — `AHSACharacter` (입력 처리 포함)

- [ ] `[Claude]` `ACharacter` 기반 `AHSACharacter` 작성.
- [ ] `[Claude]` `USpringArmComponent` + `UCameraComponent` 구성(3인칭 카메라).
- [ ] `[Claude]` `SetupPlayerInputComponent()`에서 IMC를 `UEnhancedInputLocalPlayerSubsystem`에 등록 + `IA_Move`/`IA_Look`/`IA_Jump`를 `BindAction`으로 바인딩·처리.
  - Move: 카메라(컨트롤러 yaw) 기준 방향 변환 후 `AddMovementInput`.
  - Look: `AddControllerYawInput`/`AddControllerPitchInput`.
  - Jump: `ACharacter::Jump`/`StopJumping`.
- [ ] `[Claude]` IMC/IA·mesh/ABP는 C++에서 경로 하드코딩하지 않고 `UPROPERTY(EditDefaultsOnly)`로 노출(에셋은 BP에서 할당 — 작업 7).
- [ ] `[사용자]` 빌드로 검증.

> **입력을 Character에 두는 이유는 §6 설계 노트 참조** — 이동/시점/점프는 Pawn 종속 입력이다.

### 작업 3 — `AHSAPlayerController` (전역 입력 거점)

- [ ] `[Claude]` `APlayerController` 기반 `AHSAPlayerController` 작성(Phase 1엔 **최소 골격**).
- [ ] `[Claude]` 클래스 주석에 "**Pawn 비종속(전역) 입력의 거점** — 메뉴·Pause·관전·리스폰 등은 이후 Phase에서 여기에 둔다"를 명시.

> Phase 1엔 전역 입력이 아직 없어 입력 바인딩은 비어 있다. 이 클래스를 지금 두는 이유는 ① GameMode가 지정할 `PlayerControllerClass` 자리 확보 ② 향후 전역 입력의 확장 지점 명문화다.

### 작업 4 — `AHSAGameMode`

- [ ] `[Claude]` `AGameModeBase` 기반 `AHSAGameMode` 작성.
- [ ] `[Claude]` 생성자에서 `DefaultPawnClass`/`PlayerControllerClass` 기본값 지정(실제 BP 에셋 연결은 작업 7).

> 멀티 대비 상태 배치(게임 상태는 `GameState`, 플레이어 상태는 `PlayerState`)는 그 상태가 실제로 생기는 Phase(5~)에서 다룬다. Phase 1은 GameMode로 Pawn/Controller 클래스를 묶는 데까지.

### 작업 5 — EnhancedInput 에셋

- [ ] `[사용자]` `IMC_Default`(InputMappingContext) 생성.
- [ ] `[사용자]` `IA_Move`(Axis2D) / `IA_Look`(Axis2D) / `IA_Jump`(bool) 생성.
- [ ] `[사용자]` IMC에 키 매핑: 이동(WASD, Negate/Swizzle modifier), 시점(Mouse XY, Look Y축 Negate), 점프(Space).

> IA의 value type·modifier는 binary 에셋이라 에디터 작업이다. C++ 쪽은 IA를 `UPROPERTY`로 참조만 하고 에셋은 BP에서 할당한다(작업 7).

### 작업 6 — 캐릭터 에셋 import (순수 에셋만)

- [ ] `[사용자]` Manny/Quinn **순수 에셋만** import — skeletal mesh + locomotion animation(idle/walk/run/jump) + Animation Blueprint.
- [ ] `[사용자]` **게임플레이 C++ 콘텐츠 팩은 제외**.

> Phase 0에서 C++ 기반 Third Person 콘텐츠 팩이 게임플레이 C++ 40여 클래스를 끌고 와 빌드를 깨고 HSA 자체 설계와 충돌해 롤백한 이력이 있다. 따라서 **메시·애니·ABP 같은 순수 에셋만 선별**한다.
> 방안: 임시 ThirdPerson 템플릿 프로젝트를 만들어 `Content/Characters/Mannequins`만 Migrate(BP·C++ 클래스 제외), 또는 Fab 무료 캐릭터.

### 작업 7 — BP·config 연결

- [ ] `[사용자]` `BP_HSACharacter`(`AHSACharacter` 파생) 생성 — SKM·ABP·IMC/IA 에셋 할당.
- [ ] `[사용자]` `AHSAGameMode`(또는 BP)에 `DefaultPawnClass=BP_HSACharacter`, `PlayerControllerClass=AHSAPlayerController` 연결.
- [ ] `[사용자]` `L_TestArena`의 World Settings 또는 config(`GlobalDefaultGameMode`)에 GameMode 지정.

> mesh/ABP/IMC/IA는 C++에 경로를 추측해 하드코딩하지 않는다(Phase 0 규약). C++가 노출한 `UPROPERTY`에 에디터에서 에셋을 할당한다.

### 작업 8 — 빌드·PIE 검증

- [ ] `[사용자]` 빌드 통과(→ §4).
- [ ] `[사용자]` PIE 실행 — 이동/시점/점프·애니메이션 확인(→ §4).

---

## 4. 검증

1. `[사용자]` **빌드 통과** — 일반형(경로는 사용자 환경):
   `"<UE_ROOT>/Engine/Build/BatchFiles/Build.bat" HSAEditor Win64 Development -project="<프로젝트경로>/HSA.uproject"`
2. `[사용자]` **PIE 실행 검증**:
   - WASD 이동(카메라 기준 방향), 마우스 시점 회전, Space 점프가 동작.
   - locomotion 애니메이션(idle/walk/run/jump)이 상태에 맞게 재생.
   - 스폰된 Pawn이 `BP_HSACharacter`이고 GameMode가 적용됨(다른 기본 Pawn이 아님).

빌드 에러가 나면 Claude이 그 결과를 받아 다음 작업을 이어간다(컴파일 통과를 단정하지 않음).

---

## 5. 마일스톤 (ROADMAP Phase 1)

> 거친 완료 기준 요약일 뿐, 세부 진행 상태는 **§3 체크리스트(SSOT)** 를 본다.

- [ ] 맵에서 이동/시점 조작/점프가 되는 캐릭터.
- [ ] locomotion 애니메이션이 연동된 상태.

---

## 6. 설계 노트

### 6.1 입력 성격별 분리 — Pawn 종속 입력은 Character, 전역 입력은 PlayerController

입력을 어디서 처리할지의 기준은 **"그 입력이 특정 Pawn에 종속적인가"** 다.

- **Pawn 종속 입력**(이동·점프·시점·스킬 등 그 Pawn의 행동) → **`AHSACharacter`**.
- **Pawn 비종속(전역) 입력**(메뉴·Pause·관전·리스폰 등) → **`AHSAPlayerController`**.

Phase 1 입력(Move/Look/Jump)은 전부 Pawn 종속이라 모두 Character에 두고, PlayerController는 전역 입력이 등장하는 후속 Phase의 거점으로 비워 둔다.

같은 `PlayerController`가 서로 다른 Pawn(영웅 A/B, 향후 탈것 등)을 possess하고, "이동"·"스킬"의 구체적 의미는 possess한 Pawn마다 다르다(걷는 캐릭터의 이동 ≠ 탈것의 조향, 영웅 A의 스킬1 ≠ 영웅 B의 스킬1). 그래서 **입력의 의미가 Pawn마다 달라지면 Pawn(Character)에, Pawn과 무관하면 PlayerController에** 둔다.

입력을 **Pawn에 두면** possess 교체에 입력 행동이 **다형성으로** 따라오고, Controller는 "지금 어떤 Pawn인가"를 `Cast`+분기로 떠안지 않는다 (개방-폐쇄 원칙).

이 분리가 가져오는 **부수 이점**(분류 기준이 아니라 결과적 이득):

- **입력 스택 자동 관리** — 엔진은 매 프레임 입력 스택을 구성할 때 **현재 possess된 Pawn의 InputComponent를 자동 포함**한다. 그래서 Pawn 종속 입력을 Pawn에 두면 possess/unpossess에 맞춰 그 입력의 활성/비활성이 자동으로 따라간다(별도 등록·해제 코드 불필요).
- **수명 일치** — 전역 입력은 Pawn이 없을 때(사망·관전·리스폰 등)도 유효해야 하므로, Pawn보다 오래 사는 Controller에 두면 안정적이다.

### 6.2 Look(시점) 처리 위치

시점 해석은 Pawn마다 다르므로(3인칭 캐릭터 / 탈것 / 1인칭) 시점도 **Pawn 종속 입력**이다 → 다형성 기준상 Character가 입력을 받는다. 시점 상태 자체(컨트롤러 rotation)는 Controller가 소유하지만, Character는 그것을 직접 바꾸지 않고 엔진이 제공하는 위임 통로(`AddControllerYawInput`/`AddControllerPitchInput`, `APawn` 함수)로 회전 의도만 넘긴다. 카메라가 그 시점을 어떻게 따라가는지는 `USpringArmComponent`의 역할이며 입력 처리 위치와는 무관하다.

### 6.3 C++ / BP 분리

mesh·ABP·IMC/IA 같은 에셋 참조는 C++에 경로/GUID로 하드코딩하지 않는다. C++는 `UPROPERTY(EditDefaultsOnly)`로 슬롯만 노출하고, 실제 에셋은 BP에서 할당한다(Phase 0 "에셋 경로 추측 하드코딩 금지"와 정합).

---

## 7. 범위 밖 (Phase 1에서 하지 않음)

- ASC/`UHSAAttributeSet`/`GameplayEffect`/ability·**스킬 입력** → Phase 2~3.
- 영웅별 분기·`UHSAHeroData` → Phase 3.
- 적·AI → Phase 4.
- 메뉴·관전 등 **전역 입력 실제 구현** → 그 입력이 필요한 Phase(5~6). Phase 1은 PlayerController에 거점만 마련한다.
- replication 실제 활성화(server RPC·`GetLifetimeReplicatedProps` 등) → Phase 7. Phase 1은 "분리된 구조"까지만, 네트워크 코드는 넣지 않는다.
