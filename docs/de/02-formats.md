[English](../02-formats.md) · **Deutsch** · [한국어](../ko/02-formats.md)

---

# 02 – Archiv- und Datendateiformate

Alles hier ist `[VERIFIED]` gegen die ausgelieferten Wildlands-Archive, in dem
Umfang, der in der jeweiligen Aussage genannt wird. Das Container-Layout wurde
gegen Blacksmith (MIT) gegengeprüft; Formatkonstanten wurden gegen eine
Dekompilierung von AnvilToolkit abgeglichen, ausschließlich zur Faktenprüfung
gelesen.

Aus diesen Notizen wurden sowohl ein Reader als auch ein **byte-identischer
Writer** gebaut, das Format ist also im stärksten verfügbaren Sinn verifiziert:
Ein No-Op-Rebuild aller 21 Forges einer 62-GB-Installation reproduziert jede
Datei Byte für Byte, und eine modifizierte Forge lädt im ausgelieferten Spiel.

## Der `.forge`-Container

```
header:   "scimitar" (8 Bytes), Unknown1 u8, FileVersionIdentifier i32 (= 27),
          OffsetToDataHeader u64

@OffsetToDataHeader (DataHeader1):
          NumOfEntries i32, Unknown1 i32[4], Unknown2 i64,
          MaxFilesForThisIndex i32 (= 5000), Unknown3 i32, OffsetToData i64

@OffsetToData (DataHeader2, eine pro Index-Sektion):
          IndexCount i32, Unknown1 i32,
          OffsetToIndexTable i64, OffsetToNextDataSection i64 (-1 = letzte),
          IndexStart i32, IndexEnd i32,
          OffsetToNameTable i64, Unknown2 i64

@OffsetToIndexTable:  IndexCount x {
          OffsetToRawData i64, FileDataID i64, RawDataSize i32 }      (20 Bytes)

@OffsetToNameTable:   IndexCount x {
          RawDataSize i32, Unknown1 i64, Unknown2 i32,
          ResourceIdentifier u32, Unknown3 i32[2], NextFileCount i32,
          PreviousFileCount i32, Unknown4 i32, Timestamp i32,
          Name char[128], Unknown5 i32[5] }                           (192 Bytes)
```

Hart erarbeitete Details:

- `[VERIFIED]` **Das Namensfeld beginnt bei Datensatz-Offset 44, nicht 40.** Die
  44 Byte lange Präambel plus `char[128]` plus `i32[5]` ergibt exakt den
  192-Byte-Stride. Liest man den Namen bei Offset 40, sickern die druckbaren
  Bytes des `Timestamp`-Feldes in jeden Namen. Daher stammen die falschen
  Eintragsnamen `XGlobalMetaFile` (Basis-Forges, Timestamp `0x5888C31C`) und
  `JijGlobalMetaFile` (patch_01-Forges, Timestamp `0x6A692399`), falls du sie in
  anderen Werkzeugen gesehen hast. **Timestamp liegt bei 40.**
- `[VERIFIED]` Der `i64` bei Namensdatensatz-Offset 4 ist **nicht** die
  FileDataID; er stimmt bei aktiven Datensätzen nicht mit dem Wert der
  Indextabelle überein. Die Indextabelle ist die Autorität für Datei-IDs.
- `[VERIFIED, 270.004 / 270.004 Einträge einer 62-GB-Installation]` Die
  `RawDataSize` des Namensdatensatzes entspricht immer der `RawDataSize` des
  Indexdatensatzes. Ein Repack muss also beide patchen, sonst ist das Archiv
  inkonsistent.
- `[VERIFIED]` **Eintragsnamen sind nicht eindeutig.** 5.602 Duplikate in
  `DataPC_GRN_WorldMap.forge`, 335 in `DataPC.forge`, 0 in den kleinen. Die
  FileDataIDs *sind* innerhalb einer Forge eindeutig. **Adressiere einen Eintrag
  über die file_id, niemals über den Namen.**
