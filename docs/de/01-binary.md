[English](../01-binary.md) · **Deutsch** · [한국어](../ko/01-binary.md)

---

# 01 – Das Binary

## PE-Fakten

`[VERIFIED]` Aus der `GRW.exe` des Retail-Builds der 2017er-Ära:

| Fakt | Wert |
|---|---|
| Architektur | x86-64 |
| Bevorzugte Image-Basis | `0x0000000140000000` |
| SizeOfImage | `0x1633A000` (etwa 369 MB) |
| Entry-Point-RVA | `0x162AC020`, innerhalb einer 97 Byte großen `.tls`-Sektion |
| Linker | MSVC 14.22 |
| **ASLR (`DYNAMIC_BASE`)** | **AUS** (`DllCharacteristics = 0x8120`) |
| **CFG (`GUARD_CF`)** | **AUS** (`GuardCFFunctionTable = 0`) |
| DEP (`NX_COMPAT`) | AN |
| `HIGH_ENTROPY_VA` | AN |

Dass ASLR aus ist, bedeutet: RVAs und die geladene VA unterscheiden sich um eine
Konstante, auf die man sich verlassen kann. Dass CFG aus ist, bedeutet: Ein Ziel
eines indirekten Aufrufs muss nicht registriert sein – wichtig, falls du je eines
umleitest.

## Die Sektionstabelle lügt

`[VERIFIED]` Die Sektionsnamen dieses Binaries beschreiben ihren Inhalt nicht.
Das ist der praktisch wichtigste Einzelfakt in dieser Datei.

| Name | VA | VSize | Flags | Was tatsächlich darin ist |
|---|---|---|---|---|
| `.edata` | `0x00001000` | `0x0384E800` | CODE, EXEC, R | **Jump-Thunks** (siehe unten), Entropie 1,097 |
| `.link` | `0x03850000` | `0x0083A000` | INIT_DATA, R | **die echten Read-only-Daten und Strings** |
| `.text1` | `0x0408A000` | `0x00D1A000` | CODE, R, W | **alle 1.462 RTTI-Deskriptoren** |
| `.pdata` | `0x04DA4000` | `0x0038B000` | INIT_DATA, R | Exception-Unwind-Daten (diese eine ist ehrlich) |
| `.sbss` | `0x0517C000` | `0x111266CC` | CODE, EXEC, R, W | **das echte `.text`.** Jede Export-RVA landet hier |
| `.code` | `0x162A3000` | `0x00007FE5` | CODE, EXEC, R | Stub-Region; enthält die Literale `dxgi.dll` / `d3d11.dll` / `XINPUT1_3.dll` |
| `.tls` | `0x162AC000` | `0x00000061` | CODE, EXEC, R, W | enthält den Entry Point |
| `.rdata` | `0x162B2000` | `0x000872E8` | INIT_DATA, R | Ressourcen |

**Konsequenz: Filtere einen Signatur-Scan niemals nach Sektionsnamen.** Ein
Scanner, der sich brav auf `.text` beschränkt, findet überhaupt nichts, und ein
Scanner, der `.edata` als „nicht identifizierte Daten“ überspringt, verpasst die
gesamte Thunk-Tabelle.

Im Update-Build von 2026-08 liegen die Reflection-Deskriptoren in einer Sektion
namens `.arch` (`0x040FF000..0x04E38000`).

## Code liegt im Klartext auf der Platte

`[VERIFIED]` Ein Export direkt aus der Datei auf der Platte zu disassemblieren
liefert gültige Instruktionen:

```
export ??1GraphicLibFacade@scimitar@@UEAA@XZ, RVA 0x0A8A10A0, section .sbss

0x000000014A8A10A0  48 8d 05 89 91 0a f9    lea rax, [rip - 0x6f56e77]
0x000000014A8A10A7  48 89 01                mov qword ptr [rcx], rax
0x000000014A8A10AA  c3                      ret
```

