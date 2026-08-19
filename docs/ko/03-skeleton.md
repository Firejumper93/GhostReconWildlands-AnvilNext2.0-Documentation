[English](../03-skeleton.md) · [Deutsch](../de/03-skeleton.md) · **한국어**

---

> **안내, 2026-08-19.** 이 문서의 영어 원문은 2026-08-19에 확장되었으며,
> 이 번역에 아직 반영되지 않은 내용이 있습니다. 갱신 전까지는
> [영어 원문](../03-skeleton.md)을 기준으로 삼아 주십시오. 이전에 안내한
> 날짜(2026-08-17)를 지키지 못했으므로 이번에는 날짜를 명시하지 않습니다.

# 03 - 스켈레톤, 리그, 본 명명

## 본의 식별자는 어디서나 CRC32다

`[VERIFIED]` `.Skeleton` 페이로드에는 **ASCII 본 이름이 없습니다.** 플레이어 신체
리그를 출력 가능한 문자열로 스캔해도 하나도 나오지 않습니다. 본의 식별자는 단순한 **본
이름의 CRC32**이며, 같은 CRC32가 리소스 컨테이너의 클래스 이름에도, 리플렉션 테이블의
프로퍼티 이름에도 쓰입니다.

이것이 이 엔진의 데이터 모델에 관해 가장 유용한 단일 사실입니다. 추측한 이름에서 해시
한 번으로 엔진 자체의 식별자에 도달할 수 있고, 미지의 해시는 레인보우 테이블로 깰 수
있다는 뜻이기 때문입니다.

플레이어 신체 리그의 검증된 상수들(HumanIK 이름의 CRC32):

| 본 | CRC32 | 애셋 본 인덱스 |
|---|---|---|
| `Reference` | `0x2C52CBB0` | 0 |
| `Hips` | `0xDED10611` | 1 |
| `Head` | `0x07C159A2` | 45 |
| `Neck1` | `0xB05FD12B` | |
| `LeftShoulder` | `0x2D4660A8` | |
| `LeftArm` | `0xEB830ADA` | 8 |
| `LeftForeArm` | `0x89B93A80` | |
| `LeftHand` | `0xB675F36C` | 12 |
| `RightHand` | `0x75F94D30` | 58 |

클래스 해시도 같은 방식입니다. `crc32("Skeleton") == 0x24AECB7C`,
`crc32("Bone") == 0x95741049`, `crc32("cBallisticProjectileComponent") ==
0x09BFE10E`.

## `.Skeleton` 애셋 포맷

`[VERIFIED]` 플레이어 신체 리그는 **설명되지 않은 바이트 0개로 EOF까지 바이트 단위
완전하게** 파싱됩니다. 이 포맷에 적용한 기준이 그것이었습니다.

파일 프레이밍은 `u64 오브젝트 ID` 다음에 `u32 클래스 해시`입니다.

본 레코드 레이아웃:

```
pre-byte
u64  ID              (0xF80000XX)
u32  class hash      (0x95741049 = "Bone")
u32  Name            본 이름의 CRC32
ptr  Parent          ObjectPtr
ptr  Mirror          ObjectPtr
vec4 GlobalPos
quat GlobalRot
vec4 LocalPos
quat LocalRot
u8   prio
i32  MirroringType
     modifier list
     deps list
i32  WrinkleCategory
f32  WrinkleFactor
u16  Index
u16  ChildrenCount
```

`[VERIFIED]` 와일드랜드의 **ObjectPtr 방언**: 태그 `02`는 뒤에 `u64` ID가 온다는
뜻이고, 태그 `03`은 페이로드 없는 null이라는 뜻입니다. 이걸 틀리면 파싱 전체가 어긋나므로
가장 먼저 확인해 볼 만합니다.

`[VERIFIED]` 리그 전체에서 성립하며 파서 어서션으로 쓰기 좋은 구조적 불변식: 저장된
`Index`는 배열 위치와 같고, `ChildrenCount`는 서브트리 크기와 같습니다. 100개 본 전부에
대해 그렇습니다.

### 플레이어 신체 리그, 구체적으로

`[VERIFIED]` `GR_PCF_Skeleton_Average.Skeleton`, 14,006바이트:

