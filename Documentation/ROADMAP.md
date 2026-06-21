# HSA 개발 로드맵 (ROADMAP)

이 문서는 **HSA(Heroes Survival Arena)** 의 구현 순서를 단계적으로 정리한 학습 로드맵입니다.
프로젝트 개요·게임 소개·core loop는 [README](../README.md)를 참고하세요.

HSA는 **UE5 Gameplay Systems 프로그래밍 실습**이 목적이므로, 로드맵은 "무엇을 만드는가"뿐 아니라
**"어떤 UE5 시스템을 배우는가"** 를 함께 추적합니다.

---

## 핵심 원칙(Guiding Principles)

이 세 원칙이 모든 phase의 설계 결정을 관통합니다.

### 원칙 1 — GAS 우선
영웅·스킬·버프·데미지 등 **전투/능력 관련 시스템은 GAS(Gameplay Ability System)로 구현**합니다.
GAS는 attribute, ability, effect, tag를 표준화된 방식으로 다루고 replication을 기본 지원하므로,
영웅마다 고유한 전투 방식을 data-driven하게 확장하기에 적합합니다.

### 원칙 2 — "멀티플레이 전환 가능한 싱글플레이"
co-op 멀티플레이는 README의 핵심 기능이지만, **처음부터 멀티플레이를 함께 구현하지 않습니다.**
처음부터 networked로 만들면 문제 발생 시 디버깅 복잡도가 과도하게 높아지기 때문입니다.

대신 **핵심 게임 시스템을 standalone(싱글플레이)으로 먼저 완성**하되,
**멀티플레이 전환을 의식하며 설계**합니다:

- `HasAuthority()` 분기를 가정한 구조로 작성한다.
- 클라이언트 **입력 처리**와 서버 **권위 로직(authoritative logic)** 을 분리한다.
- 게임 상태는 `GameState`에, 플레이어 상태는 `PlayerState`에 둔다.
- GAS를 활용해 ability/effect의 replication·prediction을 나중에 켤 수 있게 한다.

→ 핵심 시스템이 완성되면 **Phase 7**에서 co-op으로 전환합니다. 위 설계를 지켜왔다면 재작업이 최소화됩니다.

### 원칙 3 — Asset은 무료/마켓플레이스 + placeholder
이 프로젝트는 **아트가 아니라 게임플레이 프로그래밍** 학습이 목적입니다.
따라서 asset은 Epic 무료 에셋(Manny/Quinn, Paragon), Fab 무료 에셋, 기본 도형/placeholder로 충당하고
**직접 제작은 최소화**합니다. 각 phase에는 필요한 asset을 함께 적어 코드와 asset 의존성을 한눈에 봅니다.

---

## Phase 진행 흐름

```
Phase 0 ──▶ Phase 1 ──▶ Phase 2 ──▶ Phase 3 ──▶ Phase 4 ──▶ Phase 5 ──▶ Phase 6 ──▶ Phase 7 ──▶ Phase 8
 기반        캐릭터       GAS 토대     영웅         적/AI       Wave        휴식/상점    멀티플레이    폴리시
                                                                         (코어 루프    전환         /확장
                                                                          완성)       (co-op)     (지속)
        └──────────────── 싱글플레이로 핵심 시스템 완성 ────────────────┘   └ 멀티 전환 ┘
```

각 phase는 **목표 / 구현 항목 / Asset / 산출물(마일스톤) / 학습 포인트** 로 구성됩니다.
마일스톤은 체크박스로 진행을 추적합니다.

---

## Phase 0 — 프로젝트 기반

> 일부 완료: UE 5.7 단일 `HSA` 런타임 모듈, `EnhancedInput` 의존성.
> 실행 체크리스트·담당 구분·검증 방법은 [Phase 0 작업 계획](Phase0-Plan.md)을 본다.

