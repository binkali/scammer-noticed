# Scammer erkannt
**Lokale KI-Erkennung von Betrugsanrufen am Festnetz – mit adaptiver Prüfrate**

*Jugend-forscht-Projekt*

---

## Inhaltsverzeichnis

1. [Worum geht es?](#worum-geht-es)
2. [Grundprinzipien](#grundprinzipien)
3. [Systemüberblick](#systemüberblick)
4. [Hardware](#hardware)
5. [Verwendete Modelle und Software](#verwendete-modelle-und-software)
6. [Ablauf eines Anrufs](#ablauf-eines-anrufs)
7. [Gleitende Fenster](#gleitende-fenster)
8. [Adaptive Prüfrate](#adaptive-prüfrate)
9. [Entscheidung: Wann gibt es Alarm?](#entscheidung-wann-gibt-es-alarm)
10. [Der Betrugs-Klassifikator](#der-betrugs-klassifikator)
11. [Forschungsfragen und Messgrößen](#forschungsfragen-und-messgrößen)
12. [Projektstruktur](#projektstruktur)
13. [Datenschutz und rechtliche Hinweise](#datenschutz-und-rechtliche-hinweise)
14. [Roadmap](#roadmap)
15. [Lizenzen](#lizenzen)

---

## Worum geht es?

Telefonbetrug – etwa durch falsche Microsoft-Mitarbeiter, falsche Polizisten oder den Enkeltrick – richtet jedes Jahr großen Schaden an, vor allem bei älteren Menschen.

Der **Anrufwächter** ist ein kleines, günstiges Gerät auf Basis eines Raspberry Pi, das an die FRITZ!Box angeschlossen wird. Auf Wunsch des Angerufenen hört es ein Telefonat mit, wandelt die Sprache **lokal** in Text um und bewertet mit einem neuronalen Netz, ob es sich um einen Betrugsversuch handelt. Wird es verdächtig, warnt das Gerät – rechtzeitig, bevor Geld fließt oder ein Fernzugriff eingerichtet wird.

---

## Grundprinzipien

| Prinzip | Umsetzung |
|---|---|
| **Opt-in pro Anruf** | Der Nutzer entscheidet bei jedem Anruf selbst, ob das Gespräch geprüft wird. Bekannte Kontakte werden nie geprüft. |
| **Alles lokal** | Spracherkennung und Klassifikation laufen vollständig auf dem Raspberry Pi. Keine Cloud, keine Datenübertragung. |
| **Nichts wird gespeichert** | Audio und Text werden nur im Arbeitsspeicher verarbeitet und nach dem Gespräch verworfen. |
| **Kostenlos und offen** | Ausschließlich freie Software und frei verfügbare Modelle. |
| **Günstig nachzubauen** | Gesamtkosten der Hardware im Bereich von ca. 100–150 € (Preise schwanken). |

---

## Systemüberblick

```
┌────────────┐   SIP    ┌─────────────────────────── Raspberry Pi 5 ───────────────────────────┐
│ FRITZ!Box  │ ───────► │                                                                    │
│ (Festnetz) │          │  SIP-Client ──► Spracherkennung ──► Text mit Sprechermarkierung    │
└────────────┘          │  (baresip)      (Vosk / Whisper /      [ANRUFER] ... [NUTZER] ...  │
                        │                  Parakeet)                       │                 │
                        │                                                  ▼                 │
                        │                          ┌───────────────────────┴──────────┐      │
                        │                          │ Ebene 1: Schlüsselwort-Prüfung   │      │
                        │                          │ (ständig, bei jedem neuen Wort)  │      │
                        │                          ├──────────────────────────────────┤      │
                        │                          │ Ebene 2: Neuronales Netz         │      │
                        │                          │ (gleitendes Fenster,             │      │
                        │                          │  adaptive Prüfrate)              │      │
                        │                          └───────────────┬──────────────────┘      │
                        │                                          ▼                         │
                        │                              Gleitender Durchschnitt               │
                        │                                          ▼                         │
                        │                         Warnung (Ton / LED / Push-Nachricht)       │
                        └────────────────────────────────────────────────────────────────────┘
```

---

## Hardware

| Komponente | Empfehlung | Hinweis |
|---|---|---|
| Einplatinenrechner | **Raspberry Pi 5, 8 GB RAM** | 4 GB reichen für kleine Modelle; 8 GB geben Spielraum für Modellvergleiche |
| Kühlung | **Aktiver Kühler** | Pflicht – Spracherkennung lastet die CPU dauerhaft aus, ohne Kühlung drosselt der Pi |
| Netzteil | Offizielles 27-W-Netzteil | Schwächere Netzteile führen unter Last zu Problemen |
| Speicher | microSD ≥ 32 GB (A2) oder NVMe-SSD | SSD optional, aber schneller |
| Bedienung | Taster für die Opt-in-Abfrage | Alternativ Tastendruck am Telefon (DTMF) |
| Warnung | LED, kleiner Lautsprecher und/oder Push-Nachricht | Push z. B. an Angehörige |
| Netzwerk | LAN-Kabel zur FRITZ!Box | Stabiler als WLAN |

Für das Telefonaudio wird **keine Soundkarte** benötigt – das Audio kommt über das Netzwerk (SIP). Die beiden Gesprächsseiten liegen dabei als **getrennte Audiospuren** vor, sodass immer bekannt ist, wer spricht.

---

## Verwendete Modelle und Software

### Telefonie

| Software | Zweck | Lizenz |
|---|---|---|
| **FRITZ!Box** | Festnetzanschluss, Pi wird als IP-Telefon angemeldet | – |
| **baresip** (alternativ PJSIP) | SIP-Client auf dem Raspberry Pi | BSD bzw. GPL |

### Spracherkennung (Speech-to-Text)

Es werden drei Modelle verglichen:

| Modell | Stärken | Schwächen | Lizenz |
|---|---|---|---|
| **Vosk** (deutsches Modell) | Sehr leicht, echtes Streaming, läuft sogar auf dem Pi 4 | Geringere Genauigkeit | Apache 2.0 |
| **Whisper** über **whisper.cpp** / **faster-whisper** (tiny, base, small – jeweils int8) | Hohe Genauigkeit, sehr verbreitet | Kein natives Streaming, 2–4 s Verzögerung mit *small* auf dem Pi 5 | MIT |
| **NVIDIA Parakeet TDT 0.6B v3** | Sehr genau, unterstützt Deutsch, streamingfähig | Mind. ca. 2 GB RAM, Echtzeit auf dem Pi muss getestet werden | CC-BY-4.0 (Namensnennung) |

**Vorgehen:** Prototyp mit Vosk, anschließend systematischer Vergleich aller drei Modelle bei Genauigkeit, Verzögerung und Auslastung.

> **Telefonqualität:** Telefonaudio hat oft nur 8 kHz Abtastrate, während die meisten Modelle mit 16 kHz trainiert wurden. Die Erkennung am Telefon wird daher schlechter sein als in Benchmarks. Wie stark das die Betrugserkennung beeinflusst, ist Teil der Untersuchung.

### Betrugs-Klassifikator

| Komponente | Zweck | Lizenz |
|---|---|---|
| **deutsches BERT-Modell** (z. B. `gbert-base`) bzw. eine kompakte Variante | Vortrainiertes Sprachmodell, wird per Fine-Tuning auf Betrugserkennung angepasst | MIT |
| **PyTorch** + **Hugging Face Transformers** | Training und Ausführung | BSD / Apache 2.0 |
| **scikit-learn** | Vergleichsverfahren (TF-IDF + logistische Regression) | BSD |

Training findet auf einem normalen PC oder kostenlos in **Google Colab** statt – **nicht** auf dem Raspberry Pi. Der Pi führt das fertige Modell nur aus.

---

## Ablauf eines Anrufs

1. **Anruf geht ein.**
2. **Kontaktprüfung:** Ist die Nummer ein bekannter Kontakt → keine Prüfung, keine Nachfrage.
3. **Opt-in-Abfrage:** Bei unbekannten Nummern fragt das System nach (Taster oder Taste am Telefon). Bei unterdrückten oder ausländischen Nummern wird die Prüfung aktiv vorgeschlagen.
4. **Bei „Nein“:** Das Gespräch läuft ganz normal, nichts wird verarbeitet.
5. **Bei „Ja“:**
   - Die Spracherkennung wandelt beide Gesprächsseiten laufend in Text um und markiert die Sprecher.
   - **Ebene 1** prüft jedes neue Wort auf kritische Schlüsselwörter.
   - **Ebene 2** bewertet das aktuelle Fenster mit dem neuronalen Netz – in einem Abstand, der sich an den Verdachtsgrad anpasst.
   - Bei Alarm: **Warnung** an den Angerufenen (und optional an Angehörige).
6. **Gesprächsende:** Alle Daten im Arbeitsspeicher werden verworfen.

---

## Gleitende Fenster

Das neuronale Netz bekommt nie einzelne Wörter, sondern immer einen **Gesprächsausschnitt** – das *Fenster*.

| Parameter | Startwert | Bedeutung |
|---|---|---|
| **Fensterlänge** | 30 Sekunden (bzw. letzte ~8 Sätze) | Wie viel Text das Modell auf einmal sieht. Bleibt **immer gleich**. |
| **Prüfabstand** | 10 Sekunden zu Beginn, danach adaptiv (3–15 s) | Wie oft das Fenster neu bewertet wird. |

Das Fenster schaut immer **zurück**: Bei jeder Prüfung werden die jeweils letzten 30 Sekunden bewertet, inklusive des gerade Gesagten. Da der Prüfabstand kürzer ist als die Fensterlänge, **überlappen** sich aufeinanderfolgende Fenster:

```
Zeit (s):    0    10    20    30    40    50    60
             |-----|-----|-----|-----|-----|-----|
Fenster 1:   [=================]                      0–30 s
Fenster 2:         [=================]                10–40 s
Fenster 3:               [=================]          20–50 s
Fenster 4:                     [=================]    30–60 s
```

**Warum Überlappung?** Ohne sie könnte ein verdächtiger Zusammenhang genau auf der Grenze zwischen zwei Fenstern liegen und von keinem vollständig gesehen werden.

**Warum bleibt die Fensterlänge gleich?** Das Modell wurde mit Ausschnitten dieser Länge trainiert. Gerade wenn es verdächtig wird, braucht es den vollen Zusammenhang.

### Sprechermarkierung

Da die Gesprächsseiten als getrennte Audiospuren vorliegen, wird der Text so markiert:

```
[ANRUFER] Guten Tag, hier ist der technische Support.
[NUTZER] Ja, worum geht es?
[ANRUFER] Ihr Computer sendet Fehlermeldungen an uns.
```

Verdächtig ist vor allem, was der **Anrufer** sagt. Die Trainingsdaten verwenden dasselbe Format.

---

## Adaptive Prüfrate

Kern des Projekts: Der **Abstand zwischen zwei Prüfungen** hängt vom aktuellen Verdachtsgrad ab.

- **Harmloses Gespräch** → selten prüfen → spart Rechenleistung
- **Verdächtiges Gespräch** → häufig prüfen → schnellere Reaktion

```python
START_ABSTAND = 10  # Sekunden, erste Prüfung
MIN_ABSTAND   = 3   # Sekunden, bei sehr verdächtigem Gespräch
MAX_ABSTAND   = 15  # Sekunden, bei völlig harmlosem Gespräch

def naechster_abstand(wert: float) -> float:
    """wert = Betrugswahrscheinlichkeit des letzten Fensters (0..1)"""
    return MAX_ABSTAND - (MAX_ABSTAND - MIN_ABSTAND) * wert
```

| Betrugswert | Abstand bis zur nächsten Prüfung |
|---|---|
| 0,0 | 15,0 s |
| 0,3 | 11,4 s |
| 0,5 | 9,0 s |
| 0,8 | 5,4 s |
| 1,0 | 3,0 s |

### Absicherung durch die Schlüsselwort-Ebene

Problem: Wirkt ein Gespräch zunächst harmlos, wird nur alle 15 Sekunden geprüft. Kippt es plötzlich, würde das neuronale Netz das verspätet bemerken.

Lösung: **Ebene 1** prüft ständig jedes neue Wort auf kritische Begriffe (z. B. „AnyDesk“, „TeamViewer“, „Gutscheinkarte“, „Bargeld abholen“). Taucht ein solcher Begriff beim **Anrufer** auf, wird **sofort** eine Prüfung durch das neuronale Netz ausgelöst – unabhängig vom aktuellen Abstand.

### Bedingung für den Echtzeitbetrieb

Die Bewertung eines Fensters muss **kürzer dauern als der minimale Prüfabstand** (hier 3 s). Sonst hinkt das System immer weiter hinterher. Das wird bei jedem Modell gemessen.

---

## Entscheidung: Wann gibt es Alarm?

Einzelne Fenster können sich irren. Deshalb wird nicht nach einem Einzelwert entschieden.

### Zwei Alarmstufen

| Stufe | Auslöser | Zweck |
|---|---|---|
| **Sofortalarm** | Einzelnes Fenster > 0,95 | Eindeutige Fälle ohne Verzögerung |
| **Normaler Alarm** | Gleitender Durchschnitt > 0,7 | Robust gegen Ausreißer, erkennt sich langsam aufbauenden Betrug |

### Varianten des gleitenden Durchschnitts

Da die Prüfabstände variieren, umfassen „die letzten 3 Fenster“ je nach Verdachtsgrad unterschiedlich lange Zeiträume. Deshalb werden drei Varianten verglichen:

**A) Letzte N Fenster**

```python
from collections import deque

letzte_werte = deque(maxlen=3)

def durchschnitt_fenster(wert):
    letzte_werte.append(wert)
    return sum(letzte_werte) / len(letzte_werte)
```

**B) Fester Zeitraum** (alle Werte der letzten 30 Sekunden, unabhängig von ihrer Anzahl)

```python
import time

werte_mit_zeit = []  # Liste von (zeitstempel, wert)

def durchschnitt_zeitraum(wert, zeitraum=30):
    jetzt = time.time()
    werte_mit_zeit.append((jetzt, wert))
    aktuelle = [w for (t, w) in werte_mit_zeit if jetzt - t <= zeitraum]
    return sum(aktuelle) / len(aktuelle)
```

**C) Exponentieller gleitender Durchschnitt** (neuere Werte zählen stärker)

```python
ALPHA = 0.5
ema = None

def durchschnitt_exponentiell(wert):
    global ema
    ema = wert if ema is None else ALPHA * wert + (1 - ALPHA) * ema
    return ema
```

### Beispiel: Wie der Durchschnitt Fehlalarme verhindert

**Harmloser Anruf** (Ausreißer in Fenster 3, z. B. „Bitte bestätigen Sie Ihre Adresse“):

| Fenster | Wert | Ø letzte 3 | Alarm? |
|---|---|---|---|
| 1 | 0,10 | 0,10 | nein |
| 2 | 0,20 | 0,15 | nein |
| 3 | 0,85 | 0,38 | nein |
| 4 | 0,20 | 0,42 | nein |
| 5 | 0,10 | 0,38 | nein |

**Betrugsanruf** (Verdacht steigt dauerhaft):

| Fenster | Wert | Ø letzte 3 | Alarm? |
|---|---|---|---|
| 1 | 0,20 | 0,20 | nein |
| 2 | 0,40 | 0,30 | nein |
| 3 | 0,75 | 0,45 | nein |
| 4 | 0,85 | 0,67 | nein |
| 5 | 0,90 | **0,83** | **ja** |

---

## Der Betrugs-Klassifikator

### Ansatz: Transfer Learning

Statt ein Netz von Grund auf zu trainieren, wird ein **vortrainiertes deutsches Sprachmodell** per **Fine-Tuning** auf die Aufgabe „Betrug / normal“ angepasst. Dafür genügen einige hundert bis wenige tausend Beispiele. Das Modell kann bis zu 512 Tokens (ca. 300–400 Wörter) auf einmal verarbeiten.

### Trainingsdaten

Ziel für den Start: **ca. 500–1000 Beispiele pro Klasse**, jeweils als Gesprächsausschnitt in Fensterlänge mit Sprechermarkierung.

| Quelle | Klasse | Hinweis |
|---|---|---|
| Selbst geschriebene Dialoge nach Warnungen von Polizei und Verbraucherzentralen | Betrug | Typische Maschen: falscher Support, falsche Polizei, Enkeltrick, falsche Bank |
| Scambaiting-Mitschnitte (meist Englisch) | Betrug | Als Vorlage oder übersetzt |
| KI-generierte Dialoge | beide | Schnell, aber Risiko: Modell lernt den KI-Schreibstil statt Betrugsmuster |
| Normale Telefonate, besonders schwierige Fälle | normal | Paketdienst, Arztpraxis, **echte** Rückrufe der Bank |

> Das Modell wird **immer auch mit Daten getestet, die nicht KI-generiert sind**, um zu prüfen, ob es echte Muster gelernt hat.

### Vergleichsverfahren (Baseline)

Um zu zeigen, was das neuronale Netz wirklich bringt, wird es verglichen mit:

1. reiner **Schlüsselwortliste**
2. **TF-IDF + logistischer Regression**

### Mögliche Erweiterung: Zweistufiges Modell

BERT erzeugt für jedes Fenster einen Zahlenvektor; ein kleines **LSTM** verarbeitet diese Vektoren der Reihe nach und lernt den **Gesprächsverlauf** (freundlich → Problem → Druck → Forderung).

---

## Forschungsfragen und Messgrößen

### Forschungsfragen

1. Wie zuverlässig erkennt ein lokal laufendes neuronales Netz Betrugsanrufe anhand des transkribierten Gesprächs?
2. Bringt die **adaptive Prüfrate** eine ähnlich schnelle Warnung wie eine häufige feste Prüfung – bei deutlich geringerer Rechenlast?
3. Wie stark beeinflusst die Qualität der Spracherkennung (Modellgröße, Telefonqualität 8 kHz) die Betrugserkennung?
4. Wie viel besser ist das neuronale Netz als eine Schlüsselwortliste oder ein klassisches Verfahren?
5. Wird gewarnt, **bevor** der Betrüger Geld oder Fernzugriff verlangt?

### Zu vergleichende Prüfstrategien

| Strategie | Beschreibung |
|---|---|
| Fest | Prüfung alle 10 Sekunden |
| Satzbasiert | Prüfung nach jedem abgeschlossenen Satz |
| **Adaptiv** | Abstand 3–15 s je nach Verdachtsgrad (+ Schlüsselwort-Auslöser) |

### Messgrößen

| Messgröße | Bedeutung |
|---|---|
| **Erkennungsrate** (Recall) | Anteil der Betrugsanrufe, bei denen gewarnt wurde |
| **Fehlalarmrate** | Anteil harmloser Anrufe mit Warnung |
| **Zeit bis zur Warnung** | Vom ersten verdächtigen Satz bis zum Alarm |
| **Vorsprung** | Zeit zwischen Warnung und erster Forderung (Geld / Fernzugriff) |
| **CPU-Auslastung** | Durchschnitt und Spitze auf dem Pi |
| **Verarbeitungszeit pro Fenster** | Muss unter dem minimalen Prüfabstand liegen |
| **Wortfehlerrate** (WER) | Qualität der Spracherkennung |

---

## Projektstruktur

```
anrufwaechter/
├── README.md
├── requirements.txt
├── config.yaml              # Fensterlänge, Abstände, Schwellen, Modellwahl
├── src/
│   ├── telefonie.py         # SIP-Anbindung, Opt-in-Abfrage, Kontaktprüfung
│   ├── spracherkennung.py   # Einheitliche Schnittstelle für Vosk / Whisper / Parakeet
│   ├── fenster.py           # Gleitendes Fenster mit Sprechermarkierung
│   ├── schluesselwoerter.py # Ebene 1: Schlüsselwort-Prüfung
│   ├── klassifikator.py     # Ebene 2: neuronales Netz
│   ├── pruefrate.py         # Adaptive Prüfrate
│   ├── entscheidung.py      # Gleitender Durchschnitt, Alarmstufen
│   ├── warnung.py           # LED, Ton, Push-Nachricht
│   └── main.py
├── training/
│   ├── daten/               # Trainings- und Testdaten (keine echten Gesprächsaufnahmen!)
│   ├── baseline.py          # Schlüsselwortliste, TF-IDF + logistische Regression
│   └── finetuning.ipynb     # Fine-Tuning des BERT-Modells (z. B. in Google Colab)
└── auswertung/
    ├── messungen/           # Ergebnisse der Testläufe
    └── diagramme.ipynb      # Auswertung und Diagramme
```

---

## Datenschutz und rechtliche Hinweise

- Die Prüfung erfolgt **nur nach ausdrücklicher Zustimmung** des Angerufenen, pro Anruf.
- Audio und Text werden **ausschließlich im Arbeitsspeicher** verarbeitet und **nicht gespeichert**.
- Es findet **keine Übertragung** an Server oder Dritte statt.
- Auch die Person am anderen Ende der Leitung ist betroffen (§ 201 StGB – Vertraulichkeit des Wortes). Die genaue Ausgestaltung wird mit der Projektbetreuung abgestimmt.
- Für Tests werden **gespielte Gespräche** mit eingeweihten Testpersonen verwendet – keine Mitschnitte echter Telefonate ohne Einwilligung.
- Das System warnt nur; es greift nicht in das Gespräch ein und meldet nichts an Behörden.

---

## Roadmap

- [ ] Raspberry Pi als IP-Telefon an der FRITZ!Box anmelden
- [ ] Audio beider Gesprächsseiten getrennt abgreifen
- [ ] Prototyp mit Vosk: Live-Transkription mit Sprechermarkierung
- [ ] Opt-in-Abfrage und Kontaktprüfung
- [ ] Trainingsdaten sammeln (Ziel: 500–1000 pro Klasse)
- [ ] Baseline-Verfahren umsetzen und auswerten
- [ ] BERT-Modell fine-tunen
- [ ] Gleitendes Fenster + feste Prüfrate
- [ ] Adaptive Prüfrate + Schlüsselwort-Auslöser
- [ ] Varianten des gleitenden Durchschnitts vergleichen
- [ ] Whisper und Parakeet integrieren und vergleichen
- [ ] Messreihen auf dem Raspberry Pi
- [ ] Testanrufe mit Testpersonen
- [ ] Auswertung, Diagramme, schriftliche Arbeit

---

## Lizenzen

| Komponente | Lizenz |
|---|---|
| Whisper, whisper.cpp, faster-whisper | MIT |
| Vosk | Apache 2.0 (Modelllizenzen einzeln prüfen) |
| NVIDIA Parakeet TDT 0.6B v3 | CC-BY-4.0 – **Namensnennung von NVIDIA erforderlich** |
| Deutsches BERT-Modell | Lizenz auf der jeweiligen Modellseite prüfen |
| PyTorch, scikit-learn | BSD |
| Hugging Face Transformers | Apache 2.0 |
| baresip | BSD |

Alle Modelle werden **lokal heruntergeladen** (z. B. von Hugging Face oder GitHub). Es werden **keine** kostenpflichtigen API-Dienste genutzt.
