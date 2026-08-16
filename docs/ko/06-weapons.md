[English](../06-weapons.md) · [Deutsch](../de/06-weapons.md) · **한국어**

---

> **안내, 2026-08-16.** 이 문서의 영어 원문은 2026-08-16에 갱신되었으며,
> 아직 이 번역에 반영되지 않은 정정 사항이 포함되어 있습니다.
> 번역은 2026-08-17에 갱신될 예정입니다.
> 그때까지는 [영어 원문](../06-weapons.md)을 기준으로 삼아 주십시오.

# 06 - 무기와 어태치먼트 시스템

이 문서의 RVA는 리테일이라고 표시하지 않은 한 **2026-08 업데이트 빌드** 기준입니다.
2026년 빌드의 `SkeletonPostUpdate` 본문은 리테일 빌드와 명령어 단위로 동일하므로, 구조에
관한 설명은 그대로 넘어가고 주소만 옮겨졌습니다.

## 무기는 하나의 객체가 아니라 여러 개다

`[VERIFIED, 배포된 셰이더에서]` 무기는 **부품별로 분리된 메시**로 저작되며, 각각 하나의
`.Mesh` 애셋입니다.

```
W_ASR_AK-12_body_LOD0
W_ASR_AK-12_Barrel_LOD0
W_ASR_AK-12_Stock_LOD0
W_ASR_AK-12_StockFolded_LOD0
W_ASR_AK-12_Magazine_LOD0
W_ASR_AK-12_FlashHider_LOD0
W_ASR_AK-12_Ironsights_LOD0
```

`[VERIFIED]` 컴포넌트 레이아웃이 이를 독립적으로 뒷받침합니다.
`cWeaponAttachmentHolder`에는 **부품마다 슬롯이 하나씩** 있습니다.

실무적 결과이며, 신비해 보이는 부류의 버그를 설명해 줍니다. 각 부품이 자기 트랜스폼을 가진
독립된 드로우 객체이기 때문에, **무기 본체를 컬링하거나 숨겨도 그 부착물은 함께 사라지지
않습니다.** 총이 없는 채로 공중에 떠 있는 소음기는 본체를 숨겼을 때 나오는 당연한
결과이지 글리치가 아닙니다.

`[INFERRED, 강함]` 따라서 본 18~21개짜리 무기 리그는 메시 하나를 변형하기 위해서가 아니라
**교체 가능한 부품들을 배치하기 위해** 존재합니다. 부품 수에 부착점, 그리고 총구, 탄피
배출, 조준경 소켓을 더하면 정확히 그 범위에 들어맞고, Gunsmith 커스터마이즈 시스템은
부품별 배치를 요구합니다.

## 프레임 단위로 본 무기 부착 체인

`[VERIFIED, 정적 분석]` 손에 든 무기에 대해, 처음부터 끝까지:

```
캐릭터 SkeletonPostUpdate                0x0F085A50  (썽크 0x0189B430)
  -> Skeleton::PoseRefresh(캐릭터)       0x0F091720  (썽크 0x018A09A0)
     -> Pose::SetRootTransform(Pose A)   0x0F203FA0  (썽크 0x01904570)
     -> Pose::SetRootTransform(Pose B)   같은 함수, 소스 = 소유 엔티티 노드의 4x4
     -> Skeleton::PublishAttachments     0x0F090D00  (썽크 0x018A03B0)
        -> 캐릭터 Pose B에서 손 본의 월드 트랜스폼을 합성
           (본 레코드 + 플래그 비트 26이 꺼져 있으면 rootT/rootQ)
        -> TransformNode::SetWorldTransform(무기 노드)
                                         0x0EAB1C60  (썽크 0x017E1770)
           -> vtable 알림 -> Entity::OnTransformChanged  0x0C426AC0
              -> Skeleton::MarkPoseDirty(무기)  0x0F080AA0 (썽크 0x018996B0)
        -> Skeleton::PoseRefresh(무기 스켈레톤), 인라인, 7개 명령어 뒤
           -> 무기 포즈 루트 둘 다 다시 도출하고 dirty 비트를 지움
```