- `[VERIFIED]` Der Header und beide Tabellen liegen **unterhalb** der ersten
  Nutzlast. Die Datei verläuft: Header, Indextabelle, Namenstabelle, ein kleiner
  `_Lost&Found`-Datensatz, reservierter Raum, dann Nutzlasten. Es ist kein
  TOC am Dateiende.
- `[VERIFIED]` Nutzlasten sind **zusammenhängend in Indexreihenfolge** abgelegt,
  ohne Lücken und ohne Überlappungen, bei allen 21 Forges.
- `[VERIFIED]` Die Gesamtdateigröße ist das Ende der letzten Nutzlast, aufgerundet
  auf **`0x8000`**, mit Nullen aufgefüllt. Exakt bei 21/21 Forges. `0x4000` passt
  nur bei 10/21, das Alignment ist also tatsächlich `0x8000`.
- `[VERIFIED]` Forges mit mehreren Sektionen verketten sich über
  `OffsetToNextDataSection`. `DataPC.forge` hat zufällig nur eine Sektion mit
  29.640 Einträgen, aber im Allgemeinen muss der Kette gefolgt werden; mindestens
  ein bekannter Reader liest nur die erste Sektion.
- Wenn du ein byte-identisches Repack willst, **sortiere Einträge beim Lesen
  nicht um.** Die Dateireihenfolge muss erhalten bleiben.

## Die Nutzlast: Anvil-Datendatei-Blocksätze

Die Nutzlast jedes Eintrags besteht aus einem oder mehreren Blocksätzen:

```
pro Satz: magic u64 = 0x1004FA9957FBAA33
          version i16 (= 1)
          compression u8
          maxBlockSize u16 (= 0x8000), maxBlockSize2 u16
          blockCount i32
          blockCount x { uncompressedSize u16, compressedSize u16 }
          blockCount x { checksum u32, data[compressedSize] }
```

- `[VERIFIED]` Die Blockgröße ist `0x8000` (32.768). Die Größenfelder sind
  **u16**, nicht der i32, den manche Reader für andere Anvil-Titel verwenden;
  dieser Dialekt unterscheidet sich.
- `[VERIFIED]` Ein Block, bei dem `compressedSize == uncompressedSize` gilt, ist
  unkomprimiert gespeichert.
- `[VERIFIED]` Die Kompressionsbytes 0 und 1 sind **beide LZO1X**. (2 = LZO1A,
  5 = LZO1C, beide bisher nicht in Wildlands gesehen.)

### Der Kompressor ist LZO1X-999, nicht LZO1X-1

`[VERIFIED]`, und das ist das stärkste Einzelergebnis in dieser Datei. Das
Dekomprimieren und Neucodieren **jedes Eintrags** von
`DataPC_GRN_TitleScreen_patch_01.forge` mit `lzo1x_999_compress` reproduziert die
ausgelieferte Datei **Byte für Byte** (SHA-256 `1A7C3B8C...`). `lzo1x_1` erzeugt
gültige Streams, die etwa 15 % größer sind.

Der Kompressor der Toolchain ist also nicht bloß kompatibel mit dem des Spiels,
er *ist* der des Spiels. Wenn du ein byte-identisches Repack unveränderter
Inhalte brauchst, verwende LZO1X-999.

### Die Prüfsumme pro Chunk

`[VERIFIED, 789 / 789 Chunks über drei Forges]` Es ist **Adler-32 mit beiden
Akkumulatoren auf NULL initialisiert**. Standard-Adler-32 startet mit `a = 1`,
weshalb eine Standardimplementierung hier fehlschlägt:

```python
a = 0; b = 0
for byte in chunk_bytes:      # die GESPEICHERTEN (möglicherweise komprimierten) Bytes
    a = (a + byte) % 65521
    b = (b + a)    % 65521
checksum = (b << 16) | a
```

