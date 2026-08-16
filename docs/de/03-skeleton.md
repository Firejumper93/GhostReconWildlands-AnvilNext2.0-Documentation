[English](../03-skeleton.md) · **Deutsch** · [한국어](../ko/03-skeleton.md)

---

> **Hinweis, 2026-08-16.** Die englische Fassung dieses Kapitels wurde am
> 2026-08-16 erweitert und enthaelt Korrekturen, die hier noch fehlen.
> Diese Uebersetzung wird nachgezogen, voraussichtlich am 2026-08-17.
> Bis dahin ist die [englische Fassung](../03-skeleton.md) massgeblich.

# 03 – Skelette, Rigs und Knochenbenennung

## Knochenidentität ist überall CRC32

`[VERIFIED]` In einer `.Skeleton`-Nutzlast gibt es **keine ASCII-Knochennamen**.
Ein Scan nach druckbaren Strings im Rig des Spielerkörpers findet keine. Die
Knochenidentität ist ein schlichtes **CRC32 des Knochennamens**, und dasselbe
CRC32 wird für Klassennamen im Ressourcen-Container und für Eigenschaftsnamen in
den Reflection-Tabellen verwendet.

Das ist die nützlichste Einzeltatsache zum Datenmodell der Engine, denn sie
bedeutet: Von einem geratenen Namen kommst du mit einem einzigen Hash zur
engine-eigenen Kennung, und unbekannte Hashes lassen sich mit einer Rainbow Table
knacken.

Verifizierte Konstanten für das Rig des Spielerkörpers (CRC32 des HumanIK-Namens):

| Knochen | CRC32 | Knochenindex im Asset |
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

Klassen-Hashes funktionieren genauso: `crc32("Skeleton") == 0x24AECB7C`,
`crc32("Bone") == 0x95741049`, `crc32("cBallisticProjectileComponent") ==
0x09BFE10E`.

## Das `.Skeleton`-Assetformat

`[VERIFIED]` Das Rig des Spielerkörpers parst **byte-vollständig bis EOF mit 0
unerklärten Bytes** – das ist der Standard, an dem dieses Format gemessen wurde.

Das Datei-Framing ist `u64 Objekt-ID`, dann `u32 Klassen-Hash`.

Layout eines Knochendatensatzes:

