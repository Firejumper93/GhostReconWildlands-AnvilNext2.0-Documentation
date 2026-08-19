[English](../08-reflection.md) · **Deutsch** · [한국어](../ko/08-reflection.md)

---

> **Hinweis, 2026-08-19. Diese Uebersetzung enthaelt eine Regel, die inzwischen
> widerlegt ist.** Die hier beschriebene Offset-Dekodierung `packed >> 18` gilt
> fuer diesen Build NICHT. Sie weist zwei benachbarten Feldern denselben Offset
> zu, was unmoeglich ist. Richtig ist **`(packed & 0xFFFF) >> 3`**.
>
> Die englische Fassung wurde am 2026-08-19 um diese Korrektur und um ein
> Verfahren zum Knacken zusammengesetzter Namen erweitert. Bis zur
> Aktualisierung ist die [englische Fassung](../08-reflection.md) massgeblich.
> Ein Termin wird diesmal bewusst nicht genannt, da der zuvor zugesagte Termin
> (2026-08-17) nicht eingehalten wurde.

# 08 – Die Reflection-Tabellen und das Zurückgewinnen von Engine-Namen

Dies ist die wirkungsvollste einzelne Technik bei dieser Engine, daher bekommt
sie eine eigene Datei. Sie ist das, was aus „einem unbekannten Dword bei `+0xB0`“
ein `m_GunRootBone : BoneHandle` macht.

## Das Problem

`[VERIFIED NEGATIVE]` **Im Image existieren überhaupt keine
Gameplay-Klassennamen im Klartext.** Nicht ein einziger. Die 1.462
wiederherstellbaren MSVC-RTTI-Deskriptoren decken Engine-Infrastruktur ab und
sonst nichts.

Der übliche Weg (den Klassennamen-String finden, ihn querverweisen, im Konstruktor
landen) existiert hier also nicht. Der CRC32-Weg war keine Abkürzung, er war der
einzige Weg.

## Die Reflection-Tabellen

`[VERIFIED, Update-Build 2026-08]` Anvil führt ein vollständiges
Laufzeit-Reflection-System als statische Daten mit, in einer Sektion namens
`.arch` (`0x040FF000..0x04E38000`). Eine einzige Registrar-Funktion durchläuft
alles davon: **RVA `0x017582D0`**.

Tatsächlich wiederhergestellte Abdeckung: **4.491 Klassendeskriptoren** und
**24.642 Eigenschaftsdatensätze**, davon 2.564 Klassen und 12.813 Datensätze
**benannt**.

Layout eines Eigenschaftsdatensatzes, Stride `0x38`:

| Offset | Inhalt |
|---|---|
| `+0x04` | CRC32 des Eigenschaftsnamens |
| `+0x08` | CRC32 des Typnamens |
| `+0x0C` | Art (kind) |
| `+0x10` | gepackt; **Byte-Offset = packed >> 18** |

Die letzte Zeile ist der eigentliche Gewinn. Sobald du eine Eigenschaft benennen
kannst, liefert dir derselbe Datensatz ihren **Byte-Offset innerhalb des
Objekts** – du bekommst also ein vollständig annotiertes Struct-Layout gratis
dazu.

(Beachte: Die Verankerung ist auf diesem Build +4 gegenüber einer älteren
Konvention; prüfe deine eigene.)

## Die Methode: CRC32-Rainbow-Cracking

Alles wird mit schlichtem CRC32 gehasht, und CRC32 ist kein kryptographischer
Hash. Also baue eine Rainbow Table `hash -> Name` und schlage die Antworten nach.

Woher die Kandidatennamen stammen, geordnet nach ihrem Beitrag:

1. **Ein Community-Hash-Wörterbuch.** AnvilToolkit bringt eines mit (etwa 506.000
   Namen, Oodle-komprimiert in seinen Ressourcen). Gute Abdeckung von
   Knochennamen und Assetnamen. **Für sich allein reichte es nicht.**
2. **Die Klartext-Strings des Images selbst.** Alle 1,2 Millionen bereits in der
   ausführbaren Datei vorhandenen Klartext-Strings zu hashen, plus deren
   Token-Aufspaltungen, ergab etwa 490.000 zusätzliche Einträge. **Damit wurden
   die Hand- und Waffenknochennamen geknackt.** Die Engine enthält die Wörter,
   auch wenn sie die Bezeichner nicht enthält.
3. **Gezieltes Raten mit Positivkontrollen.** Sobald du die Namenskonvention
   kennst (`m_` als Präfix für Member, `v_` für autorierte Strings, `GR_` für
   Ghost-Recon-spezifische Klassen, `c` für Klassen, `s` für Structs), lassen sich
   Kandidaten billig erzeugen und testen.

Kombinierte Tabelle: **1.616.260 Einträge.**

### Führe immer Positivkontrollen mit