- **본 100개**, 단일 루트(본 0, `Reference`), 계층 깊이 12
- 바인드 포즈는 Z-업, **키 1.77m**, 루트 골반은 z = 0.964
- 본 이름 100개 중 86개 해석 완료. 해석되지 않은 14개는 소품 및 어태치먼트 헬퍼이며,
  그중 둘은 `LeftWristTarget` / `RightWristTarget`으로 끝난다는 것이 입증됨
- 이 리그에서 `SkeletonKey == SkeletonHierarchyKey == 0x3121DFFF`이며, 프로세스
  메모리에서 플레이어 스켈레톤을 찾는 데 쓸 만한 스캔 앵커가 됨

`[VERIFIED]` **본 이름은 오토데스크 HumanIK 규약 그대로입니다.** `Reference`, `Hips`,
Spine 체인, `Neck`, `Neck1`, `Head`, 전체 손가락 체인을 갖춘
`LeftShoulder`/`Arm`/`ForeArm`/`Hand`와 그 우측 대칭, 여기에 헬퍼 접두사 `O_`와
`L_`(예: `O_01LeftForeArm` 트위스트 체인, `O_LeftElbow`, `L_01LeftArm`)이 붙습니다.

이는 명명 이상의 의미가 있습니다. 런타임 리그가 진짜 HumanIK 캐릭터라는 뜻이고, 따라서
HumanIK 자체의 의미론이 그대로 적용된다는 뜻입니다.

### 애셋 안의 IK 데이터

`[VERIFIED]` 애셋의 `IkChainDescriptor`는 다음과 같습니다.

```
u32   ContactBoneID          (본 이름 해시)
u32   EndEffectorBoneID
u32   IkChainStartBoneID
vec3  ContactOffset
bool  InvertIkResolution
ptr   MirrorIkChain
u8    IkChainSize
```

`[VERIFIED]` **와일드랜드 시기에는 `IKData`에 페이로드가 없으며**, 플레이어 리그의
`IkChainsDefinitions` / `IKData` 본문은 null입니다. **이펙터 테이블은 런타임에만
존재합니다.** 이는 HumanIK가 애셋마다 저작되는 대신 런타임에 데이터를 공급받는다는 것과
일관됩니다. 구워진 이펙터 목록을 찾아다니지 마세요. 그런 것은 없습니다.

### 애셋이 있는 위치

`[VERIFIED]` `GR_PLAYER_Template`은 압축을 풀면(1.8MB → 5.27MB) 약 1,240개의 타입 지정
오브젝트가 되며, 베이스 forge에만도 **`.Skeleton` 인스턴스 143개**가 들어 있습니다.
신체, 수염, 머리카락, 배낭, 조끼의 부착점 스켈레톤들입니다. 눈에 띄는 항목으로는
`GR_PCF_Skeleton_Average`(신체 리그), `Child_Skeleton`, 그리고 머리별 스켈레톤이
있습니다.

애니메이션 페이로드에도 마찬가지로 ASCII 본 이름은 없습니다. 작은 forge 하나에서 나온 씬
디스크립터는 약 2,800개의 타입 지정 오브젝트를 내놓는데 그중 2,177개가 `.Animation`이며,
이쪽은 읽을 수 있는 유비소프트식 이름을 *가지고* 있습니다(`sb_`는 군인/플레이어 클래스,
`civ_`와 `rbl_`은 민간인과 반군).

## 런타임 리그 디스크립터

`[VERIFIED, 생성자와 초기화 패스의 디스어셈블리에서]` 리그 객체는 전역 매니저에서 dword
ID로 캐시되는, **공유되고 참조 카운트되는 `0xF8`바이트짜리 스켈레톤 디스크립터**입니다.
같은 스켈레톤 데이터블록을 쓰는 캐릭터들은 **하나의 리그 인스턴스를 공유합니다.** 리그는
개별 캐릭터가 아니라 스켈레톤 *클래스*를 식별합니다. 트랜스폼은 전혀 담고 있지 않습니다.

주요 필드(리테일 빌드 오프셋):