22 Instruktionen dekodiert, 0 ungültig. **Die statische Ableitung von
AOB-Signaturen gegen die Datei auf der Platte ist damit gültig**, und du brauchst
nie einen laufenden Prozess, um eine Funktion zu finden. Der Namespace in diesem
Exportnamen, `scimitar`, ist der interne Name der Engine und taucht durchgehend
auf.

## Jump-Thunks: die Hooking-Fläche

`[VERIFIED]` Die Engine ruft ihre eigenen Funktionen nicht direkt auf. Aufrufe
laufen über einen 5-Byte-Relativsprung, der allein in einem 16 Byte großen, mit
`int3` aufgefüllten Slot liegt:

```
0x01347280:  E9 5B 4E 1C 0B  CC CC CC CC CC CC CC CC CC CC CC
0x0135F720:  E9 BB 50 28 0B  CC CC CC CC CC CC CC CC CC CC CC
0x01349DF0:  E9 2B 6D 1C 0B  CC CC CC CC CC CC CC CC CC CC CC
0x013329A0:  E9 3B 84 1A 0B  CC CC CC CC CC CC CC CC CC CC CC
```

Die Form ist also `Aufrufer -> E9 rel32 (Thunk, .edata) -> echte Funktion
(.sbss)`.

Das ist die beste Hooking-Fläche auf diesem Ziel, und sie ist aus Gründen, die
spezifisch für dieses Binary sind, besser als ein Prolog-Detour:

- Ein 14 Byte großes `FF 25 00 00 00 00` (`jmp qword ptr [rip+0]`) plus eine
  8-Byte-Absolutadresse **passt vollständig in den 16-Byte-Slot des Thunks
  selbst**. Du leitest einen Aufruf um, indem du 14 Bytes in eine Tabelle von
  Stubs schreibst, und rührst dabei überhaupt keinen echten Code an.
- Dein Ersatz wird per `jmp` aus dem `call` des Aufrufers betreten und sieht
  daher identische Register, identisches Stack-Layout und identische
  Rücksprungadresse. Es ist eine gewöhnliche Funktion mit derselben Signatur.
  **Kein Trampolin, keine gestohlenen Instruktionen, kein Längen-Disassembler.**
- `[VERIFIED]` Es umgeht eine echte Falle: Beide Kameramathematik-Funktionen
  beginnen mit **rsp-relativen** Instruktionen (`mov rax, rsp`,
  `mov [rsp+8], rbx`). Ein Detour mit kopiertem Prolog würde diese in der
  falschen Stacktiefe ausführen und den Frame beschädigen. Beim Umschreiben eines
  Thunks stellt sich die Frage gar nicht erst.
- Das Wiederherstellen der ursprünglichen 16 Bytes macht die Änderung im Prozess
  exakt umkehrbar.

Manche Slots sind keine `E9`-Thunks, sondern Virtual-Dispatch-Stubs der Form
`mov rax,[rcx] / jmp qword ptr [rax+disp]`, 10 Bytes plus `int3`-Padding. Diese
lassen sich auf dieselbe Weise hooken, aber es gibt keine „Originalfunktion“, an
der man verifizieren oder zu der man zurückkehren könnte. Verifiziere stattdessen
die exakt erwartete Byte-Sequenz und implementiere den Dispatch selbst nach.

### Eine Funktion beobachten, deren Prototyp du nicht kennst

Viele der interessanten Funktionen hier haben Prototypen, die man nicht
zuverlässig rekonstruieren kann (eine Kamerafunktion nimmt 21 Argumente). Der
robuste Ansatz ist ein Assembler-Stub, der jedes Register sichert, das unter der
Microsoft-x64-ABI ein Argument tragen kann, einen Rekorder mit einem Zeiger auf
den gesicherten Block aufruft, alles wiederherstellt und dann per **Tail-Jump**
zur echten Funktion springt. Weil er springt statt aufzurufen, bleiben
stack-übergebene Argumente und die Rücksprungadresse unangetastet, und die echte
Funktion kann keinen Unterschied feststellen.

