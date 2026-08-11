[English](../10-negatives.md) · **Deutsch** · [한국어](../ko/10-negatives.md)

---

# 10 – Verifizierte Negativbefunde

Dinge, die definitiv **nicht** zutreffen, und Ansätze, die definitiv nicht
funktionieren. Bei einer geschlossenen Engine sind sie so viel wert wie die
positiven Befunde – und sie sind der Teil, den niemand veröffentlicht.

Jeder Eintrag nennt, was versucht wurde, was der Beleg war und was das bedeutet.

## Datenmodell und Strukturen

**`Skeleton+0x120`, `Skeleton+0x250` und `owner+0x050` speisen den Renderer
nicht.** `[VERIFIED]` Alle drei sind veraltete Spiegel derselben vorgelagerten
Entity-Position. `Skeleton+0x120` wird nur von einem Reset-/Teleport-Pfad
herangezogen, der nicht pro Frame läuft. Von keinem von ihnen führt ein Codepfad
zum Renderer, sie zu beschreiben ist also **konstruktionsbedingt** unsichtbar,
nicht zufällig. Eine Laufzeitbeobachtung, dass eines von ihnen bit-identisch mit
der Pose-Wurzel ist, ist gemeinsame Herkunft, keine kausale Verbindung.

**Pose A (`skel+0x230`) zu beschreiben ist sinnlos.** `[VERIFIED]` Sie wird im
selben Update über Pose B kopiert. Pose B bei `skel+0x238` ist die finale Pose.

**Ein Schreibzugriff auf die Pose-Wurzel kann bei einem angehängten Objekt nicht
überleben.** `[VERIFIED]` Der Attachment-Publish des Elternobjekts markiert die
Pose des Kindes über den Transform-Changed-Notifier als dirty und ruft sieben
Instruktionen später inline `PoseRefresh` darauf auf, was beide Wurzeln neu
herleitet.

**Der Pose-Stride ist nicht `0x80`, und der Puffer enthält nicht zwei
4x4-Matrizen pro Knoten.** `[VERIFIED, Korrektur]` Diese Lesart entsteht, wenn man
die „Einheit“ des Rig-Init-Codes als 4 Bytes behandelt. Die Einheit ist 1 Byte,
der Stride ist `0x20`, und der Datensatz ist `{float4 Translation, float4
Quaternion}`. Das `shl rax, 5` des Accessors ist eindeutig.

**Der Zeiger auf den Pose-Puffer pro Charakter liegt nicht in einem
Instanz-Pool-Slot.** `[VERIFIED]` Er ist `Pose+0x178`.

**`IKData` trägt in der Wildlands-Ära keine Nutzlast.** `[VERIFIED]` Der Body der
IK-Kettendefinitionen des Spieler-Rigs ist null. Die Effektortabelle existiert
nur zur Laufzeit, es gibt also keine eingebackene Effektorliste, die man im Asset
finden könnte.

## Kamera

**Ein Schreibzugriff auf `Camera+0x4A0` (den Slot der Pose-Matrix) bewirkt
nichts.** `[VERIFIED]` Etwa 400 Schreibvorgänge pro Sekunde mit 15 Grad
Gierwinkel erzeugten null Faults und null visuelle Veränderung, und Dumps zeigten
keine Spur der injizierten Rotation. Es ist eine abgeleitete Ausgabe, die die
Kamerafunktion vor der Verwendung neu schreibt. `Camera+0x000` ist die
maßgebliche Transformation.

**`worldMatrixOverride` liegt nicht bei `+576`.** `[VERIFIED]` Es liegt bei
`Camera+0x2A0` (672). Die Zahl `+576` stammt aus Pseudocode der Origins-Ära, und
der umgebende Block ist ansonsten Instruktion für Instruktion identisch – genau
das macht den falschen Offset überzeugend.

