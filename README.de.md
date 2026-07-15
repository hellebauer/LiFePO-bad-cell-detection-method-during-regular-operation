# Eine defekte Zelle in einem parallelen LiFePO4-Verbund finden

Eine Methode zur Diagnose der Degradation einzelner Zellen in parallelen
LiFePO4-Batterieverbünden (Pylontech US2000/US3000/US5000 und ähnliche) – mit nichts weiter
als der Telemetrie, die das BMS ohnehin schon liefert.

Das hier existiert, weil die Information sonst nirgends zu finden ist. Pylontech
veröffentlicht keine Diagnoseprozedur. In den Foren kursiert Halbwissen – „deine Zellen
driften, lade häufiger auf 100 %" – aber keine Methode, um herauszufinden, *was tatsächlich
nicht stimmt*, und eine wirklich ausfallende Zelle von normaler Fertigungsstreuung zu
unterscheiden.

Erarbeitet wurde das an einem realen Verbund: 4× US3000C an einem Victron MultiPlus-II 3000,
~475 Zyklen, auf Basis von ~11 Monaten protokollierter Daten plus gezielter Tests. Alle
Zahlen unten stammen von diesem Verbund.

---

## Inhalt

- [Was das Ganze schwierig macht](#was-das-ganze-schwierig-macht)
- [Die Kernidee](#die-kernidee)
- [Wann deine Messung gültig ist](#wann-deine-messung-gültig-ist)
- [An die Daten kommen](#an-die-daten-kommen)
- [Die Tests](#die-tests)
  - [1. Zuerst die langweiligen Erklärungen ausschließen](#1-zuerst-die-langweiligen-erklärungen-ausschließen)
  - [2. Pack-Widerstand aus den Logs schätzen](#2-pack-widerstand-aus-den-logs-schätzen)
  - [3. Der Stromumkehr-Test](#3-der-stromumkehr-test)
  - [4. Der Dauerlast-Test – der, auf den es wirklich ankommt](#4-der-dauerlast-test--der-auf-den-es-wirklich-ankommt)
  - [5. Der Erholungstest](#5-der-erholungstest)
  - [6. Wird es schlimmer? Der Langzeittest](#6-wird-es-schlimmer-der-langzeittest)
- [Der passive Balancer macht es schlimmer](#der-passive-balancer-macht-es-schlimmer)
- [Was du redlich schlussfolgern kannst](#was-du-redlich-schlussfolgern-kannst)
- [Geltungsbereich und Grenzen](#geltungsbereich-und-grenzen--bitte-lesen)
- [Wie man das mit Claude nutzt](#wie-man-das-mit-claude-nutzt)

---

## Was das Ganze schwierig macht

LiFePO4 hat eine berüchtigt flache Entladekurve. Zwischen etwa 20 % und 90 % Ladezustand
bewegt sich die Zellspannung kaum. Diese Flachheit ist das ganze Problem – und, sobald man
sie versteht, die ganze Lösung.

**Bei mittlerem SOC sind echte Unterschiede zwischen Zellen unsichtbar.** Zwei Zellen, deren
Ladezustände um mehrere Prozent auseinanderliegen, zeigen Werte innerhalb eines Millivolts.
Die Kurve ist zu flach, um es zu zeigen.

**An den Enden werden winzige Unterschiede verstärkt.** Nahe voll oder leer ist die Kurve
steil. Eine Zelle, die sich kaum von ihren Nachbarn unterscheidet, weicht plötzlich um 30 mV
ab. Deshalb geraten Leute bei 100 % SOC über „Zell-Imbalance" in Panik, obwohl nichts falsch
ist.

**Unter Strom fügt der Widerstand eine Spreizung hinzu, die nichts mit dem Ladezustand zu tun
hat.** Eine Zelle mit etwas höherem Innenwiderstand sackt unter Last ab und steigt beim Laden
– rein aus `U = I·R`, unabhängig davon, wie voll sie ist.

Zusammengenommen ergibt das die Falle: **Dieselbe Zelle kann als die schlechteste im Pack,
als die beste im Pack oder als völlig gewöhnlich erscheinen – abhängig einzig davon, wann man
zufällig hinschaut.**

Ich habe das selbst durchgemacht. Meine erste Lesart der Daten sprach die verdächtige Zelle
komplett frei. Meine zweite gab der Verkabelung die Schuld. Meine dritte einem anderen
Mechanismus. Erst durch Testen über verschiedene *Zustände* hinweg kam die wahre Antwort
zum Vorschein. Jede Diagnose aus einem einzigen Screenshot ist wertlos.

---

## Die Kernidee

Es gibt drei unterschiedliche Gründe, aus denen eine Zelle abweichen kann, und **sie haben
unterschiedliche Signaturen je nach Stromrichtung:**

| | Entladen | In Ruhe (0 A) | Laden |
|---|---|---|---|
| **Hoher Innenwiderstand** | zeigt **niedrig** | zeigt **normal** | zeigt **hoch** |
| **Wenig Ladung / verlorene Kapazität** | zeigt niedrig | zeigt **niedrig** | zeigt niedrig |
| **Diffusionslimitierung** | fällt mit der Zeit **immer weiter** | erholt sich langsam und unvollständig | steigt fortschreitend |

Die Ruhemessung ist das Unterscheidungsmerkmal. Eine Zelle, die *bei null Strom völlig normal*
ist, aber unter Last abweicht, hat kein Ladungsproblem. Sie hat ein **Widerstandsproblem**.

Das ist das mit Abstand Nützlichste in diesem Dokument. Es kostet nichts, es zu testen: eine
Messung unter Last, eine beim Laden, eine in Ruhe, alle bei mittlerem SOC.

---

## Wann deine Messung gültig ist

Der meiste Aufwand hier liegt nicht im Messen – sondern im Wissen, wann man dem Gemessenen
*nicht* trauen darf.

| Situation | Verwendbar für | Nicht verwendbar für |
|---|---|---|
| Mittlerer SOC (30–80 %), hoher Strom | **Widerstand** | — |
| Mittlerer SOC, null Strom, eingeschwungen | **Echter Ladezustand** | — |
| SOC über ~90 % | *nichts Verlässliches* | Kurve ist steil; alles ist verfälscht |
| Balancer aktiv (Zellen >3,45 V) | *nichts* | Der Balancer verzerrt die Messung der Zelle, die er gerade entlädt |
| Innerhalb ~2 min nach einer Laständerung | *nichts* | Zellen sind noch nicht eingeschwungen |
| Kleiner Strom (<5 A pro Pack) | Ladezustand | Widerstand – das Signal liegt bei der 1-mV-Auflösung des BMS |
| Ein Dauerlast-Test bei kleinem Strom | *nichts* | **Liefert einen falschen Negativbefund – siehe Test 4** |
| „In Ruhe", aber weniger als eine Stunde seit Stromende | *wenig* | Eine degradierte Zelle relaxiert womöglich noch – siehe unten |

Die letzte Zeile hat mich wiederholt gebissen. Bei 4 A pro Pack sah ich Spreizungen von
4–6 mV und errechnete Widerstandswerte, die zwischen 0,7 und 1,5 mΩ streuten. Die Zahlen
waren Rauschen. Bei 13 A war dieselbe Messung sauber.

**Faustregel: Miss den Widerstand unter so viel Strom, wie dein System hergibt, in der Mitte
des SOC-Bereichs, und niemals während der Balancer aktiv ist. Miss den Ladezustand bei null
Strom, in der Mitte des Bereichs, sobald sich wirklich alles eingeschwungen hat.**

### „In Ruhe" ist nicht dasselbe wie „eingeschwungen"

Das hatte ich falsch, und es lohnt sich, das ausdrücklich zu sagen.

Eine degradierte Zelle kann eine übel gedehnte Relaxationszeitkonstante haben – das ist Teil
dessen, was mit ihr nicht stimmt. „Ruhe" ist also kein Zustand, den man fünf Minuten nach
Lastende erreicht.

Sieh dir das an. Die ganze Zeit null Strom, Verbund steht im Leerlauf bei 100 %:

| Zeit | Zelle 13 | Höchste Zelle im Pack | Spreizung |
|---|---|---|---|
| 16:01 | 3,433 V | 3,459 V | 26 mV |
| 17:06 | **3,383 V** | 3,411 V | **28 mV** |

Fünfundsechzig Minuten, kein Strom, und die Zelle fällt *immer noch*. Ihre Nachbarn stehen
stabil. Sie relaxiert zu ihrem wahren (niedrigeren) Ladezustand hin, und sie lässt sich dabei
Zeit.

**Also: Nimm eine Reihe von Ruhemessungen, nicht eine.** Wenn sich der Wert bei deiner letzten
Messung noch bewegt, sag das – und extrapoliere, statt so zu tun, als sei der letzte Punkt die
Antwort. Die gedehnte Zeitkonstante ist selbst schon ein Befund.

---

## An die Daten kommen

Du brauchst zwei verschiedene Dinge, und ein einzelnes Werkzeug liefert dir nicht beide.

### Die CSV liefert nur Daten auf Pack-Ebene

MultiSIBControl (siehe [multisibcontrol.net](https://www.multisibcontrol.net) – die
Einrichtung ist dort dokumentiert und wird hier nicht wiederholt) kann deinen Verbund
protokollieren und als CSV exportieren. Dieser Export enthält pro Pack: Spannung, Strom,
Temperatur, SOC.

Das reicht für **Test 2** (Pack-Widerstand) und zum Verfolgen des Langzeittrends. Für alles
andere reicht es *nicht*, denn –

### Die CSV enthält **keine** Zellspannungen

Einzelne Zellspannungen erscheinen nur in der Live-Monitoring-Ansicht, nicht im Export. Und
in den Zellspannungen steckt die gesamte Diagnose.

Also: **Du musst Screenshots machen.** Daran führt kein Weg vorbei.

### Was screenshoten, und wann

Nicht einen Screenshot pro Test. Eine *Reihe*.

| Test | Screenshot-Takt |
|---|---|
| Stromumkehr (Test 3) | Einen beim Entladen, einen in Ruhe, einen beim Laden |
| Dauerlast (Test 4) | **Alle 10–15 Minuten für die ganze Stunde** |
| Erholung (Test 5) | Bei 1, 10, 30, 60, 90 min nach Lastende – dann noch einmal nach einer Nachladung |

Gerade der Dauerlast-Test ist mit zwei Datenpunkten wertlos. Der ganze Befund ist die *Form*
der Kurve, und eine Form siehst du nur, wenn du sie abtastest.

### Der Zeitstempel ist es, der die beiden Hälften verbindet

Das ist der Punkt, den du aus diesem Abschnitt am ehesten mitnehmen solltest.

Eine Zellspannung für sich genommen bedeutet nichts. Dieselben 3,24 V sind in Ruhe unauffällig
und unter 13 A alarmierend. Und dieselben 13 A bedeuten zwei Minuten nach Entladebeginn etwas
anderes als nach fünfzig Minuten. Jeder Screenshot braucht also seinen **Lastkontext**: Strom,
SOC, wie lange der Strom schon fließt, wie viel Ladung durchgesetzt wurde, wie lange seit
Stromende.

Du könntest das alles von Hand aufschreiben. Tu es nicht. **Bring die Uhr ins Bild.**

Wenn die Taskleisten-Uhr im Screenshot sichtbar ist, dann hat der Screenshot einen
Zeitstempel; und wenn du zusätzlich die CSV für diesen Zeitraum hast, dann lässt sich jeder
Kontext von oben *herleiten*, indem man die beiden abgleicht. Phase, Dauer, Durchsatz, Ruhezeit
– alles davon, berechnet, nicht erinnert.

Genau das macht das Paar aus Quellen mehr als die Summe seiner Teile. Die CSV hat keine
Zellspannungen. Der Screenshot hat keine Historie. Der Zeitstempel ist das Scharnier.

**Praktisch: Sorg dafür, dass die Windows-Uhr jedes Mal im Bild ist. Behalte die CSV für
denselben Zeitraum. Alles Weitere ergibt sich daraus.**

(Ein Blick auf die Zelltemperaturen lohnt sich ebenfalls – Wärme senkt den Innenwiderstand
und kann den Effekt maskieren. Die Monitoring-Ansicht von MultiSIBControl zeigt sie neben den
Zellspannungen an, du bekommst sie also gratis.)

### Traue den BMS-eigenen min/max/Imbalance-Zeilen nicht

Die Monitoring-Ansicht zeigt dir `Cell volt (min)`, `Cell volt (max)` und `Cell imbalance`.
Sie zeigt dir außerdem alle fünfzehn Einzelzellen. **Diese stimmen nicht überein.**

Nicht, weil etwas kaputt wäre – die Zusammenfassungs-Register und die Einzelzell-Register
werden schlicht zu leicht unterschiedlichen Zeitpunkten abgefragt. Meist liegt die Abweichung
unter einem Millivolt. Ich habe sie bis auf 12 mV steigen sehen.

Wenn du einem Signal nachjagst, das *selbst* nur ein paar Millivolt beträgt, ist das relevant.

**Berechne die Spreizung selbst, aus den fünfzehn Werten.** Zitiere die Imbalance-Zahl niemals,
als wäre sie eine Messung.

---

## Die Tests

### 1. Zuerst die langweiligen Erklärungen ausschließen

Bevor du auf die Jagd nach einer defekten Zelle gehst, prüfe zwei Dinge.

**Erreicht der Verbund überhaupt jemals wirklich voll?** Wenn dein System nur selten alle Packs
auf echte 100 % lädt, setzen sich die Coulomb-Zähler nie zurück, und die Packs driften aus
völlig harmlosen Gründen im gemeldeten SOC auseinander. In meinen Logs waren nur 2,5 % der Zeit
alle vier Packs gleichzeitig über 99 %. Das allein erklärt schon viel scheinbare „Drift".

**Ist es nur ein schlechter Anschluss?** Ein Kabel oder Kontakt mit hohem Widerstand ist
symmetrisch – er begrenzt den Strom in beiden Richtungen gleich – und erzeugt Wärme
(`P = I²R`). Prüfe auf lastkorrelierte Erwärmung, aber prüfe *richtig*: Thermische
Zeitkonstanten liegen bei Minuten bis Zehnminuten, wer also Momentantemperatur gegen
Momentanstrom korreliert, findet nichts, selbst wenn ein Hotspot existiert. Glätte den
Wärmeterm über ein realistisches thermisches Fenster, oder schau, ob der Pack *relativ zu
seinen Nachbarn* über einen anhaltenden Hochlastblock hinweg wärmer wird.

In meinem Fall fiel dieser Test negativ aus – keine lastkorrelierte Erwärmung – was die
Diagnose schließlich von der Verkabelung in die Zellen hinein verschob.

### 2. Pack-Widerstand aus den Logs schätzen

Wenn dein Monitoring-Tool CSV exportiert (MultiSIBControl tut das), kannst du den effektiven
Widerstand jedes Packs ohne Spezialausrüstung schätzen.

Innerhalb eines schmalen SOC-Bandes ist die Leerlaufspannung annähernd konstant. Trage also
innerhalb dieses Bandes die Klemmenspannung jedes Packs gegen seinen eigenen Strom auf: Die
Steigung ist sein Widerstand.

```python
slopes = []
for lo in range(30, 100, 5):
    m = (soc >= lo) & (soc < lo + 5)
    if m.sum() > 200 and (I[m].max() - I[m].min()) > 15:  # echte Stromspreizung nötig
        slope, _ = np.polyfit(I[m], V[m], 1)
        slopes.append(slope)
R_eff = np.average(slopes)   # Ohm
```

**Traue dem Absolutwert nicht.** Die Spannungsauflösung des BMS ist grob, und innerhalb jedes
Bandes gibt es noch Restbewegung im SOC. Meine Absolutwerte reichten je nach Datensatz von 60
bis 104 mΩ – die Streuung ist methodisch, nicht physikalisch.

**Traue dem relativen Vergleich und dem Trend.** Vier identische Packs, dieselbe Messung,
dieselben Bedingungen: Wenn einer davon heraussticht, ist das real.

Das hier kam an meinem Verbund über elf Monate heraus:

| | Aug 2025 | Mai 2026 | Jul (früh) | Jul (spät) |
|---|---|---|---|---|
| **P4 Mehrwiderstand ggü. den anderen drei** | **−4 %** | **+12 %** | **+24 %** | **+27 %** |

Pack 4 begann *besser* als der Durchschnitt. Genau darauf kommt es an. Ein systematischer
Fehler in der Methode hätte P4 vom ersten Tag an erhöht gezeigt. Stattdessen kreuzte er
monoton von negativ zu deutlich positiv. Das ist eine reale, fortschreitende Veränderung.

**Während du in der CSV bist, prüfe die Durchsatzverteilung.** Jeder Pack sollte 1/N der Arbeit
tragen. Meiner tat das nicht: P4 lieferte nur **86 %** seines Anteils, und P3 schulterte
**108 %**.

Das ist nicht bloß Buchhaltung. Das degradierte Modul wird *geschont*, und die gesunden werden
*härter beansprucht* – also altern sie schneller. Ein schwacher Pack belastet still den Rest
des Verbunds. Gut zu wissen, bevor du entscheidest, ob du ihn drinlässt.

**Etwas, worüber Leute stolpern:** Es gibt keine modulweise Stromregelung. In einem
Parallelverbund entscheidet das BMS nicht, wie viel Strom jedes Modul nimmt. Die Verteilung ist
passiv – `I = (U_bus − OCV) / R` – und das Master-BMS gibt einen einzigen **gemeinsamen**
Stromgrenzwert an den Wechselrichter aus. Ein schwaches Modul nimmt weniger Strom, weil es
schwach *ist*, nicht weil irgendetwas das so festgelegt hätte.

### 3. Der Stromumkehr-Test

Das ist der billigste und aussagekräftigste Test der ganzen Methode.

Nimm drei Messungen der Zellspannungen, alle bei mittlerem SOC:
- Unter nennenswerter Entladung (>10 A pro Pack)
- In Ruhe, fünf Minuten nachdem die Last endet
- Unter nennenswerter Ladung (>10 A pro Pack)

Eine Zelle mit erhöhtem Widerstand **kehrt das Vorzeichen um**. An meinem Verbund, Zelle 13
von Pack 4:

| Zustand | Zelle 13 | Ihre Position im Pack |
|---|---|---|
| Entladen, 13 A | 3,245 V | **niedrigste** |
| In Ruhe | 3,321 V | nicht vom Rest unterscheidbar |
| Laden, 9–21 A | 3,349 V | **höchste** |

Dieselbe Zelle. Kehrt sich mit der Stromrichtung vollständig um. Kehrt zum Pack-Durchschnitt
zurück, wenn kein Strom fließt. Das ist eine lehrbuchreine Widerstandssignatur und kann nichts
anderes sein.

Die anderen drei Packs lagen unterdessen in jedem Zustand bei 1–2 mV Spreizung.

**Und jetzt hör nicht dort auf – miss es.** Der Vorzeichenwechsel sagt dir den *Mechanismus*.
Ihn zu regressieren sagt dir die *Größe* und trennt die beiden Dinge, die in jeder Messung
unter Strom miteinander verwoben sind.

Nimm Messungen über einen Strombereich hinweg, alle im flachen Bereich, und trage den Abstand
der verdächtigen Zelle zur höchsten Zelle in ihrem Pack gegen den Strom dieses Packs auf:

| Strom durch P4 | Abstand Zelle 13 zur Spitzenzelle | Position |
|---|---|---|
| −1,93 A (Entladen) | 15 mV | niedrigste |
| +0,87 A (Laden) | 12 mV | niedrigste |
| +2,94 A | 8 mV | niedrigste |
| +8,52 A | 3 mV | niedrigste |
| +8,84 A | 0 mV | **höchste** |

Es ist eine Gerade, und sie liefert dir drei Zahlen:

- **Die Steigung** → der Mehrwiderstand. Hier ~**1,1 mΩ** über den Nachbarn.
- **Der Achsenabschnitt bei 0 A** → **das echte Ladungsdefizit, mit herausgerechnetem
  I·R-Term.** Hier ~**13 mV**. *Das ist die Zahl, die du eigentlich willst.*
- **Der Nulldurchgang** → der Strom, bei dem die Zelle von niedrigster zu höchster kippt. Hier
  ~**8,8 A**.

Jede einzelne Messung unter Strom vermischt Widerstand und Ladungsdefizit, und du kannst nicht
sagen, wie viel wovon ist. Die Extrapolation auf null Strom ist es, die die beiden auseinander
zieht.

### 4. Der Dauerlast-Test – der, auf den es wirklich ankommt

Alles oben findet einen Widerstandsunterschied. Dieser Test sagt dir, ob dieser Unterschied
*harmlose Streuung* oder *echte Degradation* ist, und es ist der Test, den fast niemand macht.

**Das Prinzip:** Ohmscher Widerstand ist zeitinvariant. `U = I·R` kümmert es nicht, wie lange
du schon Strom ziehst. Wenn der Spannungsabstand einer Zelle durch schlichten Widerstand
verursacht ist, hält dieser Abstand bei konstantem Strom unbegrenzt stabil.

Diffusionslimitierung ist anders. Wenn der interne Ionentransport einer Zelle nicht mithält,
baut sich der Konzentrationsgradient an der Elektrode über Minuten auf, und der
Spannungseinbruch **wächst** – selbst bei konstantem Strom.

**Bevor du ihn durchführst – eine Warnung, die mich den ganzen ersten Versuch gekostet hat.**

Mein erster Dauerlast-Test lief bei etwa 14 A pro Pack. Er fand **nichts**. Der Abstand blieb
flach und unauffällig, und ich schloss, die Zelle sei in Ordnung. Ich lag falsch, und ich blieb
eine Weile im Irrtum.

Bei 56 A Verbundlast gab dieselbe Zelle die Signatur innerhalb von zwanzig Minuten preis.

Diffusionslimitierung zeigt sich nur unter echtem Strom. **Ein unterdimensionierter Test
liefert nicht „gesund" – er liefert nichts, was identisch aussieht mit gesund.** Das ist die
gefährliche Richtung dieses Fehlers: Ein Fehlalarm wird überprüft, eine falsche Entwarnung
beendet die Untersuchung.

**Wenn dein Lasttest sauber zurückkommt, schau auf den Strom, bevor du ihm glaubst.**

**Durchführung:** Starte bei mittlerem SOC (~60 %), lege so viel konstante Entladung an, wie
dein System zulässt – so viel du bekommst – und zeichne die Zellspannungen alle 10–15 Minuten
für mindestens eine Stunde auf. Verfolge den Abstand zwischen deiner verdächtigen Zelle und der
höchsten Zelle im selben Pack.

Das hier passierte an meinem Verbund, bei annähernd konstantem Strom:

| Verstrichen | Strom durch P4 | Abstand Zelle 13 | Impliziter Widerstand |
|---|---|---|---|
| 2 min | 13,5 A | 12 mV | 0,9 mΩ |
| 17 min | 13,2 A | 19 mV | 1,4 mΩ |
| 23 min | 13,0 A | 22 mV | 1,7 mΩ |
| 37 min | 12,2 A | 29 mV | 2,4 mΩ |
| 42 min | 11,9 A | 34 mV | 2,9 mΩ |
| 55 min | 11,6 A | 40 mV | 3,4 mΩ |
| 61 min | 11,6 A | 44 mV | 3,8 mΩ |
| 66 min | 11,1 A | 46 mV | 4,1 mΩ |

Lies das genau. **Der Strom ging um 18 % *zurück*. Der Spannungsabstand ging um 283 % *hoch*.**
Das Ohmsche Gesetz sagt, der Abstand hätte schrumpfen müssen. Stattdessen hat er sich beinahe
vervierfacht.

Das ist Diffusionslimitierung. Es ist ein echter Alterungsmechanismus, und du wirst ihn in
keinem Schnappschuss sehen – nach zwei Minuten sah diese Zelle aus wie ein leicht erhöhter
Widerstand, nichts Beunruhigendes. Nach einer Stunde war sie unverkennbar.

**Wichtige Ehrlichkeit:** Die anderen drei Packs drifteten ebenfalls, nur weit weniger. P1 ging
von 1 auf 3 mV, P2 von 2 auf 5 mV, P3 von 2 auf 6 mV über dieselbe Stunde. Eine kleine
fortschreitende Spreizung unter Dauerlast ist *normal* – sie passiert auch in gesunden Zellen.
Worauf es ankommt, ist die Größenordnung. Die verdächtige Zelle von Pack 4 bewegte sich um
34 mV, wo ihre gesunden Gegenstücke sich um 2–4 mV bewegten. Es ist ein Faktor von 8–16, keine
binäre Unterscheidung gesund/kaputt. Sag es so.

**Beende den Test**, bevor deine Referenzpacks unter ~30–35 % SOC fallen, sonst verlässt du
den flachen Bereich und der Vergleich ist nicht mehr sauber.

### 5. Der Erholungstest

Nach der Last schalte den Strom ab und beobachte. Kommt die verdächtige Zelle zurück?

| Zeit nach Lastende | Abstand Zelle 13 |
|---|---|
| unter Last | 46 mV |
| 1 min | 35 mV |
| 10 min | 22 mV |
| 18 min | 21 mV |
| 50 min | 17 mV |
| ~90 min | 15 mV |

Vor dem Test lag die Spreizung dieses Packs in Ruhe bei **2 mV**. Neunzig Minuten danach lag
sie immer noch bei 15 mV und bewegte sich kaum.

An diesem Punkt schloss ich, die Zelle habe ein bleibendes Kapazitätsdefizit. **Ich lag
falsch.** Nach dem Nachladen zurück in den mittleren SOC-Bereich war die Spreizung vollständig
zusammengefallen: in Ruhe bei 61–63 % SOC war der Pack wieder bei **2 mV** – identisch mit dem
Zustand vor dem Test und mit den gesunden Packs.

Die Erholung ist also *vollständig*, nur extrem langsam. Die Zelle hat eine übel gedehnte
Erholungszeitkonstante, aber sie verliert keine Ladung dauerhaft. Das ist ein deutlich milderer
Befund, als es zunächst schien, und es gehört ins Protokoll.

**Die Lehre:** Ein Erholungstest, der nach 90 Minuten aufhört, kann wie ein Kapazitätsdefizit
aussehen, obwohl es in Wahrheit nur langsame Erholung ist. Gib ihm einen vollen Nachladezyklus,
bevor du irgendetwas schließt.

**Ein Vorbehalt, den du berücksichtigen musst.** In einem Parallelverbund tauschen die Packs,
sobald die Last endet, still kleine Balancing-Ströme aus (ich habe ~0,3–0,5 A gemessen), während
sich ihre Ruhespannungen angleichen. Das *verwässert* eine naive Erholungsmessung tatsächlich.

Aber es macht die wichtige Zahl nicht ungültig. Dieser Balancing-Strom fließt durch alle 15
Zellen eines Packs in Reihe und hebt sie ungefähr gleichmäßig an. Er verschiebt das *absolute*
Niveau des Packs relativ zu den anderen Packs – aber er berührt kaum die **Spreizung zwischen
den Zellen innerhalb dieses Packs**, die du eigentlich misst. Verfolge die Intra-Pack-Spreizung,
nicht die Pack-gegen-Pack-Spannung, und der Test übersteht es.

(Ehre, wem Ehre gebührt: Ich habe das anfangs übersehen und musste darauf gestoßen werden.)

**Heute reversibel heißt nicht für immer reversibel.** Wenn der Test „reversible Polarisation,
kein Kapazitätsdefizit" zurückgibt, beschreibt das die Zelle *jetzt gerade* – es ist keine
dauerhafte Entwarnung. Das zugrunde liegende Problem (steigender Widerstand, der die
Balancer-Schleife speist) schreitet weiter fort, eine Zelle, die diesen Monat vollständig
heilt, kann nächsten Monat einen Rest hinterlassen. Lege den Erholungstest nicht als erledigt
ab. Führe ihn erneut durch, und beobachte eine Zahl:

**Die zu verfolgende Anzeige ist der Nullstrom-Achsenabschnitt.** Nimm die Stromumkehr-Auftragung
aus Test 3 – den Abstand der verdächtigen Zelle gegen den Pack-Strom – und lies ihren Wert
extrapoliert auf 0 A ab. Das rechnet den Widerstandsterm heraus und lässt den echten Ladezustand
der Zelle übrig. Protokolliere ihn bei mittlerem SOC, eingeschwungen, alle paar Wochen. Solange
dieser Achsenabschnitt nach einer vollen Nachladung immer wieder nahe null zurückkehrt, ist die
Kapazität in Ordnung, und du beobachtest eine langsame Widerstandsdrift, mit der du leben kannst.
**Das erste Mal, dass er nach einer ordentlichen Nachladung *nicht* nahe null zurückkehrt, ist
der Tag, an dem der Schaden von reversibel zu permanent wurde** – und weil du ihn protokolliert
hast, siehst du ihn kommen, statt überrascht zu werden. Diese eine Zahl ist deine Prognose. Es
gibt keine redliche Möglichkeit, aus solchen Daten die Restlebensdauer vorherzusagen; also sag
nichts voraus – miss die Anzeige.

### 6. Wird es schlimmer? Der Langzeittest

Alles oben sagt dir, in welchem Zustand die Zelle *ist*. Nichts davon sagt dir, wohin sie
*geht* – und das ist die Frage, die dich eigentlich interessiert, denn mit einem stabilen
Defekt kannst du leben, mit einem fortschreitenden nicht.

Der Pack-Trend aus Test 2 bringt dich einen Teil des Weges. Aber die Version auf Zellebene ist
schärfer, und sie ist einfach durchzuführen: **Nimm dieselbe Messung, im selben Zustand, an
verschiedenen Tagen, und beobachte die Zahl.**

„Derselbe Zustand" leistet in diesem Satz die ganze Arbeit. Es bedeutet:

- **Mittlerer SOC**, im flachen Bereich
- **Null Strom** – nicht „klein", *null*. Jeder Strom überhaupt zieht den I·R-Term wieder herein.
- **Eingeschwungen** – und jedes Mal gleich lang eingeschwungen
- **Balancer inaktiv** (bei mittlerem SOC ist er das)

Das natürliche Zeitfenster ist gleich morgens: Lass den Verbund über Nacht entladen, dann halte
das Laden zurück, bis du deine Messungen gemacht hast. Das gibt dir einen ausgeruhten Pack in
der Mitte des flachen Bereichs, genau dort, wo ein echtes Ladungsdefizit sichtbar wird und
sonst nichts.

Vergleiche dann den Abstand der verdächtigen Zelle – berechnet aus den fünfzehn Zellwerten – Tag
für Tag. Wenn er von Zyklus zu Zyklus wächst, gewinnt der Balancer und die Schleife unten ist
quantitativ bestätigt. Wenn er stabil bleibt, dann wird das, was der Balancer während der
Absorption herausnimmt, bei der nächsten Ladung wieder hineingelegt, und die Lage ist stabil.

**Ein Vergleich zwischen verschiedenen Zuständen ist nichts wert.** Ich verglich einmal zwei
Messungen beim selben Strom (1,0 A) und war recht zufrieden mit mir – bis sich herausstellte,
dass eine bei 40 % SOC und die andere bei 98 % war. Eine flach, eine steil. Der Vergleich war
sinnlos, und ich hatte bereits eine Schlussfolgerung daraus gezogen.

Strom *und* SOC *und* Einschwingzeit müssen übereinstimmen. Sonst vergleichst du zwei
verschiedene Messungen und nennst es einen Trend.

---

## Der passive Balancer macht es schlimmer

Das ist der Teil, der mich am meisten überrascht hat, und er hat reale Konsequenzen.

Pylontech – und im Grunde jedes LFP-BMS dieser Klasse – verwendet **passives** Balancing. Es
kann Ladung von einer hohen Zelle über einen Widerstand ableiten (etwa 50 mA). Es **kann keine**
Ladung in eine niedrige Zelle hineindrücken. Dafür gibt es keinen Mechanismus.

Denk jetzt darüber nach, was eine Zelle mit hohem Widerstand beim Laden tut. `U = I·R` treibt
ihre Klemmenspannung **über** die ihrer Nachbarn. Also erreicht sie die Balancing-Schwelle
(~3,45 V) **zuerst** – obwohl sie in Wirklichkeit die *leerste* Zelle im Pack ist.

Das BMS sieht eine hohe Spannung und tut das Einzige, was es kann: **Es entlädt die schwächste
Zelle im Pack.**

Das schließt eine Schleife:

```
höherer Widerstand
    → liest beim Laden künstlich hoch
        → Balancer entlädt sie
            → verliert echte Ladung
                → sackt unter Last stärker ab
                    → gemessener Widerstand steigt
                        → (Wiederholung, jeden Zyklus)
```

Dieser Mechanismus erklärt die elfmonatige Beschleunigung in meinen Daten besser als bloße
Alterung. Und er ist *nicht* etwas, das der Nutzer verursacht oder abschalten kann. So verhält
sich das Produkt.

**Das diagnostische Merkmal:** Während der Absorption liest die verdächtige Zelle als die
**niedrigste** in ihrem Pack – während sie unter Ladestrom, Minuten zuvor, als die **höchste**
las. Wenn du diese Umkehr siehst, hat der Balancer gegen diese Zelle gearbeitet.

Hier ist das Ganze in Aktion, an einer Zelle, über einen Ladezyklus, an einem Tag:

| Zeit | Zustand | Zelle 13 | Rang im Pack |
|---|---|---|---|
| 09:47 | Entladen, nach der Nacht | 3,265 V | **niedrigste** (15 mV drunter) |
| 10:26 | Laden, 8,8 A | 3,329 V | **höchste** |
| 13:41 | Laden, 5,8 A | 3,370 V | **höchste** |
| 15:18 | Absorption, Balancer aktiv | 3,461 V | 11. von 15 – *wird entladen* |
| 15:27 | Absorption, Balancer aktiv | 3,467 V | **niedrigste** |
| 15:34 | wieder Entladen | 3,374 V | **niedrigste** (22 mV drunter) |

Lies die erste und die letzte Zeile zusammen. Sie ging mit 15 mV Rückstand in diesen Ladezyklus
hinein. Sie kam mit **22 mV** Rückstand heraus. Die Ladung sollte ihr eigentlich helfen.

Und den Mechanismus kannst du in der Mitte beobachten: Sie klettert unter Ladestrom an die Spitze
des Packs (`I·R`, nicht Ladung), überschreitet die Balancing-Schwelle *als Erste*, obwohl sie die
leerste Zelle im Pack ist, wird entladen und rutscht im Rang auf den letzten Platz, während der
Balancer noch läuft.

**Praktische Konsequenz:** Erwarte nicht, dass der Balancer eine widerstandsdegradierte Zelle
repariert. Er tut das Gegenteil. Den Ladestromgrenzwert zu senken (über DVCC oder gleichwertig)
reduziert die `I·R`-Verzerrung und lässt den Balancer auf etwas näher am wahren Zellzustand
agieren – das ist eine echte Milderung, behandelt aber das Symptom.

**Du kannst den Balancer auch an seiner Wärme erkennen – und das ist eine völlig getrennte
Bestätigung, unabhängig von den Spannungen.** Der Pack, der am meisten balanciert, läuft etwas
wärmer als seine Nachbarn, aber *nur* während und nach der Absorption. Zwei Dinge sind dabei
richtig zu machen:

Achte auf die **Pack-Temperatur, nicht auf die Temperatur der entladenen Zelle.** Die
Balancing-Widerstände sitzen auf der BMS-Platine, nicht an der Zelle – die Wärme zeigt sich also
in der Pack-/BMS-Temperaturanzeige, nicht in der Temperatur der Zelle, die abgeleitet wird. (Ich
hatte das zuerst verkehrt herum und suchte nach einer warmen Zelle. Es gibt keine. Die Platine ist
es, die sich erwärmt.)

Und der Grund, warum es überzeugend statt zufällig ist: **Die Wärme überlebt den Strom.** Sobald
der Pack 100 % erreicht und der Ladestrom aufhört, ist die gewöhnliche Ladeerwärmung vorbei – aber
der Balancer entlädt weiter, und der Pack bleibt noch eine Weile warm. Wärme ohne Strom ist der
verräterische Hinweis. Sie hinkt außerdem um mehrere Minuten hinterher (Dinge erwärmen sich
langsam), erwarte den Temperaturanstieg also nicht im selben Augenblick wie das Spannungsereignis
– schau über das ganze Absorptionsfenster. Wie immer gilt: Lies es als *wärmer als die anderen
Packs*, nicht als Absolutwert.

---

## Was du redlich schlussfolgern kannst

Sei vorsichtig mit dem, was du behauptest.

**Gut fundiert:**
- Identische Packs unter identischen Bedingungen gegeneinander vergleichen
- Der Trend über die Zeit – das ist die stärkste Evidenz, die du haben wirst
- Direkt gemessene Zellspannungen
- Die Stromumkehr-Signatur
- Der Dauerlast-Verlauf

**Nicht gut fundiert:**
- Absolute Widerstandswerte aus BMS-Telemetrie – nur als Größenordnung behandeln
- Jede Extrapolation auf die Restlebensdauer
- Ein Modul „defekt" zu nennen, wenn es keine Fehler wirft und die Nennkapazität liefert

Die Behauptung, die tatsächlich standhält, ist vergleichend: *Dieses Modul degradiert messbar
schneller als identische Module unter identischen Bedingungen, und der Mechanismus ist
identifiziert.* Das ist vertretbar. „Es ist kaputt" ist es nicht, solange es noch funktioniert.

In meinem Fall funktioniert der Verbund weiterhin einwandfrei. Keine Fehler, keine
Schutzabschaltungen, und weil er im Normalbetrieb nie in den niedrigen SOC-Bereich hinunterläuft,
ist das Kapazitätsdefizit im Alltag nicht einmal spürbar. Das Problem ist die *Entwicklung*,
nicht der aktuelle Zustand.

---

## Geltungsbereich und Grenzen – bitte lesen

Diese Methode wurde aus **einem Verbund** abgeleitet. Vier Pylontech US3000C-Module, ein
Wechselrichter, ein Haushalts-Lastprofil, elf Monate Daten. Das reicht, um die Methode zu
begründen, aber nicht, um die konkreten Zahlen für maßgeblich zu erklären.

Wovon ich überzeugt bin, dass es sich verallgemeinern lässt:
- Das Drei-Signaturen-Schema (Widerstand / Kapazität / Diffusion)
- Der Stromumkehr-Test und die Extrapolation auf null Strom
- Die Anforderung, unter *Dauerlast* zu testen, nicht mit Schnappschüssen
- Die Anforderung, unter *genug* Last zu testen – unterdimensionierte Tests geben falsche
  Entwarnungen
- Die Gültigkeitsfenster (flacher Bereich, hoher Strom, Balancer aus, wirklich eingeschwungen)
- Die passive-Balancer-Falle – das folgt daraus, wie passives Balancing funktioniert, und ist
  durch die Umkehr in der Absorptionsphase bestätigt, aber ich habe es **nicht** gegen Pylontechs
  tatsächliche Firmware verifiziert. Es ist mechanistisch stichhaltige Schlussfolgerung, keine
  Datenblatt-Tatsache.

Wofür ich mehr Fälle bräuchte, bevor ich ihnen traue:
- Konkrete mΩ-Schwellen für „bedenklich"
- Wie schnell Diffusionslimitierung typischerweise fortschreitet
- Ob dieselben Signaturen in anderen BMS-Familien auftreten

Wenn du das an deinem eigenen Verbund durchführst, werden die Zahlen, die du bekommst, anders
sein. Die *Formen* sollten es nicht.

---

## Wie man das mit Claude nutzt

`SKILL.md` in diesem Repo ist für Claude geschrieben, nicht für dich. Es kodiert den
Entscheidungsbaum, die Gültigkeitsfenster und die Fallen – damit das Modell sie nicht erst
dadurch wiederentdecken muss, dass es sechsmal danebenliegt.

(Was, fürs Protokoll, genau die Art ist, wie diese Methode entstanden ist.)

**Zur Verwendung:**

1. Lade `SKILL.md` aus diesem Repo herunter.
2. Beginne ein Gespräch mit Claude und hänge es an, zusammen mit deinen Daten.
3. Sag etwas wie: *„Nutze das angehängte Skill, um meine Batteriedaten zu analysieren."*

Dann füttere es mit dem, was du gesammelt hast:

- **Deinen CSV-Export** – Claude kann die Widerstandsregression direkt darauf laufen lassen und
  dir den Pack-Vergleich und den Trend geben.
- **Deine Screenshots** – füg sie ein, während du sie machst. Claude liest die
  Zellspannungstabellen direkt aus dem Bild.

Die Screenshots sind der Teil, den Leute überspringen, und es ist der Teil, auf den es ankommt.
Schick sie während eines Tests fortlaufend weiter.

### Was du über Claude wissen musst: Es hat keine Uhr

Das ist keine Marotte. Es ist ein Loch, und es wird deine Analyse still ruinieren, wenn du es
nicht umgehst.

Claude kann Zeit nicht wahrnehmen. Es weiß nicht, wann deine Nachricht ankam, wie lange du für
die nächste gebraucht hast oder wie viel Zeit zwischen zwei Screenshots verging. Es hat überhaupt
kein Gefühl für Dauer – und, schlimmer, **kein Gefühl dafür, dass das Gefühl fehlt.** Eine Zahl,
die es geraten hat, fühlt sich genauso an wie eine, die es gemessen hat.

Wenn du ihm also sagst *„das war etwa vierzig Minuten drin"*, wird es dir glauben, die Zahl
verwenden und sie dir drei Antworten später in einer Tabelle zurückreichen, als wäre sie Daten.
Ist sie nicht. Es ist deine Erinnerung, reingewaschen.

Ich weiß das, weil es wiederholt passiert ist, und jede Zeitangabe in der frühen Version dieser
Arbeit kam aus meinem eigenen Kopf, nicht aus irgendeiner Messung.

**Die Lösung ist trivial, und du solltest sie einfach umsetzen:**

> **1. Bring die Windows-Taskleisten-Uhr in jeden Screenshot.**
> **2. Gib Claude die CSV, die denselben Zeitraum abdeckt.**

Jetzt ist der Zeitstempel *im Bild* – er ist Daten, nicht Zeugenaussage. Und sobald Claude diesen
Zeitstempel gegen das Log abgleichen kann, kann es selbst herausrechnen, wie lange der Strom schon
fließt, wie viele Amperestunden durchgegangen sind, wie lange der Pack geruht hat. Alles davon
hergeleitet. Nichts davon geraten.

Erzähle Claude nicht das Timing. **Zeig ihm die Uhr und gib ihm das Log**, und lass es die
Arithmetik machen.

### Und der Rest

Wenn du historische Logs von vor Monaten hast, nimm die auch mit auf. **Der Trend über die Zeit
ist die stärkste Evidenz, die du produzieren wirst.** Ein einzelner Tag an Daten kann dir zeigen,
*dass* etwas nicht stimmt; nur der Trend zeigt dir, dass es schlimmer wird.

Zwei weitere Dinge, die du ihm sagen solltest, weil es vielleicht nicht daran denkt, sie zu
prüfen:

- **Berechne die Spreizung aus den fünfzehn Einzelzellwerten**, nicht aus den BMS-eigenen
  min/max/Imbalance-Zeilen. Sie stimmen nicht überein.
- **Ein Lasttest, der nichts fand, war vielleicht einfach zu sanft.** Frag nach dem Strom, bevor
  du ein sauberes Ergebnis akzeptierst.

---

*Daten von einem 4× Pylontech US3000C-Verbund an einem Victron MultiPlus-II 3000, ~475 Zyklen.
Seriennummern und Netzwerkdetails entfernt. Zeitstempel relativ.*
