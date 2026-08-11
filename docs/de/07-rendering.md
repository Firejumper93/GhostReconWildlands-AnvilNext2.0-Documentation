[English](../07-rendering.md) · **Deutsch** · [한국어](../ko/07-rendering.md)

---

# 07 – Wie Geometrie tatsächlich platziert und gezeichnet wird

Alles in dieser Datei wurde aus **den ausgelieferten Shadern des Spiels selbst**
gewonnen, disassembliert mit der `d3dcompiler_47.dll`, die neben dem Spiel
mitgeliefert wird. Das ist erwähnenswert, weil es eine weit stärkere Quelle ist
als Schlussfolgerungen aus Code: Der Shader sagt dir genau, was die GPU liest.

Ein praktischer Hinweis zum Auffinden. Das DXBC der Material-Shader liegt in den
`.forge`-Daten, **nicht** in der Shader-Container-DLL. Jener Container enthält
ausschließlich Post-Process-, Terrain- und Wasser-Shader. Ein Scan darin nach
geskinnten Vertex-Shadern liefert nichts, was sich liest wie „diese Engine macht
kein Vertex-Skinning“ – und schlicht der falsche Container ist.

## Die drei Platzierungspfade

### Pfad A: GPU-instanziiert starr (Requisiten, Vegetation)

Strukturierte SRV bei **`t15`, Stride 56**. Die Bytes `0x00..0x2F` sind eine
`float3x4`-Weltmatrix; `0x30..0x37` sind zusätzliche Daten pro Instanz,
`[UNKNOWN]`.

Gesteuert durch die Geometrieoption `UseInstancing` (Bit 13 des Shader-Keys).

### Pfad B: nicht instanziiert starr

**Das ist der dominierende Pfad der Waffe.** Die Vertexposition liegt im
Objektraum und geht direkt in den Clip-Raum:

```
dp4 o0.x, r0.xyzw, cb4[0].xyzw     ; Objekt zu CLIP
...
dp3 r1.x, normal_obj, cb4[4].xyz   ; Objekt zu WORLD 3x3
```

Kein `t15`, kein Instanzpuffer, keine Knochenpalette. **Die gesamte Platzierung
liegt im Konstantenpuffer der Vertex-Stufe an Slot `b4`**, deklariert als
`CB4[20]` = 320 Bytes:

| Offset | Inhalt |
|---|---|
| `+0x00..0x3F` | Objekt-zu-Clip 4x4 (die Zeilen sind Koeffizientenvektoren) |
| `+0x40, +0x50, +0x60` | Objekt-zu-World 3x3 (`.xyz`; die `.w`-Lanes sind `[UNKNOWN]`) |
| `+0x130` (`cb4[19]`) | `.y` UV-/Detailskalierung, `.z` Instanz-Basisindex (nur Pfad A) |

`[VERIFIED]` Derselbe `b4`-Slot bedient beide Pfade: Instanziierte Draws erhalten
in `[0..3]` reine ViewProjection, Einzelobjekt-Draws erhalten
World*ViewProjection. Damit hast du bei jedem Requisiten-Draw eine kostenlose
Laufzeit-Gegenprobe für deine Matrixkonvention.

### Pfad C: Vertex-Shader-Skinning

Die Knochenpalette ist eine strukturierte SRV bei **`t6`, Stride 48,
`float3x4`**. Indizes werden roh und ohne Basisoffset verwendet, und weil die
geskinnte Position ohne Weltmatrix direkt in `cb4[0..3]` geht, liegen **die
Paletteninhalte im WORLD-Space**.

`t8` ist die Palette des vorherigen Frames, verwendet, wenn `UseMotionVector`
gesetzt ist.

### Die `t3`-gegen-`t6`-Falle

`[VERIFIED, Korrektur]` Es gibt eine zweite Palette mit Stride 48 bei **`t3`**,
und das ist der Pfad des **Compute-Pre-Skinnings**
(`PSComputeSkinningPositions4Bones_cs` / `8Bones_cs`, Ausgabe-UAV `u0` mit
Stride 48).

Der von Waffen- und Charaktermaterialien verwendete **Vertex-Shader**-Skinning-Pfad
bindet seine Palette an **`t6`**. Gleiches 48-Byte-`float3x4`-Layout, anderer
Slot.

**Eine Draw-Zeit-Sonde, die nur auf `t3` schaut, sieht bei einem Charakter- oder
Waffen-Draw nichts.** Das ist ein sauberer Negativbefund, der exakt aussieht wie
„die Palette ist nicht hier“, und er kostete zwei unabhängige Untersuchungen,
bevor das Shader-Listing die Sache klärte.

## Anzahl der Shader-Permutationen, die dir die Architektur verraten

`[VERIFIED]`

| Material | VS-Permutationen | Starr | Geskinnt | Instanziiert |
|---|---|---|---|---|
| `SHD_Weapon_InGame` | 142 | **122** | 20 | **0** |
| `SHD_Basic` (generische Requisiten) | 357 | | 36 | **192** |