```
pre-byte
u64  ID              (0xF80000XX)
u32  class hash      (0x95741049 = "Bone")
u32  Name            CRC32 des Knochennamens
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

`[VERIFIED]` Der **ObjectPtr-Dialekt von Wildlands**: Tag `02` bedeutet, es folgt
eine `u64`-ID; Tag `03` bedeutet null ohne Nutzlast. Wer das falsch macht,
desynchronisiert den gesamten Parse-Vorgang – es lohnt sich also, das zuerst zu
prüfen.

`[VERIFIED]` Strukturelle Invarianten, die über das ganze Rig hinweg gelten und
sich gut als Parser-Assertions eignen: Der gespeicherte `Index` entspricht der
Array-Position, und `ChildrenCount` entspricht der Größe des Teilbaums – bei
allen 100 Knochen.

### Das Rig des Spielerkörpers, konkret

`[VERIFIED]` `GR_PCF_Skeleton_Average.Skeleton`, 14.006 Bytes:

- **100 Knochen**, einzelne Wurzel (Knochen 0, `Reference`), Hierarchietiefe 12
- Bind-Pose ist Z-oben, **1,77 m groß**, Wurzelbecken bei z = 0,964
- 86 von 100 Knochennamen aufgelöst. Die 14 nicht aufgelösten sind Requisiten-
  und Attachment-Helfer, von denen zwei nachweislich auf `LeftWristTarget` /
  `RightWristTarget` enden
- `SkeletonKey == SkeletonHierarchyKey == 0x3121DFFF` für dieses Rig, was einen
  brauchbaren Scan-Anker liefert, um das Spielerskelett im Prozessspeicher zu
  finden

`[VERIFIED]` **Die Knochennamen folgen wortwörtlich der
Autodesk-HumanIK-Konvention**: `Reference`, `Hips`, die Spine-Kette, `Neck`,
`Neck1`, `Head`, `LeftShoulder`/`Arm`/`ForeArm`/`Hand` mit vollständigen
Fingerketten, rechts gespiegelt, plus die Helferpräfixe `O_` und `L_` (zum
Beispiel die Twist-Kette `O_01LeftForeArm`, `O_LeftElbow`, `L_01LeftArm`).

Das ist über die Benennung hinaus wichtig: Es bedeutet, dass das Laufzeit-Rig ein
echter HumanIK-Charakter ist, für den also die eigene Semantik von HumanIK gilt.

### IK-Daten im Asset

`[VERIFIED]` Der `IkChainDescriptor` im Asset lautet:

```
u32   ContactBoneID          (ein Knochen-NAMENS-Hash)
u32   EndEffectorBoneID
u32   IkChainStartBoneID
vec3  ContactOffset
bool  InvertIkResolution
ptr   MirrorIkChain
u8    IkChainSize
```

`[VERIFIED]` **`IKData` trägt in der Wildlands-Ära keine Nutzlast**, und der
Body von `IkChainsDefinitions` / `IKData` im Spieler-Rig ist null. **Die
Effektortabelle existiert nur zur Laufzeit**, passend dazu, dass HumanIK zur
Laufzeit gefüttert wird statt pro Asset autoriert zu sein. Suche nicht nach einer
eingebackenen Effektorliste; es gibt keine.

### Wo die Assets liegen

`[VERIFIED]` `GR_PLAYER_Template` dekomprimiert (1,8 MB auf 5,27 MB) zu ~1.240
typisierten Objekten, darunter allein in der Basis-Forge **143
`.Skeleton`-Instanzen**: Skelette für Körper, Bart, Haare sowie Rucksack- und
Westen-Attach-Punkte. Bemerkenswerte Mitglieder sind `GR_PCF_Skeleton_Average`
(das Körper-Rig), `Child_Skeleton` und Skelette pro Kopf.

Animations-Nutzlasten tragen ebenfalls keine ASCII-Knochennamen. Ein
Szenen-Deskriptor aus einer kleinen Forge liefert ~2.800 typisierte Objekte,
davon 2.177 `.Animation` – diese *mit* lesbaren Ubisoft-Namen (`sb_` für die
Soldaten-/Spielerklasse, `civ_` und `rbl_` für Zivilisten und Rebellen).

## Der Laufzeit-Rig-Deskriptor

`[VERIFIED, aus der Disassemblierung seines Konstruktors und der Init-Durchläufe]`
Das Rig-Objekt ist ein **gemeinsam genutzter, refcount-verwalteter, `0xF8` Byte
großer Skelett-DESKRIPTOR**, in einem globalen Manager nach Dword-ID
zwischengespeichert. Charaktere mit demselben Skelett-Datablock **teilen sich eine
Rig-Instanz**: Das Rig identifiziert die Skelett-*Klasse*, nicht den einzelnen
Charakter. Es hält überhaupt keine Transformationen.

Schlüsselfelder (Offsets des Retail-Builds):

| Offset | Inhalt |
|---|---|
| `+0x08` | Refcount |
| `+0x20` | Zeiger auf einen `0x60` Byte großen Instanz-Pool-Helfer |
| `+0x28` | Vektor aus 8-Byte-Kanaldatensätzen `{u32 nameHash, u16 Pose-Puffer-Offset, u16 Typ-/Indexbits}` |
| `+0x34` | Gesamtgröße des Pose-Puffers |
| `+0x3C` | Dword-Rig-ID |
| `+0x42` | Anzahl der Layer |
| **`+0x50`** | **sortierte NAMENSKARTE `{u32 CRC32-Knochennamen-Hash, u16 Knotenindex}`** |
| **`+0x5A`** | **Anzahl der Einträge in der Namenskarte** |
| `+0x68` | Knoten-zu-Datensatz-Remap |
| `+0x80` | Knotendatensätze, je 16 Bytes: Parent-Word bei `+0`, Gruppen-Byte bei `+9`, Flag-Byte bei `+0xA` (Bit 3 = HIK-markiert, Bits 4-6 LOD) |
| **`+0x8A`** | **Knochenanzahl** |
| `+0x98` | Hash-Sortier-Permutation |
| `+0xA4` | Liste HIK-markierter Knoten |
| `+0xD0` | `CRITICAL_SECTION` |

`[VERIFIED]` Die Namenskarte bei `+0x50` ist nach Hash sortiert, eine
Binärsuche darüber löst also einen Knochennamen-Hash zu einem Knotenindex auf,
ohne Engine-Code aufzurufen. Die engine-eigene Suche ist
`Rig::BoneIndexFromNameHash`, Retail-Implementierung `0x0A85F0F0` über Thunk
`0x00CF90F0`, Signatur `u16(Rig* rcx, u32 crc32 edx)`, liefert bei einem
Fehltreffer `0xFFFF`.

Der Instanz-Pool-Helfer ist `0x60` Bytes groß mit einem Pool aus 64 Slots (eine
`0x1808`-Allokation von 64 x `0x60`), Free-List-Kopf bei `+0x50`,
Active-List-Sentinel bei `+0x58`, Slots verkettet über next bei `slot+0x48` und
prev bei `slot+0x40`. Also bis zu **64 lebende Charakterinstanzen pro
Rig-Klasse**.

`[VERIFIED]` Ein globales HIK-Manager-Singleton hält eine Hash-Map, deren
Schlüssel die Rig-ID ist, mit dem Rig-Zeiger bei `node+0x10`. Wer sie
durchzählt, erhält **jedes lebende Rig**.

`[VERIFIED]` Identitätskette auf Skelettseite: `[skel+0x30]` führt zu einem
Schutzblock `{verschlüsseltes qword +0, Schlüssel-Dword +0xC, KLARER
Datablock-Zeiger bei +0x10}`, wobei der Datablock-Zeiger lesbar ist, ohne
irgendetwas zu dekodieren. `[skel+0x2C8]` ist eine sortierte Override-Liste
`{u32 Hash, u32 LOD}` pro Knochen.

## `Skeleton::BipedBoneID`: das Attachment-Punkt-Enum

`[VERIFIED, Update-Build 2026-08]` Tabelle bei RVA `0x046F5080`, **143 Einträge,
Stride 8**, jeder Eintrag `{u32 CRC32(Knochenname), u32 CRC32(BIPEDBONE_*-Tag)}`
mit dem Index als Ordinalzahl. 132 von 143 Namen wiederhergestellt.

Das ist die engine-eigene Aufzählung der Attachment-Slots, und darauf bildet das
Konzept der „Attachment-Slots“ in späteren Anvil-Titeln ab. Slot-Nummern aus
anderen Titeln lassen sich **nicht** übertragen.

| idx | Hash | Name |
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

Beachte das Paar bei 115/116: Es gibt eine **visuelle** Waffenwurzel und eine
separate **Gameplay**-Waffenwurzel, und das sind unterschiedliche Knochen. Welche
die Engine wählt, entscheidet eine Auswahlfunktion, die zuerst
`FakeGunRoot_Gameplay` auflöst und auf `Fake_gunroot` zurückfällt.

`[VERIFIED]` Ebenfalls bei `0x03AD1230`: ein Quadrupel aus vier Dwords
`{LeftHand, RightHand, LeftHand_Weapon_Ref, RightHand_Weapon_Ref}`, unmittelbar
gefolgt vom String `"BipedIkParamsRoot"`. Das ist die
Hand-/Waffenreferenzmenge der IK-Parameter.

Das Waffen-Rig hat sein eigenes Knochenpräfix, `wb-`: `wb-gunroot`,
`wb-ref-anim` (`0x8CDA0E3F`), `wb-LightRoot`.

## HumanIK: nur als Daten vorhanden

`[VERIFIED]` HumanIK ist statisch gelinkt, mit **entfernten Namen**. Nirgends im
Installationsbaum existiert eine HumanIK-DLL, und die Daten-Tags stecken in der
ausführbaren Datei:

| RVA (Retail) | Tag |
|---|---|
| `0x03A81BF8` | `HIKCHARACTER000` |
| `0x03A81C63` | `HIKSTATE0000000` |
| `0x03A81CD8` | `HIKEFFECTOR0000` |
| `0x03A81CF0` | `HIKPROPERTY0000` |
| `0x03A81D08` | `HIKDATABLOCK000` |

Die **Effektor-Namenstabelle** liegt zusammenhängend (Retail
`0x03C7FE30..0x03C80238`; Update-Build `0x03CD8370..0x03CD8790`, 44 Einträge) und
ist faktisch das Effektor-Enum: `HipsEffector`, `LeftWristEffector`,
`RightWristEffector`, `HeadEffector`, `LeftHandEffector`, `RightHandEffector`,
`ChestOriginEffector`, `ChestEndEffector`, dazu jeder Finger sowie eine
korrespondierende `*Tip`-Markertabelle.

Die **Eigenschaftstabelle** enthält wortwörtlich die Eigenschaftsnamen von
Autodesk HumanIK 4.x: `ReachActorLeftShoulder`, `ReachActorRightShoulder`,
`SnSReachLeftWrist`, `SnSReachRightWrist`, `SnSReachHead`,
`ParamRealisticArmSolving` und so weiter.

`[VERIFIED]` Diese Arrays werden über **absolute gespeicherte Zeiger**
referenziert (relokiert), sind also in Laufzeittabellen eingebunden und keine
toten Daten.

`[VERIFIED NEGATIVE]` **Nirgends im Image existiert ein Symbolname der
öffentlichen HumanIK-API.** `HIKSetEffectorStateTQSfv`, `HIKSolveForEffectorSet`,
`HIKSetNodeStateTQSfv`, `HIKCharacterCreate` und `HIKEffectorSetStateCreate`
liefern über die gesamte 405 MB große Stringmenge hinweg null Treffer. Die
einzigen HIK-Treffer sind jene fünf Daten-Tags plus Rauschen aus gepackten Blobs.

Der Solver ist also nur über seine Form oder über die Effektor-Namenstabellen
erreichbar. Ob seine Effektor-Schnittstelle von außen angesteuert werden kann,
ist **`[UNKNOWN]`**, und die Namenstabellen sind der Anker, von dem aus man es
versuchen würde.
