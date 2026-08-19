[English](../01-binary.md) · [Deutsch](../de/01-binary.md) · **한국어**

---

> **안내, 2026-08-19.** 이 문서의 영어 원문은 2026-08-19에 확장되었으며,
> 이 번역에 아직 반영되지 않은 내용이 있습니다. 갱신 전까지는
> [영어 원문](../01-binary.md)을 기준으로 삼아 주십시오. 이전에 안내한
> 날짜(2026-08-17)를 지키지 못했으므로 이번에는 날짜를 명시하지 않습니다.

# 01 - 바이너리

## PE 관련 사실

`[VERIFIED]` 2017년 무렵의 리테일 `GRW.exe` 기준:

| 항목 | 값 |
|---|---|
| 아키텍처 | x86-64 |
| 선호 이미지 베이스 | `0x0000000140000000` |
| SizeOfImage | `0x1633A000` (약 369MB) |
| 진입점 RVA | `0x162AC020`, 97바이트짜리 `.tls` 섹션 내부 |
| 링커 | MSVC 14.22 |
| **ASLR (`DYNAMIC_BASE`)** | **꺼짐** (`DllCharacteristics = 0x8120`) |
| **CFG (`GUARD_CF`)** | **꺼짐** (`GuardCFFunctionTable = 0`) |
| DEP (`NX_COMPAT`) | 켜짐 |
| `HIGH_ENTROPY_VA` | 켜짐 |

ASLR이 꺼져 있다는 것은 RVA와 로드된 VA의 차이가 신뢰할 수 있는 상수라는 뜻입니다.
CFG가 꺼져 있다는 것은 간접 호출 대상이 등록되어 있을 필요가 없다는 뜻이며, 간접 호출을
리다이렉트할 일이 있다면 중요한 사실입니다.

## 섹션 테이블은 거짓말을 한다

`[VERIFIED]` 이 바이너리의 섹션 이름은 그 내용을 설명하지 않습니다. 이 문서에서 실무상
가장 중요한 단일 사실입니다.

| 이름 | VA | VSize | 플래그 | 실제로 들어 있는 것 |
|---|---|---|---|---|
| `.edata` | `0x00001000` | `0x0384E800` | CODE, EXEC, R | **점프 썽크** (아래 참조), 엔트로피 1.097 |
| `.link` | `0x03850000` | `0x0083A000` | INIT_DATA, R | **진짜 읽기 전용 데이터와 문자열** |
| `.text1` | `0x0408A000` | `0x00D1A000` | CODE, R, W | **1,462개 RTTI 디스크립터 전부** |
| `.pdata` | `0x04DA4000` | `0x0038B000` | INIT_DATA, R | 예외 언와인드 데이터 (이것만은 정직함) |
| `.sbss` | `0x0517C000` | `0x111266CC` | CODE, EXEC, R, W | **진짜 `.text`.** 모든 익스포트 RVA가 여기로 떨어짐 |
| `.code` | `0x162A3000` | `0x00007FE5` | CODE, EXEC, R | 스텁 영역. `dxgi.dll` / `d3d11.dll` / `XINPUT1_3.dll` 리터럴 보관 |
| `.tls` | `0x162AC000` | `0x00000061` | CODE, EXEC, R, W | 진입점을 포함 |
| `.rdata` | `0x162B2000` | `0x000872E8` | INIT_DATA, R | 리소스 |

**결론: 시그니처 스캔을 섹션 이름으로 절대 걸러내지 마세요.** 얌전하게 `.text`로만
범위를 좁힌 스캐너는 아무것도 찾지 못하고, `.edata`를 "정체불명 데이터"라며 건너뛰는
스캐너는 썽크 테이블 전체를 놓칩니다.

2026-08 업데이트 빌드에서는 리플렉션 디스크립터가 `.arch`라는 이름의 섹션
(`0x040FF000..0x04E38000`)에 있습니다.

## 코드는 디스크에 평문으로 있다

`[VERIFIED]` 디스크상의 파일에서 익스포트를 곧바로 디스어셈블하면 유효한 명령어가
나옵니다.

```
export ??1GraphicLibFacade@scimitar@@UEAA@XZ, RVA 0x0A8A10A0, section .sbss

0x000000014A8A10A0  48 8d 05 89 91 0a f9    lea rax, [rip - 0x6f56e77]
0x000000014A8A10A7  48 89 01                mov qword ptr [rcx], rax
0x000000014A8A10AA  c3                      ret
```

