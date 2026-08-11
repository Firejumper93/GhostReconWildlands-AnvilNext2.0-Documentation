[English](../11-other-anvil-titles.md) · **Deutsch** · [한국어](../ko/11-other-anvil-titles.md)

---

# 11 – Dies auf anderen Anvil-/AnvilNext-2.0-Titeln nutzen

> **Lies diese Datei skeptischer als die anderen.**
>
> Alles in den Dateien 01 bis 10 wurde speziell gegen *Ghost Recon Wildlands*
> verifiziert. **Diese Datei ist weitgehend Extrapolation.** Sie ist eine
> informierte Vermutung darüber, was sich innerhalb der Engine-Familie überträgt,
> geschrieben, um dir Zeit zu sparen – nicht, um dir das Nachprüfen zu ersparen.
> Aussagen hier sind mit `[CROSS-TITLE, UNVERIFIED]` gekennzeichnet, sofern sie
> nicht durch eine konkrete titelübergreifende Beobachtung gestützt sind.
>
> Behandle jede Adresse als falsch, bis du sie neu hergeleitet hast, und jede
> Struktur als Hypothese, bis dein eigener Parser sie fehlerfrei hin- und
> zurückverarbeitet.

## Wo Wildlands in der Familie steht

Wildlands erschien im **März 2017**, was es früh in der AnvilNext-2.0-Ära
verortet, vor der Reihe von Assassin's-Creed-Titeln, auf die sich die meisten
Anvil-Werkzeuge richten. Das macht es in zwei Richtungen als Bezugspunkt
nützlich: Es zeigt, wie die Engine aussah, bevor jene Titel sie veränderten, und
es ist ihnen nahe genug, dass überraschend viel erkennbar derselbe Code ist.

Titel, die üblicherweise als Anvil oder AnvilNext 2.0 identifiziert werden, sind
unter anderem Assassin's Creed Unity, Syndicate, Origins, Odyssey, Valhalla und
Mirage, For Honor, Steep, Ghost Recon Wildlands und Breakpoint, Immortals Fenyx
Rising, Riders Republic sowie Skull and Bones. **Diese Liste ist Allgemeinwissen,
kein Befund dieser Recherche**, und die Engine hat sich über diese Spanne
erheblich verändert. Nimm nicht an, ein Titel sei „dieselbe Engine“, nur weil er
auf einer Liste auftaucht.

### Konkrete titelübergreifende Beobachtungen aus dieser Arbeit

Dies sind die wenigen Stellen, an denen tatsächlich ein direkter Vergleich
gezogen wurde, und sie lohnen genaues Lesen, denn sie zeigen das Muster: **Der
Code ist oft buchstäblich derselbe, und die Offsets sind es nicht.**

- `[VERIFIED]` **Eine Signatur einer Projektionsfunktion aus Odyssey passte Byte
  für Byte auf Wildlands**, ein einziger Treffer im gesamten 369-MB-Image.
  Dieselbe Funktion, unverändert, über einen Titel und ein Jahr hinweg. Siehe aber
  die Falle weiter unten.
- `[VERIFIED]` **Pseudocode aus der Origins-Ära für den
  `worldMatrixOverride`-Block der Kamera reproduziert sich in Wildlands
  Instruktion für Instruktion, an einem anderen Struct-Offset**: hier `+0x2A0`
  gegenüber dort `+576`. Der umgebende Code ist identisch, und genau das macht den
  falschen Offset überzeugend.
- `[VERIFIED]` **`+Z` oben und `+Y` vorwärts stimmen exakt mit Odyssey und
  Valhalla überein**, Code zur Basiskonvertierung für jene Titel lässt sich also
  unverändert übernehmen.
- `[VERIFIED]` **Die „Attachment-Slot“-Nummern aus einem Leitfaden der
  Mirage-Ära übertragen sich nicht.** Wildlands hat sein eigenes
  `Skeleton::BipedBoneID`-Enum mit 143 Einträgen und eigenen Ordinalzahlen.
- `[VERIFIED]` **Ein weit verbreitetes Community-Toolkit hat überhaupt keine
  Wildlands-Unterstützung**: Ein Byte-Scan seiner Hauptassembly findet Strings zu
  Odyssey, Origins, Steep und Breakpoint und null Wildlands-Treffer. Die
  Werkzeugabdeckung ist titelspezifisch, und beworbene Unterstützung ist eher
  nachprüfenswert als vertrauenswürdig.