Ein Detail zur Ausrichtung, denn ein Fehler hier erzeugt einen Bug, der wochenlang
latent bleibt: Der Eintritt per `jmp` aus einem Thunk hinterlässt `rsp` kongruent
zu 8 mod 16. Die ABI will `rsp` unmittelbar vor einem `call` kongruent zu 0
mod 16. Also muss die eigene Frame-Reservierung des Stubs selbst 8 mod 16 sein
(zum Beispiel `0xB8`, nicht `0xB0`). Reserviere den falschen Betrag, und alles
funktioniert – bis der Rekorder xmm-Register mit `movaps` sichert, woraufhin der
erste ausgerichtete Store einen Fault auslöst.

## Zur Laufzeit aufgelöste Grafik- und Eingabe-APIs

`[VERIFIED]` `d3d11.dll`, `dxgi.dll`, `XINPUT1_3.dll` und `DINPUT8.dll` werden
**nicht** statisch importiert. Sie werden zur Laufzeit über den Namen aufgelöst,
und die Literale liegen in `.code`:

| String | RVA |
|---|---|
| `CreateDXGIFactory1` | `0x162A8458` |
| `dxgi.dll` | `0x162A846B` |
| `D3D11CreateDevice` | `0x162A84B9` |
| `d3d11.dll` | `0x162A84CB` |
| `XINPUT1_3.dll` | `0x162A897E` |
| `DINPUT8.dll` | `0x162AADCF` |

`HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\KnownDLLs` hat keinen
Eintrag für `dxgi`, `d3d11` oder `xinput`, daher wird zuerst im
Anwendungsverzeichnis gesucht.

`[VERIFIED]` Speziell XInput wird von Hand aufgelöst (`GetModuleHandle` +
`GetProcAddress`), in ein festes Daten-Global geschrieben und über dieses Global
aufgerufen, **womit die echte Importtabelle vollständig umgangen wird**. Wenn du
nach dem Import-Slot zum Patchen suchst: Es gibt keinen. Das Daten-Global ist der
einzige Zeiger, und seine RVA verschiebt sich bei jeder Neukompilierung.

`opengl32.dll` wird statisch importiert, aber nur für `glGetString`,
`wglCreateContext`, `wglDeleteContext` und `wglMakeCurrent`, also zur
GPU-Identifikation, nicht zum Rendern.

## Im Image vorhandene Middleware

| Middleware | Nachweis |
|---|---|
| **Autodesk HumanIK** | Statisch gelinkt, Namen entfernt. Nirgends eine DLL im Installationsbaum. Siehe [03-skeleton.md](03-skeleton.md). |
| **Havok** | Physik und das gepoolte Placement-Subsystem. Benannt anhand eigener String-Literale aus Profiling-Timern wie `TtWorldCastRay`. |
| **NVIDIA Ansel** | Genau drei Symbole aus `anselsdk64.dll` importiert: `setConfiguration`, `updateCamera`, `isAnselAvailable`. |
| **Tobii EyeX** | 37 Symbole aus `tobii.eyex.client.dll`, darunter `tobii_head_pose_subscribe` und `tobii_gaze_point_subscribe`. |
| **NVIDIA TurfEffects, GFSDK SSAO, Volumetric Lighting** | Eigenständige DLLs im Installationsverzeichnis. |

Dass Havok sich in Profiling-Strings selbst benennt, ist als Technik
erinnerungswürdig: `HK_TIMER_BEGIN` schreibt ein Literal in den Profiling-Strom
*aus dem Inneren genau der Funktion heraus, die es benennt*. Eine
String-Querverweissuche bringt dich also direkt in den richtigen Funktionsrumpf.

## Koordinatenkonventionen

`[VERIFIED]` Dies sind die **von der Engine selbst angegebenen Basisvektoren**,
keine Schlussfolgerung. Das Ansel-SDK verlangt, dass ein Titel seine
Koordinatenkonvention deklariert. Wer also `ansel::setConfiguration` hookt und
die Struktur ausgibt, die das Spiel übergibt, bekommt die Antwort aus dem Mund
der Engine:

```
right   = (+1, 0, 0)   = +X
up      = ( 0, 0,+1)   = +Z
forward = ( 0,+1, 0)   = +Y
```

