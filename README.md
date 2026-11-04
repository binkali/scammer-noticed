# Scammer Noticed

**Lokale KI-Erkennung von Betrugsanrufen am Festnetz – Jugend-forscht-Projekt**

## Idee

Ein Raspberry Pi wird an der FRITZ!Box als IP-Telefon angemeldet. Auf Wunsch des Angerufenen hört er das Gespräch mit, wandelt es **lokal** in Text um und erkennt mit einem neuronalen Netz, ob es ein Betrugsversuch ist (z. B. falscher Microsoft-Support, Enkeltrick). Bei Verdacht warnt er – bevor Geld fließt oder ein Fernzugriff eingerichtet wird.

## Grundprinzipien

- **Opt-in pro Anruf:** Der Nutzer entscheidet bei jedem Anruf selbst. Bekannte Kontakte werden nie geprüft.
- **Alles lokal, nichts gespeichert:** Keine Cloud, Daten nur im Arbeitsspeicher.
- **Kostenlos:** Nur freie Software und frei verfügbare Modelle.

## Hardware

Raspberry Pi 5 (8 GB), aktiver Kühler, 27-W-Netzteil, microSD ≥ 32 GB, Taster/LED/Lautsprecher für Abfrage und Warnung.

## Modelle

| Aufgabe | Modell |
|---|---|
| Spracherkennung (Vergleich) | **Vosk** (Prototyp), **Whisper** via whisper.cpp, **NVIDIA Parakeet TDT 0.6B v3** |
| Betrugserkennung | Deutsches **BERT**-Modell mit Fine-Tuning |
| Vergleichsverfahren | Schlüsselwortliste, TF-IDF + logistische Regression |

## Funktionsweise

1. **Sprechermarkierung:** Beide Gesprächsseiten kommen getrennt an → Text wird mit `[ANRUFER]` / `[NUTZER]` markiert.
2. **Gleitendes Fenster:** Das Netz bewertet immer die letzten ~30 Sekunden Gespräch. Fenster überlappen sich, damit kein Zusammenhang zerschnitten wird. Die Fensterlänge bleibt immer gleich.
3. **Adaptive Prüfrate:** Start mit 10 s Abstand, danach abhängig vom Verdacht – harmlos = alle 15 s, sehr verdächtig = alle 3 s:
   ```python
   abstand = 15 - (15 - 3) * betrugswert
   ```
4. **Schlüsselwort-Ebene:** Prüft ständig jedes Wort (z. B. „AnyDesk“, „Gutscheinkarte“) und löst bei einem Treffer sofort eine Prüfung durch das Netz aus.
5. **Alarm:**
   - Sofortalarm, wenn ein Fenster > 0,95
   - Normaler Alarm, wenn der gleitende Durchschnitt > 0,7 (glättet Fehlalarme durch Ausreißer)

## Forschungsfragen

- Wie zuverlässig erkennt das Netz Betrug (Erkennungsrate, Fehlalarme)?
- Warnt die **adaptive Prüfrate** genauso schnell wie häufiges Prüfen, aber mit weniger Rechenlast?
- Wie stark leidet die Erkennung unter Telefonqualität und schwächeren Spracherkennungsmodellen?
- Kommt die Warnung **bevor** der Betrüger Geld oder Fernzugriff verlangt?

## Datenschutz

Prüfung nur mit Zustimmung, keine Speicherung, keine Übertragung. Tests mit gespielten Gesprächen. Rechtliche Details (§ 201 StGB) werden mit der Betreuung abgestimmt.
