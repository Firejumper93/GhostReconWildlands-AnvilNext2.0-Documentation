[English](../04-pose.md) · **Deutsch** · [한국어](../ko/04-pose.md)

---

# 04 – Das Pose-Objekt und das tatsächliche Knochenlayout

Dies ist der Teil der Engine, der entscheidet, wo die Knochen einer Figur
tatsächlich sind, und hier stellte sich eine frühere, selbstbewusste Antwort als
falsch heraus. Beide Fassungen stehen unten, denn die falsche ist eine plausible
Lesart des Init-Codes, und jemand anderes wird ebenfalls darauf kommen.

## Die Kette von einem Skelett zu einem animierten Knochen

`[VERIFIED]`, auf drei unabhängigen Wegen: die Layout-Arithmetik in drei
getrennten Leaf-Accessoren.

```
Skeleton (0xCA0 Bytes)
  +0x010 -> besitzende ENTITY
  +0x120    eine vollständige {float4 T, float4 Q}-Transformation (ein veralteter Spiegel, siehe unten)
  +0x1F0    das Skelett-Lock
  +0x220 -> Rig-Deskriptor       pro Skelett-KLASSE geteilt, hält keine Transformationen
  +0x228 -> Rig-Instanz
  +0x230 -> Pose A               die Animationsausgabe
  +0x238 -> Pose B               DIE FINALE POSE (aliasiert bei +0x240 und +0x260)
  +0x268, +0x270 -> Posen C, D
  +0xC7C    Versionsstempel der Namenskarte
  +0xC90    Bit 7 = Pose dirty

Pose (0x190 Bytes)
  +0x000 float4   Wurzeltranslation
  +0x010 float4   Wurzelquaternion   (dessen w AUCH eine uniforme Skalierung trägt)
  +0x08C dword    Flags
  +0x090/+0x0A0   ein ZWEITES Transformationspaar, mit einem begleitenden Dword bei +0x160
  +0x170 ptr      das Rig (dasselbe Objekt wie skel+0x220)
  +0x178 ptr      DER KNOCHENTRANSFORMATIONSPUFFER

Knochenpuffer, Stride 0x20 pro Knotenindex i
  buf + 0x20*i + 0x00   float4 Translation
  buf + 0x20*i + 0x10   float4 Quaternion
```

## Die Accessoren, die die Grundwahrheit zum Layout sind

`[VERIFIED]` Jeder ist über einen einzelnen Jump-Thunk erreichbar:

| Was | Retail-Impl | Retail-Thunk | Update-Impl | Update-Thunk | Vertrag |
|---|---|---|---|---|---|
| `Rig::BoneIndexFromNameHash` | `0x0A85F0F0` | `0x00CF90F0` | | | `u16(Rig* rcx, u32 crc32 edx)`, `0xFFFF` bei Fehltreffer |
| `Pose::GetBoneRecord` | `0x0DC3F730` | `0x018BA4F0` | `0x0F1A51C0` | `0x018F00E0` | liefert `buf + idx*0x20` |
| `Pose::GetBoneTranslation` | `0x0DC3F9C0` | `0x018BA500` | | | `void(Pose*, u16 idx, float4* out r8)` |
| `Pose::GetBoneRotation` | `0x0DC3FCF0` | `0x018BA550` | | | `void(Pose*, u16 idx, float4* out r8)` |
| `Pose::Bind` (allokiert den Puffer) | `0x0DC4DAA0` | | `0x0F1B55C0` | `0x018F3460` | `mov [rdi+0x178], rax` |
| `Pose::SetRootTransform` | | | `0x0F203FA0` | `0x01904570` | |
| `PoseCopy` (nur Knochen) | | | `0x0F1CC7A0` | `0x018FA690` | |
| `Skeleton::PoseRefresh` | | | `0x0F091720` | `0x018A09A0` | |
| `Skeleton::PublishAttachments` | | | `0x0F090D00` | `0x018A03B0` | |
| `Skeleton::MarkPoseDirty` | | | `0x0F080AA0` | `0x018996B0` | |
| `SkeletonPostUpdate` | `0x0DA1A990` | `0x01865A10` | `0x0F085A50` | `0x0189B430` | `rcx` = Skelett |

`GetBoneRecord` ist vier Instruktionen lang und klärt die Stride-Frage
vollständig:

```
movzx eax, dx
shl   rax, 5           ; x 0x20
add   rax, [rcx+0x178]
ret
```