Das Waffenmaterial ist durchgehend mit **ausgeschaltetem** `UseInstancing`
kompiliert. Null von 142 Permutationen nehmen den instanziierten Pfad, für Waffen
wird Pfad A also nie genommen.

## Der Shader-Key, entschlüsselt

`[VERIFIED]` Aus dem engine-eigenen Debug-Printer für Shader-Permutationen,
verankert an dessen Formatstrings `ShaderConfig: %s` und `Vertex Format: %s`:

- **ShaderConfig-Enum**: 0 = DepthOnly, 1 = GBuffer, 2 = Forward.
- **Namen der Geometrieoptions-Flags**, 14 Einträge = Bits 8..21 des
  Shader-Key-Dwords: `Is2Sided`, `Opaque`, `UseLowShaderLOD`,
  `UseTransparencyWithAlphaTest`, `IsLayer`, `IsParticle2D`,
  `IsParticleDepthFade`, `UseMotionVector`, `UseClustering`,
  `UseCharacterDecals`, `UseEnvInfluence`, `UseDissolve`, `UseClutterDissolve`,
  **`UseInstancing`**.
- **Vertexformat-Namen**: `Skinning4Bones`, `Skinning8Bones`, die
  `+Fat`-Varianten, `StaticFat`, `StaticQuantizedPosition` (plus `Fat` /
  `NoColor` / `NoColorFat`), `Position3f_*`, `BulkInstance`, `FakeMesh`,
  `Particle2D`, `TerrainMeshFormat`.

`[VERIFIED]` Render-Traversierungs-Tasks: `Graphic::Render::SplitInit`,
`SplitNodes`, `SplitEnd`, `ClearNodeBuckets`, alle in einem zusammenhängenden
Block registriert. Unter den Pass-Namen befindet sich **`PlayerCharactersPass`**:
Der Spieler und seine Ausrüstung werden in einem **eigenen Pass** gezeichnet,
getrennt von anderen Charakteren.

## Der Produzent der GPU-Knochenpalette

`[VERIFIED]` `BuildBoneMatrixPalette` bei `0x0DBCE0C0`, Thunk `0x013A7AB0`, mit
einer im gesamten Image eindeutigen Signatur, die keine rel32- oder
rip-Operanden enthält:

```
49 89 E3 49 89 5B 10 49 89 73 18 57 48 81 EC 90 00 00 00 41 0F 29 73 E8 48 89 D6 0F B7 51 1A
```

```c
BuildBoneMatrixPalette(Rig* rcx, F3x4** pLocal, F3x4** pOut)
```

Phase 1 normalisiert jedes Quaternion (`rsqrtps` plus ein Newton-Schritt) und
konvertiert `{quat, translation}` an Ort und Stelle zu einer `float3x4` **mit der
Translation in den `.w`-Lanes**. Phase 2 verkettet die Hierarchie hinab:

```
O[0] = L[0]
O[i].row = O[parent].row.xyz * L[i].rows + (O[parent].row & (0,0,0,1))
```

Die Indexarithmetik ist `lea edx,[rax+rax*2]; shl eax,4`, also x48.

### Dieser Pfad liest die Pose nie

`[VERIFIED]`, und das ist wichtig, falls du hoffst, dass ein CPU-Pose-Schreibzugriff
auf dem Bildschirm auftaucht. Ein Scan auf Byte-Ebene über **jede Funktion der
Kette** nach den Displacements `+0x178`, `+0x238`, `+0x8C` und nach `shl reg,5`
liefert **null bei allen vieren**.

Ihre Quelle ist ein **separates Array mit Stride 48 aus `{quat, translation}`,
gefüllt durch das Abtasten von Animationsclips**, elternrelativ – also der
entgegengesetzte Raum zum Pose-Puffer im Model-Space. Die Weltplatzierung wird
anschließend eingefaltet, indem jeder Eintrag mit einer 4x4-Matrix pro Instanz
nachmultipliziert wird.

Alle 74 Stellen im gesamten Image, die das Pose-Space-Flag prüfen, wurden
aufgezählt, und keine davon ist eine Massenschleife mit 48-Byte-Stride.

**Das CPU-Skelett und die GPU-Palette werden also aus verschiedenen Quellen
gespeist.** Das ist die konkrete Form einer Regel, die man sich bei jeder Engine
einprägen sollte: Die GPU-Palette zu beschreiben bewegt das, was du siehst, und
ändert nichts an dem, was das Spiel denkt; das CPU-Skelett vor dem finalen
Pose-Read zu beschreiben ändert Reichweite, Kollision und Mündungsposition. Das
ist nicht dasselbe, und eine Änderung, die das Bild bewegt, aber nicht die
Kugeln, ist keine Lösung.

### Die Übergabe an die GPU

`[VERIFIED]` Es ist ein **Map/Unmap in einen doppelt gepufferten dynamischen
Buffer**, kein `UpdateSubresource` und kein UAV-Dispatch. Die Map-Funktion hat
**38 Aufrufstellen** und ist der universelle Allokator der Engine für dynamische
Puffer; die Frame-Parität kommt aus einem Global, das mit `^ 1` umgeschaltet wird.

