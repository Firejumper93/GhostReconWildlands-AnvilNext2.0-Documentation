[English](../02-formats.md) · [Deutsch](../de/02-formats.md) · **한국어**

---

# 02 - 아카이브 및 데이터 파일 포맷

여기 실린 내용은 모두 배포된 와일드랜드 아카이브를 대상으로, 각 주장에 명시된 범위에서
`[VERIFIED]`되었습니다. 컨테이너 레이아웃은 Blacksmith(MIT)와 교차 확인했고, 포맷
상수는 AnvilToolkit의 디컴파일 결과와 대조했습니다(사실 확인 목적으로만 열람).

이 기록을 바탕으로 리더와 **바이트 단위로 동일한 결과를 내는 writer**를 모두 만들었으므로,
이 포맷은 가능한 가장 강한 의미로 검증되었습니다. 62GB 설치본의 forge 21개를 아무 변경
없이 다시 빌드하면 모든 파일이 바이트 단위로 재현되며, 수정한 forge는 배포된 게임에서
로드됩니다.

## `.forge` 컨테이너

```
header:   "scimitar" (8바이트), Unknown1 u8, FileVersionIdentifier i32 (= 27),
          OffsetToDataHeader u64

@OffsetToDataHeader (DataHeader1):
          NumOfEntries i32, Unknown1 i32[4], Unknown2 i64,
          MaxFilesForThisIndex i32 (= 5000), Unknown3 i32, OffsetToData i64

@OffsetToData (DataHeader2, 인덱스 섹션마다 하나):
          IndexCount i32, Unknown1 i32,
          OffsetToIndexTable i64, OffsetToNextDataSection i64 (-1 = 마지막),
          IndexStart i32, IndexEnd i32,
          OffsetToNameTable i64, Unknown2 i64

@OffsetToIndexTable:  IndexCount x {
          OffsetToRawData i64, FileDataID i64, RawDataSize i32 }      (20바이트)

@OffsetToNameTable:   IndexCount x {
          RawDataSize i32, Unknown1 i64, Unknown2 i32,
          ResourceIdentifier u32, Unknown3 i32[2], NextFileCount i32,
          PreviousFileCount i32, Unknown4 i32, Timestamp i32,
          Name char[128], Unknown5 i32[5] }                           (192바이트)
```

어렵게 알아낸 세부 사항:

- `[VERIFIED]` **이름 필드는 레코드 오프셋 40이 아니라 44에서 시작합니다.** 44바이트의
  앞부분에 `char[128]`과 `i32[5]`를 더하면 정확히 192바이트 스트라이드가 됩니다. 오프셋
  40에서 이름을 읽으면 `Timestamp` 필드의 출력 가능한 바이트가 모든 이름에 새어 듭니다.
  다른 도구에서 본 적이 있다면, 엉뚱한 항목 이름 `XGlobalMetaFile`(베이스 forge, 타임스탬프
  `0x5888C31C`)과 `JijGlobalMetaFile`(patch_01 forge, 타임스탬프 `0x6A692399`)이 바로
  여기서 나옵니다. **Timestamp는 40에 있습니다.**
- `[VERIFIED]` 이름 레코드 오프셋 4의 `i64`는 FileDataID가 **아닙니다.** 살아 있는
  레코드에서 인덱스 테이블의 값과 일치하지 않습니다. 파일 ID에 관해서는 인덱스 테이블이
  기준입니다.
- `[VERIFIED, 62GB 설치본 전체에서 270,004 / 270,004개 항목]` 이름 레코드의
  `RawDataSize`는 언제나 인덱스 레코드의 `RawDataSize`와 같습니다. 따라서 리패킹 시
  둘 다 고쳐야 하며, 그러지 않으면 아카이브가 불일치 상태가 됩니다.
- `[VERIFIED]` **항목 이름은 고유하지 않습니다.** `DataPC_GRN_WorldMap.forge`에 중복
  5,602개, `DataPC.forge`에 335개, 작은 것들에는 0개입니다. FileDataID는 forge 안에서
  고유*합니다*. **항목은 이름이 아니라 file_id로 지정하세요.**