`GetBoneRotation` liest `+0x10` des Datensatzes und komponiert es, wenn das unten
genannte Flag nicht gesetzt ist, mit dem Wurzelquaternion bei `[pose+0x10]`. Das
beweist, dass der Datensatz `{Translation bei +0x00, Quaternion bei +0x10}` ist
und nicht umgekehrt.

Im gesamten Image eindeutige Byte-Signaturen (Retail):

```
GetBoneRecord       0F B7 C2 48 C1 E0 05 48 03 81 78 01 00 00 C3
GetBoneTranslation  40 53 48 83 EC 30 48 8B 81 78 01 00 00
Pose::Bind          48 89 5C 24 08 57 48 83 EC 20 8B 81 8C 00 00 00
SkeletonPostUpdate  F6 81 92 0C 00 00 01 48 89 CF
```

## Raum: Model, nicht World

`[VERIFIED]` Bei der Pose an `skel+0x238` wird Flag-Bit 26 bei der Erzeugung
ausdrücklich **gelöscht**, ihr Puffer liegt also im **MODEL-Space**:

```
Weltposition = rootQ angewandt auf boneT, plus rootT
Weltrotation = rootQ komponiert mit boneQ
```

Genau das macht die Engine selbst. **Normalisiere das Wurzelquaternion, bevor du
es als Rotation verwendest**, denn sein `w` trägt eine uniforme Skalierung; lässt
man das weg, multipliziert die Skalierung stillschweigend deine
Knochen-Offsets.

### Flag-Bits von `[pose+0x8C]`

Aus `Pose::Bind`:

| Bit | Maske | Bedeutung |
|---|---|---|
| 26 | `0x04000000` | Knochenpuffer liegt bereits im World-Space. Bei Pose B in der Konstruktion gelöscht, weshalb gehaltene Waffen als „gelöscht“ gelesen werden |
| 27 | `0x08000000` | Knochen seit der letzten Kopie geändert (gesetzt, nachdem der Solver schreibt, gelöscht von `PoseCopy`) |
| 28 | `0x10000000` | `[rig+0x36]` statt `[rig+0x34]` für die Knochenanzahl verwenden |
| 24, 25, 29, 31 | | sticky, über `Bind` hinweg erhalten (Maske `0xAF000000`); 25 und 31 sind `[UNKNOWN]` |
| 0..23 | | nie verwendet, `Bind` maskiert sie aus |

Respektiere Bit 26, auch wenn es in der Praxis immer gelöscht ist. Ein
zukünftiger Pfad, der dir einen World-Space-Puffer übergibt, würde sonst doppelt
transformiert – und das ist ein subtiler, schwer sichtbarer Fehler statt eines
offensichtlichen.

## Das durchgerechnete Beispiel der Engine selbst

`[VERIFIED]` Bei Retail `0x0A85B172` lädt die Engine `edx = 0x89B93A80` (CRC32
von `LeftForeArm`), legt das Rig in `rcx`, ruft die Namenssuche auf und übergibt
den zurückgegebenen Index an `GetBoneTranslation`. Anschließend wiederholt sie
das für `LeftHand`, `RightForeArm` und `RightHand`.

Das ist das kanonische Rezept, von der Engine selbst: **Namen hashen, Index gegen
das Rig auflösen, den Pose-Puffer indizieren.**

Für wiederholte Lesevorgänge gibt es bei `0x0DA15AE0` eine bessere Vorlage: ein
zwischengespeichertes Knochen-Handle `{u32 nameHash +8, u16 nodeIdx +0xC, u16
stamp +0xE}`, das nur dann neu aufgelöst wird, wenn der Stempel von `[skel+0xC7C]`
abweicht, mit einem Pose-Refresh, wenn `[skel+0xC90] & 0x80` gilt.

## Die Kette pro Frame

`[VERIFIED]` `SkeletonPostUpdate(skeleton)` ruft der Reihe nach auf: eine
Präambel, die Arbeitsfunktion (die damit endet, Pose A nach Pose B zu kopieren),
ein Culling, einen weiteren Schritt, eine Funktion, die die Wurzeltransformation
auf `+0x238` setzt und erneut kopiert, sowie zwei weitere.

`[VERIFIED]` **Ein Schreibzugriff auf `[[skel+0x238]+0x178] + idx*0x20` ist ein
CPU-Skelett-Schreibzugriff und die maßgebliche Seite.** Genau diesen Puffer geben
die drei Accessoren zurück, und sie werden von 34+ Stellen aus Animation,
Gameplay und Grafik aufgerufen.