- `[VERIFIED]` **Die Blockgrößenfelder der Datendateien sind in Wildlands u16**,
  während mindestens ein anderer Titel derselben Familie i32 verwendet. Gleiches
  Format, andere Integer-Breite, stille Korruption, wenn du rätst.

### Die Falle, die mit einer perfekten Signaturübereinstimmung kommt

`[VERIFIED]` Die Odyssey-Projektionssignatur, die Byte für Byte auf Wildlands
passte, zeigt auf eine Funktion, die die Gameplay-Kamera **nicht aufruft**. Die
Engine hat sechs Projektionsvarianten; diejenige mit der berühmten Signatur ist
hier nicht die auf dem Gameplay-Pfad.

Eine perfekte titelübergreifende Signaturübereinstimmung sagt dir also, dass
**die Funktion vorhanden und unverändert ist**. Sie sagt dir nicht, dass **das
Spiel sie für das aufruft, was dich interessiert**. Das sind verschiedene
Aussagen, und sie zu vermengen kostet einen Tag.

## Was sich wahrscheinlich überträgt

`[CROSS-TITLE, UNVERIFIED]`, sofern nicht anders vermerkt. Geordnet danach, wie
zuversichtlich ich wäre.

### Sehr wahrscheinlich: Identität und Benennung

**CRC32 ist der universelle Identifikator.** Knochennamen, Klassennamen,
Eigenschaftsnamen und Ressourcen-Klassen-Hashes sind in Wildlands allesamt
schlichtes CRC32 des Namens. Das ist eine engine-weite Designentscheidung, keine
pro Titel, und derselbe Rainbow-Table-Ansatz sollte überall in der Familie
funktionieren.

Auch der Trick mit dem zweiquelligen Wörterbuch sollte sich übertragen: Ein
Community-Hash-Wörterbuch allein reichte hier **nicht**, und die Klartext-Strings
des Images selbst zu hashen, samt deren Token-Aufspaltungen, ist das, was die
interessanten Namen geknackt hat. Die ausführbare Datei jedes Titels enthält die
Wörter, auch wenn sie die Bezeichner nicht enthält.

### Sehr wahrscheinlich: das Reflection-System

Ein Laufzeit-Reflection-System, das Klassendeskriptoren und
Eigenschaftsdatensätze als statische Daten führt und von einem einzigen Registrar
durchlaufen wird, ist Kern-Engine-Architektur. Wenn du den Registrar in einem
anderen Titel findest, bekommst du annotierte Struct-Layouts für das ganze Spiel.

Womit du an Unterschieden rechnen solltest: die Sektion, in der es liegt, der
Datensatz-Stride und die Bitpackung des Offsetfeldes (Wildlands: Stride `0x38`,
Byte-Offset = `packed >> 18`). Finde eine Klasse, deren Layout du bereits kennst,
und kalibriere damit die Packung, bevor du irgendetwas anderem traust.

### Sehr wahrscheinlich: HumanIK und Havok als Middleware

Ein statisch gelinktes HumanIK mit entfernten Namen, das nur als Daten-Tags und
Namenstabellen vorhanden ist, ist ein starkes Muster. Ebenso, dass Havok seine
eigenen Funktionen über String-Literale von Profiling-Timern benennt. Beides gibt
dir Anker in einem stringlosen Binary.

### Wahrscheinlich: der Container- und Kompressions-Stack

Der `.forge`-Container, das Blocksatz-Layout der Datendateien und der
Multi-Resource-Container innerhalb einer Nutzlast sind das Speicherdesign der
Familie.

Damit rechne pro Titel als variabel:

- **Der `FileVersionIdentifier` der Forge** (27 in Wildlands).
- **Der Kompressionsalgorithmus.** Wildlands ist LZO1X; spätere Titel gingen zu
  Oodle über, und das Kompressionsbyte zählt mehrere LZO-Varianten auf. Nicht
  hartcodieren.
- **Integer-Breiten im Blockheader** (die u16-gegen-i32-Aufteilung oben).
- **Die Prüfsumme.** Wildlands verwendet Adler-32 mit beiden Akkumulatoren auf
  **null** initialisiert statt `a = 1`. Ob diese Eigenheit engine-weit oder
  Wildlands-spezifisch ist, ist `[UNKNOWN]`, und es ist billig zu testen: Berechne
  beides über einen bekannten Chunk und sieh, welches passt.

### Wahrscheinlich: die Skelett- und Pose-Architektur

Ein gemeinsam genutzter, refcount-verwalteter Rig-**Deskriptor**, der keine
Transformationen hält, ein **Pose**-Objekt pro Charakter, dem der Knochenpuffer
gehört, eine sortierte Hash-zu-Index-Namenskarte und Knochendatensätze im
Model-Space, die gegen eine Wurzeltransformation komponiert werden. Das ist ein
stimmiges Design und wurde wahrscheinlich nicht pauschal neu geschrieben.