**Die Gameplay-Kamera verwendet nicht die Projektionsfunktion, die zur bekannten
Odyssey-Signatur passt.** `[VERIFIED, durch Messung]` Die Signaturübereinstimmung
bei `0x0C50C0E0` ist byte-perfekt, und es ist die falsche Funktion; der
Gameplay-Pfad ruft `0x0C50C420` auf. Diesen Punkt sollte man zweimal lesen: Eine
perfekte Signaturübereinstimmung auf dem richtigen Funktionsnamen kann trotzdem
der falsche Aufrufpfad sein.

**Die Engine stellt keine sechs Kameraobjekte bereit.** `[VERIFIED]` Sie stellt
**zwei** bereit, über jeden getesteten Zustand hinweg (zu Fuß, Fahrzeuge, Zielen,
Zielfernrohre, Menüs, Fotomodus). Unterscheidungsmerkmale aus späteren
Anvil-Titeln setzen eine Kamera pro Zustand voraus und hätten hier nie
funktionieren können. Verwende `mode == 0` bei `Camera+0x290`, nicht die
Zeigeridentität.

## Rendering

**Die Shader-Container-DLL enthält nicht die Material-Shader.** `[VERIFIED]` Sie
enthält nur Post-Process, Terrain und Wasser. Ein Scan darin nach geskinnten
Vertex-Shadern liefert 0 von 548, was sich wie „diese Engine hat kein
Vertex-Skinning“ liest und schlicht der falsche Container ist. Das
Material-DXBC liegt in den `.forge`-Daten.

**Die Knochenpalette liegt bei Charakter- und Waffen-Draws nicht bei `t3`.**
`[VERIFIED, Korrektur]` `t3` ist der Pfad des **Compute-Pre-Skinnings**. Der
Vertex-Shader-Skinning-Pfad bindet seine Palette bei **`t6`** (plus `t8` für den
vorherigen Frame). Gleiches 48-Byte-`float3x4`-Layout, anderer Slot. Eine Sonde,
die nur `t3` beobachtet, liefert bei jedem Charakter-Draw einen sauberen,
bedeutungslosen Negativbefund.

**Der Produzent der GPU-Knochenpalette liest die Pose nie.** `[VERIFIED]` Ein
Scan auf Byte-Ebene über jede Funktion der Kette nach den Displacements `+0x178`,
`+0x238`, `+0x8C` und nach `shl reg,5` liefert null bei allen vieren. Ihre Quelle
ist ein separates, elternrelatives Array mit Stride 48, gefüllt durch das
Abtasten von Animationsclips. Das CPU-Skelett und die GPU-Palette werden aus
verschiedenen Quellen gespeist.

**`sg_BulkSkinBuffer_vs` hat nichts mit Skinning zu tun.** `[VERIFIED]` Es ist
ein Passthrough aus zwei Instruktionen; „skin“ bedeutet dort
Subsurface-Haut**schattierung**.

**`0x0DBDEDD0` ist nicht der Palettenproduzent.** `[VERIFIED]` Es liest das
Layout des Pose-Datensatzes und schreibt 48-Byte-Datensätze mit Stride `0x30`,
was exakt richtig aussieht, aber sein Layout ist `{float3 T, float3 A, float3 B,
float3 AxB}` mit der Translation in den Floats 0..2 statt in den `.w`-Lanes.

**Die D3D-Schicht des Renderers enthält keine Strings.** `[VERIFIED]` Die im Image
vorhandenen D3D11-Fehlerstrings gehören zu NVIDIA TurfEffects, nicht zum
Hauptrenderer, und können ihn daher nicht verankern. Die CPU-Funktion, die den
Platzierungs-Konstantenpuffer füllt, wurde über keinen String-, Map-Idiom- oder
strukturellen Weg je gefunden.

**Vtable-Patching ist bei D3D11-Draws die falsche Ebene.** `[VERIFIED]` Jeder
Device Context hat seine eigene Heap-Vtable; eine zu patchen patcht einen
Kontext.

