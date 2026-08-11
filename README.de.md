<div align="center">

**🌐 Language · Sprache · 언어**

[English](README.md) · **Deutsch** · [한국어](README.ko.md)

</div>

---

# Ghost Recon Wildlands / AnvilNext 2.0 – Engine-Dokumentation

Reverse-Engineering-Notizen zu Ubisofts AnvilNext 2.0 in der Fassung, die in
*Tom Clancy's Ghost Recon Wildlands* (2017, x86-64, Direct3D 11) ausgeliefert
wurde.

Alles hier wurde aus erster Hand aus der ausgelieferten ausführbaren Datei, den
ausgelieferten Shadern und den ausgelieferten Datenarchiven abgeleitet und
anschließend verifiziert. Veröffentlicht wird es, weil die Engine ansonsten
undokumentiert ist, weil jede einzelne dieser Tatsachen echte Zeit gekostet hat
und weil insbesondere die negativen Ergebnisse zu der Sorte gehören, die nie
jemand aufschreibt.

Wildlands ist ein nützlicher Bezugspunkt für die Anvil-Familie. Das Spiel
erschien im März 2017, früh in der AnvilNext-2.0-Ära und vor der Reihe von
Assassin's-Creed-Titeln, auf die sich die meisten Anvil-Werkzeuge richten. Es
zeigt also, wie die Engine vor diesen Änderungen aussah, und ist dabei immer
noch erkennbar dieselbe Engine.

---

## Genauigkeitshinweis, gleich vorweg

**Dies ist Reverse Engineering nach bestem Wissen. Es ist keine Dokumentation
von Ubisoft, es wurde von niemandem mit Zugriff auf den Quellcode geprüft, und
Teile davon sind möglicherweise nicht 100 % korrekt. Dies ist ein laufendes
Forschungsprojekt.**

Lies es in diesem Geist:

- **Befunde wurden bis zu dem Standard verifiziert, der in der jeweiligen
  Aussage genannt wird, und nicht weiter.** „Verifiziert“ heißt hier, dass ein
  konkretes Artefakt die Aussage stützt. Es heißt nicht, dass jemand anderes sie
  unabhängig reproduziert hat oder dass sie in Fällen gilt, die niemand getestet
  hat.
- **Diese Engine hat bereits bewiesen, dass sie selbstbewusste falsche Antworten
  hervorbringen kann.** Mehrere Schlussfolgerungen in diesen Dateien haben eine
  frühere Fassung ersetzt, die plausibel, in sich stimmig und falsch war. Wo das
  passiert ist, bleibt die Korrektur im Text stehen, statt herausredigiert zu
  werden – gerade weil die falsche Lesart nachvollziehbar war und jemand anderes
  ebenfalls darauf kommen wird.
- **Adressen veralten.** RVAs sind build-spezifisch. Ein Spiel-Update verschiebt
  sie alle, und zwei Retail-Builds dieses Spiels unterscheiden sich bereits in
  jeder einzelnen Funktionsadresse.
- **Ein Teil davon ist Schlussfolgerung.** `[INFERRED]` bedeutet genau das: aus
  verifizierten Fakten hergeleitet, tragend und nicht bewiesen.
- **Datei 11 ist die unzuverlässigste und sagt das oben auch selbst.**
  Titelübergreifende Aussagen sind Extrapolation aus einem einzigen Titel.
- **Das Fehlen eines Befunds ist kein Beweis.** Ein `[UNKNOWN]` bedeutet
  üblicherweise, dass noch niemand gründlich genug nachgesehen hat, nicht dass
  die Antwort unerkennbar wäre.

Wenn du darauf aufbaust: leite alles Tragende gegen dein eigenes Binary neu her,
bevor du dich darauf verlässt, und bevorzuge die Byte-Signaturen gegenüber den
Adressen – genau dafür sind sie da.

**Korrekturen sind ausdrücklich willkommen.** Öffentlich falsch zu liegen und
korrigiert zu werden ist besser, als privat falsch zu liegen. Öffne ein Issue
mit dem, was du gemessen hast, und wie.

---

## Wie diese Dokumentation zu lesen ist

Jede Aussage trägt eine Vertrauensmarkierung, und sie bedeuten genau das, was
sie sagen:

| Marker | Bedeutung |
|---|---|
| `[VERIFIED]` | Durch ein konkretes Artefakt belegt: ein Disassembly, ein Byte-Muster, ein Shader-Listing, ein Round-Trip-Test oder eine Laufzeitmessung. Beweise, auf die man zeigen kann. |
| `[INFERRED]` | Eine begründete Schlussfolgerung aus verifizierten Fakten. Plausibel, tragend, nicht bewiesen. |
| `[UNKNOWN]` | Offen. Festgehalten, damit niemand annimmt, die Frage sei beantwortet. |
| `[VERIFIED NEGATIVE]` | Etwas, das definitiv **nicht** zutrifft, oder ein Ansatz, der definitiv nicht funktioniert. |

Die Negativbefunde sind kein Füllmaterial. Bei einer geschlossenen Engine ist
das Wissen, dass ein Ansatz tot ist, genauso viel wert wie das Wissen, dass
einer funktioniert – und es gibt eine ganze Datei davon.

**Adressen sind RVAs und build-spezifisch.** Zwei Builds tauchen hier auf und
sind stets gekennzeichnet: der Retail-Build aus der 2017er-Ära und das Update
von 2026-08. Wo eine Struktur die Neukompilierung überlebt hat, aber verschoben
wurde, sind beide Adressen angegeben.

---

## Inhalt

Alle Notizen liegen in [`docs/`](docs/). Diese Übersicht verweist auf die
deutschen Fassungen in [`docs/de/`](docs/de/).

| Datei | Was darin steht |
|---|---|
| [01-binary.md](docs/de/01-binary.md) | PE-Layout, die verwürfelte Sektionstabelle, die Jump-Thunk-Aufrufarchitektur, Koordinatensystem, der Engine-Task-Graph |
| [02-formats.md](docs/de/02-formats.md) | Der `.forge`-Container, das Anvil-Datenblockformat, die LZO-Variante, die Prüfsumme, der Multi-Resource-Container, Auflösung von Base gegen Patch |
| [03-skeleton.md](docs/de/03-skeleton.md) | Das `.Skeleton`-Assetformat, der Laufzeit-Rig-Deskriptor, CRC32-Knochenbenennung, das 143-Einträge-Biped-Knochen-Enum, HumanIK |
| [04-pose.md](docs/de/04-pose.md) | Das Pose-Objekt, der Knochentransformationspuffer, Model- gegen World-Space, die Skelettkette pro Frame, wo ein Knochenschreibzugriff überlebt |
| [05-camera.md](docs/de/05-camera.md) | Die Kamera-Struktur, die neun Matrizen, die maßgebliche Transformation, die sechs Projektionsfunktionen |
| [06-weapons.md](docs/de/06-weapons.md) | Die Attachment-Kette von Anfang bis Ende, `TransformNode::SetWorldTransform`, Layout der Waffenkomponenten, Projektile, Havok |
| [07-rendering.md](docs/de/07-rendering.md) | Die drei Pfade zur Geometrieplatzierung, die GPU-Knochenpalette, Shader-Permutationsschlüssel, Registerslots und Strides |
| [08-reflection.md](docs/de/08-reflection.md) | Die Klassen-/Eigenschafts-Reflection-Tabellen und wie man daraus per CRC32 Engine-Namen zurückgewinnt |
| [09-methodology.md](docs/de/09-methodology.md) | Die Techniken, die funktioniert haben, in der Reihenfolge, in der sie einen Versuch wert sind |
| [10-negatives.md](docs/de/10-negatives.md) | Alles, was definitiv **nicht** zutrifft, und jeder Ansatz, der definitiv scheitert |
| [11-other-anvil-titles.md](docs/de/11-other-anvil-titles.md) | **Extrapolation.** Was auf andere Anvil-/AnvilNext-2.0-Spiele übertragbar sein sollte, was nicht, eine Portierungs-Checkliste |

---

## Wie diese Dokumentation zu benutzen ist

### Wenn du die Archive lesen willst

Beginne mit [02-formats.md](docs/de/02-formats.md). Es ist eine vollständige
Spezifikation zum Lesen und – wichtiger noch – zum **Schreiben** einer `.forge`:
Container-Layout, das Nutzdatenformat der Blocksätze, die exakte LZO-Variante,
die nicht standardkonforme Prüfsumme und das Multi-Resource-Verzeichnis
innerhalb einer Nutzlast.

Die nicht offensichtlichen Teile, gelernt durch das Kaputtmachen des Spiels,
sind die Einschränkungen für den Writer: die ursprüngliche Blocksatzstruktur und
das Kompressionsbyte erhalten, ausschließlich Einträge ersetzen, Einträge über
die Datei-ID statt über den Namen adressieren und das Ende auf `0x8000`
auffüllen.