저 인라인 `PoseRefresh`를 눈여겨보세요. 조사의 한 갈래를 통째로 닫아 버리기 때문입니다.
**포즈 루트에 쓴 값은 들고 있는 무기에서는 살아남지 못합니다.** 부모의 퍼블리시가 자식을
dirty로 표시한 뒤 곧바로 그 루트들을 다시 도출합니다.

## `TransformNode::SetWorldTransform`

**구현 `0x0EAB1C60`, 썽크 `0x017E1770`.**

```c
__fastcall(TransformNode* rcx, const float4x4* rdx, bool r8b, bool r9b)
```

`rdx`가 가리키는 행렬의 레이아웃: `+0x00`, `+0x10`, `+0x20`의 행이 회전 기저이고,
**이동 성분은 `+0x30`의 네 번째 행입니다.**

`[VERIFIED]` 부착 객체의 배치에 관해서는 이것이 진실의 출처입니다. 무기의 포즈 루트가
여기*로부터* 재생성되고, 트랜스폼 변경 알림도 여기*로부터* 전파됩니다.

세 가지 특성 덕분에 이곳은 타협이 아니라 좋은 가로채기 지점이 됩니다.

- `[VERIFIED]` 어태치먼트 퍼블리시는 **새 행렬이 노드의 현재 행들과 다를 때만** 이 함수를
  호출합니다(바로 앞에 `cmpeqps` 비교 블록이 있습니다). 주입한 포즈는 항상 다르므로,
  가로채기가 항상 마지막 발언권을 갖습니다.
- `[VERIFIED]` 프레임 안에서 무기 배치를 **마지막으로 쓰는** 주체입니다.
- `[VERIFIED]` 썽크는 `int3`로 패딩된 슬롯 안의 표준 5바이트 jmp입니다.

`[VERIFIED]` **rel32 호출 지점이 9곳** 있으므로 무기 전용이 아니라 범용 배치 API입니다.
`rcx`로 게이팅하는 것은 필수입니다. 게이팅 없이는 트랜스폼 노드를 쓰는 세상의 모든 것을
움직이게 됩니다.

`[UNKNOWN]`, 분명히 말해 둡니다. 트랜스폼 노드를 옮기는 것은 **배치와 소유 관계**를 증명할
뿐 GPU 업로드 경로를 증명하지는 않습니다. 모든 경우에 시각적으로도 충분한지는 그 객체의
지오메트리가 어떻게 그려지는지에 달려 있습니다([07-rendering.md](07-rendering.md) 참조).

## 대안이 되는 쓰기 대상: 노드 자신의 로컬 행렬

`[VERIFIED]` `PublishAttachments`는 `world = nodeLocal x boneWorld`를 계산하고 네 번째
인자를 설정한 채 커밋하는데, 이는 로컬 재계산을 건너뜁니다. 따라서 무기 자신의
`TransformNode+0x00` 로컬 행렬에 쓴 값은 매 프레임 소비됩니다.

이 방식은 그 객체가 소유한 64바이트만 건드립니다. 공유 포즈 버퍼도, 스키닝 위험도 없고,
어느 gun-root 본이 진짜인지에 관한 문제도 비켜 갑니다. 먼저 두 가지 값이 예상대로 읽혀야
합니다(`[node+0x48] == 0`이면 상속, 그리고 `[node+0x88] == null`). 로컬 행렬에 스케일을
넣지 마세요. 정규 직교화 과정에서 사라집니다.

## 부착 API

`[VERIFIED]` **`AttachEntityToBoneByNameHash`**, 썽크 `0x0085DBB0`에서 본문
`0x094BF1A0`으로, 호출 지점 17곳:

```c
(rcx = 자식 엔티티, rdx = 부모 Entity, r8d = CRC32(본 이름))
```

부모의 SkeletonComponent를 해석하고, 해시로 FindBone을 수행하며, **리그에 그 본이 없으면
거부하고**, `SetParent`를 호출한 뒤, 기본 로컬 오프셋을 제공하는 심을 통해 어태치먼트를
등록합니다.

