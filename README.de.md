# Eine schwache Zelle im LiFePO4-Parallelverbund finden

Eine Methode, um Degradation einzelner Zellen in parallelen LiFePO4-Speichern
(Pylontech US2000/US3000/US5000 und ähnliche) allein aus der Telemetrie zu
diagnostizieren, die das BMS ohnehin liefert.

Das hier existiert, weil es diese Information sonst nirgends gibt. Pylontech
veröffentlicht kein Diagnoseverfahren. In den Foren kursiert Erfahrungswissen — „deine
Zellen driften, lade öfter auf 100 %" — aber keine Methode, um herauszufinden, was
*tatsächlich* nicht stimmt, und um eine wirklich versagende Zelle von normaler
Serienstreuung zu unterscheiden.

Entwickelt wurde das an einem realen Speicher: 4× US3000C an einem Victron
MultiPlus-II 3000, rund 475 Zyklen, auf Basis von etwa elf Monaten Logdaten plus
gezielten Tests. Alle Zahlen unten stammen von diesem Speicher.

---

## Inhalt

- [Warum das schwierig ist](#warum-das-schwierig-ist)
- [Die Kernerkenntnis](#die-kernerkenntnis)
- [Wann eine Messung gültig ist](#wann-eine-messung-gültig-ist)
- [Die Daten erheben](#die-daten-erheben)
- [Die Tests](#die-tests)
  - [1. Zuerst die banalen Erklärungen ausschließen](#1-zuerst-die-banalen-erklärungen-ausschließen)
  - [2. Modulwiderstand aus den Logs schätzen](#2-modulwiderstand-aus-den-logs-schätzen)
  - [3. Der Stromrichtungs-Test](#3-der-stromrichtungs-test)
  - [4. Der Dauerlasttest — der Test, auf den es ankommt](#4-der-dauerlasttest--der-test-auf-den-es-ankommt)
  - [5. Der Erholungstest](#5-der-erholungstest)
- [Der passive Balancer macht es schlimmer](#der-passive-balancer-macht-es-schlimmer)
- [Was du ehrlich behaupten kannst](#was-du-ehrlich-behaupten-kannst)
- [Reichweite und Grenzen](#reichweite-und-grenzen--bitte-lesen)
- [Nutzung mit Claude](#nutzung-mit-claude)

---

## Warum das schwierig ist

LiFePO4 hat eine berüchtigt flache Entladekurve. Zwischen etwa 20 % und 90 %
Ladezustand bewegt sich die Zellspannung kaum. Genau diese Flachheit ist das ganze
Problem — und, sobald man sie versteht, die ganze Lösung.

**Im mittleren SOC sind reale Unterschiede zwischen Zellen unsichtbar.** Zwei Zellen,
deren Ladung sich um mehrere Prozent unterscheidet, liegen spannungsmäßig im
Millivoltbereich beieinander. Die Kurve ist zu flach, um es zu zeigen.

**An den Enden werden winzige Unterschiede verstärkt.** Nahe voll oder leer ist die
Kurve steil. Eine Zelle, die sich kaum von ihren Nachbarn unterscheidet, liest plötzlich
30 mV daneben. Deshalb geraten Leute bei 100 % SOC über „Zell-Imbalance" in Panik,
obwohl nichts ist.

**Unter Strom addiert der Innenwiderstand eine Spreizung, die nichts mit dem
Ladezustand zu tun hat.** Eine Zelle mit leicht erhöhtem Innenwiderstand sackt unter
Last ab und steigt beim Laden — rein aus `U = I·R`, unabhängig davon, wie voll sie ist.

Zusammengenommen ergibt das die Falle: **Dieselbe Zelle kann als die schlechteste im
Pack, als die beste, oder als völlig unauffällig erscheinen — je nachdem, wann man
zufällig hinschaut.**

Ich bin selbst hineingetappt. Mein erster Blick auf die Daten sprach die verdächtige
Zelle komplett frei. Mein zweiter beschuldigte die Verkabelung. Mein dritter einen
anderen Mechanismus. Erst durch Tests über verschiedene *Zustände* kam die richtige
Antwort heraus. Jede Diagnose aus einem einzelnen Screenshot ist wertlos.

---

## Die Kernerkenntnis

Es gibt drei verschiedene Gründe, warum eine Zelle abweichen kann — und **sie haben
unterschiedliche Signaturen je nach Stromrichtung:**

| | Entladen | Ruhezustand (0 A) | Laden |
|---|---|---|---|
| **Erhöhter Innenwiderstand** | liest **niedrig** | liest **normal** | liest **hoch** |
| **Weniger Ladung / Kapazitätsverlust** | niedrig | **niedrig** | niedrig |
| **Diffusionslimitierung** | sackt **immer weiter** ab | erholt sich langsam und unvollständig | steigt progressiv |

Der Ruhezustand ist das Unterscheidungsmerkmal. Eine Zelle, die bei *null Strom völlig
normal* ist, aber unter Last abweicht, hat kein Ladungsproblem. Sie hat ein
**Widerstandsproblem**.

Das ist die nützlichste Einzelerkenntnis in diesem Dokument. Der Test kostet nichts:
eine Messung unter Last, eine beim Laden, eine im Ruhezustand — alle im mittleren SOC.

---

## Wann eine Messung gültig ist

Der meiste Aufwand hier liegt nicht im Messen, sondern darin zu wissen, wann man dem
Gemessenen *nicht* trauen darf.

| Situation | Brauchbar für | Nicht brauchbar für |
|---|---|---|
| Mittlerer SOC (30–80 %), hoher Strom | **Innenwiderstand** | — |
| Mittlerer SOC, kein Strom, eingeschwungen | **Echten Ladezustand** | — |
| SOC über ~90 % | *nichts Verlässliches* | Kurve ist steil, alles verfälscht |
| Balancer aktiv (Zellen >3,45 V) | *nichts* | Der Balancer verfälscht die Anzeige der Zelle, die er gerade ableitet |
| Innerhalb ~2 min nach Laständerung | *nichts* | Zellen sind noch nicht eingeschwungen |
| Kleiner Strom (<5 A pro Pack) | Ladezustand | Widerstand — das Signal liegt an der 1-mV-Auflösung des BMS |

Die letzte Zeile hat mich mehrfach reingelegt. Bei 4 A pro Pack sah ich 4–6 mV
Spreizung und rechnete Widerstandswerte aus, die zwischen 0,7 und 1,5 mΩ streuten. Die
Zahlen waren Rauschen. Bei 13 A war dieselbe Messung sauber.

**Faustregel: Widerstand mit so viel Strom messen, wie das System hergibt, in der Mitte
des SOC-Bereichs, und niemals während der Balancer läuft.**

---

## Die Daten erheben

Du brauchst zwei verschiedene Dinge, und ein Tool allein liefert dir nicht beides.

### Das CSV liefert nur die Pack-Ebene

MultiSIBControl (siehe [multisibcontrol.net](https://www.multisibcontrol.net) — die
Einrichtung ist dort dokumentiert und wird hier nicht wiederholt) kann deinen Speicher
loggen und CSV exportieren. Dieser Export enthält pro Pack: Spannung, Strom, Temperatur,
SOC.

Das reicht für **Test 2** (Modulwiderstand) und um den Langzeittrend zu verfolgen. Für
alles andere reicht es *nicht*, denn —

### Das CSV enthält **keine** Zellspannungen

Die einzelnen Zellspannungen erscheinen nur in der Live-Monitoring-Ansicht, nicht im
Export. Und in den Zellspannungen steckt die gesamte Diagnose.

Also: **Du musst Screenshots machen.** Daran führt kein Weg vorbei.

### Was screenshotten, und wann

Nicht ein Screenshot pro Test. Eine *Serie*.

| Test | Screenshot-Takt |
|---|---|
| Stromrichtungs-Test (Test 3) | Einer unter Entladung, einer im Ruhezustand, einer beim Laden |
| Dauerlasttest (Test 4) | **Alle 10–15 Minuten, über die ganze Stunde** |
| Erholungstest (Test 5) | Bei 1, 10, 30, 60, 90 min nach Lastende — und nochmal nach dem Wiederaufladen |

Gerade der Dauerlasttest ist mit zwei Datenpunkten wertlos. Der ganze Befund ist die
*Form* der Kurve, und eine Form siehst du nur, wenn du sie abtastest.

### Notiere zu jedem Screenshot den Kontext

Eine Zellspannung für sich bedeutet nichts. Dieselben 3,24 V sind im Ruhezustand
unauffällig und unter 13 A alarmierend. Halte für jede Aufnahme fest:

- **Strom** (pro Pack — nicht nur die Summe)
- **SOC der Referenz-Packs** (nicht des Verdachts-Packs, dessen SOC-Anzeige ist verzerrt)
- **Wie lange** der Strom in dieser Höhe schon fließt
- **Zelltemperaturen** (Wärme senkt den Innenwiderstand und verdeckt den Effekt)

Wenn deine Screenshots die Tabelle mit Pack-Strom, SOC und Temperaturen neben den
Zellspannungen zeigen — wie es die Monitoring-Ansicht von MultiSIBControl tut — ist das
alles automatisch mit erfasst. Das ist der Hauptgrund, es zu benutzen.

---

## Die Tests

### 1. Zuerst die banalen Erklärungen ausschließen

Bevor du eine schlechte Zelle jagst, prüfe zwei Dinge.

**Erreicht der Speicher überhaupt jemals wirklich 100 %?** Wenn dein System selten alle
Packs auf echte 100 % lädt, werden die Coulomb-Zähler nie zurückgesetzt, und die Packs
driften im gemeldeten SOC aus völlig harmlosen Gründen auseinander. In meinen Logs waren
nur 2,5 % der Zeit alle vier Packs gleichzeitig über 99 %. Das allein erklärt einen
Großteil des scheinbaren „Drifts".

**Ist es nur eine schlechte Klemme?** Ein hochohmiges Kabel oder ein schlechter Kontakt
wirkt symmetrisch — er begrenzt den Strom in beide Richtungen gleich — und erzeugt Wärme
(`P = I²R`). Prüfe auf lastkorrelierte Erwärmung, aber prüfe *richtig*: Thermische
Zeitkonstanten liegen bei Minuten bis Zehnminuten. Wer die Momentantemperatur gegen den
Momentanstrom korreliert, findet nichts — selbst wenn ein Hotspot existiert. Glätte die
Verlustleistung über ein realistisches thermisches Fenster, oder schau, ob sich der Pack
*relativ zu seinen Nachbarn* über eine anhaltende Hochlastphase erwärmt.

In meinem Fall war dieser Test negativ — keine lastkorrelierte Erwärmung — und genau das
verschob die Diagnose schließlich von der Verkabelung in die Zellen.

### 2. Modulwiderstand aus den Logs schätzen

Wenn dein Monitoring-Tool CSV exportiert (MultiSIBControl tut das), kannst du den
effektiven Widerstand jedes Packs ohne Spezialausrüstung schätzen.

Innerhalb eines schmalen SOC-Bands ist die Leerlaufspannung annähernd konstant. Also:
Trag innerhalb dieses Bands die Klemmenspannung jedes Packs gegen seinen eigenen Strom
auf — die Steigung ist sein Widerstand.

```python
slopes = []
for lo in range(30, 100, 5):
    m = (soc >= lo) & (soc < lo + 5)
    if m.sum() > 200 and (I[m].max() - I[m].min()) > 15:  # braucht Stromspreizung
        slope, _ = np.polyfit(I[m], V[m], 1)
        slopes.append(slope)
R_eff = np.average(slopes)   # Ohm
```

**Traue dem Absolutwert nicht.** Die Spannungsauflösung des BMS ist grob, und innerhalb
jedes Bands bewegt sich der SOC noch etwas. Meine Absolutwerte lagen je nach Datensatz
zwischen 60 und 104 mΩ — die Streuung ist methodisch, nicht physikalisch.

**Traue dem relativen Vergleich und dem Trend.** Vier baugleiche Packs, dieselbe
Messung, dieselben Bedingungen: Wenn einer davon heraussticht, ist das real.

Das kam bei meinem Speicher über elf Monate heraus:

| | Aug 2025 | Mai 2026 | Jul (früh) | Jul (spät) |
|---|---|---|---|---|
| **P4 Mehrwiderstand ggü. den anderen drei** | **−4 %** | **+12 %** | **+24 %** | **+27 %** |

Pack 4 war anfangs *besser* als der Durchschnitt. Das ist der entscheidende Teil. Ein
systematischer Methodenfehler hätte P4 von Anfang an erhöht gezeigt. Stattdessen ging er
monoton von negativ ins deutlich Positive. Das ist eine reale, fortschreitende
Veränderung.

### 3. Der Stromrichtungs-Test

Das ist der billigste und aussagekräftigste Test der ganzen Methode.

Nimm drei Messungen der Zellspannungen, alle im mittleren SOC:
- Unter spürbarer Entladung (>10 A pro Pack)
- Im Ruhezustand, fünf Minuten nachdem die Last endet
- Unter spürbarer Ladung (>10 A pro Pack)

Eine Zelle mit erhöhtem Innenwiderstand **kehrt ihr Vorzeichen um**. Bei meinem
Speicher, Zelle 13 in Pack 4:

| Zustand | Zelle 13 | Position im Pack |
|---|---|---|
| Entladung, 13 A | 3,245 V | **niedrigste** |
| Ruhezustand | 3,321 V | nicht vom Rest zu unterscheiden |
| Ladung, 9–21 A | 3,349 V | **höchste** |

Dieselbe Zelle. Kehrt sich mit der Stromrichtung komplett um. Kommt bei null Strom auf
den Pack-Durchschnitt zurück. Das ist eine Lehrbuch-Widerstandssignatur und kann nichts
anderes sein.

Die anderen drei Packs lagen dabei in jedem Zustand bei 1–2 mV Spreizung.

### 4. Der Dauerlasttest — der Test, auf den es ankommt

Alles oben findet einen Widerstandsunterschied. Dieser Test sagt dir, ob dieser
Unterschied *harmlose Streuung* oder *echte Degradation* ist — und fast niemand führt ihn
durch.

**Das Prinzip:** Ohmscher Widerstand ist zeitunabhängig. `U = I·R` interessiert sich
nicht dafür, wie lange du schon Strom ziehst. Wenn die Spannungslücke einer Zelle von
schlichtem Widerstand kommt, bleibt sie bei konstantem Strom konstant — beliebig lange.

Diffusionslimitierung ist anders. Wenn der Ionentransport in einer Zelle nicht mehr
hinterherkommt, baut sich über Minuten ein Konzentrationsgradient an der Elektrode auf,
und der Spannungseinbruch **wächst** — bei konstantem Strom.

**Durchführung:** Starte im mittleren SOC (~60 %), lege so viel konstante Entladelast an,
wie dein System zulässt, und erfasse die Zellspannungen alle 10–15 Minuten über
mindestens eine Stunde. Verfolge den Abstand zwischen deiner Verdachtszelle und der
höchsten Zelle im selben Pack.

Das passierte bei meinem Speicher, bei etwa konstantem Strom:

| Dauer | Strom durch P4 | Abstand Zelle 13 | Rechn. Widerstand |
|---|---|---|---|
| 2 min | 13,5 A | 12 mV | 0,9 mΩ |
| 17 min | 13,2 A | 19 mV | 1,4 mΩ |
| 23 min | 13,0 A | 22 mV | 1,7 mΩ |
| 37 min | 12,2 A | 29 mV | 2,4 mΩ |
| 42 min | 11,9 A | 34 mV | 2,9 mΩ |
| 55 min | 11,6 A | 40 mV | 3,4 mΩ |
| 61 min | 11,6 A | 44 mV | 3,8 mΩ |
| 66 min | 11,1 A | 46 mV | 4,1 mΩ |

Lies das genau. **Der Strom ging um 18 % *runter*. Die Spannungslücke ging um 283 %
*hoch*.** Nach dem Ohmschen Gesetz hätte die Lücke schrumpfen müssen. Sie hat sich
stattdessen fast vervierfacht.

Das ist Diffusionslimitierung. Es ist ein echter Alterungsmechanismus, und du wirst ihn
in keiner Momentaufnahme sehen — nach zwei Minuten sah diese Zelle nach mildem
Mehrwiderstand aus, nichts Alarmierendes. Nach einer Stunde war es unverkennbar.

**Wichtige Ehrlichkeit:** Die anderen drei Packs driften ebenfalls, nur weit weniger. P1
ging von 1 auf 3 mV, P2 von 2 auf 5 mV, P3 von 2 auf 6 mV über dieselbe Stunde. Eine
kleine progressive Spreizung unter Dauerlast ist *normal* — sie tritt auch bei gesunden
Zellen auf. Worauf es ankommt, ist das Ausmaß. Die Verdachtszelle in Pack 4 bewegte sich
um 34 mV, wo ihre gesunden Gegenstücke sich um 2–4 mV bewegten. Das ist ein Faktor 8–16,
kein binäres gesund/kaputt. Formuliere es auch so.

**Brich den Test ab**, bevor deine Referenz-Packs unter ~30–35 % SOC rutschen, sonst
verlässt du den flachen Bereich und der Vergleich wird unsauber.

### 5. Der Erholungstest

Nach der Last: Strom weg und beobachten. Kommt die Verdachtszelle zurück?

| Zeit nach Lastende | Abstand Zelle 13 |
|---|---|
| unter Last | 46 mV |
| 1 min | 35 mV |
| 10 min | 22 mV |
| 18 min | 21 mV |
| 50 min | 17 mV |
| ~90 min | 15 mV |

Vor dem Test lag die Spreizung dieses Packs im Ruhezustand bei **2 mV**. Neunzig Minuten
danach war sie immer noch bei 15 mV und bewegte sich kaum.

An dieser Stelle schloss ich auf ein bleibendes Kapazitätsdefizit. **Das war falsch.**
Nach dem Wiederaufladen in den mittleren SOC-Bereich war die Spreizung vollständig
verschwunden: im Ruhezustand bei 61–63 % SOC lag der Pack wieder bei **2 mV** — identisch
mit dem Zustand vor dem Test und mit den gesunden Packs.

Die Erholung ist also *vollständig*, nur extrem langsam. Die Zelle hat eine stark
gedehnte Erholungs-Zeitkonstante, verliert aber keine Ladung dauerhaft. Das ist ein
deutlich milderer Befund als es zunächst aussah, und er gehört ins Protokoll.

**Die Lehre:** Ein Erholungstest, der nach 90 Minuten abbricht, kann wie ein
Kapazitätsdefizit aussehen, obwohl es nur langsame Erholung ist. Fahre einen vollen
Ladezyklus, bevor du etwas schlussfolgerst.

**Eine Einschränkung, die du berücksichtigen musst.** In einem Parallelverbund tauschen
die Packs nach Lastende leise kleine Ausgleichsströme aus (ich habe ~0,3–0,5 A gemessen),
während sich ihre Ruhespannungen angleichen. Das verwässert eine naive Erholungsmessung
tatsächlich.

Aber es entwertet die wichtige Zahl nicht. Dieser Ausgleichsstrom fließt durch alle 15
Zellen eines Packs in Serie und hebt sie ungefähr gleichmäßig an. Er verschiebt das
*absolute* Niveau des Packs gegenüber den anderen Packs — aber er berührt kaum die
**Spreizung zwischen den Zellen innerhalb dieses Packs**, und genau die misst du.
Verfolge die pack-interne Spreizung, nicht die Spannung Pack-gegen-Pack, und der Test
hält stand.

(Fairerweise: Ich hatte das zunächst übersehen und musste darauf gestoßen werden.)

---

## Der passive Balancer macht es schlimmer

Das ist der Teil, der mich am meisten überrascht hat, und er hat echte Konsequenzen.

Pylontech — und im Grunde jedes LFP-BMS dieser Klasse — nutzt **passives** Balancing. Es
kann Ladung aus einer hohen Zelle über einen Widerstand ableiten (rund 50 mA). Es kann
**keine** Ladung in eine niedrige Zelle schieben. Dafür gibt es keinen Mechanismus.

Nun überlege, was eine Zelle mit hohem Innenwiderstand beim Laden tut. `U = I·R` treibt
ihre Klemmenspannung **über** die ihrer Nachbarn. Sie erreicht die Balancing-Schwelle
(~3,45 V) also **zuerst** — obwohl sie in Wahrheit die *leerste* Zelle im Pack ist.

Das BMS sieht eine hohe Spannung und tut das Einzige, was es kann: **Es leitet Ladung
aus der schwächsten Zelle des Packs ab.**

Das schließt einen Kreis:

```
höherer Widerstand
    → liest beim Laden künstlich hoch
        → Balancer leitet Ladung ab
            → verliert tatsächlich Ladung
                → sackt unter Last stärker ab
                    → gemessener Widerstand steigt
                        → (wiederholt sich, jeden Zyklus)
```

Dieser Mechanismus erklärt die elfmonatige Beschleunigung in meinen Daten besser als
Alterung allein. Und er ist *nicht* etwas, das der Anwender verursacht oder abschalten
kann. Es ist das Verhalten des Produkts.

**Das diagnostische Merkmal:** In der Absorptionsphase liest die Verdachtszelle als die
**niedrigste** in ihrem Pack — während sie unter Ladestrom, Minuten zuvor, die
**höchste** war. Wenn du diese Umkehr siehst, hat der Balancer gegen diese Zelle
gearbeitet.

Bei meinem Speicher las Zelle 13 während der Absorption bei 99 % SOC 3,485 V gegen einen
Pack-Durchschnitt von rund 3,50 V. Niedrigste im Pack. Sie war bereits heruntergezogen
worden.

**Praktische Konsequenz:** Erwarte nicht, dass der Balancer eine widerstandsdegradierte
Zelle wieder hochbringt. Er tut das Gegenteil. Den Ladestrom zu begrenzen (über DVCC oder
äquivalent) reduziert die `I·R`-Verzerrung und lässt den Balancer auf etwas reagieren,
das dem echten Zellzustand näher kommt — das ist eine echte Milderung, behandelt aber das
Symptom.

---

## Was du ehrlich behaupten kannst

Sei vorsichtig mit dem, was du behauptest.

**Gut belegt:**
- Der Vergleich baugleicher Packs untereinander, unter identischen Bedingungen
- Der Trend über die Zeit — das ist das stärkste Beweismittel, das du haben wirst
- Direkt gemessene Zellspannungen
- Die Stromrichtungs-Signatur
- Der Verlauf im Dauerlasttest

**Nicht gut belegt:**
- Absolute Widerstandswerte aus BMS-Telemetrie — nur als Größenordnung behandeln
- Jede Extrapolation auf die Restlebensdauer
- Ein Modul als „defekt" zu bezeichnen, wenn es keine Fehler wirft und die Nennkapazität
  liefert

Die Behauptung, die wirklich hält, ist vergleichend: *Dieses Modul degradiert messbar
schneller als baugleiche Module unter identischen Bedingungen, und der Mechanismus ist
identifiziert.* Das ist vertretbar. „Es ist kaputt" ist es nicht, solange es funktioniert.

In meinem Fall funktioniert der Speicher weiterhin einwandfrei. Keine Fehler, keine
Schutzabschaltungen, und weil er im Normalbetrieb nie in den unteren SOC-Bereich
hinunterfährt, ist der Kapazitätsverlust im Alltag nicht einmal spürbar. Das Problem ist
die *Entwicklung*, nicht der aktuelle Zustand.

---

## Reichweite und Grenzen — bitte lesen

Diese Methode wurde an **einem** Speicher entwickelt. Vier Pylontech-US3000C-Module, ein
Wechselrichter, ein Haushalts-Lastprofil, elf Monate Daten. Das reicht, um die Methode zu
etablieren, aber nicht, um die konkreten Zahlen als verbindlich zu erklären.

Was meiner Einschätzung nach verallgemeinerbar ist:
- Das Drei-Signaturen-Schema (Widerstand / Kapazität / Diffusion)
- Der Stromrichtungs-Test
- Die Notwendigkeit, unter *anhaltender* Last zu testen, nicht in Momentaufnahmen
- Die Gültigkeitsfenster (flacher Bereich, hoher Strom, Balancer aus)
- Die Balancer-Falle — sie folgt daraus, wie passives Balancing funktioniert, und wird
  durch die Umkehr in der Absorptionsphase bestätigt. Ich habe sie aber **nicht** gegen
  Pylontechs tatsächliche Firmware verifiziert. Es ist mechanistisch schlüssige
  Schlussfolgerung, keine Datenblatt-Tatsache.

Wo ich mehr Fälle sehen wollte, bevor ich es für gesichert halte:
- Konkrete mΩ-Schwellen für „bedenklich"
- Wie schnell Diffusionslimitierung typischerweise fortschreitet
- Ob dieselben Signaturen bei anderen BMS-Familien auftreten

Wenn du das an deinem eigenen Speicher durchführst, werden deine Zahlen abweichen. Die
*Formen* sollten es nicht.

---

## Nutzung mit Claude

`SKILL.md` in diesem Repo ist für Claude geschrieben, nicht für dich. Es kodiert den
Entscheidungsbaum, die Gültigkeitsfenster und die Fallen — damit das Modell sie nicht
erst durch sechsmaliges Danebenliegen wiederentdecken muss.

(Was, der Vollständigkeit halber, genau die Art ist, wie diese Methode entstanden ist.)

**So gehst du vor:**

1. Lade `SKILL.md` aus diesem Repo herunter.
2. Starte ein Gespräch mit Claude und hänge die Datei an, zusammen mit deinen Daten.
3. Schreib etwas wie: *„Nutze den angehängten Skill, um meine Batteriedaten zu
   analysieren."*

Dann übergib ihm, was du gesammelt hast:

- **Deinen CSV-Export** — Claude kann die Widerstands-Regression direkt darauf rechnen
  und dir den Pack-Vergleich sowie den Trend ausgeben.
- **Deine Screenshots** — schick sie ihm, während du sie machst. Claude liest die
  Zellspannungs-Tabellen direkt aus dem Bild.

Die Screenshots sind der Teil, den die meisten überspringen — und der Teil, auf den es
ankommt. Schick sie während eines Tests **laufend** weiter. Sag Claude dazu, wie hoch der
Strom ist, wo der SOC steht und wie lange die Last schon läuft. Ohne diesen Kontext kann
er nicht interpretieren, was er sieht — und rät dann (schlecht).

Wenn du historische Logs von vor Monaten hast, gib sie ebenfalls dazu. **Der Trend über
die Zeit ist das stärkste Beweismittel, das du erzeugen wirst.** Ein einzelner Tag zeigt
dir, *dass* etwas nicht stimmt. Nur der Trend zeigt dir, dass es schlimmer wird.

---

*Daten von einem 4× Pylontech US3000C Speicher an einem Victron MultiPlus-II 3000,
~475 Zyklen. Seriennummern und Netzwerkdetails entfernt. Zeitangaben relativ.*
