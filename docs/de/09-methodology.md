[English](../09-methodology.md) · **Deutsch** · [한국어](../ko/09-methodology.md)

---

# 09 – Methodik: was tatsächlich funktioniert hat

Techniken, ungefähr in der Reihenfolge, in der sie bei dieser Engine einen
Versuch wert sind. Die meisten lassen sich auf andere geschlossene Engines
übertragen; einige wenige sind spezifisch für Anvil.

## Reihenfolge des Vorgehens

1. **Community-Formatspezifikationen vor der Disassemblierung.** Bei Dateiformaten
   bringt dich ein existierender quelloffener Reader an einem Nachmittag durch den
   Großteil eines Container-Layouts, und der Rest ist die eigentliche Arbeit. Lies
   ihn auf Fakten hin und verifiziere dann jedes Feld gegen deine eigenen Dateien,
   denn die Dialekte unterscheiden sich zwischen Titeln genau in den Feldern, die
   niemand dokumentiert.
2. **Statische Analyse an der Datei auf der Platte.** Code liegt hier im Klartext
   auf der Platte, die gesamte ausführbare Datei lässt sich also offline
   analysieren, ohne Prozess und ohne Debugger. Fast alles in diesem Repository
   wurde so gewonnen.
3. **CRC32-Namenswiederherstellung** (siehe [08-reflection.md](08-reflection.md)).
   Bei dieser Engine ist das nicht eine Technik unter mehreren, es ist *die*
   Technik.
4. **Laufzeitmessung zuletzt**, und nur, um konkrete Ja/Nein-Fragen zu
   beantworten.

## Benenne Funktionen danach, was ihre Arithmetik berechnet

Ohne Gameplay-Strings und ohne Methodennamen ist die zuverlässigste
Identifikation die mathematische. Eine Funktion, die für vier Zeilen
`m[row][0] + m[row][3]` berechnet, nach der Länge des xyz-Anteils normalisiert
und negiert, **ist** die Gribb-Hartmann-Extraktion von Frustumebenen. Nichts
anderes sieht so aus.

Ebenso ist eine Funktion, deren gesamter Rumpf `movzx / shl rax,5 /
add rax,[rcx+X] / ret` lautet, ein Array-Accessor mit Stride `0x20`, und sie
belegt diesen Stride mit mehr Autorität als jede Menge umgebender Kontext.

Middleware hilft, weil sie sich selbst benennt. Havoks `HK_TIMER_BEGIN` schreibt
ein Literal wie `TtWorldCastRay` **aus dem Inneren der Funktion heraus, die es
benennt**, in den Profiling-Strom, sodass ein String-Querverweis direkt im
richtigen Rumpf landet. Die Daten-Tags und Effektor-Namenstabellen von HumanIK
leisten auf der Animationsseite dasselbe.

## Verifiziere ein Ziel auf drei Wegen, bevor du irgendetwas schreibst

Das Muster, das sich bewährt hat:

1. **Eine eindeutige Byte-Signatur**, die im gesamten Image **genau einmal**
   passen muss. Eine Signatur, die zweimal passt, ist keine Signatur.
2. **Ein Abgleich mit der erwarteten RVA** gegen eine Adresstabelle pro Build.
3. **Eine Byte-Verifikation im Moment der Verwendung**: Bei einem Jump-Thunk muss
   es `E9 rel32` gefolgt von `int3`-Padding sein, und sein Sprung muss zu der
   Funktion auflösen, die du erwartest.

Fällt auch nur eines davon aus, heißt das: nichts tun und laut protokollieren.
Eine falsche Tabelle kann sich dann weigern zu handeln, aber sie kann nicht auf
den falschen Bytes handeln.

Falle niemals auf eine hartcodierte Adresse aus einem anderen Build zurück.
Manche Frameworks liefern bei einem Signatur-Fehltreffer einen veralteten
Fallback, und der Aufrufer verwendet ihn ungeprüft – was aus einem Spiel-Update
einen Absturz statt einer sauberen Verweigerung macht.

## Verwende `.pdata`, nicht rückwärtige Disassemblierung

Um den Anfang einer Funktion von einer Adresse in ihrem Inneren aus zu finden,
schlage sie in der Exception-Unwind-Tabelle nach, statt rückwärts zu
disassemblieren. Rückwärtige Disassemblierung ist auf x86-64 Rätselraten;
`.pdata` ist eine sortierte Tabelle exakter Funktionsgrenzen und
konstruktionsbedingt korrekt.

## Beweise das Instrument, bevor du einem Negativbefund glaubst