22개 명령어 디코딩, 무효 0개. **따라서 디스크상의 파일을 대상으로 한 정적 AOB 시그니처
도출은 유효하며**, 함수를 찾기 위해 프로세스를 실행할 필요가 전혀 없습니다. 저 익스포트
이름에 들어 있는 네임스페이스 `scimitar`는 엔진의 내부 이름이며, 곳곳에 등장합니다.

## 점프 썽크: 후킹 지점

`[VERIFIED]` 엔진은 자기 함수를 직접 호출하지 않습니다. 호출은 `int3`로 패딩된
16바이트 슬롯에 홀로 놓인 5바이트 상대 점프를 거칩니다.

```
0x01347280:  E9 5B 4E 1C 0B  CC CC CC CC CC CC CC CC CC CC CC
0x0135F720:  E9 BB 50 28 0B  CC CC CC CC CC CC CC CC CC CC CC
0x01349DF0:  E9 2B 6D 1C 0B  CC CC CC CC CC CC CC CC CC CC CC
0x013329A0:  E9 3B 84 1A 0B  CC CC CC CC CC CC CC CC CC CC CC
```

즉 구조는 `호출자 -> E9 rel32 (썽크, .edata) -> 실제 함수 (.sbss)`입니다.

이것이 이 대상에서 가장 좋은 후킹 지점이며, 이 바이너리에 특유한 이유들로 프롤로그
디투어보다 낫습니다.

- 14바이트짜리 `FF 25 00 00 00 00`(`jmp qword ptr [rip+0]`)에 8바이트 절대 주소를
  더한 것이 **썽크 자신의 16바이트 슬롯 안에 통째로 들어갑니다.** 스텁 테이블에
  14바이트를 쓰는 것만으로 호출을 리다이렉트하며, 진짜 코드는 전혀 건드리지 않습니다.
- 대체 함수는 호출자의 `call`에서 이어진 `jmp`로 진입하므로, 레지스터도 스택 레이아웃도
  반환 주소도 완전히 동일하게 보게 됩니다. 같은 시그니처를 가진 평범한 함수일 뿐입니다.
  **트램펄린도, 훔쳐 온 명령어도, 길이 디스어셈블러도 필요 없습니다.**
- `[VERIFIED]` 실제 함정 하나를 피해 갑니다. 두 카메라 수학 함수는 모두 **rsp 기준**
  명령어(`mov rax, rsp`, `mov [rsp+8], rbx`)로 시작합니다. 프롤로그를 복사하는 디투어는
  그 명령어들을 잘못된 스택 깊이에서 실행해 프레임을 망가뜨립니다. 썽크를 고쳐 쓰는
  방식에서는 이 문제가 아예 생기지 않습니다.
- 원래의 16바이트를 되돌리면 변경 사항이 프로세스 내에서 정확히 원상 복구됩니다.

일부 슬롯은 `E9` 썽크가 아니라 `mov rax,[rcx] / jmp qword ptr [rax+disp]` 형태의 가상
디스패치 스텁이며, 10바이트에 `int3` 패딩이 붙습니다. 같은 방식으로 후킹할 수 있지만,
검증하거나 되돌아갈 "원본 함수"가 없습니다. 대신 정확히 기대되는 바이트 시퀀스를
검증하고 디스패치를 직접 다시 구현하세요.

### 프로토타입을 모르는 함수 관찰하기

여기서 흥미로운 함수 상당수는 프로토타입을 확실하게 재구성할 수 없습니다(어떤 카메라
함수는 인자를 21개 받습니다). 견고한 접근법은 어셈블리 스텁입니다. Microsoft x64 ABI에서
인자를 나를 수 있는 모든 레지스터를 저장하고, 저장된 블록을 가리키는 포인터와 함께
레코더를 호출한 뒤, 전부 복원하고 실제 함수로 **테일 점프**하는 것입니다. 호출이 아니라
점프이기 때문에 스택으로 전달된 인자와 반환 주소가 그대로 유지되고, 실제 함수는 차이를
알아챌 수 없습니다.

정렬에 관한 세부 사항 하나. 이걸 틀리면 몇 주 동안 잠복하는 버그가 생깁니다. 썽크에서
`jmp`로 진입하면 `rsp`는 16으로 나눈 나머지가 8인 상태입니다. ABI는 `call` 직전에
`rsp`의 나머지가 0이기를 요구합니다. 따라서 스텁 자신의 프레임 예약량도 16으로 나눈
나머지가 8이어야 합니다(예: `0xB0`이 아니라 `0xB8`). 잘못된 값을 예약하면 레코더가
`movaps`로 xmm 레지스터를 저장하기 전까지는 모든 것이 잘 동작하다가, 첫 정렬 저장에서
폴트가 납니다.

