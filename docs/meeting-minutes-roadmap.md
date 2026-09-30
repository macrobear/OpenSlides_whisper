# Umsetzungsplan: KI-Meetingprotokolle in OpenSlides

Wer macht was, wann? Ergänzt das [Architekturkonzept](meeting-minutes-concept.md).

**Rollen**

- **DU**: Hardware, Tests mit echten Aufnahmen, Entscheidungen, Review,
  Organisation (Datenschutz, Betriebsrat, Kolleg:innen)
- **CLAUDE**: Code, Tests, Installationsskript, Dokumentation, Pull Requests

**Annahmen**

- Die Wochen sind relativ (Woche 1 = Start). Du investierst etwa 4–8 Stunden
  pro Woche.
- Claude arbeitet nur in einer Session, die du startest. Jede Phase beginnt
  also mit einer Nachricht von dir. Zwischenstände werden immer committed
  und gepusht, weil die Session-Umgebung nicht dauerhaft ist.
- Jede Phase endet mit einer **Abnahme**. Erst wenn sie erfüllt ist, geht es
  weiter.

---

## Übersicht

| Woche | Phase | DU (Stunden ca.) | CLAUDE |
|---|---|---|---|
| 0–1 | Vorbereitung + **Phase 0: Machbarkeitstest** | KI-PC, Testaufnahmen, PoC ausführen (6–8 h) | PoC-Skript + Anleitung |
| 1 | **Go/No-Go** | Ergebnisse bewerten, entscheiden (1 h) | Auswertung, Empfehlung |
| 2 | Vorbereitung Integration | Forks anlegen, offene Fragen klären (2–3 h) | Submodule umstellen, Entwicklungsumgebung |
| 2–3 | **Phase 1: Service + Datenmodell + Deployment** | Review, Installation testen (5–8 h) | Minutes-Service, Backend, Installationsskript |
| 4 | **Phase 2: Aufnahme-Button** | Browser/Mikrofon testen (3–4 h) | Client-Modul Aufnahme + Upload |
| 5 | **Phase 3: Stimmprofile** | Kolleg:innen einlernen, Erkennung prüfen (4–6 h) | Enrollment, Sprecherzuordnung |
| 6–7 | **Phase 4: Protokoll + Review** | Vorlage liefern, Ergebnisse bewerten (6–8 h) | LLM-Schritt, Review-Oberfläche, PDF |
| 8 | **Phase 5: Hybrid + Feinschliff** | Teams-Export testen (3–4 h) | VTT-Import, Offline-Paket, Löschfristen |
| 9–10 | **Pilot** | 3–5 echte Meetings, Feedback (4–6 h) | Fehler beheben, Feinjustierung |
| parallel | **Recht und Organisation** | Datenschutz, Betriebsrat, Einwilligungen | Textentwürfe als Vorlage |

---

## Parallel ab Woche 0: Recht und Organisation (DU)

Diese Spur bremst das Projekt am ehesten. Deshalb sofort starten, parallel
zur Technik.

| Schritt | DU | CLAUDE |
|---|---|---|
| Datenschutzbeauftragte informieren | Termin, Konzept + Präsentation zeigen | – |
| Betriebsrat einbinden | Vorhaben vorstellen, Betriebsvereinbarung anstoßen | Entwurf „Eckpunkte Betriebsvereinbarung“ (Zweck, keine Leistungskontrolle, Löschfristen) |
| Datenschutz-Folgenabschätzung | mit DSB durchführen | Vorlage mit technischer Beschreibung der Datenflüsse |
| Einwilligungen | Texte freigeben lassen | Entwürfe: Einwilligung Aufnahme, Einwilligung Stimmprofil, Info-Ansage vor dem Meeting |
| Teams-Richtlinien | IT fragen: Aufzeichnung + Export erlaubt? | – |

**Wichtig:** Für die Testaufnahmen in Phase 0 brauchst du bereits die
Einwilligung der Beteiligten. Am einfachsten sind Testmeetings mit
Freiwilligen zu einem unkritischen Thema.