### Wenn du mit Charakteren oder Animation arbeiten willst

[03-skeleton.md](docs/de/03-skeleton.md) enthält ein byte-vollständiges
`.Skeleton`-Format ohne einen einzigen unerklärten Byte, und die Knochennamen
lösen sich in die Standardkonvention Autodesk HumanIK auf.
[04-pose.md](docs/de/04-pose.md) liefert die Laufzeitseite: wie man von einem
Skelettzeiger zur Weltposition eines animierten Knochens kommt und welcher
Puffer maßgeblich ist gegenüber welchem, der ein veralteter Spiegel ist.

### Wenn du einen Kamera-Mod schreiben willst

[05-camera.md](docs/de/05-camera.md) enthält die maßgebliche
Kameratransformation, was passiert, wenn man sie schreibt (getestet), was
passiert, wenn man die naheliegende falsche schreibt (ebenfalls getestet), und
das eine sichtbare Artefakt, das sie erzeugt.

### Wenn du etwas hooken willst

[01-binary.md](docs/de/01-binary.md) beschreibt die Jump-Thunk-Architektur, die
die sauberste Abfangfläche dieses Binaries ist und aus Gründen, die spezifisch
für dieses Binary sind, besser als ein Prolog-Detour.
[09-methodology.md](docs/de/09-methodology.md) enthält die Verifikationsdisziplin,
die dafür sorgt, dass ein Spiel-Update eine saubere Verweigerung statt eines
Absturzes erzeugt.

### Wenn du eine Woche sparen willst

Lies zuerst [10-negatives.md](docs/de/10-negatives.md). Es ist der kürzeste Weg
dazu, keine Arbeit zu wiederholen, die bereits erledigt wurde und gescheitert
ist.

### Wenn du an einem anderen Anvil-Titel arbeitest

[11-other-anvil-titles.md](docs/de/11-other-anvil-titles.md), mit seiner eigenen
Warnung versehen. Es enthält die Handvoll direkter titelübergreifender
Vergleiche, die tatsächlich angestellt wurden, eine Portierungs-Checkliste und
eine ehrliche Aufteilung zwischen dem, was übertragbar sein sollte, und dem, was
es definitiv nicht ist.

---

## Laufende Projekte

Diese Dokumentation ist ein Nebenprodukt. Sie existiert, weil diese Projekte die
Antworten gebraucht haben.

### GRW-XR, eine native OpenXR-VR-Konversion (in Entwicklung, Alpha-Releases öffentlich)

Ein nativer VR-Mod für Wildlands: echtes OpenXR, Stereo-Rendering auf dem
spieleigenen D3D11-Device, kopfgesteuerte Kamera, Touch-Controller und eine
Egoperspektiv-Kamera, die am animierten Kopfknochen der Spielfigur verankert
ist.

Stand zum Zeitpunkt der Niederschrift: spielbar im Alphastadium, mit einem im
Headset bestätigten Ergebnis, das – soweit diese Recherche feststellen konnte –
**keinen Vorläufer in dieser Engine-Familie hat**: Die Waffe im Spiel folgt dem
Motion-Controller 1:1 in Position und Rotation, angetrieben durch einen
Knochenschreibzugriff am engine-eigenen Attachment-Publish. Die ehrliche Lücke,
dokumentiert statt beschönigt: Projektile folgen weiterhin der Kamera statt der
Waffe. An dieser Arbeit wird weitergearbeitet.

**Releases, Installer und Issue-Tracker:
<https://github.com/Firejumper93/GhostReconWildlandsVR>**

### GRW-FP, ein Egoperspektiv-Mod für den Flachbildschirm (in Entwicklung, unveröffentlicht)

Das Nicht-VR-Geschwister. Derselbe Kopfknochen-Kameraanker für gewöhnliches
Flachbildschirmspiel, wobei die spieleigene Maussteuerung die volle Hoheit über
das Zielen behält, sodass Ballistik, Fadenkreuz und jedes am Zielen verankerte
UI-Element sich exakt so verhalten wie im Auslieferungszustand. Er verschiebt
die Kameraposition und sonst nichts.

Er existiert, weil diese Einschränkung ihn zu einem viel kleineren Problem macht
als die VR-Konversion: Fast die gesamte VR-Arbeit ist Rotationsarbeit, und
nichts davon wird für flache Egoperspektive gebraucht.