- `[VERIFIED]` 헤더와 두 테이블은 첫 페이로드보다 **앞쪽에** 있습니다. 파일 구성은
  헤더, 인덱스 테이블, 이름 테이블, 작은 `_Lost&Found` 레코드, 예약 공간, 그다음
  페이로드 순입니다. 꼬리에 붙는 TOC가 아닙니다.
- `[VERIFIED]` 페이로드는 21개 forge 전부에서 빈틈도 겹침도 없이 **인덱스 순서대로
  연속** 저장됩니다.
- `[VERIFIED]` 전체 파일 크기는 마지막 페이로드의 끝을 **`0x8000`**으로 올림한 값이며,
  0으로 패딩됩니다. 21/21 forge에서 정확히 일치합니다. `0x4000`은 10/21에서만 맞으므로
  정렬 단위는 실제로 `0x8000`입니다.
- `[VERIFIED]` 여러 섹션을 가진 forge는 `OffsetToNextDataSection`으로 연결됩니다.
  `DataPC.forge`는 마침 29,640개 항목의 단일 섹션이지만, 일반적으로는 체인을 따라가야
  합니다. 잘 알려진 리더 중 최소 하나는 첫 섹션만 읽습니다.
- 바이트 단위로 동일한 리패킹을 원한다면 **읽을 때 항목을 정렬하지 마세요.** 파일 순서가
  그대로 유지되어야 합니다.

## 페이로드: Anvil 데이터 파일 블록 세트

각 항목의 페이로드는 하나 이상의 블록 세트입니다.

```
세트마다: magic u64 = 0x1004FA9957FBAA33
         version i16 (= 1)
         compression u8
         maxBlockSize u16 (= 0x8000), maxBlockSize2 u16
         blockCount i32
         blockCount x { uncompressedSize u16, compressedSize u16 }
         blockCount x { checksum u32, data[compressedSize] }
```

- `[VERIFIED]` 블록 크기는 `0x8000`(32,768)입니다. 크기 필드는 **u16**이며, 일부 리더가
  다른 Anvil 타이틀에 쓰는 i32가 아닙니다. 그 방언은 다릅니다.
- `[VERIFIED]` `compressedSize == uncompressedSize`인 블록은 압축 없이 그대로
  저장됩니다.
- `[VERIFIED]` 압축 바이트 0과 1은 **둘 다 LZO1X**입니다. (2 = LZO1A, 5 = LZO1C이며,
  와일드랜드에서는 아직 둘 다 관측되지 않았습니다.)

### 압축기는 LZO1X-1이 아니라 LZO1X-999다

`[VERIFIED]`이며, 이 문서에서 가장 강력한 단일 결과입니다.
`DataPC_GRN_TitleScreen_patch_01.forge`의 **모든 항목**을 압축 해제한 뒤
`lzo1x_999_compress`로 다시 인코딩하면 배포된 파일이 **바이트 단위로** 재현됩니다
(SHA-256 `1A7C3B8C...`). `lzo1x_1`은 유효하지만 약 15% 더 큰 스트림을 만듭니다.

따라서 이 툴체인의 압축기는 단지 게임의 것과 호환되는 정도가 아니라 게임의 것 *그
자체*입니다. 변경되지 않은 콘텐츠를 바이트 단위로 동일하게 리패킹해야 한다면 LZO1X-999를
쓰세요.

### 청크 단위 체크섬

`[VERIFIED, forge 3개에 걸쳐 789 / 789 청크]` **양쪽 누산기를 모두 0으로 초기화한
Adler-32**입니다. 표준 Adler-32는 `a = 1`로 시작하며, 그래서 기성 구현이 여기서
실패합니다.

```python
a = 0; b = 0
for byte in chunk_bytes:      # 저장된(압축되었을 수도 있는) 바이트
    a = (a + byte) % 65521
    b = (b + a)    % 65521
checksum = (b << 16) | a
```