Rechne damit, dass **die Offsets und der Stride abweichen**. Wildlands ist
`{float4 T, float4 Q}` bei Stride `0x20`. Bestätige deinen aus der Arithmetik des
Accessors selbst (`shl reg, N` liefert dir den Stride direkt), nicht aus einer
Struct-Definition, die jemand gepostet hat.

### Plausibel, aber vorher prüfen: die Jump-Thunk-Aufrufarchitektur

Jeder Engine-Aufruf in Wildlands läuft über ein 5-Byte-`E9 rel32`, das allein in
einem 16 Byte großen, mit `int3` aufgefüllten Slot liegt. Das ist ebenso sehr
eine Build-/Link-Eigenschaft wie eine Engine-Eigenschaft, es kann für einen
anderen Titel also gelten oder auch nicht, selbst bei derselben Engine.

Es ist trivial zu testen: Disassembliere ein beliebiges Aufrufziel und sieh, ob
du auf einem jmp landest. Wenn ja, hast du dieselbe bequeme Abfangfläche; wenn
nicht, brauchst du konventionelle Techniken.

### Plausibel: Koordinatenkonventionen

`+Z` oben, `+Y` vorwärts, rechtshändig, 1 Einheit = 1 Meter. Als übereinstimmend
mit Odyssey und Valhalla verifiziert.

Wenn der Titel NVIDIA Ansel einbindet, kannst du das aus dem Mund der Engine
selbst bekommen, statt es zu erschließen: Das Ansel-SDK verlangt, dass ein Titel
seine Koordinatenkonvention deklariert; den Konfigurationsaufruf abzufangen und
die Struktur auszugeben liefert dir also die deklarierten Basisvektoren und den
Weltmaßstab direkt. Diese Technik ist titelunabhängig und dauert einen
Nachmittag.

## Was sich definitiv nicht überträgt

- **Jede RVA.** Zwei Retail-Builds *desselben Spiels* haben jede Funktion an
  einer anderen Adresse. Über Titel hinweg ist das nicht einmal in der Nähe.
- **Struct-Offsets, allgemein.** Beachte aber die innerhalb von Wildlands über
  ein Update von neun Jahren beobachtete Asymmetrie: **Das Datenlayout überlebte,
  die Codeadressen nicht.** Vier Eigenschafts-Offsets der Waffenkomponente wurden
  identisch neu hergeleitet, während sich jede Funktion verschob. Offsets sind
  also portabler als Adressen – und trotzdem nicht portabel genug, um ihnen zu
  trauen.
- **Enum-Ordinalzahlen.** Das Biped-Knochen-Enum ist titelspezifisch.
- **Konventionen zur Asset-Benennung.** Die Präfixe `GR_`, das
  Waffenknochenpräfix `wb-` und die Animationspräfixe `sb_`/`civ_`/`rbl_` sind
  Ghost-Recon-spezifisch.
- **Alles zum Schutz eines Titels.** Hier außerhalb des Umfangs, variiert pro
  Titel und pro Release und wird in diesem Repository nicht behandelt.

## Eine Portierungs-Checkliste

Wenn du diese Befunde auf einen anderen Titel bringen willst, ist dies die
Reihenfolge, die im Rückblick die meiste Zeit gespart hätte.

1. **Identifiziere das Binary.** Lies TimeDateStamp und SizeOfImage aus dem
   PE-Header und mache sie von Anfang an zu deiner Build-Fixierung. Eine
   Fixierung nach einem Spiel-Update nachzurüsten ist schmerzhaft; eine zu haben
   bedeutet, dass ein Update eine saubere Verweigerung statt eines Absturzes
   erzeugt.
2. **Prüfe, ob die Sektionsnamen lügen.** Gib die Sektionstabelle aus und
   lokalisiere, wo der echte Code und die echten Strings tatsächlich liegen. Bei
   Wildlands ist der Code in `.sbss` und die Strings sind in `.link`. Machst du
   das falsch, ist jeder Scan danach auf die falschen Bytes begrenzt.
3. **Prüfe, ob Code im Klartext auf der Platte liegt.** Wenn ja, kann deine
   gesamte statische Arbeit offline gegen eine Kopie stattfinden, ganz ohne
   Prozess.
4. **Teste die Thunk-Hypothese.** Disassembliere ein paar Aufrufziele und sieh,
   ob es jmp-Stubs in gepolsterten Slots sind.