## 런타임에 해석되는 그래픽 및 입력 API

`[VERIFIED]` `d3d11.dll`, `dxgi.dll`, `XINPUT1_3.dll`, `DINPUT8.dll`은 정적으로
임포트되지 **않습니다.** 런타임에 이름으로 해석되며, 리터럴은 `.code`에 있습니다.

| 문자열 | RVA |
|---|---|
| `CreateDXGIFactory1` | `0x162A8458` |
| `dxgi.dll` | `0x162A846B` |
| `D3D11CreateDevice` | `0x162A84B9` |
| `d3d11.dll` | `0x162A84CB` |
| `XINPUT1_3.dll` | `0x162A897E` |
| `DINPUT8.dll` | `0x162AADCF` |

`HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\KnownDLLs`에는 `dxgi`,
`d3d11`, `xinput` 항목이 없으므로 애플리케이션 디렉터리가 먼저 검색됩니다.

`[VERIFIED]` 특히 XInput은 수동으로 해석되어(`GetModuleHandle` + `GetProcAddress`)
고정된 데이터 전역 변수에 저장되고 그 전역을 통해 호출되며, **실제 임포트 테이블을 완전히
우회합니다.** 패치할 임포트 슬롯을 찾고 있다면, 그런 것은 없습니다. 그 데이터 전역이
유일한 포인터이고, 그 RVA는 재컴파일할 때마다 옮겨집니다.

`opengl32.dll`은 정적으로 임포트되지만, `glGetString`, `wglCreateContext`,
`wglDeleteContext`, `wglMakeCurrent`만 사용합니다. 즉 렌더링이 아니라 GPU 식별
용도입니다.

## 이미지에 존재하는 미들웨어

| 미들웨어 | 근거 |
|---|---|
| **Autodesk HumanIK** | 정적 링크, 이름은 제거됨. 설치 트리 어디에도 DLL이 없음. [03-skeleton.md](03-skeleton.md) 참조. |
| **Havok** | 물리 및 풀링된 배치 서브시스템. `TtWorldCastRay` 같은 자체 프로파일링 타이머 문자열 리터럴로 식별. |
| **NVIDIA Ansel** | `anselsdk64.dll`에서 정확히 세 개의 심벌 임포트: `setConfiguration`, `updateCamera`, `isAnselAvailable`. |
| **Tobii EyeX** | `tobii.eyex.client.dll`에서 37개 심벌, `tobii_head_pose_subscribe`와 `tobii_gaze_point_subscribe` 포함. |
| **NVIDIA TurfEffects, GFSDK SSAO, Volumetric Lighting** | 설치 루트에 별도 DLL로 존재. |

Havok이 프로파일링 문자열에서 스스로를 밝힌다는 점은 기법으로 기억해 둘 만합니다.
`HK_TIMER_BEGIN`은 *자신이 이름 붙이는 바로 그 함수 안에서* 프로파일링 스트림에 리터럴을
씁니다. 따라서 문자열 상호 참조를 따라가면 곧바로 해당 함수 본문에 도달합니다.

## 좌표 규약

`[VERIFIED]` 이것은 추론이 아니라 **엔진 스스로 명시한 기저 벡터**입니다. Ansel SDK는
타이틀이 자신의 좌표 규약을 선언하도록 요구하므로, `ansel::setConfiguration`을 후킹해
게임이 넘기는 구조체를 덤프하면 엔진의 입에서 직접 답을 얻을 수 있습니다.

```
right   = (+1, 0, 0)   = +X
up      = ( 0, 0,+1)   = +Z
forward = ( 0,+1, 0)   = +Y
```

| 속성 | 값 |
|---|---|
| 상방 축 | **+Z** |
| 전방 축 | **+Y** |
| 우측 축 | **+X** |
| 손 방향 | **오른손 좌표계** |
| 월드 스케일 | `metersInWorldUnit = 1.0`, 즉 **1 월드 단위 = 1미터** |

손 방향 도출: `right x up = X x Z = -Y`이고 `(right x up) . forward = -1.0`입니다.
기저 `(right, up, forward)`에 대해 `right x up = +forward`이면 왼손 좌표계,
`-forward`이면 오른손 좌표계입니다.

선언된 구조체, 일부만 식별됨:

```
+0x000  right   (1,0,0)
+0x00C  up      (0,0,1)
+0x018  forward (0,1,0)
+0x024  1.0        metersInWorldUnit
+0x028  45.0       [INFERRED] 속도 또는 각도 제한값
+0x02C  1   (int)
+0x030  8   (int)
+0x034  1.0
+0x038  01 01 01 01  [INFERRED] 불리언 4개
```