Das war die lange bestehende Blockade beim Schreiben eines gültigen Archivs, und
es ist tatsächlich nur dieser eine Unterschied in der Initialisierung.

### Die Satzstruktur ist tragend

`[VERIFIED]` Jede Original-Nutzlast verwendet Kompressionsbyte 1 und trägt einen
**kleinen führenden Blocksatz** vor dem eigentlichen Satz: 16 Bytes für ein
Material, 128 für einen Zellen-Datablock, 41.806 Bytes in zwei Blöcken für einen
Szenen-Deskriptor. Dieser führende Satz ist der Header-Datensatz der Ressource,
und die Engine liest ihn getrennt.

Eine als **einzelner** Satz mit Kompressionsbyte 0 neu gepackte Nutzlast
dekomprimiert in einem unabhängigen Reader korrekt, **lässt das Spiel aber in
einem Schwarzbild hängen.** Am Retail-Spiel verifiziert. Ein Repack muss also die
ursprüngliche Satzstruktur und das Kompressionsbyte erhalten, nicht bloß einen
gültigen Stream erzeugen.

## Der Multi-Resource-Container innerhalb einer Nutzlast

`[VERIFIED]` Die dekomprimierte Nutzlast eines Forge-Eintrags ist üblicherweise
nicht eine Ressource. Sie ist ein Verzeichnis von Ressourcen:

```
u16 count
count x { u64 file_id, u32 record_size, u16 flags }        (14 Bytes, das TOC)
count x {
    u32 class_hash        # CRC32 des Klassennamens
    u32 body_size
    u32 name_len
    char name[name_len]
    u8  gap[...]          # normalerweise das einzelne NUL, das den Namen beendet
    u8  body[body_size]   # beginnt erneut mit u64 file_id, u32 class_hash
}
```

- `record_size` umfasst den **gesamten** Datensatz (12 + name_len + gap +
  body_size), der Body wird also als `record_start + record_size - body_size`
  lokalisiert. Leite ihn so her, statt ein einzelnes NUL anzunehmen: Mindestens
  ein realer Datensatz hat `name_len == 0` und eine 8 Byte große Lücke. Übernimm
  die Lücke wortwörtlich.
- `[VERIFIED]` Drei unabhängige Gegenproben gelten bei jedem Datensatz: die
  TOC-ID entspricht der ID im Body, der äußere Klassen-Hash entspricht dem Hash im
  Body, und die `record_size` des TOC entspricht der beim Durchlaufen ermittelten
  Datensatzlänge. Erzwinge alle drei, dann scheitert eine Nutzlast anderer Form
  laut statt still.
- `[VERIFIED]` `class_hash` ist schlicht **CRC32 des Klassennamens**:
  `crc32("Skeleton") == 0x24AECB7C`. 49 der 50 verschiedenen Klassen-Hashes, die
  über 6.563 Ressourcen hinweg auftauchen, lösen sich auf diesem Weg in echte
  Namen auf (Animation, Mesh, Material, Skeleton, Entity und so weiter).
- `[VERIFIED]` Abdeckung: 473 von 479 Nutzlasten über drei komplette Forges plus
  eine Stichprobe von 400 Einträgen aus `DataPC.forge` lassen sich als Container
  parsen, und alle 473 bauen byte-identisch wieder auf. Die 6 Ausnahmen sind 3
  `GlobalMetaFile`-Einträge (eine andere Struktur) und 3 `PrefetchingFileInfos`
  (siehe unten).

### `PrefetchingFileInfos` und `GlobalMetaFile`

`[VERIFIED]` Jede Forge endet mit einem `PrefetchingFileInfos`-Eintrag, der die
Datendatei-Magic trägt, **nicht** aber das Blocksatz-Layout; sein Blockzähler
parst als Unsinn. Er ist der verbleibende Ausreißer bei der Dekompression.