`[VERIFIED]` `BoneHandle`은 16바이트입니다.
`{+0x00에 소유자 포인터, +0x08에 int32 index = -1, +0x0C에 u16 slotNum = 0xFFFF}`.

`[VERIFIED]` **손을 고르는 호출 지점**은 함수 `0x067EA090` 안의 `0x067EA0EB`입니다.
`[rcx+0x40]`의 선택자가 0이면 `Prop_RightHand`(`0x53135E44`), 1이면
`Prop_LeftHand`(`0x85562B5C`)를 고른 뒤 부착 API를 호출합니다.

생성자 `0x13ADBFB0`에 있는 두 번째 데이터 주도 명세는 결합의 **양쪽 끝**을 모두 명시합니다.
캐릭터 쪽 본(`Prop_RightHand` / `Prop_LeftHand`)과 무기 쪽 본
**`wb-ref-anim`**(`0x8CDA0E3F`)입니다. `wb-` 접두사는 무기 리그 고유의 네임스페이스입니다.
`wb-gunroot`, `wb-ref-anim`, `wb-LightRoot`.

### 이름 해시로 본 찾기

`[VERIFIED]` `FindBone`에 해당하는 것은 썽크 `0x0188D690`에서 `0x0EF071F0`입니다.

```c
(rcx = 스켈레톤/모델 객체, rdx = 출력 BoneHandle, r8d = CRC32(이름))
```

해석하기 전에 `[rdx+8] = 0xFFFFFFFF`와 `[rdx+0xC] = 0xFFFF`(유효하지 않은 본을 뜻하는
센티널)를 씁니다. **직접 호출 지점이 295곳**으로, 엔진에서 가장 많이 연결된 함수 중
하나이며, 게임이 무엇을 언제 조회하는지 배우기 좋은 지점입니다.

### gun-root 선택기

`[VERIFIED]` `0x080F8F10..0x080F9022`는 먼저 **`FakeGunRoot_Gameplay`**(`0x08B4DDD5`)를
해석하고 없으면 **`Fake_gunroot`**(`0x826846F3`)로 폴백하며, 채택된 해시를 반환합니다.
무기 컴포넌트의 `m_GunRootBone`이 무엇이 될지를 결정하는 것이 바로 이것입니다.

이 구분은 실재하고 중요합니다. 저 두 본 중 하나는 시각용 폴백이고 다른 하나는 게임플레이
장착점이며, 잘못된 쪽을 구동하면 아무것도 움직이지 않습니다. 가정하지 말고 측정해 볼 가치가
있습니다.

## 컴포넌트 레이아웃

`[VERIFIED, 리플렉션 테이블에서]` `GR_cWeaponComponent`, 해시 `0x1B15F6FA`, 디스크립터
`0x04AF60E0`, 크기 `0x260`:

| 오프셋 | 프로퍼티 | 타입 |
|---|---|---|
| `+0xA8` | `m_AssociatedEssence` | `WeaponEssence` |
| **`+0xB0`** | **`m_GunRootBone`** | `BoneHandle` |
| `+0xC0` | `m_MuzzleShootAnchor` | `BoneHandle` |
| `+0xD0` | `m_AimingPointAnchor` | `BoneHandle` |
| **`+0xE0`** | **`m_AttachmentHolder`** | `cWeaponAttachmentHolder` |
| `+0x100` | `m_SpreadSubComponent` | `sSpreadWeaponSubComponent` |
| `+0x158` | | `sBallisticControllerSubComponent` |
| `+0x17C` | `m_eWeaponCategory` | `EWeaponCategory` |

`[VERIFIED]` 이 네 오프셋은 **2026-08 업데이트를 거치며 바뀌지 않았습니다.** 이 엔진의
데이터 레이아웃이 코드 주소에 비해 얼마나 안정적인지를 보여 주는 유용한 데이터 포인트입니다.

`cWeaponAttachmentHolder`, 해시 `0xECFEF6C0`, 크기 `0xD8`:

| 오프셋 | 슬롯 |
|---|---|
| `+0x98` | `m_ScopeSlot` |
| `+0xA0` | `m_BulletSlot` |
| `+0xA8` | `m_MuzzleSlot` |
| `+0xB0` | `m_TriggerSlot` |
| `+0xB8` | `m_BarrelSlot` |
| `+0xC0` | `m_UnderBarrelSlot` |
| `+0xC8` | `m_RailSlot` |
| `+0xD0` | `m_MagazineSlot` |

전부 `GR_SingleSlot`이며, 그 `m_Stuff +0x40`이 슬롯을 차지한
`GR_EquipmentEssence`를 가리킵니다.

`GR_cInventoryHolder`: `m_PrimaryWeapon +0xF0`, `m_SecondaryWeapon +0xF8`,
`m_ThirdWeapon +0x100`, `m_CurrentHandledCategory +0x128`.

`UniquePropEssence`: `m_CurrentBoneHandle +0x70`, `m_iAttachmentBone +0x80`,
`v_cAttachmentBone +0x88`, `m_Incarnation +0x90`. `[INFERRED]`
`m_iAttachmentBone`은 `BipedBoneID` 서수이고 `v_cAttachmentBone`은 그것의 저작된
문자열입니다. 투척물 부착 정의 클래스가 `m_iBoneID +0x00` / `v_cBoneName +0x08`로 그 짝을
명시적으로 보여 줍니다.

`[UNKNOWN]` `m_AttachmentHolder`를 매 프레임 읽는 함수는 찾지 못했습니다.

## 발사체

`[VERIFIED]` 총알은 `cBallisticProjectileComponent`이며, 크기 `0x180`, 클래스 해시
`0x09BFE10E`입니다. 스폰 함수가 이를 할당하고 채우며, 필드 이름은 엔진 자체의 리플렉션
데이터에서 나온 것입니다.

```
movaps xmm1, [owner+0x150]  ->  [proj+0x50]   m_vBulletShootOrigin
movaps xmm0, [owner+0x140]  ->  [proj+0x100]  m_vBulletSimulationDirection
mov    [proj+0x20], owner                     역참조 포인터
```

즉 **발사 방향은 스폰 시점에 계산되지 않고** `[owner+0x140]`에서 읽어 옵니다.

`[VERIFIED NEGATIVE]`이며, 사격 방향을 바꾸려는 사람에게 정말로 유용한 부정 결과입니다.
눈에 보이는 발사체는 **명중을 결정하는 대상이 아닙니다.** 세 가지 개입을 각각 살아 있는
필드에서 실행되도록 만들고 카운터로 실제 적용을 확인했지만 탄착점은 움직이지 않았습니다.
발사체 자신의 위치 필드 재배치, `owner+0x140`의 스폰 방향 덮어쓰기, 발사마다 조준값을 읽는
코드 오버라이드가 그것입니다. 결정은 이 셋 모두보다 상류의 어딘가에서 이루어집니다.

아직 해결되지 않은 남은 실마리: 특정 Havok `castRay` 호출자가 모든 발사체 스폰 직전에
집중적으로 호출되고, 이후에는 전혀 호출되지 않습니다.

## Havok

`[VERIFIED]` `hknpWorld::castRay`는 Havok 자체의 모니터 타이머 문자열 리터럴
`TtWorldCastRay`를 상호 참조해 찾았습니다. `HK_TIMER_BEGIN`이 자신이 이름 붙이는 함수
**안에서** 프로파일링 스트림에 그 리터럴을 씁니다. 시그니처는
`castRay(hknpWorld* rcx, RayInput* rdx, Collector* r8)`입니다. 이미지 전체에 호출 지점은
**여덟 곳**뿐입니다.

또한 `TtCastRay` 본문이 따로 있는데, 세 개의 호출 지점이 런타임에 구성된 가상 테이블을
통해 `hknpWorld::castRay` 래퍼를 우회해 **직접** 도달합니다. 래퍼에만 후킹하면 그 셋을 보지
못하며, 이는 자신만만한 거짓 음성을 만들어 내는 전형적인 상황입니다.

본 수집 소비자 뒤에 있는 풀링된 배치 서브시스템은 렌더러의 것이 아니라 Havok의 것입니다.