5. **Baue die CRC32-Rainbow-Table früh**, aus einem Community-Wörterbuch plus den
   eigenen Strings des Images. Alles andere wird leichter, sobald sich Namen
   auflösen.
6. **Finde den Reflection-Registrar** und gib die Klassen- und
   Eigenschaftstabellen aus. Das ist der mit Abstand größte Kraftverstärker.
7. **Parse die Archive** und hole die Material-Shader heraus. Ausgelieferte
   Shader sind eine zu wenig genutzte Quelle: Sie sagen dir genau, welche
   Registerslots und Strides die GPU liest, ganz ohne Laufzeit und ohne
   Capture-Werkzeug.
8. **Erst dann** beginne mit Laufzeitarbeit, und nur, um konkrete Ja/Nein-Fragen
   zu beantworten, die die statische Analyse wirklich nicht beantworten kann.

## Wofür diese Recherche ein Sprungbrett sein könnte

Als Richtungen angeboten, nicht als Behauptung, irgendetwas davon sei leicht.

**Asset- und Archiv-Werkzeuge.** Die Layouts von Container, Blocksatz,
Kompression, Prüfsumme und Multi-Resource in
[02-formats.md](02-formats.md) sind eine vollständige Spezifikation zum Lesen
und – wichtiger noch – zum **Schreiben** einer `.forge`. Die dort ermittelten
Writer-Einschränkungen (Satzstruktur und Kompressionsbyte erhalten, nur Einträge
ersetzen, auf `0x8000` auffüllen) sind der nicht offensichtliche Teil, und sie
wurden gelernt, indem ein Spiel in ein Schwarzbild gefahren wurde.

**Skelett- und Animationswerkzeuge.** [03-skeleton.md](03-skeleton.md) enthält
ein byte-vollständiges `.Skeleton`-Format ohne einen einzigen unerklärten Byte,
und die Knochennamen lösen sich in die Standard-HumanIK-Konvention auf. Das
genügt für einen Rig-Viewer, einen Bind-Pose-Editor oder einen Konverter in ein
DCC-Format.

**Kamera-Mods** (FOV, Third-to-First-Person, freie Kamera, Cinematic-Werkzeuge).
Die maßgebliche Kameratransformation, was passiert, wenn man sie schreibt, und
das eine sichtbare Artefakt, das sie erzeugt, stehen alle in
[05-camera.md](05-camera.md). Der Befund, dass die Engine die Transformation
jeden Frame neu aufbaut, ist das, was einen absoluten Schreibzugriff pro Frame
sicher macht.

**VR-Konversionen.** Die Kameraarbeit ist das Fundament; die
Koordinatenkonventionen und der Maßstab „1 Einheit = 1 Meter“ bedeuten, dass
keine Kalibrierung nötig ist. Das bekannte Artefakt (Culling, das vorgelagert zum
Kamera-Schreibzugriff entschieden wird) ist dokumentiert, damit es nicht neu
entdeckt werden muss.

**Waffen- und Attachment-Mods.** [06-weapons.md](06-weapons.md) enthält die
vollständige Attachment-Kette pro Frame und die Identität des letzten Schreibers
der Platzierung eines angehängten Objekts. Der Befund, dass eine Waffe aus vielen
starren Teileobjekten und nicht aus einem geskinnten Mesh besteht, rahmt eine
ganze Problemklasse neu.

**Render-Forschung.** [07-rendering.md](07-rendering.md) enthält die drei
Platzierungspfade mit exakten Registerslots und Strides, und genau das brauchst
du, bevor du irgendeinen Draw hookst.

**Alles davon auf einem anderen Titel**, mit diesem hier als Karte dessen, wonach
zu suchen ist, und – noch nützlicher – mit
[10-negatives.md](10-negatives.md) als Karte dessen, wofür man keine Woche
verschwenden sollte.

## Eine abschließende Warnung

Die wertvollste einzelne Gewohnheit aus dieser Arbeit: **Zeige, dass dein
Instrument bei etwas, von dem du bereits weißt, dass es da ist, im selben Lauf
einen Positivbefund liefert, bevor du einem Negativbefund glaubst.** Nahezu jeder
teure Fehler, der in diesen Dateien dokumentiert ist, war eine Sonde, die an der
falschen Stelle suchte und ein ehrliches, bedeutungsloses Nichts meldete.

Das gilt doppelt beim Portieren auf einen Titel, den du nicht untersucht hast,
denn dort hast du kein Gefühl dafür, welches Nichts echt ist.
