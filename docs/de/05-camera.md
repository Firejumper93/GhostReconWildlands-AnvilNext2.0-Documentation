[English](../05-camera.md) · **Deutsch** · [한국어](../ko/05-camera.md)

---

# 05 – Die Kamera

Alle RVAs in dieser Datei beziehen sich auf den Retail-Build der 2017er-Ära,
sofern nicht anders angegeben.

## Die eine Funktion, auf die es ankommt

**`UpdateCameraMatricesAndFrustum`**, Implementierungs-RVA `0x0C5E47E0`, Thunk
`0x0135F720`. Die Anvil-VR-Community nennt sie üblicherweise `on_calc_mvp`. Sie
nimmt **21 Argumente**, `rcx` = die Kamera, und läuft für die Spielerkamera etwa
**2 Aufrufe pro Frame** (gemessen: 146.628 Aufrufe über 73.311 Frames).

`[VERIFIED]` Die Argumentzuordnung, zur Laufzeit über 55 vollständige
Argument-Snapshots in Menü, zu Fuß, langsamer Rotation und frei fliegendem
Fotomodus bestätigt, mit **null Abweichungen**:

| Arg | Wert |
|---|---|
| 1 (`rcx`) | `cam` |
| 2 (`rdx`) | `cam + 0x290` |
| 3 (`r8`) | `cam + 0x314` |
| 4 (`r9`) | `cam + 0x334` |
| 5 bis 11 | `cam + 0x324`, `cam`, `+0x360`, `+0x420`, `+0x4A0`, `+0x4E0`, `+0x520` |
| 12, 14, 15 | Floats (gemessen `0.8`/`0.846`, `0.1`, `0.2`) |
| 13 | `cam + 0x79C` |
| 16 | **ein Zeiger** außerhalb der Kamerastruktur, Ziel nicht identifiziert |
| 17 bis 21 | `cam + 0x5A0`, `+0x560`, `+0x460`, `+0x5E0`, `+0x620` |

Beachte, dass Argument 6 die Kamerabasis selbst ist. Das war der bezweifelte Teil
der statischen Lesart, und er hielt in 55 von 55 Stichproben stand.

## Die Kamerastruktur

| Offset | Inhalt |
|---|---|
| **`+0x000`** | **die Welttransformation der Kamera, 64 Bytes. DIE maßgebliche** |
| `+0x290` | int, Kameramodus-/Typ-Enum |
| `+0x2A0` | `Matrix4x4*` `worldMatrixOverride` |
| `+0x2B0` | float |
| `+0x2BC` | float, Basis-Sichtfeld |
| `+0x2C4`, `+0x2C8` | float, Projektionsversatz x und y |
| `+0x420 .. +0x620` | neun 4x4-Matrizen, zusammenhängend im Abstand `0x40` |
| `+0x748` | wird von einem Modus-Wechsel-Synchronisierer am Funktionseintritt erreicht |

### `worldMatrixOverride` liegt bei `+0x2A0`

`[VERIFIED]` Wenn du von Pseudocode aus der Origins-Ära portierst: Jener Code
liest `*(Matrix4x4 **)(pCamera + 576)`. Wildlands reproduziert denselben Block
**Instruktion für Instruktion** an einem anderen Offset:

```
0x0C5E4844  mov rax, [rsi + 0x2a0]     ; rsi = pCamera. Der Override-Zeiger.
0x0C5E484B  mov rdx, [rbp + 0x4f]      ; = Argument 6 = die Kamerabasis
0x0C5E484F  test rax, rax
0x0C5E4852  je  0x0C5E4872             ; wenn null, überspringen
            movaps x4                  ; 64 Bytes kopieren
0x0C5E4872  lea rcx, [rbp - 0x39]
0x0C5E4876  call ...                   ; -> 0x0CC28A10, der View-Matrix-Builder
```

