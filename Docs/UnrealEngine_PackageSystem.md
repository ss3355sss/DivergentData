# Unreal Engine 패키지 시스템 용어 정리

## 핵심 개념: Object, Package, Asset

```
┌─────────────────────────────────────────────────────────────┐
│  파일 시스템                                                  │
│  SM_Chair.uasset  ←──────── 실제 디스크의 파일               │
└─────────────────────────────────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│  Package (UPackage)                                         │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Asset (메인 UObject)                                  │  │
│  │  "SM_Chair" (UStaticMesh)  ← Export Object            │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌─────────────────┐ ┌─────────────────┐                   │
│  │ SubObject 1     │ │ SubObject 2     │  ← 내부 오브젝트들 │
│  │ (UBodySetup)    │ │ (UNavCollision) │                   │
│  └─────────────────┘ └─────────────────┘                   │
└─────────────────────────────────────────────────────────────┘
```

### 1. UObject

모든 것의 기반 클래스. UE의 모든 객체는 UObject를 상속합니다.

### 2. Package (UPackage)

UObject들을 담는 컨테이너. 직렬화(저장/로드)의 단위입니다.
- 하나의 `.uasset` 파일 = 하나의 Package

### 3. Asset

Package 내의 "메인" UObject를 부르는 용어.
- 에디터에서 Content Browser에 보이는 것이 Asset입니다.

| 개념 | 정체 | 설명 |
|------|------|------|
| **UObject** | C++ 클래스 | 모든 객체의 베이스 |
| **Package** | `UPackage` 인스턴스 | 오브젝트 컨테이너, 파일과 1:1 매핑 |
| **Asset** | 용어 (Term) | 패키지의 메인 오브젝트를 부르는 이름 |

---

## 경로 용어 정리

실제 예시로 살펴보겠습니다:

```
프로젝트 구조:
Divergent/
└── Game/
    └── Content/
        └── Divergent/
            └── Items/
                └── SM_Sword.uasset    ← 이 파일 기준
```

### 각 경로의 의미

| 용어 | 값 | 설명 |
|------|-----|------|
| **Filename** | `E:/ws/Divergent/Game/Content/Divergent/Items/SM_Sword.uasset` | 실제 파일 시스템 경로 |
| **LongPackageName** | `/Game/Divergent/Items/SM_Sword` | UE 내부의 패키지 가상 경로 |
| **ObjectPath** | `/Game/Divergent/Items/SM_Sword.SM_Sword` | 패키지 내 특정 오브젝트 경로 |
| **AssetName** | `SM_Sword` | 에셋 이름만 |
| **PackageName** | `SM_Sword` | 패키지 이름만 (보통 AssetName과 동일) |

### 시각적 분해

```
Filename (파일 시스템):
E:/ws/Divergent/Game/Content/Divergent/Items/SM_Sword.uasset
└──── 실제 경로 ─────┘└─ Content ─┘└── 폴더 ──┘└─ 파일 ──┘

LongPackageName (UE 가상 경로):
/Game/Divergent/Items/SM_Sword
  │                       │
  └─ Mount Point          └─ 확장자 없음
    (Content 폴더)

ObjectPath (오브젝트 경로):
/Game/Divergent/Items/SM_Sword.SM_Sword
└───── LongPackageName ───────┘.└ Object Name ┘
```

---

## 코드로 확인하기

```cpp
// UObject에서 각 경로 얻기
UStaticMesh* Mesh = ...;

// Package 얻기
UPackage* Package = Mesh->GetPackage();

// 각종 이름들
FString ObjectPath = Mesh->GetPathName();
// → "/Game/Divergent/Items/SM_Sword.SM_Sword"

FString LongPackageName = Package->GetName();
// → "/Game/Divergent/Items/SM_Sword"

FName ObjectName = Mesh->GetFName();
// → "SM_Sword"
```

### 경로 변환 함수들 (FPackageName)

```cpp
FString LongPackageName = TEXT("/Game/Divergent/Items/SM_Sword");

// LongPackageName → Filename
FString Filename;
FPackageName::TryConvertLongPackageNameToFilename(
    LongPackageName,
    Filename,
    FPackageName::EConvertFlags::None
);
// → "E:/ws/Divergent/Game/Content/Divergent/Items/SM_Sword.uasset"

// Filename → LongPackageName
FString ConvertedPackageName;
FPackageName::TryConvertFilenameToLongPackageName(Filename, ConvertedPackageName);
// → "/Game/Divergent/Items/SM_Sword"

// ObjectPath → LongPackageName
FString ObjectPath = TEXT("/Game/Divergent/Items/SM_Sword.SM_Sword");
FString PackageName = FPackageName::ObjectPathToPackageName(ObjectPath);
// → "/Game/Divergent/Items/SM_Sword"
```

---

## 왜 이렇게 복잡한가?

1. **플랫폼 독립성**: `LongPackageName`은 OS 파일 시스템과 무관한 가상 경로
2. **마운트 시스템**: `/Game/`, `/Engine/`, `/Plugin/` 등 여러 루트를 하나의 네임스페이스로 통합
3. **패키지 내 복수 오브젝트**: 하나의 `.uasset`에 여러 `UObject`가 있을 수 있어 `ObjectPath`로 구분
4. **Blueprint 특수성**: `BP_Weapon.uasset` 안에는 `BP_Weapon` (UBlueprint)과 `BP_Weapon_C` (UClass) 두 개가 있음

---

## 한눈에 보는 요약

```
┌─────────────────────────────────────────────────────────────────┐
│                        용어 관계도                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Filename ──────► 실제 파일 (.uasset)                          │
│       │                    │                                    │
│       │              ┌─────▼─────┐                              │
│       │              │  Package  │ ◄─── LongPackageName         │
│       │              │ (UPackage)│                              │
│       │              └─────┬─────┘                              │
│       │                    │                                    │
│       │         ┌──────────┼──────────┐                         │
│       │         ▼          ▼          ▼                         │
│       │    ┌────────┐ ┌────────┐ ┌────────┐                     │
│       │    │ Asset  │ │SubObj 1│ │SubObj 2│  ◄─ ObjectPath      │
│       │    │(메인)  │ │        │ │        │     으로 각각 참조   │
│       │    └────────┘ └────────┘ └────────┘                     │
│       │                                                         │
│   OS 영역           ◄────────►        UE 가상 영역              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 추가 참고: 주요 마운트 포인트

| Mount Point | 실제 위치 | 용도 |
|-------------|----------|------|
| `/Game/` | `{Project}/Content/` | 프로젝트 에셋 |
| `/Engine/` | `{Engine}/Content/` | 엔진 기본 에셋 |
| `/{PluginName}/` | `{Plugin}/Content/` | 플러그인 에셋 |
| `/Temp/` | 메모리 | 임시 패키지 |
| `/Script/` | - | C++ 클래스 (UClass) |

---

## 관련 클래스 및 함수

| 클래스/함수 | 용도 |
|------------|------|
| `FPackageName` | 경로 변환 유틸리티 |
| `FSoftObjectPath` | 오브젝트 경로 저장 (직렬화 가능) |
| `TSoftObjectPtr<T>` | 타입 안전한 소프트 레퍼런스 |
| `FAssetData` | 에셋 메타데이터 |
| `IAssetRegistry` | 에셋 검색 인터페이스 |