**„Hidden“-Flags pro Objekt verallgemeinern sich nicht von Köpfen auf Waffen.**
`[VERIFIED]` Ein für die Familie der Kopf-Attachments verifiziertes Flagbit
liefert bei einem imageweiten Scan nach seiner Testinstruktion genau einen
Treffer, innerhalb eines Heap-Allokators. Es wurde kein Zeigerpfad von einer
Waffenskelett-Instanz zu einem Render-Knoten gefunden.

## Reflection und Benennung

**Klassendeskriptoren und Eigenschaftstabellen haben null rip-relative
Codereferenzen.** `[VERIFIED, fünf geprüft, mit einer im selben Lauf
durchlaufenden Kontrolle]` Sie werden nur über Zeiger-Slots erreicht, die der
Registrar durchläuft. Daher **funktioniert „den Deskriptor querverweisen, um den
Code zu finden, der Feld X liest“ auf dieser Engine nicht.**

**Eigenschaftsdatensätze tragen auf diesem Build keinen Getter-Zeiger.**
`[VERIFIED]` Die Bytes `+0x14..+0x37` sind auf der Platte null.

**Methodennamen sind weitgehend nicht wiederherstellbar.** `[VERIFIED]` 11.283
von 103.014 Einträgen lösen sich auf, fast ausschließlich generische
Lifecycle-Methoden. Weder `AttachTo` noch `GetAttachmentBone` noch `FindBone`
existiert als Name in einem der beiden Wörterbücher.

**Im Image existieren überhaupt keine Gameplay-Klassennamen im Klartext.**
`[VERIFIED]` Der CRC32-Weg war keine Abkürzung, er war der einzige Weg.

**Debug-Formatstrings sind nicht immer verankerbar.** `[VERIFIED]` Ein Satz von
Formatstrings für Render-State-Dumps hat null rip-relative und null absolute
Referenzen, weil er über gepoolte Basis-plus-Offset-Adressierung erreicht wird.
Eine Kontrolle im selben Lauf lieferte ihre eine bekannte Referenz sehr wohl
zurück, der Negativbefund ist also eine Eigenschaft des Strings, nicht des
Scanners.

## HumanIK

**Nirgends im Image existiert ein Symbolname der öffentlichen HumanIK-API.**
`[VERIFIED]` `HIKSetEffectorStateTQSfv`, `HIKSolveForEffectorSet`,
`HIKSetNodeStateTQSfv`, `HIKCharacterCreate` und `HIKEffectorSetStateCreate`
liefern über die gesamte 405 MB große Stringmenge hinweg null Treffer. Die
Bibliothek ist statisch gelinkt, mit entfernten Namen. Die einzigen HIK-Treffer
sind fünf Daten-Tags plus Rauschen aus gepackten Blobs. Die
**Effektor-Namenstabellen** sind der einzige Anker.

## Waffen und Projektile

**Das sichtbare Projektil ist nicht das, was den Treffer auflöst.** `[VERIFIED]`
Drei unabhängige Eingriffe wurden jeweils auf einem aktiven Feld ausgeführt, mit
Zählern, die ihre Anwendung belegten – und die Einschläge bewegten sich nicht:
das Verschieben der eigenen Positionsfelder des Projektils, das Umschreiben der
Spawn-Richtung, die die Spawn-Funktion liest, und das Überschreiben des
Zielwert-Lesers pro Schuss. Die Entscheidung liegt vorgelagert zu allen dreien.

**Das gepoolte Placement-Subsystem gehört Havok**, es ist kein
renderer-seitiger Bone-Gather-Konsument. Dieser Untersuchungsstrang ist
geschlossen.

**Ein Hook auf dem Raycast-Wrapper ist für drei Aufrufstellen blind**, die den
Raycast-Rumpf über eine zur Laufzeit gebaute virtuelle Tabelle direkt erreichen.
Eine Erhebung des Schussfensters allein am Wrapper sieht nur
Hintergrundverkehr.

## Archive

**Eintragsnamen in einer `.forge` sind nicht eindeutig.** `[VERIFIED]` 5.602
Duplikate in einem Archiv, 335 in einem anderen. FileDataIDs sind eindeutig.
Adressiere über die file_id.