Die teuersten Fehler in diesem Projekt hatten alle dieselbe Form: Eine Sonde
meldete nichts, dem Nichts wurde geglaubt, und die wirkliche Antwort war, dass
die Sonde nicht dort hinsah, wo sie zu sehen glaubte.

Konkrete Fälle, von denen jeder einen vollen Testzyklus kostete:

- Eine Paletten-Sonde, die den Shader-Resource-Slot `t3` beobachtete, während der
  verwendete Pfad `t6` bindet. Sauberer Negativbefund, zweimal, beide
  bedeutungslos.
- Ein Scan nach geskinnten Shadern, ausgeführt gegen den falschen
  Shader-Container. Null Treffer von 548, was sich wie „diese Engine hat kein
  Vertex-Skinning“ liest.
- Ein Raycast-Beobachter auf einer Wrapper-Funktion, die drei Aufrufstellen über
  eine zur Laufzeit gebaute virtuelle Tabelle umgehen.
- Ein Hotkey, abgefragt über das „seit dem letzten Aufruf gedrückt“-Bit von
  `GetAsyncKeyState`. **Das Spiel fragt Eingaben jeden Frame ab und konsumiert
  dieses Bit**, eine Abfrage einmal pro Sekunde löst also nie aus. Zwei Tasten
  verzeichneten über eine ganze Sitzung hinweg null Tastendrücke. Lies das
  Taste-ist-gedrückt-Bit (`0x8000`) und mache deine eigene Flankenerkennung.

Bevor du einem Negativbefund glaubst, zeige, dass das Instrument bei etwas, von
dem du bereits weißt, dass es da ist, einen Positivbefund liefert – **im selben
Lauf**, nicht in einem anderen.

## Jedes Gate bekommt seinen eigenen Zähler

Ein Zähler, der hinter mehreren frühen Returns sitzt, kann die Frage „warum ist
nichts passiert“ nicht beantworten. Eine Sonde meldete null Ergebnisse, was als
„das Objekt passt nie“ gelesen wurde, während ein völlig anderes Gate alles
abwies.

Gib jedem frühen Return seinen eigenen Zähler und gib sie alle aus. Das macht aus
„Ich habe die Taste gedrückt und nichts ist passiert“ statt eines zweiten
Testzyklus eine einzige Logzeile.

Verwandt dazu: **Eine kapazitätsbegrenzte Sonde, die ihre eigene Grenze nicht
meldet, produziert selbstbewussten Unsinn.** Eine Erhebung füllte 509 von 512
Slots, sagte nichts darüber, und ihre Rangfolge wurde als „die nächstgelegenen
Objekte“ gelesen, während sie „das Nächstgelegene von dem, was zuerst
aufgezeichnet wurde“ bedeutete. Gib die Sättigung immer aus.

## Taste schneller ab als der Effekt, den du misst

Ein Test, der einen Wert um 180 Grad pro Stichprobe hochfuhr, erzeugte Daten, die
exakt so aussahen wie zwei Systeme, die sich um ein Feld streiten. Es war
Aliasing. Wenn du einen Wert abtastest, den die Engine ebenfalls schreibt, taste
schneller ab, als einer von euch beiden ihn ändert, sonst erfindest du einen
Konflikt, der nicht existiert.

## Bevorzuge absolute Werte gegenüber inkrementellen

Wo immer möglich, berechne einen absoluten Wert aus einer Quelle, die du
kontrollierst, statt ein Delta auf das zu komponieren, was gerade dort steht.

Inkrementelle Komposition ratscht immer dann hoch, wenn die Engine das Feld
nicht so oft auffrischt wie angenommen, und sie zwingt dich, Fragen zu
beantworten wie „baut die Engine das einmal pro Frame oder einmal pro Aufruf neu
auf“, die wirklich schwer zu messen sind. Ein absoluter Wert macht diese Fragen
gegenstandslos. Zwei unabhängige Konversionsprojekte auf anderen Engines kamen
zum selben Schluss, nachdem ihre Delta-Versionen hochgeratscht waren.

## Arbeite mit dem finalen Commit der Engine, nicht dagegen

Bei der Platzierung ist die verlässliche Position der letzte eigene
Schreibvorgang der Engine für einen Wert, statt von außen mit ihr um die Wette zu
laufen. Dann konkurrierst du nicht mehr um die Reihenfolge.