Der Override liegt also bei **`Camera + 0x2A0` (672), nicht bei `+576`**, und
entscheidend ist, dass der Block die Override-Matrix **nach `Camera+0x000`**
kopiert. Dasselbe Muster taucht unabhängig davon bei `0x0C508FA7` an derselben
Struktur auf.

### Die neun Matrizen bei `+0x420 .. +0x620`

`[VERIFIED]`, anhand ihrer Laufzeitinhalte identifiziert, nicht geraten. Matrizen
sind zeilenweise (row-major) mit der Translation in der vierten Zeile, die
**Engine komponiert also mit Zeilenvektoren** (`v * M`).

| Offset | Arg | Identität | Nachweis |
|---|---|---|---|
| `+0x420` | 8 | **Projektion** | perspektivische Form, `[3][2] = 0.1` = near, `[2][3] = -1` |
| `+0x460` | 19 | Projektionsvariante | gleiche x/y-Skalen, `[2][2] = -1`, `[3][2] = -0.1`. `[INFERRED]` die ferne Hälfte einer Near/Far-Tiefenpartition |
| `+0x4A0` | 9 | Kamerapose-Matrix (World) | orthonormale Zeilen, Translationszeile = Weltposition der Kamera |
| `+0x4E0` | 10 | inverse Projektion | numerisch die exakte Inverse von `+0x420` |
| `+0x520` | 11 | inverse View-Projection | entspricht zeilenweise `invProj * pose` |
| `+0x560` | 18 | inverse View-Projection, zweite Variante | Zeilen 0-1 identisch mit `+0x520`, Zeilen 2-3 vorzeichenverkehrt |
| `+0x5A0` | 17 | **View-Projection** | Spalte 3 = negierte Kamera-Vorwärtszeile |
| `+0x5E0` | 20 | Kopie von `+0x420` | in jedem Dump identisch |
| `+0x620` | 21 | Kopie von `+0x4E0` | in jedem Dump identisch |

`[INFERRED]` Die zwei Aufrufe pro Frame entsprechen je einer Hälfte der
Tiefenpartition, passend zum Projektionspaar `+0x420`/`+0x460`.

## Wohin man schreibt und wohin nicht

Das wurde experimentell geklärt, und der Negativbefund ist so nützlich wie der
positive.

`[VERIFIED NEGATIVE]` **Ein Schreibzugriff auf `Camera+0x4A0` (den Pose-Slot)
bewirkt nichts.** Eine Komposition von 15 Grad Gierwinkel am Funktionseintritt,
etwa 400 Schreibvorgänge pro Sekunde, erzeugte null Faults und **null visuelle
Veränderung**, und regelmäßige Dumps zeigten das Paar `+0x4A0`/`+0x5A0` sauber und
in sich stimmig, ohne jede Spur des injizierten Gierwinkels. `+0x4A0` ist eine
**abgeleitete Ausgabe**, die die Funktion vor der Verwendung neu schreibt.

`[VERIFIED]` **Ein Schreibzugriff auf `Camera+0x000` funktioniert, und das ist der
Injektionspunkt.** Derselbe 15-Grad-Gierwinkel, am Funktionseintritt auf die
Wurzeltransformation komponiert, erzeugte:

- eine gerenderte Ansicht mit konstantem Gierversatz gegenüber der Richtung, in
  die das Spiel zielt, im Spiel äußerst sichtbar
- **Hüftschüsse landen abseits des Fadenkreuzes**: Das Zielen des Spiels bleibt
  maßgeblich, während sich die gerenderte Ansicht bewegt – genau die beabsichtigte
  Trennung für einen Kamera-Mod
- kein aufsummierendes Drehen, was beweist, dass die Engine **`Camera+0x000` in
  jedem Frame aus ihrem eigenen Kamerazustand neu aufbaut**, sodass eine
  Komposition pro Aufruf einen konstanten Versatz statt einer Ratsche ergibt
- jede abgeleitete Matrix (View-Projection, beide Inversen, die Pose bei `+0x4A0`)
  folgte konsistent der injizierten Pose, mit einem gemessenen Gierdelta von 0,00
  Grad zwischen Wurzel und Abgeleitetem