### Archiv- und Asset-Werkzeuge (privat, Befunde hier veröffentlicht)

Ein Reader und ein byte-identischer Writer für den `.forge`-Container und die
Anvil-Datendateien, gebaut, um Fragen zu beantworten, die die Laufzeit nicht
beantworten konnte. Seine Ergebnisse sind das, was
[02-formats.md](docs/de/02-formats.md) dokumentiert: Ein No-Op-Rebuild
reproduziert alle 21 Archive einer 62-GB-Installation Byte für Byte, und ein neu
gebautes Archiv mit neu codierten Nutzdaten und umgeflossenen Offsets lädt im
Retail-Spiel korrekt.

Die Werkzeuge selbst sind nicht veröffentlicht. Das Formatwissen schon, und zwar
vollständig, damit sich jeder seine eigenen bauen kann.

---

## Umfang, und was hier bewusst nicht steht

Dies ist Dokumentation zur Engine-Architektur. Es ist kein Modding-Spickzettel
und kein Werkzeugkasten.

Bewusst ausgeschlossen und auch auf Anfrage nicht erhältlich:

- **Alles zum Kopierschutz.** Keine Anti-Tamper-Analyse, keine
  Trigger-Positionen, keine Diskussion von Umgehungen. Die Arbeit, aus der dies
  hervorging, lief innerhalb des normalerweise geschützten Prozesses, ohne diese
  Maschinerie anzurühren, und die Notizen dazu bleiben privat.
- **Alles im Umfeld von Anti-Cheat.** Keine Manipulation von Treffgenauigkeit,
  Streuung, Schaden oder Zielhilfe. Die Projekte, aus denen dies hervorging,
  sind strikt Einzelspieler, und diese Klasse von Adressen zu veröffentlichen
  dient dem Cheaten und sonst nichts.
- **Von Dritten stammendes Material.** Notizen aus fremden, in der Nutzung
  eingeschränkten Mods, dekompilierten Werkzeugen oder Cheat Tables werden hier
  nicht wiederveröffentlicht, unabhängig von ihrem Lizenzstatus.
- **Jegliche Spielinhalte.** Keine Binaries, keine extrahierten Assets, keine
  Archiv-Dumps, keine Shader-Blobs. Es wird erwartet, dass Leser das Spiel
  besitzen und selbst extrahieren.

---

## Herkunft und Vorgehen

Die Arbeit entstand für die oben genannte VR-Konversion über etwa dreißig
Arbeitssitzungen. Die statische Analyse lief offline gegen eine Kopie der
ausführbaren Datei; Laufzeitfakten wurden durch Hooken, Messen und erneutes
Testen ermittelt, unter der stehenden Regel, dass ein Ergebnis im Headset oder
auf dem Bildschirm immer über einer Logzeile oder einer plausibel aussehenden
Argumentation steht.

Ein großer Teil dessen, was hier steht, existiert, weil eine frühere
selbstbewusste Fassung davon sich als falsch herausstellte. Diese Korrekturen
bleiben im Text stehen, statt herausredigiert zu werden, weil die falschen
Lesarten nachvollziehbar waren und jemand anderes ebenfalls darauf kommen wird.

---

## Links und Unterstützung

- **Der VR-Mod** (Releases, Installer, Issues):
  <https://github.com/Firejumper93/GhostReconWildlandsVR>
- **Buy me a coffee**: <https://buymeacoffee.com/firejumper93>

Unterstützung ist völlig freiwillig und niemals erforderlich. Alles hier und
alles im VR-Mod ist kostenlos und bleibt kostenlos. Wenn diese Dokumentation dir
eine Woche vor dem Disassembler erspart hat, ist das bereits der Sinn der
Veröffentlichung.

---

## Lizenz

Notizen und Fließtext: **CC BY 4.0**. Nenne die Quelle und nutze sie frei, auch
in kommerzieller Arbeit.

Es ist kein Spielcode, keine Spieldaten und kein Code Dritter enthalten oder
weiterverbreitet. Ghost Recon, Wildlands, AnvilNext und Anvil sind Marken von
Ubisoft Entertainment. Dies ist inoffizielle Dokumentation, die durch
Interoperabilitätsanalyse entstanden ist, und steht in keiner Verbindung zu
Ubisoft, wird von Ubisoft nicht unterstützt und nicht befürwortet.

---

*Diese Übersetzung folgt dem englischen Original. Bei Abweichungen ist
[README.md](README.md) maßgeblich.*