| 오프셋 | 내용 |
|---|---|
| `+0x08` | 참조 카운트 |
| `+0x20` | `0x60`바이트짜리 인스턴스 풀 헬퍼 포인터 |
| `+0x28` | 8바이트 채널 레코드 벡터 `{u32 nameHash, u16 포즈 버퍼 오프셋, u16 타입/인덱스 비트}` |
| `+0x34` | 포즈 버퍼 전체 크기 |
| `+0x3C` | dword 리그 ID |
| `+0x42` | 레이어 개수 |
| **`+0x50`** | **정렬된 `{u32 CRC32 본 이름 해시, u16 노드 인덱스}` 이름 맵** |
| **`+0x5A`** | **이름 맵 항목 수** |
| `+0x68` | 노드 → 레코드 리맵 |
| `+0x80` | 노드 레코드, 각 16바이트: `+0`에 부모 word, `+9`에 그룹 바이트, `+0xA`에 플래그 바이트(비트 3 = HIK 표시, 비트 4-6 LOD) |
| **`+0x8A`** | **본 개수** |
| `+0x98` | 해시 정렬 순열 |
| `+0xA4` | HIK 표시된 노드 목록 |
| `+0xD0` | `CRITICAL_SECTION` |

`[VERIFIED]` `+0x50`의 이름 맵은 해시로 정렬되어 있으므로, 이진 탐색만으로 엔진 코드를
호출하지 않고 본 이름 해시를 노드 인덱스로 해석할 수 있습니다. 엔진 자체의 조회 함수는
`Rig::BoneIndexFromNameHash`이며, 리테일 구현은 `0x0A85F0F0`, 썽크는 `0x00CF90F0`,
시그니처는 `u16(Rig* rcx, u32 crc32 edx)`이고, 실패 시 `0xFFFF`를 반환합니다.

인스턴스 풀 헬퍼는 `0x60`바이트이며 64슬롯 풀(64 x `0x60`짜리 `0x1808` 할당 하나),
프리 리스트 헤드는 `+0x50`, 활성 리스트 센티널은 `+0x58`, 슬롯은 `slot+0x48`의 next와
`slot+0x40`의 prev로 연결됩니다. 즉 **리그 클래스당 최대 64개의 살아 있는 캐릭터
인스턴스**입니다.

`[VERIFIED]` 전역 HIK 매니저 싱글턴이 리그 ID를 키로 하는 해시 맵을 보유하며, 리그
포인터는 `node+0x10`에 있습니다. 이를 열거하면 **살아 있는 모든 리그**를 얻습니다.

`[VERIFIED]` 스켈레톤 쪽 식별 체인: `[skel+0x30]`은 가드 블록
`{암호화된 qword +0, 키 dword +0xC, +0x10에 평문 데이터블록 포인터}`에 도달하며, 데이터블록
포인터는 아무것도 복호화하지 않고 읽을 수 있습니다. `[skel+0x2C8]`은 본별 오버라이드
목록으로 `{u32 해시, u32 LOD}`가 정렬되어 있습니다.

## `Skeleton::BipedBoneID`: 부착점 열거형

`[VERIFIED, 2026-08 업데이트 빌드]` RVA `0x046F5080`의 테이블, **항목 143개, 스트라이드
8**, 각 항목은 `{u32 CRC32(본 이름), u32 CRC32(BIPEDBONE_* 태그)}`이고 인덱스가 서수
역할을 합니다. 143개 중 132개 이름을 복원했습니다.

이것이 엔진 자체의 부착 슬롯 열거이며, 이후 Anvil 타이틀의 "어태치먼트 슬롯" 개념이
대응되는 대상입니다. 다른 타이틀의 슬롯 번호는 **넘어오지 않습니다.**

| idx | 해시 | 이름 |
|---|---|---|
| 6 | `0x07C159A2` | `Head` |
| 15 | `0xB675F36C` | `LeftHand` |
| 38 | `0x75F94D30` | `RightHand` |
| **72** | `0x3FB256E5` | **`RightHand_Weapon_Ref`** |
| **73** | `0xA9611103` | **`LeftHand_Weapon_Ref`** |
| 115 | `0x826846F3` | `Fake_gunroot` |
| 116 | `0x08B4DDD5` | `FakeGunRoot_Gameplay` |
| 117-120 | | `FakeGunRoot_LeftTrigger` / `_LeftCannon` / `_RightTrigger` / `_RightCannon` |
| 121-124 | `0x53135E44` .. | `Prop_RightHand`, `Prop_RightHand2..4` |
| 125-128 | `0x85562B5C` .. | `Prop_LeftHand`, `Prop_LeftHand2..4` |
| 130-131 | | `FakeGunRoot_SecondHand`, `_SecondHand_Gameplay` |
| 132-136 | `0x7ECBAF84` .. | `Holster_Hips` / `_Back` / `_Chest` / `_LeftUpLeg` / `_RightUpLeg` |
| 137-138 | `0x48674E6D`, `0x776266E1` | `Backpack_Gun_AttachPoint_Primary` / `_Secondary` |