`[VERIFIED]` **Der Pose-Slot akzeptiert vollständig allgemeine
6DOF-Orientierungen.** Während des Rollens im Fotomodus enthielten die Zeilen
beliebige orthonormale Basen, und der Renderer folgte. Keine
Gimbal-Einschränkung, keine Reorthogonalisierung gegen die Schwerkraft.

### Das eine sichtbare Artefakt

`[VERIFIED, beobachtet]` **Fehlpassungen beim Vegetations-Culling am Bildrand**
auf der Seite, zu der hin der Versatz dreht. Die Sichtbarkeit von Vegetation
entscheidet etwas, das die Kamera *vor* dem Zeitpunkt konsumiert, an dem
`on_calc_mvp` die Rotation anwendet.

Bei kleinen Deltas pro Frame ist das unsichtbar. Falls es je relevant wird, ist
die Lösung, den Pose-Schreibzugriff früher im Frame anzuwenden, an der Nahtstelle
zwischen den Tasks `BeginFrame` und `UpdateCamera`.

### Idempotenz pro Frame

Wenn du diesen Slot beschreibst, schreibe einen **absoluten** Wert, der aus einer
einmal pro Frame erfassten Basis abgeleitet ist, keine inkrementelle Komposition.
Es gibt zwei Aufrufe pro Frame, und es ist nicht geklärt, ob die Engine
`Camera+0x000` vor jedem von ihnen oder einmal pro Frame auffrischt; ein absoluter
Schreibzugriff macht die Frage gegenstandslos.

## Welche Kamera die des Spielers ist

`[VERIFIED]` **Die Kamera des Spielers ist die mit `mode == 0`** bei
`Camera+0x290`.

`[VERIFIED]` **Die Engine stellt nur ZWEI Kameraobjekte bereit, nicht sechs.**
Das wurde in jedem Zustand getestet, der plausibel die Kamera hätte tauschen
können: zu Fuß, Fahrzeuge, Zielen, Zielfernrohre, Menüs, Fotomodus. Die aus
späteren Anvil-Titeln übernommenen Unterscheidungsmerkmale hätten hier nie
funktionieren können, denn sie setzen eine Kamera pro Zustand voraus.

**Verwende den Modus, nicht den Zeiger.** Zeigeridentität ist nicht stabil und
nicht das Unterscheidungsmerkmal.

Der Fotomodus ist als Forschungsinstrument erwähnenswert: Er lässt die
Modus-0-Kamera frei fliegen, mit Rollen, bis zu ~170 m von der Figur entfernt,
und alles Abgeleitete folgt. Das ist eine kostenlose Möglichkeit zu beweisen, dass
ein Kamera-Schreibzugriff vollständig allgemein ist, bevor du irgendetwas darauf
aufbaust. (In diesem Titel ist die freie Kamera der spieleigene Fotomodus, auf F10
gelegt; es ist nicht NVIDIA Ansel, obwohl Ansel eingebunden ist.)

## Die Projektionsfamilie

`[VERIFIED]` Es gibt nicht eine Projektionsfunktion, sondern **sechs**,
verteilt über einen Selektor bei `0x0C510B20`, der das Sichtfeld aus dem
Seitenverhältnis berechnet:

| Thunk | Funktions-RVA | Form |
|---|---|---|
| `0x01347280` | `0x0C50C0E0` | `sub rsp,0x90`, nimmt einen `pInputVector` |
| `0x01347460` | `0x0C50C2E0` | klein, `sub rsp,0x40`, kein Eingabevektor |
| `0x01347530` | **`0x0C50C420`** | groß, `sub rsp,0x100`, zusätzlicher Zeiger in `rdx` |
| `0x01347840` | `0x0C50C7E0` | |
| `0x01345720` | `0x0C5094D0` | |
| `0x01345800` | `0x0C509720` | |