**Das TOC liegt nicht am Dateiende.** `[VERIFIED]` Header und beide Tabellen
liegen unterhalb der ersten Nutzlast, bei allen 21 Archiven der Installation.

**Das Namensfeld liegt nicht bei Datensatz-Offset 40.** `[VERIFIED]` Es liegt bei
44. Bei 40 zu lesen lässt die druckbaren Bytes des Timestamp-Feldes in jeden
Namen sickern – daher stammen bestimmte bekannte falsche Eintragsnamen.

**Eine gültig dekomprimierbare Nutzlast ist nicht zwangsläufig eine ladbare.**
`[VERIFIED, im Retail-Spiel]` Eine als ein Blocksatz mit Kompressionsbyte 0 neu
gepackte Nutzlast durchläuft einen unabhängigen Reader einwandfrei und **lässt
das Spiel in einem Schwarzbild hängen**. Die ursprüngliche Satzstruktur und das
Kompressionsbyte müssen erhalten bleiben.

**Die Blockgrößenfelder sind u16, nicht i32.** `[VERIFIED]` Der i32-Dialekt
gehört zu einem anderen Anvil-Titel.

**Die Chunk-Prüfsumme ist kein Standard-Adler-32.** `[VERIFIED, 789/789 Chunks]`
Es ist Adler-32 mit **beiden Akkumulatoren auf null initialisiert** statt mit
`a = 1`.

**Der ausgelieferte Kompressor ist nicht LZO1X-1.** `[VERIFIED]` Es ist
**LZO1X-999**, bewiesen durch die Byte-für-Byte-Reproduktion eines
ausgelieferten Archivs.

**Eine Patch-Forge ersetzt eine Basis-Forge nicht pauschal.** `[VERIFIED]` Die
Auflösung erfolgt forge-übergreifend über die **Ressourcen**-file_id, und ein
Patch ersetzt nur die IDs, die er tatsächlich enthält. Eine nur in der Basis
vorhandene Ressource wird aus der Basis gelesen, selbst wenn der Patch einen
Eintrag desselben Namens und derselben Eintrags-file_id führt.

**Ein Community-Toolkit hat überhaupt keine Wildlands-Unterstützung
einkompiliert.** `[VERIFIED]` Ein Byte-Scan seiner Hauptassembly (ASCII und
UTF-16) findet Strings zu Odyssey, Origins, Steep und Breakpoint und **null**
Wildlands-Treffer.

## Eingabe

**Das „seit dem letzten Aufruf gedrückt“-Bit von `GetAsyncKeyState` ist hier
unbrauchbar.** `[VERIFIED]` Das Spiel fragt Eingaben jeden Frame ab und
konsumiert es, eine Abfrage einmal pro Sekunde auf dieses Bit **löst also nie
aus**. Zwei Hotkeys verzeichneten über eine ganze Sitzung hinweg null
Tastendrücke. Lies das Taste-ist-gedrückt-Bit (`0x8000`) und mache deine eigene
Flankenerkennung.

## Frame-Reihenfolge

**Es gibt kein Zeitfenster im selben Frame zwischen Skelett-Update und
Rendering.** `[VERIFIED]` Die Skelettarbeit läuft in den
Komponenten-Update-Phasen nach der Grafikphase, die Ausgabe von
`SkeletonPostUpdate` in Frame N wird also vom Renderer in Frame N+1 konsumiert.

## Werkzeuge

**RenderDoc erfasst diesen Titel nicht.** Vier getrennte Wege wurden versucht,
keiner lieferte eine Aufnahme. Die ausgelieferten Shader aus den Archiven zu
lesen erwies sich ohnehin als strikt besser, da es exakte Registerslots und
Strides ganz ohne Laufzeit liefert.

**Bei ausgelieferten Shadern ist RDEF entfernt.** Du bekommst Slots und Strides,
keine Namen.