Bei dieser Engine heißt das `TransformNode::SetWorldTransform` für die
Platzierung angehängter Objekte und der Eintritt in
`Skeleton::PublishAttachments` für einen Knochen, aus dem Attachments komponiert
werden. Zu früh, und der Animations-Solver überschreibt dich; zu spät, und der
Konsument hat bereits gelesen. Dieselbe Idee taucht in anderen
Konversionsprojekten unter Namen wie „Permanent Change“ auf, und mindestens ein
bekannter Mod hat seine Renderer- und Animations-Hooks zugunsten davon aufgegeben.

## Sage, welche Seite der CPU/GPU-Trennlinie eine Änderung betrifft

Die GPU-Knochenpalette und das CPU-Skelett werden bei dieser Engine aus
**verschiedenen Quellen** gespeist (siehe [07-rendering.md](07-rendering.md)).
Die Palette zu ändern betrifft das, was du siehst, und nichts von dem, was das
Spiel glaubt. Das CPU-Skelett vor dem finalen Pose-Read zu ändern betrifft
Reichweite, Kollision und Mündungsposition.

Sage immer, welches von beiden du angefasst hast. Eine Änderung, die das Bild
bewegt, aber nicht die Kugeln, ist keine Lösung, und sie so zu nennen kostet alle
Beteiligten den nächsten Tag.

## Eine Verhaltensänderung pro Test-Build

Wenn eine Spielsitzung die einzige Wahrheitsquelle ist, bedeuten zwei Änderungen
in einem Build ein mehrdeutiges Ergebnis, und beide müssen erneut getestet werden.
Es fühlt sich langsam an und ist die schnellste verfügbare Option.

Folgerung: Entferne gescheiterte Experimente aus dem Baum, statt schlafende
Schalter und Fallback-Pfade zurückzulassen. Ein Baum voller deaktivierter Sonden
macht das nächste Problem dreimal schwerer auffindbar.

## Fixiere den Build, scheitere geschlossen

Zwei Retail-Builds dieses Spiels liefern dieselbe Engine mit **jeder Funktion an
einer anderen RVA** aus. Erkenne den Build am PE-Header des geladenen Moduls
selbst (TimeDateStamp plus SizeOfImage), wähle daraus eine Adresstabelle pro Build
und tue bei einem Binary, das du nie analysiert hast, überhaupt nichts.

Beachte die Asymmetrie, die das billig macht: **Das Datenlayout überlebte die
Neukompilierung, die Codeadressen nicht.** Vier Eigenschafts-Offsets der
Waffenkomponente wurden über das Update 2026-08 hinweg identisch neu hergeleitet,
während sich jede Funktion verschob. Tabellen pro Build brauchen also Adressen,
keine Struct-Offsets.

## Werkzeuge, die sich bezahlt gemacht haben

- Ein Signatur-Scanner, der **Fehltreffer** und **mehrdeutig** als
  unterschiedliche Ergebnisse meldet. Null Treffer bedeutet meist, dass das Spiel
  aktualisiert wurde; mehr als einer bedeutet, dass das Muster zu kurz ist. Beides
  zu vermengen kostet einen Tag.
- Ein Daten-Querverweis-Scanner, der **sowohl** rip-relative als auch absolut
  gespeicherte Zeiger behandelt. Die HumanIK-Tabellen werden nur über absolute,
  relozierte Zeiger referenziert; ein Scanner nur für rip-relative Referenzen
  meldet sie als tote Daten.
- Ein `.pdata`-Nachschlagewerkzeug (Funktionsgrenzen aus einer inneren Adresse).
- Ein Durchlauf zur RTTI-Wiederherstellung und -Demanglierung, auch wenn er hier
  nur die Engine-Infrastruktur abdeckt.
- Ein DXBC-Disassembler, betrieben über die vom Spiel selbst mitgelieferte
  `d3dcompiler_47.dll`. Der Reflection-Abschnitt (RDEF) ist aus den
  ausgelieferten Shadern entfernt, du bekommst also Slots und Strides, aber keine
  Namen.
- Eine CRC32-Rainbow-Table, gebaut aus einem Community-Wörterbuch **plus den
  eigenen Strings des Images**. Die zweite Hälfte ist das, was die Hand- und
  Waffenknochennamen geknackt hat.

## Was versucht wurde und nicht funktionierte

RenderDoc wurde bei diesem Titel über vier getrennte Wege verfolgt und lieferte
nie eine Aufnahme. Wenn ein Plan von einem Frame-Capture abhängt, halte einen
zweiten Plan bereit.

Die ausgelieferten Shader aus den Archiven zu lesen erwies sich ohnehin als
strikt besser: Es liefert exakte Registerslots und Strides ganz ohne Laufzeit, und
es ist das, was die Frage nach dem Platzierungspfad endgültig geklärt hat.