---

## Woche 0–1: Phase 0 – Machbarkeitstest

**Ziel:** Mit echten Aufnahmen auf dem echten KI-PC herausfinden, ob Qualität
und Laufzeit reichen, bevor OpenSlides angefasst wird.

### CLAUDE (Tag 1, sofort möglich)

- [ ] Ordner `minutes-poc/` in diesem Repo:
  - Python-Kommandozeilenprogramm: `poc run meeting.m4a --voices stimmen/ --agenda agenda.txt`
  - Pipeline: ffmpeg → faster-whisper → pyannote → Abgleich mit Stimmproben → Ollama
  - Ausgabe: `transkript.md` (mit Sprechern und Zeitstempeln),
    `protokoll.md` (pro TOP: Zusammenfassung, Beschlüsse, Aufgaben),
    `bericht.json` (Laufzeit pro Schritt, Sprecher-Konfidenzen)
- [ ] `docker-compose.yml` + `setup.sh` für Debian/Fedora (Vorstufe des späteren Installationsskripts)
- [ ] Anleitung `minutes-poc/README.md`: Installation, Aufnahme der Stimmproben, Ausführen, Ergebnisse lesen
- [ ] Bewertungsbogen `minutes-poc/BEWERTUNG.md` zum Ausfüllen
- [ ] Test in der Session mit öffentlich verfügbarem Beispiel-Audio (nur Funktion, keine Qualitätsaussage)

### DU

- [ ] KI-PC mit Debian oder Fedora bereitstellen, Netzwerk, SSH-Zugang
- [ ] Hugging-Face-Konto anlegen, Nutzungsbedingungen der pyannote-Modelle akzeptieren, Token erzeugen
- [ ] Konferenzmikrofon besorgen bzw. vorhandenes auswählen
- [ ] Stimmproben von 3–5 Freiwilligen: je ca. 30 s, vorgegebener Text (kommt mit dem PoC), Dateiname = Name
- [ ] Zwei Testaufnahmen (je 20–45 min):
  1. vor Ort mit Konferenzmikrofon
  2. Teams-Meeting (Aufzeichnung exportieren, wenn möglich mit `.vtt`-Transkript)
- [ ] Für jede Aufnahme eine kurze Tagesordnung als Textdatei
- [ ] PoC nach Anleitung installieren und ausführen
- [ ] Bewertungsbogen ausfüllen und mir `bericht.json` + deine Einschätzung schicken
  (**keine** Audiodateien oder Transkripte mit vertraulichem Inhalt in die Session)

### Abnahme / Go-No-Go (Ende Woche 1)

| Kriterium | Ziel |
|---|---|
| Laufzeit | ≤ 2 h Verarbeitung pro 1 h Meeting |
| Transkript | gut lesbar, Fachbegriffe größtenteils richtig |
| Sprechererkennung | ≥ 85 % der Redezeit richtig zugeordnet |
| Protokollentwurf | mit ≤ 15 min Nacharbeit brauchbar |

Bei **No-Go** gemeinsam entscheiden: besseres Mikrofon, anderes Modell,
GPU nachrüsten oder Projekt stoppen. CLAUDE liefert dazu eine Auswertung mit
konkreten Stellschrauben.

---

## Woche 2: Vorbereitung der Integration

### DU

- [ ] Auf GitHub unter `macrobear` forken:
  `openslides-backend`, `openslides-meta`, `openslides-client`, `openslides-proxy`
- [ ] Neues Repo `openslides-minutes-service` anlegen (oder entscheiden: Ordner in diesem Repo)
- [ ] Die Repos für die Claude-Session freigeben
- [ ] Offene Fragen aus dem Konzept (Kapitel 14) beantworten, mindestens:
  Wer darf aufnehmen, wer freigeben? Löschfristen? Gibt es eine Protokollvorlage?