유효한 아카이브를 쓰지 못하게 오랫동안 가로막던 장애물이 이것이었고, 실제로는 딱 저
초기화 차이 하나였습니다.

### 세트 구조는 논지의 핵심이다

`[VERIFIED]` 모든 순정 페이로드는 압축 바이트 1을 사용하며, 본문 세트 앞에 **작은 선행
블록 세트**를 하나 둡니다. 머티리얼 하나는 16바이트, 셀 데이터블록은 128바이트, 씬
디스크립터는 두 블록에 41,806바이트입니다. 그 선행 세트는 리소스의 헤더 레코드이며 엔진은
이를 별도로 읽습니다.

압축 바이트 0으로 **단일** 세트로 리패킹한 페이로드는 독립 리더에서는 제대로 압축이
풀리지만 **게임에서는 검은 화면이 됩니다.** 리테일 게임에서 확인했습니다. 따라서 리패킹은
단지 유효한 스트림을 만드는 데 그치지 않고 원래의 세트 구조와 압축 바이트를 보존해야
합니다.

## 페이로드 안의 다중 리소스 컨테이너

`[VERIFIED]` forge 항목의 압축을 푼 페이로드는 대개 리소스 하나가 아닙니다. 리소스들의
디렉터리입니다.

```
u16 count
count x { u64 file_id, u32 record_size, u16 flags }        (14바이트, TOC)
count x {
    u32 class_hash        # 클래스 이름의 CRC32
    u32 body_size
    u32 name_len
    char name[name_len]
    u8  gap[...]          # 보통은 이름을 끝내는 NUL 한 바이트
    u8  body[body_size]   # 다시 u64 file_id, u32 class_hash로 시작
}
```

- `record_size`는 레코드 **전체**(12 + name_len + gap + body_size)를 포함하므로, 본문
  위치는 `record_start + record_size - body_size`로 구합니다. NUL 하나를 가정하지 말고
  이렇게 도출하세요. 실제 레코드 중 최소 하나는 `name_len == 0`에 8바이트 갭을 가집니다.
  갭은 그대로 보존하세요.
- `[VERIFIED]` 모든 레코드에서 세 가지 독립적인 교차 검사가 성립합니다. TOC의 id가 본문
  안의 id와 같고, 바깥쪽 클래스 해시가 본문 안의 해시와 같으며, TOC의 `record_size`가
  실제로 훑어 나간 레코드 길이와 같습니다. 셋 다 강제하면 형태가 다른 페이로드가 조용히
  넘어가지 않고 요란하게 실패합니다.
- `[VERIFIED]` `class_hash`는 단순히 **클래스 이름의 CRC32**입니다.
  `crc32("Skeleton") == 0x24AECB7C`. 리소스 6,563개에서 관측된 서로 다른 클래스 해시 50개
  중 49개가 이 방식으로 실제 이름(Animation, Mesh, Material, Skeleton, Entity 등)으로
  해석됩니다.
- `[VERIFIED]` 커버리지: forge 3개 전체와 `DataPC.forge`에서 뽑은 400개 항목 표본을 합쳐
  페이로드 479개 중 473개가 컨테이너로 파싱되고, 그 473개는 모두 바이트 단위로 동일하게
  재빌드됩니다. 예외 6개는 `GlobalMetaFile` 항목 3개(구조가 다름)와
  `PrefetchingFileInfos` 3개(아래 참조)입니다.

### `PrefetchingFileInfos`와 `GlobalMetaFile`

`[VERIFIED]` 모든 forge는 `PrefetchingFileInfos` 항목으로 끝납니다. 이 항목은 데이터
파일 매직은 가지고 있지만 블록 세트 레이아웃은 **가지고 있지 않으며**, 블록 개수를 읽으면
말이 되지 않는 값이 나옵니다. 압축 해제에 계속 저항하고 있는 항목입니다.

