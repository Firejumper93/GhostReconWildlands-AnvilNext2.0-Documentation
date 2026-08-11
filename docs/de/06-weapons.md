[English](../06-weapons.md) · **Deutsch** · [한국어](../ko/06-weapons.md)

---

# 06 – Waffen und das Attachment-System

Die RVAs in dieser Datei stammen aus dem **Update-Build 2026-08**, sofern nicht
als Retail gekennzeichnet. Der 2026er-Rumpf von `SkeletonPostUpdate` ist
Instruktion für Instruktion identisch mit dem des Retail-Builds, die strukturellen
Anmerkungen lassen sich also übertragen, und es haben sich nur die Adressen
verschoben.

## Eine Waffe besteht aus vielen Objekten, nicht aus einem

`[VERIFIED, aus den ausgelieferten Shadern]` Waffen sind als **separate
Teile-Meshes** autoriert, je ein `.Mesh`-Asset:

```
W_ASR_AK-12_body_LOD0
W_ASR_AK-12_Barrel_LOD0
W_ASR_AK-12_Stock_LOD0
W_ASR_AK-12_StockFolded_LOD0
W_ASR_AK-12_Magazine_LOD0
W_ASR_AK-12_FlashHider_LOD0
W_ASR_AK-12_Ironsights_LOD0
```

`[VERIFIED]` Das wird unabhängig durch das Komponentenlayout bestätigt:
`cWeaponAttachmentHolder` hat **einen Slot pro Teil**.

Praktische Konsequenz, die eine Klasse mysteriös wirkender Bugs erklärt: Weil
jedes Teil ein eigenes gezeichnetes Objekt mit eigener Transformation ist,
**cullt oder versteckt man mit dem Waffenkörper nicht dessen Attachments.** Ein
in der Luft schwebender Schalldämpfer ohne angeschlossene Waffe ist das erwartete
Ergebnis, wenn man den Körper versteckt, kein Fehler.

`[INFERRED, stark]` Das Waffen-Rig mit 18 bis 21 Knochen existiert daher, um die
**austauschbaren Teile zu positionieren**, nicht um ein Mesh zu deformieren.
Teileanzahl plus Attachment-Punkte plus Mündungs-, Auswurf- und Visier-Sockets
landen exakt in dieser Größenordnung, und das Gunsmith-Anpassungssystem verlangt
Platzierung pro Teil.

## Die Attachment-Kette der gehaltenen Waffe, pro Frame

`[VERIFIED, statisch]` Von Anfang bis Ende, für eine in der Hand gehaltene Waffe:

```
character SkeletonPostUpdate            0x0F085A50  (Thunk 0x0189B430)
  -> Skeleton::PoseRefresh(character)   0x0F091720  (Thunk 0x018A09A0)
     -> Pose::SetRootTransform(Pose A)  0x0F203FA0  (Thunk 0x01904570)
     -> Pose::SetRootTransform(Pose B)  dieselbe Fn, Quelle = 4x4 des OWNER-ENTITY-Knotens
     -> Skeleton::PublishAttachments    0x0F090D00  (Thunk 0x018A03B0)
        -> Welttransformation des Handknochens aus Pose B des Charakters komponieren
           (Knochendatensatz + rootT/rootQ, wenn Flag-Bit 26 gelöscht ist)
        -> TransformNode::SetWorldTransform(Waffenknoten)
                                        0x0EAB1C60  (Thunk 0x017E1770)
           -> vtable-Notifier -> Entity::OnTransformChanged  0x0C426AC0
              -> Skeleton::MarkPoseDirty(Waffe)  0x0F080AA0 (Thunk 0x018996B0)
        -> Skeleton::PoseRefresh(Waffenskelett), inline, 7 Instruktionen später
           -> leitet BEIDE Waffen-Pose-Wurzeln neu her, löscht das Dirty-Bit
```