- [ ] Entscheiden: volle Integration oder leichte Integration (Konzept Kapitel 13)
- [ ] Eine OpenSlides-Testinstanz bereitstellen (getrennt von einer produktiven)

### CLAUDE

- [ ] Submodule in diesem Repo auf die Forks umstellen, Branch-Strategie festlegen
  (`main` = Upstream-Stand, `minutes` = unsere Erweiterung)
- [ ] Anleitung: Upstream-Updates in die Forks übernehmen
- [ ] Entwicklungsumgebung prüfen (Backend-Tests lauffähig), Stolpersteine dokumentieren

---

## Woche 2–3: Phase 1 – Service, Datenmodell, Deployment

### CLAUDE

- [ ] `openslides-meta`: neue Collections `minutes_recording`, `minutes_transcript`,
  `minutes_entry`, `voice_profile`, Rechte `minutes.*`
- [ ] `openslides-backend`: Actions (public + internal), Upload-Token, Migration, Tests
- [ ] `openslides-minutes-service`: FastAPI, Upload-API, Job-Queue, Pipeline aus dem PoC,
  Rückkanal über `/internal/handle_request`, Tests
- [ ] `install.sh` für Debian/Fedora (Kapitel 8 im Konzept) mit `--update`, `--status`, `--uninstall`
- [ ] `openslides-proxy`: Route `/minutes/`
- [ ] Minimale Client-Ansicht: Datei hochladen, Status sehen, Transkript lesen
- [ ] Je Repo ein Pull Request mit Testanleitung

### DU

- [ ] Pull Requests durchsehen (Schwerpunkt: Verständlichkeit, nicht jede Zeile)
- [ ] `install.sh` auf dem KI-PC ausführen, Secret in der OpenSlides-Testinstanz eintragen
- [ ] Testaufnahme aus Phase 0 in OpenSlides hochladen, Ergebnis prüfen
- [ ] Fehler und Wünsche in der Session melden

### Abnahme

Datei-Upload in OpenSlides → Fortschritt sichtbar → Transkript mit Sprechern
erscheint im Meeting. Das Installationsskript läuft auf deinem System
fehlerfrei und wiederholbar.

---

## Woche 4: Phase 2 – Aufnahme-Button

### CLAUDE

- [ ] Client: Aufnahme-Ansicht mit Start/Stopp, Pegel, Mikrofonwahl, Timer
- [ ] Button „Nächster TOP“ (Marker)
- [ ] Chunk-Upload mit Puffer in IndexedDB, Warnung beim Schließen, Wake Lock
- [ ] Einwilligungsdialog vor dem Start (Pflicht-Checkbox)

### DU

- [ ] Im Besprechungsraum testen: Chrome/Edge, Konferenzmikrofon, 30 min Aufnahme
- [ ] Störfälle testen: WLAN kurz trennen, Tab neu laden, Laptop zuklappen
- [ ] Rückmeldung zur Bedienung

### Abnahme

Eine 60-minütige Aufnahme vor Ort läuft ohne Datenverlust durch und wird
automatisch verarbeitet.

---

## Woche 5: Phase 3 – Stimmprofile

### CLAUDE

- [ ] Benutzerprofil: Einwilligung, Stimmprobe aufnehmen, Status, Löschen
- [ ] Minutes-Service: Embedding berechnen, Audio sofort löschen, verschlüsselt speichern
- [ ] Automatische Zuordnung mit Ampel (hoch / mittel / niedrig)
- [ ] Review: Sprecher umbenennen, gilt für alle Segmente
- [ ] Auswertungs-Skript: Trefferquote aus deinen Korrekturen berechnen

### DU

- [ ] 5–10 Kolleg:innen mit Einwilligung einlernen lassen
- [ ] 2–3 Meetings aufnehmen und im Review korrigieren
- [ ] Trefferquote mit dem Auswertungs-Skript ermitteln und melden

### Abnahme

≥ 85 % der Redezeit werden richtig zugeordnet, unsichere Fälle sind
erkennbar markiert.

