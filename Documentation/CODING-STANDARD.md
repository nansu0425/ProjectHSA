# CODING-STANDARD.md — HSA C++ 코딩 표준

이 문서는 HSA 프로젝트의 **C++ 코딩 표준**이다. [Epic C++ Coding Standard for Unreal Engine (5.7)](https://dev.epicgames.com/documentation/unreal-engine/epic-cplusplus-coding-standard-for-unreal-engine?application_version=5.7)을 기준으로 정리했다. Claude Code는 코드를 작성·수정할 때 이 문서를 표준으로 삼는다. 규칙은 **Unreal Engine 5.7 / C++20** 기준이다.

---

## 1. 파일 구조 / Copyright

- 공개 배포되는 모든 소스 파일(`.h`, `.cpp`)은 첫 줄에 정확히 다음 copyright를 둔다. 포맷이 틀리면 CI가 실패한다.
  ```cpp
  // Copyright Epic Games, Inc. All Rights Reserved.
  ```
- 모든 헤더는 `#pragma once`로 다중 include를 막는다(include guard 매크로 대신).
- 파일 끝에는 빈 줄을 하나 둔다(gcc 호환).

## 2. 클래스 구성 (reader-first)

- `public` → `protected` → `private` 순으로 선언한다. 읽는 사람이 public API를 먼저 보고 구현 세부는 나중에 보도록 한다.
- 멤버는 거의 항상 `private`로 둔다. public/protected interface의 일부일 때만 노출한다. 파생 클래스 전용 필드는 `protected` accessor로 제공한다.
- 파생을 의도하지 않은 클래스는 `final`을 붙인다.

## 3. 네이밍 컨벤션

기본은 **PascalCase** — 각 단어 첫 글자를 대문자로, 단어 사이 underscore 없음.

### 타입 접두사 (UHT가 강제하므로 반드시 정확히)
| 접두사 | 의미 | 예시 |
|--------|------|------|
| `U` | `UObject` 파생 | `UActorComponent` |
| `A` | `AActor` 파생 | `ACharacter` |
| `S` | `SWidget` 파생(Slate) | `SCompoundWidget` |
| `I` | 추상 interface | `IAnalyticsProvider` |
| `T` | template class | `TArray`, `TMap` |
| `F` | 그 외 struct/class | `FVector`, `FString` |
| `E` | enum | `EColorBits` |
| `b` | bool 변수 | `bPendingDestruction`, `bHasFadedIn` |

> 프로젝트 규약: 위 UE 접두사 위에 `HSA`를 덧붙인다(예: `UHSAHeroComponent`, `AHSACharacter`). 상세는 [CLAUDE.md](../CLAUDE.md).

### 네이밍 규칙
- **타입·변수명은 명사**, **함수명은 동사**(효과나 반환값을 서술).
- bool을 반환하는 함수는 true/false 질문 형태: `IsVisible()`, `ShouldClearBuffer()`.
- 절차적 함수는 강한 동사 + 대상: `DoSomething()`.
- 참조로 받아 **쓰기(write)** 하는 out parameter는 `Out` 접두사: `FThing& OutResult`. bool out은 `bOutResult`.
- template parameter는 `In` 접두사: `template <typename InElementType>`.
- 매크로는 ALL_CAPS + `UE_` 접두사: `UE_LOG`.
- 과도한 축약(over-abbreviation)을 피한다. 서술적인 Type suffix를 선호한다.
- typedef는 대상에 맞는 접두사를 붙인다. template instantiation은 template 접두사를 잃는다: `typedef TArray<FMyType> FArrayOfMyTypes;`.

```cpp
bool  IsTeaFresh(FTea Tea);   // true의 의미가 분명
float TeaWeight;
int32 TeaCount;
UClass* TeaClass;
```

## 4. 타입 / Portability

직렬화(serialize)·복제(replicate)되는 데이터에는 **크기가 명시된 타입**을 쓴다.

| 타입 | 용도 |
|------|------|
| `bool` | boolean (크기 가정 금지, `BOOL` 쓰지 않음) |
| `TCHAR` | 문자 (크기 가정 금지) |
| `uint8`/`int8` | 1 byte |
| `uint16`/`int16` | 2 byte |
| `uint32`/`int32` | 4 byte |
| `uint64`/`int64` | 8 byte |
| `float` | 4 byte 단정밀도 |
| `double` | 8 byte 배정밀도 |
| `PTRINT` | 포인터 크기 정수 (크기 가정 금지) |

- 기본 `int`/`unsigned int`는 플랫폼마다 크기가 다르다(최소 32-bit 보장). **정수 폭이 중요하지 않을 때만** 사용 가능.

## 5. 표준 라이브러리 사용

UE는 성능·일관성 때문에 표준 라이브러리를 피해왔으나, 성숙한 C++20 기능은 자체 구현보다 나을 때 허용된다.

| 헤더 | 권장 |
|------|------|
| `<atomic>` | 신규 코드 사용, 기존 코드도 손댈 때 마이그레이션 |
| `<type_traits>` | legacy UE trait과 겹치면 선호. 조합 시 소문자 `value`/`type` 사용 |
| `<initializer_list>` | braced initializer에 필요 |
| `<limits>` | `std::numeric_limits` 전면 사용 가능 |
| `<cmath>` | 모든 부동소수점 함수 허용 |
| `<cstring>` | `memcpy()`/`memset()`은 성능 이점이 분명할 때 |
| `<regex>` | editor 전용 코드에서만 |

- 표준 container·string은 interop 코드를 제외하고 피한다(`TArray`/`TMap`/`TSet`/`FString` 사용).

## 6. const correctness

- 수정하지 않는 함수 인자는 const 포인터/참조로 받는다.
- 객체를 수정하지 않는 method는 `const`로 표시한다.
- container를 수정하지 않는 순회는 `const`로 한다.
- by-value parameter·지역 변수도 함수 본문에서 수정하지 않으면 `const`를 선호(단, move하는 by-value parameter는 const 금지 — §8 move 참조).

```cpp
void SomeMutatingOperation(FThing& OutResult, const TArray<int32>& InArray);
void FThing::SomeNonMutatingOperation() const;
for (const FString& Item : StringArray) { /* StringArray 수정 안 함 */ }
```

- **반환 타입에 const 금지.** 복합 타입의 move semantics를 막고, 내장 타입은 경고를 낸다.
  ```cpp
  const TArray<FString>& GetSomeArray();  // OK: 참조-to-const
  const TArray<FString>* GetSomeArray();  // OK: 포인터-to-const
  const TArray<FString>  GetSomeArray();  // 금지: const 반환 타입
  ```
- 포인터 자체를 const로 만들 땐 const를 **끝에** 둔다: `T* const Ptr = ...;`.

## 7. 주석

- **self-documenting code를 먼저** 쓴다. 주석은 **구현(how)이 아니라 의도(why)**를 설명한다.
- 나쁜 코드에 주석을 덧대지 말고 코드를 다시 쓴다. 주석이 코드와 모순되게 두지 않는다. 의도가 바뀌면 주석도 갱신한다.
- JavaDoc 기반 시스템으로 문서를 자동 추출한다. class 주석은 "이 클래스가 푸는 문제·생성 이유"를, method 주석은 목적·parameter(단위·범위·불가능 값)·반환값을 적고 `@warning`·`@note`·`@see`·`@deprecated`는 별도 줄에 둔다.

## 8. Modern C++ (C++20 기본)

### override / final
강력 권장. overriding method에는 `virtual` + `override`를 함께 쓴다.
```cpp
class B : public A
{
public:
    virtual void F() override;
};
```

### nullptr
모든 경우 C 스타일 `NULL` 대신 `nullptr`을 쓴다.

### auto — 기본 금지
초기화하는 타입은 항상 명시한다. 예외만 허용:
- lambda를 변수에 바인딩할 때(타입 표현 불가)
- 타입이 장황한 iterator 변수
- 타입을 쉽게 알 수 없는 고급 template 코드

> IDE가 타입을 추론해줘도 merge/diff 도구나 GitHub 단독 파일 보기에는 도움이 안 된다. **C++20 structured binding은 쓰지 않는다.**

### range-based for — 선호
```cpp
// 선호
for (TPair<FString, int32>& Kvp : MyMap)
{
    UE_LOG(LogCategory, Log, TEXT("Key: %s, Value: %d"), *Kvp.Key, Kvp.Value);
}
```

### strongly-typed enum
namespace enum 대신 `enum class`. Blueprint 노출 enum은 `uint8` 기반이어야 한다.
```cpp
UENUM()
enum class EThing : uint8
{
    Thing1,
    Thing2,
};
```
flag enum은 `ENUM_CLASS_FLAGS(EnumType)` 매크로로 bitwise 연산자를 얻는다.
```cpp
enum class EFlags
{
    None  = 0x00,
    Flag1 = 0x01,
    Flag2 = 0x02,
};
ENUM_CLASS_FLAGS(EFlags)

if ((Flags & EFlags::Flag1) != EFlags::None) { }
```

### move semantics
`TArray`/`TMap`/`TSet`/`FString`은 move를 지원한다. by-value로 받아 `MoveTemp`(UE의 `std::move`)로 옮기면 복사 비용 없이 표현력을 얻는다. **move하는 by-value parameter는 const로 두지 않는다.**
```cpp
void FBlah::SetMemberArray(TArray<FString> InNewArray)
{
    MemberArray = MoveTemp(InNewArray);
}
```

### default member initializer
멤버는 선언부에서 직접 초기화한다(constructor 중복·순서 불일치 방지). 게임 코드에서 특히 권장.
```cpp
UCLASS()
class UTeaOptions : public UObject
{
    GENERATED_BODY()
public:
    UPROPERTY()
    int32 MaximumNumberOfCupsPerDay = 10;

    UPROPERTY()
    FString TeaType = TEXT("Earl Grey");
};
```

### lambda
- 자유롭게 쓰되 짧게(몇 statement). 큰 식의 일부일수록 짧게.
- **capture는 명시적으로** — `[&]`·`[=]` 금지. 의도를 드러내 코드 리뷰에서 실수를 잡는다.
- 지연 실행(deferred) lambda는 dangling 위험 — `CreateWeakLambda`/`CreateSPLambda`, capture는 `TWeakObjectPtr`/`TWeakPtr`로 잡고 내부에서 validate. 큰 lambda·함수 호출 반환은 명시적 return type을 쓴다.

## 9. 코드 포맷팅

### 중괄호 (braces)
- 여는 중괄호는 **항상 새 줄**.
- 단일 statement 블록도 **항상 중괄호**를 포함한다(편집 실수·조건부 컴파일 안전).
```cpp
// 나쁨
int32 GetSize() const { return Size; }
// 좋음
int32 GetSize() const
{
    return Size;
}

if (bThing)
{
    return;
}
```

### if-else
각 블록에 중괄호. `else if`는 첫 `if`와 같은 들여쓰기 레벨로 정렬.
```cpp
if (TannicAcid < 10)
{
    UE_LOG(LogCategory, Log, TEXT("Low Acid"));
}
else if (TannicAcid < 100)
{
    UE_LOG(LogCategory, Log, TEXT("Medium Acid"));
}
else
{
    UE_LOG(LogCategory, Log, TEXT("High Acid"));
}
```

### 들여쓰기
- 줄 시작 whitespace는 **tab**(spaces 아님), tab 크기 4. 비-tab 문자 뒤 정렬에는 space 허용.

### switch
fallthrough는 `// falls through` 주석으로 명시하거나 `break`/`return`을 둔다. 동일 코드의 다중 case는 주석 없이 묶어도 된다. 항상 `default`를 둔다.
```cpp
switch (Condition)
{
    case 1:
        // falls through
    case 2:
        break;
    default:
        break;
}
```

## 10. namespace

- UHT는 namespace를 지원하지 않는다 — `UCLASS`/`USTRUCT`/`UENUM` 등에는 namespace를 쓰지 않는다.
- 신규 비-`UCLASS` API는 `UE::` namespace, 가능하면 중첩(`UE::Audio::`). 구현 세부는 `UE::Audio::Private::`.
- 전역 scope나 `.cpp`에서 `using` 금지(unity build 깨짐). namespace 내부나 함수 본문에서는 허용.
- 매크로는 namespace에 둘 수 없다 — `UE_` 접두사로 구분.

## 11. Physical Dependencies / include 위생

- **파일명에 접두사를 붙이지 않는다**(가능한 한): `UScene.cpp`가 아니라 `Scene.cpp`.
- **forward declaration을 include보다 우선**한다.
- 필요한 헤더는 **직접 include**한다. 간접 include에 의존하지 않는다. `Core.h` 같은 광범위 헤더 대신 구체적 헤더를 include.
- 다른 헤더에서 표준 라이브러리 헤더 include를 피한다.
- 모듈은 `Public`(타 모듈이 쓰는 정의) / `Private`(그 외) 디렉터리로 나눈다.
- `FORCEINLINE`·inline 남용 금지(include하는 모든 파일을 rebuild하게 만들어 빌드 시간 악화).

## 12. API 설계

### bool parameter 피하기 — flag enum 사용
```cpp
// 나쁨: 호출부에서 의미 불명
FCup* MakeCupOfTea(FTea* Tea, bool bAddSugar, bool bAddMilk, bool bAddHoney);
// 좋음
enum class ETeaFlags { None = 0, Milk = 0x01, Sugar = 0x02, Honey = 0x04 };
ENUM_CLASS_FLAGS(ETeaFlags)
FCup* MakeCupOfTea(FTea* Tea, ETeaFlags Flags = ETeaFlags::None);
```
예외: `void SetEnabled(bool bEnabled)`처럼 완전한 상태 setter는 bool 허용.

### 긴 parameter list — 전용 struct로
parameter가 많으면 default member initializer를 가진 struct를 넘긴다.

### overloading
`bool`과 `FString` overload를 같이 두지 않는다(`Func(TEXT("..."))`가 bool overload를 호출).

### interface
추상이어야 하고 `I` 접두사, 멤버 변수 없음. inline 구현된 비-pure-virtual·static·비-virtual method는 가능.

### UObject는 포인터로 전달
참조가 아니라 **포인터**로 넘긴다. null 허용 여부를 문서화/처리한다.
```cpp
void AddActorToList(AActor* Obj);   // 좋음 (AActor& 아님)
```

## 13. 일반 스타일

- 변수는 **쓰기 직전에** 값을 설정해 의존 거리를 줄인다.
- 큰 method는 잘 명명된 helper로 쪼갠다.
- 함수명과 `(` 사이에 space 없음: `Func()` (not `Func ()`).
- **포인터/참조는 오른쪽에만 space 하나**: `FShaderType* Ptr` (not `*Ptr` / `* Ptr`).
- 컴파일러 warning은 고친다. `#pragma` 억제는 최후 수단.
- 문자열 리터럴은 항상 `TEXT()`로 감싼다.
- 변수 shadowing 금지.
- 익명 리터럴 대신 named constant: `const FName ObjectName = TEXT("Soldier");`.
- 복잡한 식은 named 중간 변수로 쪼갠다.
  ```cpp
  const bool bIsLegalWindow = Blah->WindowExists && bStuff;
  const bool bIsPlayerDead  = bPlayerExists && bGameStarted && bHasPawn;
  if (bIsLegalWindow && !bIsPlayerDead)
  {
      DoSomething();
  }
  ```
- 헤더의 non-trivial static은 `.cpp`에 정의하고 헤더엔 `extern`만 둔다.
- **whitespace/rename 같은 cosmetic 변경을 동작 변경과 섞지 않는다.** 광범위한 cosmetic 변경은 merge history를 해치므로 피한다.
- debug 코드는 다듬어 쓸 만하지 않으면 commit하지 않는다.

## 14. 플랫폼 / 서드파티 코드

- 플랫폼별 코드는 `PLATFORM_[PLATFORM]` 분기 대신 **HAL(hardware abstraction layer)을 확장**한다. 꼭 필요한 define은 속성을 서술하는 이름(`PLATFORM_USE_PTHREADS`)으로 `Platform.h`에 기본값을 두고 플랫폼별 헤더에서 override.
- cross-platform 코드는 플랫폼 디렉터리 유무와 무관하게 컴파일·실행돼야 한다.
- 서드파티 수정은 태그로 표시한다.
  ```cpp
  // @third party code - BEGIN PhysX
  #include <physx.h>
  // @third party code - END PhysX
  ```

## 15. inclusive language

- class·함수·변수·파일·주석·UI 텍스트에 존중하는 전문적 언어를 쓴다.
- `blacklist`/`whitelist` → `deny list`/`allow list`, `master`/`slave` → `primary`/`secondary`(또는 `source`/`replica` 등). 역사적 트라우마·고정관념을 강화하는 단어를 피한다.
- 비인격 대상(module·plugin·function·server)은 `it/its`, 가정의 사람은 단수라도 `they/them`. 성별 가정 표현(`guys`, `poor man's X`)·slang·profanity를 피한다.