Jeder Cracking-Lauf sollte im selben Lauf Fakten reproduzieren, die du bereits
kennst. Die hier verwendeten Kontrollen waren `crc32("Head") == 0x07C159A2`,
`crc32("cBallisticProjectileComponent") == 0x09BFE10E` sowie vier bereits im
vorherigen Build ermittelte Offsets der Waffenkomponente, die auf dem neuen Build
unabhängig neu hergeleitet wurden.

Ohne Kontrollen ist ein Cracking-Lauf, der nichts findet, nicht von einem kaputten
Skript zu unterscheiden – und das ist genau der Fall, in dem ein kaputtes Skript
wie ein sauberer Negativbefund aussieht.

## Was das eingebracht hat

Konkrete Beispiele, alle aus den Tabellen:

- Das gesamte Layout von `GR_cWeaponComponent`, mit Eigenschaftsnamen und Offsets
  (siehe [06-weapons.md](06-weapons.md)).
- `cWeaponAttachmentHolder` mit einem benannten Slot pro Waffenteil, was den
  Shader-Befund, dass eine Waffe aus vielen Objekten besteht, unabhängig
  bestätigte.
- `Skeleton::BipedBoneID`, das Attachment-Punkt-Enum mit 143 Einträgen, 132 Namen
  wiederhergestellt (siehe [03-skeleton.md](03-skeleton.md)).
- Die Feldnamen der Projektilkomponente, sodass
  `m_vBulletSimulationDirection` und `m_vBulletShootOrigin` die engine-eigenen
  Namen sind und nicht unsere Bezeichnungen für zwei Offsets.
- Die Zuordnung von Klassen-Hash zu Name für den Ressourcen-Container: 49 von 50
  verschiedenen Klassen-Hashes über 6.563 Ressourcen.

## Vier Negativbefunde, die ändern, wie man diese Engine navigiert

Diese sparen am meisten Zeit, denn jeder davon ist eine Technik, die bei anderen
Engines gut funktioniert und hier nicht.

1. `[VERIFIED NEGATIVE]` **Klassendeskriptoren und Eigenschaftstabellen haben
   null rip-relative Codereferenzen.** An fünf von ihnen geprüft, mit einer
   Kontrolle, die im selben Lauf durchlief. Sie werden nur über Zeiger-Slots
   erreicht, die der Registrar durchläuft. Damit funktioniert **„den Deskriptor
   querverweisen, um den Code zu finden, der Feld X liest“ nicht.** Code, der
   Felder liest, muss stattdessen über Offset-Muster oder über
   Knochen-Hash-Konstanten gefunden werden.
2. `[VERIFIED NEGATIVE]` **Eigenschaftsdatensätze tragen auf diesem Build keinen
   Getter-Zeiger.** Die Bytes `+0x14..+0x37` sind auf der Platte null. Eine ältere
   Notiz, die einen Getter-Thunk im Datensatz beschreibt, überträgt sich nicht.
3. `[VERIFIED NEGATIVE]` **Methodennamen sind weitgehend nicht
   wiederherstellbar.** Nur 11.283 von 103.014 Einträgen lösen sich auf, fast
   ausschließlich generische Lifecycle-Methoden. Weder `AttachTo` noch
   `GetAttachmentBone` noch `FindBone` konnte als Name wiederhergestellt werden;
   keines der beiden Wörterbücher enthält Engine-Methodenbezeichner. Plane nicht
   damit, an Methodennamen zu kommen.
4. `[UNKNOWN]` 11.124 Hashes bleiben ungelöst. Der Schwanz ist lang, und die
   verbleibenden Namen sind genau die, die in keinem Wörterbuch stehen.

## Andere Dinge, die genauso gehasht werden

Sobald du weißt, dass CRC32 der universelle Identifikator ist, öffnen sich
mehrere weitere Strukturen auf einmal:

- **Knochennamen** in `.Skeleton`-Assets und in der Namenskarte des
  Laufzeit-Rigs.
- **Klassennamen** im TOC des Multi-Resource-Containers.
- **Knochennamen im Code**: Die Engine lädt ein literales CRC32 nach `edx` und
  ruft die Knochensuche auf. Eine Suche nach einem bekannten Knochen-Hash als
  Immediate findet also jede Stelle, die diesen Knochen namentlich anfasst. So
  wurden die Gun-Root-Auswahl und die Aufrufstelle der Handauswahl gefunden.
- **Die HumanIK-Knotenlisten**, die statische CRC32-Listen sind.

Die letzte Technik verdient Nachdruck. Bei einer Engine ohne Gameplay-Strings ist
ein **Knochen-Hash als Immediate das, was einem Symbol im Code am nächsten
kommt**, und es gibt nur ein paar hundert davon. Jede Stelle aufzuzählen, die
eines lädt, liefert dir eine Karte von allem, was die Engine mit benannten
Knochen tut.