Lies dieses inline aufgerufene `PoseRefresh` aufmerksam, denn es schließt einen
ganzen Untersuchungsstrang ab: **Ein Schreibzugriff auf die Pose-Wurzel kann bei
einer gehaltenen Waffe nicht überleben.** Der Publish des Elternobjekts markiert
das Kind als dirty und leitet dessen Wurzeln unmittelbar danach neu her.

## `TransformNode::SetWorldTransform`

**Impl `0x0EAB1C60`, Thunk `0x017E1770`.**

```c
__fastcall(TransformNode* rcx, const float4x4* rdx, bool r8b, bool r9b)
```

Layout der Matrix in `rdx`: Die Zeilen bei `+0x00`, `+0x10`, `+0x20` sind die
Rotationsbasis, und **die Translation ist die vierte Zeile bei `+0x30`.**

`[VERIFIED]` Dies ist die Wahrheitsquelle für die Platzierung eines angehängten
Objekts: Die Pose-Wurzel der Waffe wird *daraus* regeneriert, und die
Transform-Changed-Benachrichtigung propagiert *daraus*.

Drei Eigenschaften machen es zu einem guten Abfangpunkt statt zu einem Kompromiss:

- `[VERIFIED]` Der Attachment-Publish ruft es **nur auf, wenn die neue Matrix von
  den aktuellen Zeilen des Knotens abweicht** (unmittelbar davor steht ein
  `cmpeqps`-Vergleichsblock). Eine injizierte Pose weicht immer ab, ein Intercept
  hat also immer das letzte Wort.
- `[VERIFIED]` Es ist der **letzte Schreiber** der Waffenplatzierung im Frame.
- `[VERIFIED]` Der Thunk ist ein standardmäßiger 5-Byte-`jmp` in einem
  `int3`-gepolsterten Slot.

`[VERIFIED]` Es hat **9 rel32-Aufrufstellen**, ist also eine allgemeine
Platzierungs-API und nicht waffenspezifisch. Eine Prüfung auf `rcx` ist
zwingend; ohne sie bewegst du alles in der Welt, das Transform-Knoten benutzt.

`[UNKNOWN]`, klar gesagt: Den Transform-Knoten zu bewegen beweist **Platzierung
und Zugehörigkeit**, nicht den GPU-Upload-Pfad. Ob es in jedem Fall auch visuell
ausreicht, hängt davon ab, wie die Geometrie dieses Objekts gezeichnet wird (siehe
[07-rendering.md](07-rendering.md)).

## Das alternative Schreibziel: die eigene lokale Matrix des Knotens

`[VERIFIED]` `PublishAttachments` berechnet `world = nodeLocal x boneWorld` und
committet mit gesetztem viertem Argument, was die lokale Neuberechnung
überspringt. Ein Wert, der in die eigene lokale Matrix der Waffe bei
`TransformNode+0x00` geschrieben wird, wird also jeden Frame konsumiert.

Das berührt nur die 64 Bytes, die dieses Objekt besitzt: kein gemeinsamer
Pose-Puffer, kein Skinning-Risiko, und es umgeht die Frage, welcher
Gun-Root-Knochen der echte ist. Zwei Stellschrauben sollten vorher wie erwartet
gelesen werden (`[node+0x48] == 0` bedeutet „erben“, und `[node+0x88] == null`).
Setze keine Skalierung in die lokale Matrix; sie wird wegorthonormalisiert.

## Die Attach-API

`[VERIFIED]` **`AttachEntityToBoneByNameHash`**, Thunk `0x0085DBB0` zum Rumpf
`0x094BF1A0`, 17 Aufrufstellen:

```c
(rcx = Kind-Entity, rdx = Eltern-Entity, r8d = CRC32(Knochenname))
```

Es löst die SkeletonComponent des Elternobjekts auf, führt ein FindBone per Hash
durch, **verweigert die Ausführung, wenn dem Rig dieser Knochen fehlt**, ruft
`SetParent` auf und registriert dann das Attachment über einen Shim, der einen
Standard-Lokaloffset liefert.