- **목표:** 빌드/모듈/입력 토대를 정리하고 GAS 사용 준비를 마친다.
- **구현 항목:**
  - `HSA` 모듈 정리, 폴더 구조 컨벤션 확립(Public/Private, 시스템별 디렉터리).
  - EnhancedInput `InputMappingContext` / `InputAction` 기본 세팅.
  - `GameplayAbilities` 플러그인 활성화.
  - `HSA.Build.cs`에 의존성 추가: `GameplayAbilities`, `GameplayTags`, `GameplayTasks`.
- **Asset:** 테스트용 기본 레벨(빈 맵 + 바닥/조명), Epic 무료 에셋(Manny/Quinn 등) 프로젝트에 추가.
- **산출물(마일스톤):**
  - [ ] GAS 모듈이 링크된 상태로 컴파일되는 프로젝트.
  - [ ] 실행 가능한 테스트 레벨.
- **학습 포인트:** UBT/모듈 구조, 플러그인 의존성, 프로젝트 디렉터리 컨벤션.

---

## Phase 1 — 플레이어 캐릭터 기반

- **목표:** 조작 가능한 third-person 캐릭터를 만든다.
- **구현 항목:**
  - `AHSACharacter`(`ACharacter` 기반), `USpringArmComponent` + `UCameraComponent`.
  - EnhancedInput 기반 이동/시점/점프.
  - `AHSAPlayerController` — 입력을 받아 캐릭터로 위임(멀티 대비 입력/로직 분리).
  - 기본 애니메이션 연동.
- **Asset:** 무료 캐릭터 skeletal mesh + locomotion animation(걷기/달리기/점프 — Manny/Quinn 또는 Paragon), Animation Blueprint.
- **산출물(마일스톤):**
  - [ ] 맵에서 이동/시점 조작/점프가 되는 캐릭터.
- **학습 포인트:** Character/Controller/Pawn 관계, EnhancedInput, 입력은 컨트롤러에서 받아 캐릭터로 위임하는 패턴.

---

## Phase 2 — GAS 토대

- **목표:** GAS 최소 구성으로 능력 1개를 입력부터 효과 적용까지 끝까지 발동한다(vertical slice).
- **구현 항목:**
  - `UAbilitySystemComponent` 부착 — 위치는 **`PlayerState` 권장**(멀티 대비 정석, 아래 *설계 노트* 참고).
  - `UHSAAttributeSet` — Health, Mana 등 기본 attribute.
  - `UGameplayEffect`로 데미지/회복 적용.
  - `UGameplayAbility` 1개 + `GameplayTag` 컨벤션 수립.
  - `AbilitySystemGlobals` 초기화.
- **Asset:** 발동 확인용 placeholder VFX/SFX(Niagara 기본 또는 무료 에셋), 디버그 표시.
- **산출물(마일스톤):**
  - [ ] 입력 → ability 발동 → effect로 attribute가 변하는 수직 슬라이스.
- **학습 포인트:** ASC, AttributeSet, GameplayEffect, GameplayTag, GAS의 prediction/authority 모델 개념.

---

## Phase 3 — 영웅 시스템

- **목표:** 영웅마다 서로 다른 고유 전투 방식을 구현한다.
- **구현 항목:**
  - `UHSAHeroData`(`UPrimaryDataAsset`) — 기본 스탯, 부여할 ability set, 메시/아이콘.
  - 영웅별 `UGameplayAbility` 세트.
  - 영웅 선택 시 ability를 부여(grant)하는 로직.
  - 간단한 영웅 선택 진입점.
- **Asset:** 영웅 2~3종 구분용 무료 메시/머티리얼(색상 변형 또는 Paragon 캐릭터), 영웅별 스킬 VFX placeholder, 영웅 아이콘.
- **산출물(마일스톤):**
  - [ ] 2~3개 영웅을 선택해 서로 다른 스킬로 플레이.
- **학습 포인트:** DataAsset 기반 data-driven 설계, ability granting, GAS 확장.

---

## Phase 4 — 적 / AI / 전투 루프

- **목표:** 적을 처치하는 전투를 성립시킨다.
- **구현 항목:**
  - `AHSAEnemy` — ASC/AttributeSet 재사용.
  - `AIController` + Behavior Tree(또는 State) — 추적·공격 로직.
  - GAS effect로 데미지 적용, 사망 처리.
  - 체력 UI(UMG) 기초.