| Eigenschaft | Wert |
|---|---|
| Hochachse | **+Z** |
| Vorwärtsachse | **+Y** |
| Rechtsachse | **+X** |
| Händigkeit | **RECHTSHÄNDIG** |
| Weltmaßstab | `metersInWorldUnit = 1.0`, also **1 Welteinheit = 1 Meter** |

Herleitung der Händigkeit: `right x up = X x Z = -Y`, und `(right x up) . forward
= -1.0`. Für eine Basis `(right, up, forward)` bedeutet `right x up = +forward`
linkshändig und `-forward` rechtshändig.

Die deklarierte Struktur, teilweise identifiziert:

```
+0x000  right   (1,0,0)
+0x00C  up      (0,0,1)
+0x018  forward (0,1,0)
+0x024  1.0        metersInWorldUnit
+0x028  45.0       [INFERRED] eine Geschwindigkeits- oder Winkelgrenze
+0x02C  1   (int)
+0x030  8   (int)
+0x034  1.0
+0x038  01 01 01 01  [INFERRED] vier Bools
```

`[INFERRED]` Diese vier aufeinanderfolgenden `01`-Bytes sind sehr wahrscheinlich
`isCameraOffcenteredProjectionSupported`, `isCameraRotationSupported`,
`isCameraTranslationSupported` und `isCameraFovSupported`, alle wahr. Wenn diese
Lesart stimmt, hat das Spiel gegenüber Ansel erklärt, dass es außermittige
Projektion, Rotation, Translation und FOV-Override unterstützt.

**+Z oben mit +Y vorwärts entspricht späteren Anvil-Titeln**, also lässt sich
Code zur Basiskonvertierung, der für Odyssey oder Valhalla geschrieben wurde,
unverändert übernehmen. Die Händigkeit sollte man gegen die jeweilige Referenz,
von der man portiert, doppelt prüfen: Mindestens eine bekannte
Anvil-VR-Codebasis behauptet linkshändiges GLM, während ihre tatsächliche
Projektionsmathematik die rechtshändige Form verwendet. Die Mathematik stimmt;
die Behauptung betrifft die GLM-Build-Konfiguration, nicht die Engine.

## Der Engine-Task-Graph

`[VERIFIED]` In den String-Daten liegt ein zusammenhängender Block von 867
benannten Engine-Tasks. Das ist eine kostenlose Karte des Frames.

Frame-Tasks:

| RVA | Task |
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

Kamera-Tasks:

| RVA | Task |
|---|---|
| `0x0394A9D8` | `UpdateCamera` |
| `0x0394A9E8` | `Ai::UpdateCamera` |
| `0x0394AD80` | `UpdateActionAfterCameraTask` |
| `0x03964DF0` | `WorldView` |
| `0x03964E30` | `CameraTransform` |

Animations- und Skelett-Tasks, in Ausführungsreihenfolge:

| RVA | Task |
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

Die `AfterCamera`-Varianten sind die interessanten: Die Engine unterscheidet
ausdrücklich zwischen Animation, die vor dem Kamera-Update läuft, und Animation,
die danach läuft.

`[VERIFIED]` **Frame-Reihenfolge, die zählt:** Die Skelettarbeit läuft in den
Komponenten-Update-Phasen innerhalb von `Engine::EngineLoop::Step`, **nach** der
Grafikphase. Die Ausgabe von `SkeletonPostUpdate` in Frame N wird also vom
Renderer in Frame N+1 konsumiert. Es gibt kein Zeitfenster für einen
Post-Palette-Schreibzugriff im selben Frame.

## RTTI

`[VERIFIED]` 1.462 MSVC-RTTI-Typdeskriptoren sind wiederherstellbar und lassen
sich sauber demanglen; sie liegen in der irreführend benannten `.text1`. Sie
decken Engine-Infrastruktur ab (den `scimitar`-Namespace), nicht
Gameplay-Klassen.

Gameplay-Klassennamen stehen **überhaupt nicht** im Klartext im Image. Siehe
[08-reflection.md](08-reflection.md) für den CRC32-Weg, der die einzige
Möglichkeit ist, sie zu bekommen.