Nützlicherweise enthält seine Nutzlast die FileDataIDs der echten Einträge und
**keine Größen und keine Offsets**. Ein Repack, das Offsets umfließen lässt und
Nutzlastgrößen ändert, macht ihn also nicht ungültig. Ihn unverändert
durchzureichen ist sicher. `GlobalMetaFile` ist überhaupt keine Datendatei (keine
Magic) und wird ebenso durchgereicht.

## Basis-Forge gegen Patch-Forge: welche Kopie das Spiel liest

`[VERIFIED]` Dies entscheidet, wohin eine Änderung gehört, und ein Fehler hier
macht einen Test still ergebnislos statt fehlschlagend.

`DataPC.forge` und `DataPC_patch_01.forge` tragen beide einen Eintrag namens
`GR_PLAYER_Template` mit **derselben Eintrags-file_id**, aber es ist nicht
derselbe Container: 1.838.722 Bytes und 1.241 Ressourcen in der Basis, 4.916.946
Bytes und 2.238 Ressourcen im Patch. Nach Ressourcen-file_id sind 1.230 in beiden,
11 nur in der Basis, 1.008 nur im Patch, und von den 1.230 gemeinsamen
**unterscheiden sich 277 in den Bytes**.

Die Nur-Basis-Menge enthält das 100-Knochen-Rig des Spielerkörpers, und das Spiel
hat offensichtlich weiterhin einen Spielerkörper. Die Engine übernimmt also nicht
pauschal den neuesten Container: **Sie löst Ressourcen forge-übergreifend über die
Ressourcen-file_id auf, und ein Patch ersetzt nur die IDs, die er tatsächlich
enthält.**

Praktische Regeln:

- Eine Ressource zu bearbeiten, die in **beiden** Forges vorhanden ist, bedeutet,
  die Patch-Kopie (oder beide) zu bearbeiten, sonst wird die Änderung maskiert und
  dein Test beweist nichts.
- Eine **nur in der Basis** vorhandene Ressource zu bearbeiten ist eindeutig: Die
  Basis ist die einzige Kopie.
- Die kleinen Forges (GhostRoom, TitleScreen und ihre Patches) enthalten
  überhaupt keine Skeleton-Ressourcen, eine Skelettänderung lässt sich also nicht
  gegen sie testen.

## Was ein Writer kann und was nicht

Ermittelt durch den Bau eines solchen:

- `[VERIFIED]` Ein No-Op-Rebuild ist **byte-identisch (SHA-256) bei allen 21
  Forges** einer 62-GB-Installation, bis hinauf zu einem Archiv von 21 GB mit
  63.351 Einträgen.
- `[VERIFIED, im Retail-Spiel]` Eine neu gebaute Forge mit **unseren**
  Nutzlast-Bytes, **unserem** Blocklayout, **unseren** Prüfsummen und **umgeflossenen
  Offsets** lädt korrekt, sofern Satzstruktur und Kompressionsbyte erhalten
  bleiben. Der gescheiterte Einzelsatz-Versuch dient zugleich als Positivkontrolle:
  Er erzeugt ein Schwarzbild, was beweist, dass die Engine dieses Archiv
  tatsächlich liest und ein späterer sauberer Start kein falsch positives Ergebnis
  ist.
- Die LZO-Ausgabe ist nicht kanonisch, ein byte-identisches Repack eines
  **geänderten** Eintrags ist also nicht erreichbar und kein sinnvolles Ziel.
  Kopiere die Nutzlast-Bytes unveränderter Einträge wortwörtlich, statt sie neu zu
  codieren.
- **Nur Einträge ersetzen.** Nichts hat bestätigt, dass das Spiel eine geänderte
  Eintragsanzahl toleriert; Einträge hinzuzufügen, zu entfernen oder umzuordnen
  ist unbewiesen.

Zwei stehende Warnungen für alle, die solche Änderungen verbreiten: Die
Steam-Dateiüberprüfung macht sie rückgängig, und jeder Spiel-Patch macht sie
ungültig.