`[VERIFIED]` `BoneHandle` ist 16 Bytes groß:
`{Owner-Ptr +0x00, int32 index = -1 bei +0x08, u16 slotNum = 0xFFFF bei +0x0C}`.

`[VERIFIED]` **Die Aufrufstelle der Handauswahl** liegt bei `0x067EA0EB` innerhalb
der Funktion `0x067EA090`: Ein Selektor bei `[rcx+0x40]` wählt 0 für
`Prop_RightHand` (`0x53135E44`) oder 1 für `Prop_LeftHand` (`0x85562B5C`) und ruft
dann die Attach-API auf.

Eine zweite, datengetriebene Spezifikation beim Konstruktor `0x13ADBFB0` benennt
**beide Enden** der Verbindung: den Charakterknochen (`Prop_RightHand` /
`Prop_LeftHand`) und den waffenseitigen Knochen **`wb-ref-anim`**
(`0x8CDA0E3F`). Das Präfix `wb-` ist der eigene Namensraum des Waffen-Rigs:
`wb-gunroot`, `wb-ref-anim`, `wb-LightRoot`.

### Knochensuche über den Namens-Hash

`[VERIFIED]` Das `FindBone`-Äquivalent ist Thunk `0x0188D690` zu `0x0EF071F0`:

```c
(rcx = Skelett-/Modellobjekt, rdx = out BoneHandle, r8d = CRC32(Name))
```

Es schreibt `[rdx+8] = 0xFFFFFFFF` und `[rdx+0xC] = 0xFFFF` (das Sentinel für
einen ungültigen Knochen), bevor es auflöst. **295 direkte Aufrufstellen**, was es
zu einer der am besten vernetzten Funktionen der Engine macht und zu einem guten
Ort, um zu lernen, was das Spiel wann nachschlägt.

### Die Gun-Root-Auswahl

`[VERIFIED]` `0x080F8F10..0x080F9022` löst zuerst **`FakeGunRoot_Gameplay`**
(`0x08B4DDD5`) auf und fällt auf **`Fake_gunroot`** (`0x826846F3`) zurück, wobei
der gewinnende Hash zurückgegeben wird. Das entscheidet, was das
`m_GunRootBone` einer Waffenkomponente wird.

Die Unterscheidung ist real und relevant: Einer dieser beiden Knochen ist der
visuelle Fallback und einer die Gameplay-Aufhängung, und den falschen anzusteuern
bewegt überhaupt nichts. Das sollte man messen, nicht annehmen.

## Komponentenlayout

`[VERIFIED, aus den Reflection-Tabellen]` `GR_cWeaponComponent`, Hash
`0x1B15F6FA`, Deskriptor `0x04AF60E0`, Größe `0x260`:

| Offset | Eigenschaft | Typ |
|---|---|---|
| `+0xA8` | `m_AssociatedEssence` | `WeaponEssence` |
| **`+0xB0`** | **`m_GunRootBone`** | `BoneHandle` |
| `+0xC0` | `m_MuzzleShootAnchor` | `BoneHandle` |
| `+0xD0` | `m_AimingPointAnchor` | `BoneHandle` |
| **`+0xE0`** | **`m_AttachmentHolder`** | `cWeaponAttachmentHolder` |
| `+0x100` | `m_SpreadSubComponent` | `sSpreadWeaponSubComponent` |
| `+0x158` | | `sBallisticControllerSubComponent` |
| `+0x17C` | `m_eWeaponCategory` | `EWeaponCategory` |

`[VERIFIED]` Diese vier Offsets sind **über das Update 2026-08 hinweg
unverändert**, ein nützlicher Datenpunkt dazu, wie stabil das Datenlayout dieser
Engine im Vergleich zu ihren Codeadressen ist.

`cWeaponAttachmentHolder`, Hash `0xECFEF6C0`, Größe `0xD8`:

| Offset | Slot |
|---|---|
| `+0x98` | `m_ScopeSlot` |
| `+0xA0` | `m_BulletSlot` |
| `+0xA8` | `m_MuzzleSlot` |
| `+0xB0` | `m_TriggerSlot` |
| `+0xB8` | `m_BarrelSlot` |
| `+0xC0` | `m_UnderBarrelSlot` |
| `+0xC8` | `m_RailSlot` |
| `+0xD0` | `m_MagazineSlot` |

Alle sind `GR_SingleSlot`, deren `m_Stuff +0x40` auf die belegende
`GR_EquipmentEssence` zeigt.

`GR_cInventoryHolder`: `m_PrimaryWeapon +0xF0`, `m_SecondaryWeapon +0xF8`,
`m_ThirdWeapon +0x100`, `m_CurrentHandledCategory +0x128`.

`UniquePropEssence`: `m_CurrentBoneHandle +0x70`, `m_iAttachmentBone +0x80`,
`v_cAttachmentBone +0x88`, `m_Incarnation +0x90`. `[INFERRED]`
`m_iAttachmentBone` ist eine `BipedBoneID`-Ordinalzahl und `v_cAttachmentBone` ihr
autorierter String; eine Definitionsklasse für Wurfobjekt-Attachments macht diese
Paarung mit `m_iBoneID +0x00` / `v_cBoneName +0x08` explizit.

`[UNKNOWN]` Die Funktion, die `m_AttachmentHolder` pro Frame liest, wurde nicht
lokalisiert.

## Projektile

`[VERIFIED]` Das Geschoss ist `cBallisticProjectileComponent`, Größe `0x180`,
Klassen-Hash `0x09BFE10E`. Die Spawn-Funktion allokiert und füllt es, und die
Feldnamen stammen aus den engine-eigenen Reflection-Daten:

```
movaps xmm1, [owner+0x150]  ->  [proj+0x50]   m_vBulletShootOrigin
movaps xmm0, [owner+0x140]  ->  [proj+0x100]  m_vBulletSimulationDirection
mov    [proj+0x20], owner                     Rückzeiger
```

**Die Schussrichtung wird also nicht beim Spawn berechnet**; sie wird aus
`[owner+0x140]` gelesen.

`[VERIFIED NEGATIVE]`, und das ist ein wirklich nützlicher Negativbefund für alle,
die Feuer umlenken wollen: Das sichtbare Projektil ist **nicht das, was den
Treffer auflöst**. Drei getrennte Eingriffe wurden jeweils nachweislich auf einem
aktiven Feld ausgeführt, mit Zählern, die ihre Anwendung belegten – und die
Einschläge bewegten sich nicht: das Verschieben der eigenen Positionsfelder des
Projektils, das Umschreiben der Spawn-Richtung bei `owner+0x140` und das
Überschreiben des Zielwert-Lesers pro Schuss. Die Entscheidung fällt irgendwo
vorgelagert zu allen dreien.

Die verbleibende, ungelöste Spur: Ein bestimmter Havok-`castRay`-Aufrufer feuert
unmittelbar vor jedem Projektil-Spawn los und nie danach.

## Havok

`[VERIFIED]` `hknpWorld::castRay` wurde über einen Querverweis auf Havoks eigenes
Monitor-Timer-String-Literal `TtWorldCastRay` lokalisiert, das `HK_TIMER_BEGIN`
**aus dem Inneren** der Funktion, die es benennt, in den Profiling-Strom schreibt.
Signatur `castRay(hknpWorld* rcx, RayInput* rdx, Collector* r8)`. Im gesamten
Image existieren nur **acht Aufrufstellen**.

Es gibt außerdem einen `TtCastRay`-Rumpf, den drei Aufrufstellen **direkt**
erreichen und dabei den `hknpWorld::castRay`-Wrapper über eine zur Laufzeit
gebaute virtuelle Tabelle umgehen. Ein Hook allein auf dem Wrapper ist für diese
drei blind – genau die Art Sache, die ein selbstbewusstes falsch-negatives
Ergebnis erzeugt.

Das gepoolte Placement-Subsystem hinter dem Bone-Gather-Konsumenten gehört Havok,
nicht dem Renderer.