- **Asset:** 적 무료 메시 + 이동/공격/사망 animation, 피격 VFX/SFX, 체력바 위젯 placeholder.
- **산출물(마일스톤):**
  - [ ] 적이 스폰되어 플레이어를 추적·공격하고, 플레이어가 적을 처치 가능.
- **학습 포인트:** AIController/BehaviorTree, GAS를 적 캐릭터에도 재사용, 데미지 파이프라인.

---

## Phase 5 — Wave 시스템

- **목표:** 난이도가 점점 올라가는 wave 진행을 구현한다.
- **구현 항목:**
  - `AHSAGameMode` / `AHSAGameState`에 wave 상태(웨이브 시작 / 클리어 / 종료).
  - 스폰 매니저, 난이도 스케일링(적 수·스탯).
  - wave 진행 UI.
- **Asset:** 아레나 레벨(무료 환경 에셋 또는 간단한 모듈러 키트), 스폰 포인트 마커, wave/타이머 HUD 위젯.
- **산출물(마일스톤):**
  - [ ] 웨이브 시작 → 전멸 → 다음 웨이브로 이어지는 루프.
- **학습 포인트:** GameMode/GameState 책임 분리(멀티 대비 상태는 GameState에), 스폰 시스템.

---

## Phase 6 — 휴식 시간 & 상점 (코어 루프 완성)

- **목표:** README의 core gameplay loop를 완성한다.
- **구현 항목:**
  - rest phase 전환.
  - shop UI(UMG), 통화/재화 attribute.
  - item·equipment 데이터(DataAsset).
  - 구매 → 능력/스탯 반영(GAS effect/ability grant 연계).
- **Asset:** 상점 UI 위젯 셋(패널/버튼/슬롯), item·equipment 아이콘(무료 아이콘 팩), 구매 SFX.
- **산출물(마일스톤):**
  - [ ] 웨이브 사이 상점에서 구매 후 강화된 상태로 다음 웨이브에 진입.
- **학습 포인트:** UMG, 인벤토리/장비 데이터 모델, GAS와 아이템 연동.

---

## Phase 7 — 멀티플레이 전환 (co-op)

- **목표:** 싱글플레이로 완성한 시스템을 networked co-op으로 전환한다.
- **구현 항목:**
  - replication 점검(Actor/Component replicate, `GetLifetimeReplicatedProps`).
  - GAS replication mode 설정(Mixed).
  - ability 입력의 server RPC / prediction 정리.
  - GameState/PlayerState 동기화, 스폰·wave의 서버 권위화.
  - 세션/접속(listen server 기준).
- **Asset:** 신규 asset 거의 없음 — 로비/접속 UI 위젯 placeholder 정도.
- **산출물(마일스톤):**
  - [ ] 2인 이상 co-op으로 wave 생존 플레이.
- **학습 포인트:** UE replication 모델, server-authoritative 설계, GAS 네트워크 prediction.
- **메모:** 앞선 phase에서 원칙 2("멀티 친화 설계")를 지켜왔다면 이 phase의 재작업이 최소화됩니다.

---

## Phase 8 — 폴리시 & 콘텐츠 확장 (지속)

코어 루프 + co-op 완성 이후 지속적으로 확장하는 항목:

- 영웅 / 적 / 아이템 추가.
- VFX / SFX 개선, 밸런싱.
- 메인 메뉴 / 로비 UI.
- 저장(save/load), 성능 프로파일링.

---

## 설계 노트(Design Notes)

- **ASC 부착 위치 — `PlayerState` 권장:** GAS의 ASC는 `PlayerState`에 두는 것을 권장합니다.
  멀티플레이에서 플레이어가 pawn을 잃거나 재스폰해도 attribute/ability가 유지되고,
  respawn·관전 등으로 확장하기 쉽기 때문입니다(원칙 2와 정합). 적 캐릭터처럼 pawn 수명과 함께
  사라져도 되는 경우에는 ASC를 pawn 자체에 두어도 무방합니다.