`[UNKNOWN]`, ehrlich gesagt: ob `0x0DBCE0C0` der Pfad ist, der **den Spieler und
seine Waffe** zeichnet. Zwei Fakten legen nahe, dass es der Instanz-/Crowd-Pfad
ist: Die Kette ist an `u16 [rig+0x5C] >= 2` gebunden (mindestens zwei Instanzen),
und ihre Ausgabe ist FP16 (12 Halbwörter, 24 Bytes pro Knochen) mit einem Stride
pro Instanz, **nicht** der float32-Puffer mit Stride 48, den der Compute-Shader
liest. Vier unabhängige heuristische Durchläufe über alle 378 MB der
Codesektionen fanden keinen zweiten `float3x4`-Produzenten, aber heuristische
Durchläufe sind kein Beweis.

Die Entscheidung ist billig zu treffen: Hooke den Thunk nur protokollierend und
zähle die Aufrufe, während nur der Spieler auf dem Bildschirm ist. Null Aufrufe
bedeutet, dass dies der Crowd-Pfad ist und der Produzent für die Heldenfigur noch
offen bleibt.

## Ein visueller Schreibzugriff als letzte Möglichkeit

Wenn der maßgebliche Weg scheitert, ist der garantiert visuelle der, `cb4[0..3]`
(Objekt zu Clip) und `cb4[4..6]` (Objekt zu World 3x3) des Konstantenpuffers der
Vertex-Stufe **im Moment des Draws** neu zu schreiben, innerhalb eines
D3D11-Detours. Es ist nachweislich der letzte Ort, an dem die Transformation
existiert, bevor die GPU sie sieht, es erfordert kein Wissen über den
spielseitigen Objektgraphen, und nichts kann es überschreiben, weil die GPU es bei
diesem Draw konsumiert. Setze
`newObjectToClip = desiredWorld * ViewProjection`; ViewProjection ist bereits aus
dem Matrixblock der Kamera verfügbar.

Für geskinnte Geometrie ist das Äquivalent, den Inhalt des strukturierten Puffers
`t6` für diesen Draw neu zu schreiben.

**Klare Einschränkung: Das ist rein visuell.** Es bewegt Pixel und sonst nichts.

### Ein billiger Laufzeit-Diskriminator, den das ermöglicht

Innerhalb eines D3D11-Detours, in einem begrenzten Stichprobenfenster: Lies
`VSGetShaderResources(6,1)` und `VSGetConstantBuffers(4,1)`. Ein Draw mit einem
nicht-null `t6` ist geskinnt; ein Draw mit null `t6` ist starr, und seine gesamte
Platzierung steckt im 320 Byte großen `b4`-Puffer.

## Zwei falsche Freunde

`[VERIFIED]` `sg_BulkSkinBuffer_vs` ist ein Passthrough aus zwei Instruktionen.
„Skin“ bedeutet dort Subsurface-**Hautschattierung**, nicht Skinning.

`[VERIFIED]` `0x0DBDEDD0` liest das Layout des Pose-Datensatzes **und** schreibt
48-Byte-Datensätze mit Stride `0x30`, wodurch es exakt wie der Palettenproduzent
aussieht. Ist es nicht: Sein Layout ist `{float3 T, float3 A, float3 B, float3
AxB}` mit der Translation in den Floats 0..2, **nicht** in den `.w`-Lanes.

## Wo man Draws hookt

`[VERIFIED]` Vtable-Patching ist bei D3D11-Draws auf diesem Ziel die falsche
Ebene, denn **jeder Device Context hat seine eigene Heap-Vtable**. Patchst du
eine, hast du einen Kontext gepatcht. Code-Detours innerhalb der `d3d11.dll`
selbst decken alle ab.

## Was hier unbekannt bleibt

- `[UNKNOWN]` Die CPU-Funktion, die `cb4` füllt, wurde nie gefunden. Scans nach
  dem Map-Idiom, strukturelle Scans und String-Anker liefen alle ins Leere.
- `[VERIFIED NEGATIVE]` Die D3D-Schicht des Renderers enthält **keine Strings**.
  Die D3D11-Fehlerstrings, die es im Image gibt, gehören zu NVIDIA TurfEffects,
  nicht zum Hauptrenderer, und können ihn daher nicht verankern.
- `[VERIFIED NEGATIVE]` Ein „hidden“-Flagbit pro Objekt, das für die Familie der
  Kopf-Attachments verifiziert wurde, zeigt **kein Anzeichen dafür, sich auf
  Waffen zu verallgemeinern**: Ein imageweiter Scan nach der entsprechenden
  Testinstruktion liefert genau einen Treffer, innerhalb eines Heap-Allokators.
  Es wurde kein Zeigerpfad von einer Waffenskelett-Instanz zu einem Render-Knoten
  gefunden.