---

## Woche 6–7: Phase 4 – Protokoll und Review

### DU (zu Beginn der Phase)

- [ ] Eure Protokollvorlage liefern (Aufbau, Pflichtfelder)
- [ ] 2–3 anonymisierte Beispielprotokolle, die als „gut“ gelten

### CLAUDE

- [ ] LLM-Schritt pro TOP mit strukturierter Ausgabe und Quellverweisen
- [ ] Wortprotokoll (bereinigt, ohne LLM)
- [ ] Review-Oberfläche: Transkript + Protokoll nebeneinander, Audio-Sprung, Bearbeiten,
  „Neu generieren“, „Freigeben“
- [ ] TOP-Grenzen auf einer Zeitleiste verschieben (für Teams-Uploads)
- [ ] PDF-Export nach eurer Vorlage
- [ ] Prompt-Varianten + Vergleich von 2–3 LLM-Modellen auf deinem KI-PC (Skript)

### DU

- [ ] Modellvergleich auf dem KI-PC laufen lassen, Favorit wählen
- [ ] Protokolle von 2–3 Meetings bewerten: Was fehlt? Was ist falsch?
- [ ] Freigabe-Workflow mit einer zweiten Person testen

### Abnahme

Protokollentwurf mit ≤ 15 min Nacharbeit, Beschlüsse und Aufgaben
vollständig, PDF entspricht eurer Vorlage.

---

## Woche 8: Phase 5 – Hybrid und Feinschliff

### CLAUDE

- [ ] Import des Teams-Transkripts (`.vtt`/`.docx`) zur Namenszuordnung
- [ ] Optional: Systemton-Aufnahme im Browser (Mikrofon + Teams)
- [ ] Automatische Löschung nach Frist, Anzeige „Audio verfügbar bis …“
- [ ] `build-bundle.sh` + `install.sh --offline`
- [ ] Health-Anzeige in der Organisationsverwaltung
- [ ] Betriebsdokumentation: Installation, Update, Backup, Fehlerbehebung

### DU

- [ ] Hybrid-Meeting (Raum in Teams eingewählt) aufnehmen und verarbeiten
- [ ] Offline-Installation testen, falls der KI-PC ohne Internet laufen soll
- [ ] Löschfristen prüfen

---

## Woche 9–10: Pilot

### DU

- [ ] 3–5 echte Meetings mit dem System protokollieren
- [ ] Teilnehmende nach ihrer Einschätzung fragen
- [ ] Liste: Fehler, Wünsche, Showstopper

### CLAUDE

- [ ] Fehler beheben, Feinjustierung (Schwellwerte, Prompts, Bedienung)
- [ ] Abschlussbericht: Stand, bekannte Grenzen, nächste Schritte

### Go-live-Voraussetzungen

- [ ] Datenschutz-Folgenabschätzung abgeschlossen
- [ ] Betriebsvereinbarung bzw. Zustimmung Betriebsrat liegt vor
- [ ] Einwilligungstexte freigegeben
- [ ] Pilot-Abnahme erfüllt

---

## So arbeiten wir zusammen

1. **Start einer Phase:** Du schreibst in der Session z. B. „Starte Phase 1“.
2. **Arbeit:** Ich arbeite auf einem Branch je Repo und öffne einen Pull
   Request mit Testanleitung.
3. **Test:** Du testest nach Anleitung und meldest Ergebnisse in der Session.
   Echte Aufnahmen und vertrauliche Inhalte bleiben bei dir. Du schickst nur
   Kennzahlen, Fehlermeldungen und Logs ohne Inhalte.
4. **Abnahme:** Du bestätigst die Abnahme, dann wird gemergt und die nächste
   Phase beginnt.

**Gesamtdauer:** ca. 10 Wochen Kalenderzeit bei 4–8 Stunden pro Woche auf
deiner Seite. Der Engpass liegt eher bei Tests, Rückmeldungen und der
Rechtsklärung als beim Code.
