# Phase 0 작업 계획 — 프로젝트 기반

이 문서는 [ROADMAP](ROADMAP.md)의 **Phase 0(프로젝트 기반)** 을 실행 가능한 체크리스트로 푼 것이다.
전체 단계 흐름·원칙은 ROADMAP을, 작업 규약(빌드 정책·편집 금지·컨벤션)은 [CLAUDE.md](../CLAUDE.md)를 본다.

> **담당 표기**: 각 작업에 `[Claude]`(코드/설정 파일 작성·수정) / `[사용자]`(빌드·에디터·에셋 작업)를 명시한다.
> Claude은 C++/설정 파일 작성까지만 하고, **빌드·컴파일·에디터 작업은 사용자가 수행**한다(CLAUDE.md 빌드 정책).

---

## 1. 목표

빌드/모듈/입력 토대를 정리하고 **GAS 사용 준비**를 마친다.
완료 시 → GAS 모듈이 링크된 채 컴파일되고, 실행 가능한 테스트 레벨이 있는 상태.

---

## 2. 전제 (착수 전 이미 갖춰진 것)

Phase 0에서 **건드리지 않는 불변 사실**이다. 작업 **진행 상태**는 여기가 아니라
**§3 체크리스트가 유일한 출처(single source of truth)** 다 — 상태를 한 곳에서만 관리해 불일치를 원천 차단한다.

| 전제 | 근거 |
|------|------|
| UE 5.7 단일 `HSA` runtime 모듈 | [HSA.uproject](../HSA.uproject), [Private/HSA.cpp](../Source/HSA/Private/HSA.cpp) |
| `EnhancedInput` 의존성 | [HSA.Build.cs](../Source/HSA/HSA.Build.cs) |
| EnhancedInput 기본 입력 클래스 설정 | [DefaultInput.ini](../Config/DefaultInput.ini) (`DefaultPlayerInputClass=EnhancedPlayerInput` 등) |

> EnhancedInput `InputMappingContext`/`InputAction` **에셋** 제작은 Phase 0이 아니라
> 실제 입력이 필요한 **Phase 1**에서 한다(Phase 0은 모듈/의존성 토대까지).

---

## 3. 작업 체크리스트 (순서대로)

### 작업 1 — 폴더 구조 컨벤션 확립 (`Public`/`Private`)

- [x] `[Claude]` `Source/HSA/Public/`, `Source/HSA/Private/` 생성.
- [x] `[Claude]` `HSA.h` → [Public/HSA.h](../Source/HSA/Public/HSA.h), `HSA.cpp` → [Private/HSA.cpp](../Source/HSA/Private/HSA.cpp) 이동.
- [x] `[사용자]` 빌드로 검증.

**시스템별 디렉터리(`Characters/`, `Abilities/` 등)는 미리 만들지 않는다** — 해당 클래스가 처음 생기는
Phase에서 그때 만든다(빈 폴더는 git에 추적되지도 않음).

> **근거**: UBT가 `Public`/`Private` 폴더명을 자동으로 public/private include path로 인식해
> 모듈 API 경계를 만든다. HSA는 현재 단일 모듈이라 기능적 필수는 아니지만, ROADMAP/CLAUDE.md 컨벤션이고
> 후일 모듈 분리·외부 참조 시 재배치 비용을 없애기 위해 Phase 0에서 확립한다. (→ §5 설계 노트)

### 작업 2 — `GameplayAbilities` 플러그인 활성화

- [x] `[Claude]` [HSA.uproject](../HSA.uproject) `Plugins`에 `GameplayAbilities` 항목 추가 (`Enabled: true`).
- [x] `[사용자]` 에디터에서 플러그인 활성화 확인(또는 에디터 *Plugins* 창에서 직접 켜기).

> `.uproject` 수정은 빌드 영향이 크다 → CLAUDE.md 정책상 **변경 전 사용자에게 알리고 진행**한다.