다행히 그 페이로드에는 실제 항목들의 FileDataID가 들어 있고 **크기도 오프셋도 없습니다.**
따라서 오프셋을 재배치하고 페이로드 크기를 바꾸는 리패킹이 이 항목을 무효화하지 않습니다.
그대로 통과시키는 것이 안전합니다. `GlobalMetaFile`은 애초에 데이터 파일이 아니며(매직이
없음) 마찬가지로 그대로 통과시킵니다.

## 베이스 forge 대 패치 forge: 게임이 읽는 사본은 어느 쪽인가

`[VERIFIED]` 이 사실이 수정을 어디에 넣어야 하는지를 결정하며, 이를 틀리면 테스트가
실패하는 대신 조용히 결론 없는 상태가 됩니다.

`DataPC.forge`와 `DataPC_patch_01.forge`에는 둘 다 `GR_PLAYER_Template`라는 이름의 항목이
**같은 항목 file_id**로 들어 있지만, 같은 컨테이너가 아닙니다. 베이스는 1,838,722바이트에
리소스 1,241개, 패치는 4,916,946바이트에 리소스 2,238개입니다. 리소스 file_id 기준으로
1,230개가 양쪽에 있고, 11개는 베이스에만, 1,008개는 패치에만 있으며, 공통인 1,230개 중
**277개는 바이트가 다릅니다.**

베이스에만 있는 쪽에는 100본짜리 플레이어 신체 리그가 포함되어 있고, 게임에는 분명히
여전히 플레이어 신체가 있습니다. 따라서 엔진은 가장 최신 컨테이너를 통째로 취하지
않습니다. **엔진은 forge를 가로질러 리소스 file_id로 리소스를 해석하며, 패치는 자신이
실제로 담고 있는 id만 대체합니다.**

실무 규칙:

- **양쪽** forge에 모두 존재하는 리소스를 수정한다면 패치 사본(또는 양쪽)을 수정해야
  합니다. 그러지 않으면 변경이 가려져 테스트가 아무것도 증명하지 못합니다.
- **베이스에만** 있는 리소스를 수정하는 것은 모호하지 않습니다. 베이스가 유일한
  사본입니다.
- 작은 forge들(GhostRoom, TitleScreen과 그 패치)에는 Skeleton 리소스가 전혀 없으므로,
  스켈레톤 수정은 그것들로 테스트할 수 없습니다.

## writer가 할 수 있는 것과 할 수 없는 것

직접 하나 만들어 보며 확인한 내용입니다.

- `[VERIFIED]` 아무 변경 없는 재빌드는 62GB 설치본의 **forge 21개 전부에서 바이트 단위로
  동일(SHA-256)**하며, 여기에는 21GB / 63,351개 항목짜리 아카이브도 포함됩니다.
- `[VERIFIED, 리테일 게임에서]` **우리가 만든** 페이로드 바이트, **우리가 만든** 블록
  레이아웃, **우리가 계산한** 체크섬, **재배치된 오프셋**으로 다시 빌드한 forge는 세트
  구조와 압축 바이트를 보존한다는 전제 아래 정상적으로 로드됩니다. 실패했던 단일 세트
  시도는 양성 대조군 역할도 합니다. 검은 화면이 났다는 것은 엔진이 실제로 그 아카이브를
  읽는다는 뜻이고, 따라서 이후의 정상 부팅이 거짓 통과가 아님을 증명합니다.
- LZO 출력은 정규형이 아니므로, **변경된** 항목을 바이트 단위로 동일하게 리패킹하는 것은
  불가능하며 합리적인 목표도 아닙니다. 변경되지 않은 항목은 다시 인코딩하지 말고 페이로드
  바이트를 그대로 복사하세요.
- **항목 교체만 하세요.** 게임이 항목 개수 변경을 견딘다는 것은 아무것도 검증되지
  않았으므로, 항목을 추가하거나 제거하거나 순서를 바꾸는 것은 미검증 영역입니다.

이런 종류의 수정을 배포하는 사람을 위한 상시 주의 사항 두 가지: Steam 파일 검사는 수정을
되돌리며, 게임 패치는 무엇이든 수정을 무효화합니다.