`[INFERRED]` 연속된 저 네 개의 `01` 바이트는 거의 확실히
`isCameraOffcenteredProjectionSupported`, `isCameraRotationSupported`,
`isCameraTranslationSupported`, `isCameraFovSupported`이며 전부 참입니다. 이 해석이
맞다면, 게임은 Ansel에 대해 비중심 투영, 회전, 이동, FOV 오버라이드를 지원한다고 선언한
셈입니다.

**+Z 상방에 +Y 전방은 이후의 Anvil 타이틀과 일치하므로**, 오디세이나 발할라용으로 작성한
기저 변환 코드가 그대로 넘어갑니다. 다만 손 방향은 이식해 오는 참조 구현이 무엇이든
한 번 더 확인할 가치가 있습니다. 잘 알려진 Anvil VR 코드베이스 중 최소 하나는 GLM이
왼손 좌표계라고 단언하면서 실제 투영 계산은 오른손 형태를 씁니다. 계산은 맞습니다. 그
단언은 엔진이 아니라 GLM 빌드 설정에 관한 것입니다.

## 엔진 태스크 그래프

`[VERIFIED]` 문자열 데이터 안에 이름이 붙은 엔진 태스크 867개가 연속된 블록으로
존재합니다. 프레임의 지도를 공짜로 얻는 셈입니다.

프레임 태스크:

| RVA | 태스크 |
|---|---|
| `0x0394A4B0` | `BeginFrame` |
| `0x0394A4C0` | `Engine::BeginFrame` |
| `0x0394A4D8` | `BeginGraphicFrame` |
| `0x0394A548` | `GraphicFrame` |
| `0x0394A570` | `EndGraphicFrame` |
| `0x0394A5A0` | `BeginEngineFrame` |
| `0x0394AC88` | `EndEngineFrame` |
| `0x0394ACD8` | `EndFrame` |
| `0x03A29338` | `ViewPreRender` |

카메라 태스크:

| RVA | 태스크 |
|---|---|
| `0x0394A9D8` | `UpdateCamera` |
| `0x0394A9E8` | `Ai::UpdateCamera` |
| `0x0394AD80` | `UpdateActionAfterCameraTask` |
| `0x03964DF0` | `WorldView` |
| `0x03964E30` | `CameraTransform` |

애니메이션 및 스켈레톤 태스크, 실행 순서대로:

| RVA | 태스크 |
|---|---|
| `0x03980110` | `SkeletonGatherComponents` |
| `0x03980168` | `SkeletonUpdate` |
| `0x03980178` | `SkeletonUpdateInteractions` |
| `0x0397FF10` | `AnimUpdateBones` |
| `0x0397FD18` | `AnimUpdateBonesAfterCamera` |
| `0x0397FC78` | `SkelUpdateBeginAnimateAfterCamera` |
| `0x0397FCA0` | `Anim::UpdateEndAnimateAfterCamera` |
| `0x03980408` | `SkeletonPostUpdate` |
| `0x039803C8` | `Anim::SkeletonPostUpdate` |
| `0x039801B0` | `AtomAfterSkeleton` |

`AfterCamera` 변종들이 흥미로운 부분입니다. 엔진은 카메라 업데이트 이전에 도는
애니메이션과 이후에 도는 애니메이션을 명시적으로 구분합니다.

`[VERIFIED]` **중요한 프레임 순서:** 스켈레톤 작업은 `Engine::EngineLoop::Step` 내부의
컴포넌트 업데이트 단계에서, 그래픽 단계 **이후에** 실행됩니다. 따라서 프레임 N의
`SkeletonPostUpdate` 출력은 프레임 N+1에서 렌더러가 소비합니다. 같은 프레임 안에서 팔레트
이후에 값을 쓸 수 있는 창은 없습니다.

## RTTI

`[VERIFIED]` MSVC RTTI 타입 디스크립터 1,462개를 복원할 수 있고 깔끔하게 디맹글되며,
이름이 오해를 부르는 `.text1`에 들어 있습니다. 이들은 게임플레이 클래스가 아니라 엔진
인프라(`scimitar` 네임스페이스)를 다룹니다.

게임플레이 클래스 이름은 이미지 안에 평문으로 **전혀** 존재하지 않습니다. 유일한 획득
경로인 CRC32 방식은 [08-reflection.md](08-reflection.md)를 참조하세요.