`[VERIFIED, durch Messung]` **Die Gameplay-Kamera verwendet `0x0C50C420`, nicht
`0x0C50C0E0`.** `0x0C50C0E0` ist diejenige, die Byte für Byte zur bekannten
Odyssey-Signatur passt, was sie zum naheliegenden – und falschen – Kandidaten
macht. Das ist ausdrücklich festzuhalten, weil die Signaturübereinstimmung wirklich
perfekt ist: Es ist dieselbe Funktion, sie ist nur nicht die, die der
Gameplay-Pfad aufruft.

Ihr fünftes Argument ist das **vertikale Sichtfeld im Bogenmaß**, übergeben auf
dem Stack des Aufrufers. Gemessene Bereiche: 0,78 bis 0,87 im normalen Spiel, etwa
1,22 beim Sprinten oder in Fahrzeugen, 0,49 bis 0,52 beim einfachen Zielen über
Kimme und Korn, etwa 0,17 durch einen 3-fach-Vergrößerer, 0,33+ in Menüs und
exakt pi/2 (1,5708) für Cubemap-Flächenaufnahmen (Himmel- und Reflexionssonden).
Falls du diesen Wert je überschreibst, begrenze ihn: Weitet man die
Cubemap-Aufnahmen, verschiebt sich der Himmel gegenüber der Welt.

`0x0C50C0E0` bleibt der nützliche **Anker**, um zu verifizieren, dass du das
Binary vor dir hast, das du erwartest, denn seine Signatur ist im gesamten Image
eindeutig:

```
48 89 E0 53 48 81 EC 90 00 00 00 0F 29 70 E8 48 89 CB F3
```

## Unterstützende Mathematikfunktionen, benannt nach ihrer Mathematik

| RVA | Was es ist | Wie es identifiziert wurde |
|---|---|---|
| `0x0C4DADE0` | `UpdateFrustumPlanesFromVPMatrix` | Gribb-Hartmann-Ebenenextraktion aus einer View-Projection-Matrix, normalisiert und negiert. 11 Aufrufstellen, Thunk `0x013329A0` |
| `0x0CC28A10` | View-Matrix-Builder | kopiert vier Zeilen, multipliziert eine Zeile mit einer Vorzeichenkonstante, ruft die Matrixinverse auf. Kamera-Weltmatrix rein, View-Matrix raus |
| `0x05FE48C0` | 4x4-Matrixmultiplikation | Broadcast-Shuffle, `mulps`, akkumulieren |
| `0x06B1DBB0` | 4x4-Matrixinverse | Transposition per `shufps`, Kofaktorentwicklung |
| `0x0C510B20` | Projektionsmodus-Selektor | berechnet fov aus dem Seitenverhältnis, verteilt auf die sechs Varianten |

Eine Funktion danach zu benennen, was ihre Arithmetik nachweislich berechnet, ist
hier weit robuster, als sie nach einem String oder einer Vermutung zu benennen,
denn es gibt keine Gameplay-Strings, an denen man sich orientieren könnte. Der
Frustum-Builder ist ein gutes Beispiel: Nichts benennt ihn, aber nichts anderes
berechnet `m[row][0] + m[row][3]` für vier Zeilen, normalisiert nach der
xyz-Länge und negiert.

## Eine Methodennotiz, die eine Sitzung gekostet hat

**Verwende keinen kontinuierlich variierenden Float als Schlüssel einer
Diagnosetabelle.** Eine frühe Sonde zeichnete eine Zeile pro eindeutigem
Argumentwert auf, und ein Float, der sich jeden Frame ändert, verbrauchte die
gesamte Tabelle innerhalb von Sekunden. Sammle stattdessen Bereiche (Min/Max)
statt exakter Werte und gib immer aus, ob die Tabelle voll gelaufen ist. Eine
kapazitätsbegrenzte Sonde, die ihre eigene Grenze nicht meldet, produziert
selbstbewussten Unsinn: Es sieht aus wie „das sind die nächstgelegenen Objekte“,
bedeutet aber „das sind die ersten Objekte, die zufällig die Tabelle gefüllt
haben“.