### 작업 3 — `Build.cs` 의존성 추가

- [x] `[Claude]` [HSA.Build.cs](../Source/HSA/HSA.Build.cs) `PublicDependencyModuleNames`에
      `GameplayAbilities`, `GameplayTags`, `GameplayTasks` 추가.
- [x] `[사용자]` 빌드로 GAS 링크 검증 (→ §4).

> GAS 코드(`UAbilitySystemComponent` 등)를 쓰려면 플러그인 활성화(작업 2)와 의존성 선언(작업 3)이 **둘 다** 필요하다.

### 작업 4 — 테스트 레벨 + Epic 무료 에셋

- [ ] `[사용자]` 빈 맵 + 바닥/조명으로 테스트 레벨 생성, 기본 맵으로 지정.
- [ ] `[사용자]` Epic 무료 에셋(Manny/Quinn 등) 프로젝트에 추가.

> 에셋은 binary(.uasset/.umap)라 에디터 작업이다. C++에서 에셋 **경로/GUID를 추측해 하드코딩하지 않는다**.

---

## 4. 검증

1. `[사용자]` **빌드 통과** — 일반형(경로는 사용자 환경):
   `"<UE_ROOT>/Engine/Build/BatchFiles/Build.bat" HSAEditor Win64 Development -project="<프로젝트경로>/HSA.uproject"`
2. `[사용자/Claude]` **GAS 링크 확인** — 빌드 성공이면 충분. 더 확실히 하려면 임시로
   `#include "AbilitySystemComponent.h"`가 컴파일되는지 확인 후 제거.
3. `[사용자]` **테스트 레벨 PIE 실행** — 에디터에서 정상 실행.

빌드 에러가 나면 Claude이 그 결과를 받아 다음 작업을 이어간다(컴파일 통과를 단정하지 않음).

---

## 5. 마일스톤 (ROADMAP Phase 0)

> 거친 완료 기준 요약일 뿐, 세부 진행 상태는 **§3 체크리스트(SSOT)** 를 본다.

- [x] GAS 모듈이 링크된 상태로 컴파일되는 프로젝트.
- [ ] 실행 가능한 테스트 레벨.

---

## 6. 설계 노트 — `Public`/`Private` 분리

- **모듈**: UE에서 모듈은 C++ 컴파일 단위(빌드 결과 DLL 하나). `HSA`가 이 프로젝트의 단일 runtime 모듈이다.
- **UBT의 폴더 인식**: 모듈의 `Source/<Module>/Public`은 자동으로 *public include path*,
  `Private`는 *private include path*가 된다. → `Public/` 헤더는 **다른 모듈에서 include 가능**,
  `Private/`는 모듈 내부 전용. 즉 폴더 위치가 곧 API 공개 범위다.
- **trade-off**: 단일 모듈에선 "다른 모듈이 우리 헤더를 include"할 일이 아직 없어 기능적 이득은 약하다.
  그럼에도 ① 컨벤션 일관성 ② 후일 에디터 전용 모듈 분리·외부 참조 대비 재배치 비용 제거를 위해 Phase 0에서 확립한다.
- **공식 표준 여부**: Epic *코딩 표준* 문서가 규정하는 건 네이밍/중괄호 같은 코드 스타일이다.
  `Public`/`Private` 폴더는 그 문서의 규정이 아니라 **UBT 모듈 레이아웃 관례**로, 엔진 소스 대부분이 따르는 사실상의 표준이다.

---

## 7. 범위 밖 (Phase 0에서 하지 않음)

- `AbilitySystemGlobals` 초기화, ASC/`UHSAAttributeSet`/`GameplayEffect` 등 → **Phase 2(GAS 토대)**.
- `InputMappingContext`/`InputAction` 에셋, 캐릭터/카메라 → **Phase 1**.
- 시스템별 디렉터리 일괄 생성 → 각 시스템이 처음 등장하는 Phase에서.