`[VERIFIED]` **Ein Schreibzugriff auf `[skel+0x230]` (Pose A) ist sinnlos**: Sie
wird im selben Update wieder über `+0x238` kopiert.

`[VERIFIED]` **Die Pose-Wurzel ist kein Zustand, sondern eine abgeleitete Kopie
pro Frame.** `Pose::SetRootTransform` ist der einzige Schreiber von
`[pose+0x00]` / `[pose+0x10]` im gesamten Image (Signatur mit einem einzigen
Treffer: `48 83 EC 28 0F 28 02 0F 29 01 0F 28 4A 10 0F 29 49 10 45 84 C0`) und
wird aus dem Transform-Knoten der besitzenden Entity gespeist,
`[[skel+0x10]+0x18]`, dessen Translation die **vierte Zeile** bei `node+0x30`
ist. Sie wird niemals in die Gegenrichtung zurückgelesen.

`[VERIFIED]` `PoseCopy` kopiert **nur den Knochenpuffer** und rührt die Wurzel
nicht an.

## Veraltete Spiegel: drei Felder, die maßgeblich aussehen und es nicht sind

`[VERIFIED]` `Skeleton+0x120`, `Skeleton+0x250` und `owner+0x050` sind allesamt
veraltete Spiegel derselben vorgelagerten Entity-Position. `Skeleton+0x120` wird
nur von einem Reset-/Teleport-Pfad herangezogen, der nicht pro Frame läuft.

**Von keinem von ihnen führt ein Codepfad zum Renderer, sie zu beschreiben ist
also konstruktionsbedingt unsichtbar.** Wenn du beobachtest, dass eines von ihnen
bit-identisch mit der Pose-Wurzel ist, ist das gemeinsame Herkunft und kein
Beleg dafür, dass es die Wurzel speist.

Das hat zweimal echte Zeit gekostet. Wiederhole es nicht.

## Das Schreibfenster und das Lock

Wenn du einen Knochen ändern und die Engine deinen Wert verwenden lassen willst,
ist das Fenster schmal und genau festgelegt:

- **Zu früh** (vor dem Animations-Solver), und dein Schreibzugriff wird
  überschrieben.
- **Zu spät** (nach dem Konsumenten), und er hat keine Wirkung.
- Für angehängte Objekte ist der richtige Moment der **Eintritt in
  `Skeleton::PublishAttachments`**, denn dort komponiert die Engine die
  Welttransformation jedes angehängten Objekts aus der Pose und platziert es dann.

`[VERIFIED]` Die gesamte Sequenz läuft unter dem Skelett-Lock (`skel+0x1F0`) auf
einem Engine-Worker-Thread. Ein unsynchronisierter, hochfrequenter Schreibzugriff
von einem anderen Thread rennt dagegen an und kann ein `movaps`-Paar zerreißen –
hier sollte man also nicht nachlässig sein.

`[VERIFIED]` Ein Schreibzugriff auf die Pose-Wurzel **kann bei einem angehängten
Objekt nicht überleben**: Der Attachment-Publish des Elternobjekts markiert die
Pose des Kindes über den Transform-Changed-Notifier als dirty und ruft sieben
Instruktionen später inline `PoseRefresh` darauf auf, was beide Wurzeln neu
herleitet.

## Die Korrektur, bewusst beibehalten

Eine frühere Lesart des fünften Init-Durchlaufs des Rigs schloss, der Pose-Puffer
enthalte **zwei 4x4-Matrizen pro Knoten bei Stride 0x80**. Diese Lesart behandelte
die „Einheit“ des Init-Codes als 4 Bytes.

**Sie ist falsch.** Die Einheit ist 1 Byte, und der Stride ist `0x20` als
`{float4 T, float4 Q}`. Das `shl rax, 5` in `GetBoneRecord` ist eindeutig.

Zwei Folgekorrekturen aus demselben Fehler:

1. Der Zeiger auf den Pose-Puffer pro Charakter liegt **nicht** in einem
   Instanz-Pool-Slot. Er ist `Pose+0x178`.
2. Eine Laufzeitbeobachtung von „einer animierten 4x4-Matrix bei rig+0x430 mit
   einer Kopie bei +0x470“ passt nicht zu einem `0x20`-Stride und darf nicht als
   Beleg für irgendetwas weitergetragen werden. Es war ein Pose-Puffer, der im
   Heap in der Nähe des Rigs allokiert war, nicht das Rig selbst.