115/116의 쌍에 주목하세요. **시각용** 총기 루트와 별도의 **게임플레이용** 총기 루트가
있고, 둘은 서로 다른 본입니다. 엔진이 어느 쪽을 고르는지는 선택 함수가 결정하는데, 먼저
`FakeGunRoot_Gameplay`를 해석하고 없으면 `Fake_gunroot`로 폴백합니다.

`[VERIFIED]` `0x03AD1230`에도 하나 있습니다. dword 네 개짜리 묶음
`{LeftHand, RightHand, LeftHand_Weapon_Ref, RightHand_Weapon_Ref}` 바로 뒤에 문자열
`"BipedIkParamsRoot"`가 따라옵니다. IK 파라미터의 손/무기 참조 세트입니다.

무기 리그는 자체 본 접두사 `wb-`를 씁니다. `wb-gunroot`, `wb-ref-anim`(`0x8CDA0E3F`),
`wb-LightRoot`.

## HumanIK: 데이터로만 존재

`[VERIFIED]` HumanIK는 **이름이 제거된 채** 정적 링크되어 있습니다. 설치 트리 어디에도
HumanIK DLL이 없으며, 데이터 태그는 실행 파일 안에 있습니다.

| RVA (리테일) | 태그 |
|---|---|
| `0x03A81BF8` | `HIKCHARACTER000` |
| `0x03A81C63` | `HIKSTATE0000000` |
| `0x03A81CD8` | `HIKEFFECTOR0000` |
| `0x03A81CF0` | `HIKPROPERTY0000` |
| `0x03A81D08` | `HIKDATABLOCK000` |

**이펙터 이름 테이블**은 연속되어 있고(리테일 `0x03C7FE30..0x03C80238`, 업데이트 빌드
`0x03CD8370..0x03CD8790`, 항목 44개) 사실상 이펙터 열거형입니다. `HipsEffector`,
`LeftWristEffector`, `RightWristEffector`, `HeadEffector`, `LeftHandEffector`,
`RightHandEffector`, `ChestOriginEffector`, `ChestEndEffector`에 모든 손가락, 그리고
대응하는 `*Tip` 마커 테이블이 함께 있습니다.

**프로퍼티 테이블**은 오토데스크 HumanIK 4.x의 프로퍼티 이름 그대로입니다.
`ReachActorLeftShoulder`, `ReachActorRightShoulder`, `SnSReachLeftWrist`,
`SnSReachRightWrist`, `SnSReachHead`, `ParamRealisticArmSolving` 등입니다.

`[VERIFIED]` 이 배열들은 **저장된 절대 포인터**로 참조되며(재배치 대상), 따라서 죽은
데이터가 아니라 런타임 테이블에 실제로 연결되어 있습니다.

`[VERIFIED NEGATIVE]` **이미지 어디에도 HumanIK 공개 API 심벌 이름은 존재하지
않습니다.** `HIKSetEffectorStateTQSfv`, `HIKSolveForEffectorSet`,
`HIKSetNodeStateTQSfv`, `HIKCharacterCreate`, `HIKEffectorSetStateCreate` 모두 405MB
문자열 집합 전체에서 히트가 0입니다. HIK 관련 매치는 저 다섯 개 데이터 태그와 패킹된
블롭에서 나온 잡음뿐입니다.

따라서 솔버에는 형태로 접근하거나 이펙터 이름 테이블을 통해서만 도달할 수 있습니다. 그
이펙터 인터페이스를 외부에서 구동할 수 있는지는 **`[UNKNOWN]`**이며, 시도한다면 이름
테이블이 출발점이 될 앵커입니다.
