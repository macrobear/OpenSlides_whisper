# Konzept: KI-gestützte Meetingprotokolle in OpenSlides

Status: Entwurf / Architekturkonzept (noch keine Implementierung)
Basis: OpenSlides 4.3.4 (dieses Repository)

---

## 1. Ziel und Rahmenbedingungen

OpenSlides soll neben Versammlungen auch für **Arbeitsmeetings** genutzt werden
können – vor Ort, hybrid oder rein über Microsoft Teams. Aus einer Audioaufnahme
soll automatisch ein Protokoll entstehen, das in OpenSlides geprüft, freigegeben
und exportiert wird.

Festgelegte Rahmenbedingungen:

| Thema | Entscheidung |
|---|---|
| Verarbeitung | **Vollständig lokal** auf einem separaten PC (i7, 16–32 GB RAM, keine GPU). Keine Cloud-KI. |
| Einsatz | Arbeitskontext (→ DSGVO, Betriebsrat, Einwilligungen, siehe Kapitel 11) |
| Teams | Die Aufzeichnung aus Teams wird exportiert und in OpenSlides hochgeladen. Es gibt **keinen** Bot im Meeting. |
| Vor Ort | Die Aufnahme wird **per Button in OpenSlides** im Browser gestartet (Client-PC mit Mikrofon). |
| Zeitpunkt | Die Verarbeitung erfolgt **nachträglich** (Batch). Eine Live-Transkription ist nicht nötig. |
| Ergebnis | Wortprotokoll, Ergebnisprotokoll (Beschlüsse, Aufgaben) und Zusammenfassung **pro Tagesordnungspunkt** |
| Sprecher | Die Teilnehmenden werden über **Stimmproben** erkannt (Enrollment). |

---

## 2. Architektur-Überblick

```
                          ┌──────────────────────────── OpenSlides-Server (bestehend) ───────────────────────────┐
                          │                                                                                      │
 ┌──────────────┐  HTTPS  │  ┌────────┐   ┌──────────────┐   ┌──────────────┐   ┌────────────┐   ┌───────────┐  │
 │  Browser     │────────▶│  │ proxy  │──▶│ backend      │──▶│ postgres     │◀──│ autoupdate │──▶│ (Client)  │  │
 │  (Client)    │         │  │        │   │ (+ neue      │   │ (+ neue      │   │            │   │           │  │
 │              │         │  │        │   │  Actions)    │   │  Collections)│   └────────────┘   └───────────┘  │
 │ ● Aufnahme   │         │  │        │   └──────▲───────┘   └──────────────┘                                   │
 │ ● Upload     │         │  │        │          │ /internal/handle_request (internes Passwort)                  │
 │ ● Review-UI  │         │  │ /minutes/*        │                                                               │
 └──────────────┘         │  └───┬────┘          │                                                               │
                          └──────┼───────────────┼───────────────────────────────────────────────────────────────┘
                                 │ Audio-Upload  │ Ergebnisse / Status
                                 ▼               │
                  ┌──────────────────────────────┴──────────── Separater KI-PC (neu) ──────────────────┐
                  │  openslides-minutes-service                                                        │
                  │   ├─ Upload-API (Chunks, Wiederaufnahme)                                           │
                  │   ├─ Job-Queue (SQLite/Redis) + Worker (sequenziell)                               │
                  │   ├─ Pipeline: ffmpeg → Whisper → Diarization → Sprecher-ID → TOP-Zuordnung → LLM  │
                  │   ├─ Stimmprofil-Speicher (nur Embeddings, verschlüsselt)                          │
                  │   └─ Audio-Speicher (temporär, automatische Löschung)                              │
                  │  Ollama / llama.cpp (lokales LLM)                                                  │
                  └────────────────────────────────────────────────────────────────────────────────────┘
```

**Grundprinzipien**

1. **Audio und biometrische Daten verlassen den KI-PC nicht.** In OpenSlides
   landen nur Texte, Status und Zuordnungen. Der Media-Service von OpenSlides
   speichert Dateien in Postgres und eignet sich nicht für große Audiodateien.
2. **Die Rechteprüfung bleibt im OpenSlides-Backend.** Der Minutes-Service
   vertraut nur Tokens, die das Backend ausgestellt hat.
3. **Die Ergebnisse fließen über normale Backend-Actions zurück.** Dadurch
   funktionieren Autoupdate (Live-Status im Client), History und Rechte ohne
   Sonderlogik.
4. **Die KI liefert einen Entwurf, der Mensch gibt frei.** Kein Protokoll wird
   ohne Review veröffentlicht.

---

## 3. Komponenten

### 3.1 Neuer Microservice `openslides-minutes-service` (auf dem KI-PC)

- **Sprache:** Python, weil Whisper, pyannote und die LLM-Tools dort nativ
  laufen. FastAPI als HTTP-Schicht.
- **Deployment:** Docker Compose auf dem KI-PC. Der OpenSlides-Proxy routet
  `/minutes/*` per Traefik bzw. Caddy-Route auf diesen Host (LAN, TLS).
- **Endpunkte (Entwurf):**

| Methode | Pfad | Zweck |
|---|---|---|
| `POST` | `/minutes/upload/{recording_id}/chunk` | Audio-Chunk anhängen (Browser-Aufnahme, alle ~10 s) |
| `POST` | `/minutes/upload/{recording_id}/file` | Komplette Datei hochladen (Teams-MP4, M4A, WAV, …) |
| `POST` | `/minutes/upload/{recording_id}/finish` | Upload abschließen, Job einreihen |
| `POST` | `/minutes/voice-enrollment/{user_id}` | Stimmprobe hochladen, Embedding berechnen, Audio löschen |
| `DELETE` | `/minutes/voice-enrollment/{user_id}` | Stimmprofil löschen |
| `POST` | `/minutes/jobs/{recording_id}/rerun` | Einzelne Pipeline-Schritte neu ausführen (z. B. nur LLM nach Sprecherkorrektur) |
| `GET` | `/minutes/health` | Status, Queue-Länge, Modellversionen |

- **Authentifizierung:** Jeder Aufruf trägt ein **kurzlebiges, signiertes
  Upload-Token** (JWT, HMAC mit einem gemeinsamen Secret). Das Backend stellt es
  bei `minutes_recording.create` bzw. `voice_profile.start_enrollment` aus. Es
  enthält `recording_id`, `meeting_id`, `user_id`, `scope` und `exp`. So muss der
  Minutes-Service keine OpenSlides-Rechte kennen.
- **Rückkanal:** Der Service ruft `POST /internal/handle_request` am Backend auf
  und authentifiziert sich mit dem vorhandenen *internal auth password*, genau
  wie andere OpenSlides-Dienste. Er nutzt ausschließlich **interne Actions**
  (`ActionType.BACKEND_INTERNAL`), siehe 3.3.
- **Job-Verarbeitung:** Die Jobs laufen strikt **nacheinander**, weil CPU und
  RAM begrenzt sind. Der Fortschritt wird pro Schritt an OpenSlides gemeldet.
  Jeder Schritt speichert sein Zwischenergebnis, damit ein Neustart ab einem
  Schritt möglich ist.

### 3.2 OpenSlides-Client (Angular, `openslides-client`)

Ein neues Modul **„Protokolle“** in der Meeting-Navigation:

1. **Aufnahme-Ansicht**
   - Button **„Aufnahme starten“** / **„Stopp“** mit großem, sichtbarem
     Aufnahmeindikator und Timer.
   - Auswahl des Mikrofons und Pegelanzeige (WebAudio `AnalyserNode`).
   - Button **„Nächster TOP“** bzw. eine TOP-Liste zum Anklicken. Jeder Klick
     setzt einen Zeitstempel-Marker. Das ist die zuverlässigste Grundlage für
     die Zuordnung zu Tagesordnungspunkten.
   - Optional **„Teams-/Systemton mit aufnehmen“** für hybride Meetings, siehe
     Kapitel 5.
   - Technik: `getUserMedia` + `MediaRecorder` (Opus/WebM, ca. 32 kbit/s,
     etwa 15 MB pro Stunde). Die Chunks werden fortlaufend hochgeladen,
     zusätzlich in IndexedDB gepuffert (Schutz bei Netzabbruch oder Tab-Crash).
     `navigator.wakeLock` verhindert den Standby, `beforeunload` warnt beim
     Schließen des Tabs.
2. **Upload-Ansicht:** Teams-Aufzeichnung (MP4/M4A) und optional das
   Teams-Transkript (`.vtt`/`.docx`) per Drag & Drop hochladen.
3. **Status-Ansicht:** Pipeline-Fortschritt live über Autoupdate
   (z. B. „Transkription 45 %“).
4. **Review-Ansicht (Kernstück)**
   - Links das Transkript mit Sprechern und Zeitstempeln, rechts der
     Protokollentwurf pro TOP.
   - **Sprecher-Zuordnung korrigieren:** „Sprecher 3“ → Person auswählen. Die
     Änderung gilt für alle Segmente dieses Sprechers.
   - Klick auf ein Segment spielt die passende Audiostelle ab (Streaming vom
     Minutes-Service, solange das Audio noch nicht gelöscht ist).
   - Zusammenfassung, Beschlüsse und Aufgaben editieren (bestehender
     TinyMCE-Editor).
   - Button **„Neu generieren“** (nur den LLM-Schritt, z. B. nach
     Sprecherkorrektur).
   - Button **„Freigeben“**: setzt den Status `approved`, danach ist das
     Protokoll schreibgeschützt.
5. **Export:** PDF über die bestehende pdfmake-Exportinfrastruktur des
   Clients. Optional DOCX/Markdown.
6. **Stimmprofil (im Benutzerprofil):** Einwilligungstext, Aufnahme eines
   vorgegebenen Textes (ca. 30 s), Status „vorhanden seit …“, Button
   „Löschen“.

### 3.3 OpenSlides-Backend (`openslides-backend` + `meta`)

**Neue Collections** (in `openslides-backend/meta/collections/*.yml`, Entwurf):

```yaml
# minutes_recording.yml – eine Aufnahme / ein Verarbeitungsjob
fields:
  id:            {type: number, constant: true, required: true}
  meeting_id:    {type: relation, to: meeting/minutes_recording_ids, required: true}
  title:         {type: string}
  source:        {type: string, enum: [browser, file_upload, teams_upload]}
  state:         {type: string, enum: [recording, uploading, queued, preprocessing,
                  transcribing, diarizing, identifying, summarizing, review, approved, failed]}
  progress:      {type: number}           # 0–100 für den aktuellen Schritt
  error_message: {type: text}
  started_at:    {type: timestamp}
  ended_at:      {type: timestamp}
  duration:      {type: number}           # Sekunden
  agenda_markers: {type: JSON}            # [{agenda_item_id, offset_s}]
  speaker_mapping: {type: JSON}           # {"SPEAKER_03": {meeting_user_id, confidence, confirmed}}
  participant_ids: {type: relation-list, to: meeting_user/minutes_recording_ids}
  audio_available: {type: boolean}        # false nach Löschung auf dem KI-PC
  audio_delete_at: {type: timestamp}
  transcript_id: {type: relation, to: minutes_transcript/recording_id}
  entry_ids:     {type: relation-list, to: minutes_entry/recording_id}

# minutes_transcript.yml – Rohtranskript (1:1 zur Aufnahme)
fields:
  recording_id:  {type: relation, to: minutes_recording/transcript_id, required: true}
  language:      {type: string}
  model_info:    {type: JSON}             # Whisper-/pyannote-Versionen (Nachvollziehbarkeit)
  segments:      {type: JSON}             # [{start, end, speaker, meeting_user_id, text, agenda_item_id}]

# minutes_entry.yml – Protokoll pro Tagesordnungspunkt
fields:
  recording_id:   {type: relation, to: minutes_recording/entry_ids, required: true}
  agenda_item_id: {type: relation, to: agenda_item/minutes_entry_ids}  # null = „Sonstiges“
  weight:         {type: number}
  summary:        {type: HTMLStrict}      # Zusammenfassung
  verbatim:       {type: HTMLStrict}      # bereinigtes Wortprotokoll
  decisions:      {type: JSON}            # [{text, vote_result?}]
  tasks:          {type: JSON}            # [{text, assignee_meeting_user_id, due_date}]
  state:          {type: string, enum: [draft, approved]}
  approved_by_id: {type: relation, to: meeting_user/...}
  approved_at:    {type: timestamp}

# voice_profile.yml – NUR Metadaten, KEINE Biometrie
fields:
  user_id:        {type: relation, to: user/voice_profile_id, required: true}
  consent_at:     {type: timestamp}
  consent_text_version: {type: string}
  state:          {type: string, enum: [none, pending, active]}
  enrolled_at:    {type: timestamp}
```

Dazu kommen die Rückrelationen in `meeting`, `meeting_user`, `agenda_item` und
`user`.

**Neue Rechte** (`meta/permission.yml`):
`minutes.can_see` < `minutes.can_record` < `minutes.can_manage`
(die Freigabe erfordert `can_manage`).

**Actions:**

| Action | Typ | Zweck |
|---|---|---|
| `minutes_recording.create` | public | Aufnahme oder Upload anlegen, gibt `id` + Upload-Token zurück |
| `minutes_recording.update` | public | Titel, TOP-Marker, Teilnehmer, Sprecherzuordnung ändern |
| `minutes_recording.delete` | public | inklusive Löschauftrag an den KI-PC |
| `minutes_recording.set_state` | **internal** | Status/Fortschritt vom Minutes-Service |
| `minutes_transcript.create` / `.update` | **internal** | Transkript schreiben |
| `minutes_entry.create` | **internal** | Protokollentwurf schreiben |
| `minutes_entry.update` | public | Review/Bearbeitung |
| `minutes_entry.approve` | public | Freigabe (`can_manage`) |
| `voice_profile.start_enrollment` | public | Einwilligung speichern, gibt Upload-Token zurück (nur für sich selbst) |
| `voice_profile.set_state` | **internal** | vom Minutes-Service nach der Embedding-Berechnung |
| `voice_profile.delete` | public | Löschung, triggert die Löschung auf dem KI-PC |

Außerdem ist eine **Migration** in `openslides_backend/migrations` für die
neuen Felder nötig.

### 3.4 Proxy (`openslides-proxy`)

Eine zusätzliche Route `/minutes/` → `http://<ki-pc>:<port>/` mit einem
erhöhten Upload-Limit (z. B. 2 GB für Teams-MP4-Dateien).

---

## 4. Verarbeitungspipeline (auf dem KI-PC)

```
Audio ─▶ 1 Vorverarbeitung ─▶ 2 Transkription ─▶ 3 Diarization ─▶ 4 Sprecher-ID ─▶ 5 TOP-Zuordnung ─▶ 6 LLM ─▶ OpenSlides
```

| # | Schritt | Werkzeug (Vorschlag) | Ergebnis |
|---|---|---|---|
| 1 | Vorverarbeitung | `ffmpeg`: Audio aus MP4 extrahieren, 16 kHz mono, Lautheit normalisieren; optional Rauschunterdrückung | `audio.wav` |
| 2 | Transkription | **faster-whisper** (CTranslate2, int8 auf CPU), Modell `large-v3-turbo` (Qualität) oder `medium` (Tempo), Sprache `de`, VAD-Filter, Wort-Zeitstempel | Segmente mit Text und Zeiten |
| 3 | Diarization | **pyannote.audio** (`speaker-diarization-3.x`), Anzahl der Sprecher als Hinweis aus der Teilnehmerliste | „SPEAKER_00…n“ mit Zeitbereichen |
| 4 | Sprecher-Identifikation | Speaker-Embeddings (pyannote/WeSpeaker oder SpeechBrain ECAPA), Vergleich mit Stimmprofilen | Zuordnung Label → Person + Konfidenz |
| 5 | TOP-Zuordnung | TOP-Marker, OpenSlides-Redeliste, LLM-Fallback | Segment → `agenda_item_id` |
| 6 | Protokoll-Generierung | Lokales LLM über **Ollama** oder llama.cpp | Zusammenfassung, Beschlüsse, Aufgaben pro TOP |

Die Transkription (2) und die Diarization (3) werden über die Zeitstempel
zusammengeführt (Wort → Sprecher mit der größten Überlappung). Werkzeuge wie
*WhisperX* bündeln genau das und können als Vorlage dienen.

### 4.1 Sprechererkennung im Detail

Die Erkennung erfolgt **nicht über die LLM**, sondern über Stimm-Embeddings.
Eine LLM kann Stimmen nicht hören, sie sieht nur Text. Das Verfahren:

1. **Enrollment (einmalig pro Person):** Die Person liest ca. 30 s einen
   vorgegebenen Text vor. Ein „kurzer Satz“ reicht meist nicht zuverlässig aus,
   30 s sind ein guter Kompromiss. Daraus entsteht ein Embedding-Vektor
   (192–512 Zahlen). Die **Audiodatei wird sofort gelöscht**, gespeichert wird
   nur der Vektor, verschlüsselt auf dem KI-PC.
2. **Pro Meeting:** Für jeden Diarization-Sprecher wird ein
   Durchschnitts-Embedding aus seinen Segmenten gebildet.
3. **Abgleich:** Die Kosinus-Ähnlichkeit wird gegen die Stimmprofile **nur der
   Meeting-Teilnehmenden** berechnet. Das verkleinert den Suchraum und erhöht
   die Trefferquote deutlich. Die optimale 1:1-Zuordnung erfolgt über den
   Hungarian Algorithm.
4. **Schwellwerte:**
   - ≥ hoch → automatisch zugeordnet (grün)
   - mittel → Vorschlag, bitte bestätigen (gelb)
   - niedrig oder kein Profil → „Unbekannt 1“ (rot), manuelle Zuordnung im Review
5. **Zusätzliche Signale (gewichtet):**
   - **Redeliste:** Wird die Redeliste in OpenSlides genutzt, hat
     `speaker.begin_time`/`end_time` bereits Zeitstempel und Person. Das ist
     ein sehr starkes Signal (setzt eine synchron gestartete Aufnahme voraus).
   - **Teams-Transkript (`.vtt`):** Enthält für Remote-Teilnehmende bereits
     Namen. Diese werden per Zeitüberlappung übernommen, das Embedding dient
     nur zur Plausibilisierung.
   - **LLM-Hinweise (schwach):** Anreden wie „Danke, Frau Müller“ können
     unklare Fälle als Vorschlag markieren, aber nie automatisch zuordnen.
6. **Lernen:** Manuell bestätigte Zuordnungen können (mit Einwilligung) das
   Stimmprofil verbessern (gleitender Mittelwert der Embeddings).

Grenzen: Überlappendes Sprechen, schlechte Raummikrofone und sehr kurze
Wortbeiträge (< 2 s) bleiben fehleranfällig. Deshalb ist die Review-Ansicht
Pflicht.

### 4.2 Zuordnung zu Tagesordnungspunkten

Priorität der Quellen:

1. **TOP-Marker** aus der Aufnahme-Ansicht („Nächster TOP“-Button). Das ist
   exakt und einfach.
2. **Projektor/Redeliste:** Wann wurde welcher TOP projiziert bzw. welche
   Redeliste geöffnet? Das setzt voraus, dass dies in OpenSlides mitprotokolliert
   wird. Heute gibt es dafür keine Historie, daher optional mit
   Erweiterungsaufwand.
3. **Fallback LLM:** Die TOP-Titel und -Texte gehen als Kontext an die LLM, die
   die Übergänge im Transkript bestimmt. Das Ergebnis ist ein Vorschlag, der
   im Review verschoben werden kann.
4. Alles ohne Zuordnung landet unter „Sonstiges“.

Bei **Teams-Uploads** gibt es keine Live-Marker. Hier sind Variante 3 oder das
nachträgliche Setzen von Markern auf einer Zeitleiste in der Review-Ansicht
möglich.

### 4.3 LLM-Protokollerstellung

- **Chunking pro TOP:** Die LLM erhält immer nur das Transkript eines TOPs mit
  Sprechernamen. Das hält den Kontext klein, was auf der CPU entscheidend für
  die Laufzeit ist.
- **Strukturierte Ausgabe** (JSON-Schema bzw. Ollama `format`):
  `{summary, decisions[], tasks[{text, assignee, due}], open_questions[]}`.
- **Wortprotokoll:** Kommt direkt aus dem Transkript (Füllwörter entfernen,
  Absätze pro Sprecher). Dafür ist keine LLM nötig, so bleibt es
  halluzinationsfrei.
- **Prompt-Kontext:** TOP-Titel, TOP-Text (Topic), Teilnehmerliste mit Rollen,
  eine firmenspezifische Protokollvorlage (konfigurierbar pro Meeting).
- **Qualitätssicherung:** Jede Aufgabe und jeder Beschluss verweist auf die
  Quell-Segmente (Zeitstempel). Im Review springt ein Klick zur Stelle im
  Transkript.
- **Modellwahl (CPU, quantisiert Q4):** 7–9 B-Modelle mit gutem Deutsch
  (z. B. aktuelle Qwen-, Llama- oder Mistral-Varianten) bei 16 GB RAM,
  bis ca. 14 B bei 32 GB. Das Modell sollte konfigurierbar sein und vor der
  Entscheidung mit echten Beispieltranskripten verglichen werden.

---

## 5. Aufnahmeszenarien

| Szenario | Empfohlener Weg | Anmerkung |
|---|---|---|
| **Vor Ort** | Browser-Aufnahme per Button in OpenSlides auf einem Raum-PC oder Laptop mit **Konferenzmikrofon** (Konferenzspinne, USB) | Die Mikrofonqualität ist der größte Qualitätshebel, ein Laptopmikrofon reicht meist nicht. |
| **Rein Teams** | Teams-Aufzeichnung exportieren → in OpenSlides hochladen (+ optional das Teams-Transkript `.vtt`) | Das Teams-Transkript liefert Namen gratis, die Stimmprofile sind dann nur Backup. |
| **Hybrid, Raum ist in Teams eingewählt** | Wie „Rein Teams“: Die Teams-Aufzeichnung enthält Raum und Remote | Alle Personen im Raum erscheinen in Teams als *ein* Teilnehmer, dort übernehmen Diarization und Stimmprofile. |
| **Hybrid, Alternative** | Browser-Aufnahme mit **Mikrofon + Systemton** (`getDisplayMedia` mit Audio, per WebAudio gemischt) auf dem Raum-PC, auf dem Teams läuft | Ohne Teams-Export, aber Systemton-Aufnahme im Browser funktioniert nur in Chromium-Browsern (Chrome/Edge) unter Windows zuverlässig. Vorher testen. |

Zwei getrennte Aufnahmen (Raum + Teams) zeitlich zusammenzuführen, ist bewusst
**nicht** vorgesehen (fehleranfällige Synchronisierung).

---

## 6. Ablauf (Sequenz)

```
Nutzer            Client                 Backend                    Minutes-Service (KI-PC)
  │ Start ─────────▶│ minutes_recording.create ─▶│ Rechte prüfen,          │
  │                 │◀──── id + Upload-Token ────│ Datensatz anlegen       │
  │                 │── Audio-Chunks (Token) ───────────────────────────▶│ speichern
  │ „Nächster TOP“ ▶│ minutes_recording.update (agenda_markers) ▶│        │
  │ Stopp ─────────▶│── finish ─────────────────────────────────────────▶│ Job in Queue
  │                 │                            │◀── set_state(transcribing, 30 %)
  │   (Autoupdate zeigt den Fortschritt live)    │◀── minutes_transcript.create
  │                 │                            │◀── minutes_entry.create (je TOP)
  │                 │                            │◀── set_state(review)
  │ Review/Korrektur▶ minutes_entry.update ─────▶│                         │
  │ „Neu generieren“▶│── rerun(LLM) ───────────────────────────────────▶│
  │ Freigeben ─────▶│ minutes_entry.approve ────▶│                         │
  │                 │                            │ (nach Frist) ──────────▶│ Audio löschen
```

---

## 7. Hardware und Laufzeiten (Schätzung, per Benchmark zu verifizieren)

**Empfehlung: 32 GB RAM.** Mit 16 GB funktioniert es, wenn die Schritte strikt
nacheinander laufen und ein 7–8 B-LLM genutzt wird. Weitere Anforderungen:
SSD, Linux (Debian/Ubuntu) + Docker, kabelgebundenes LAN.

Grobe Laufzeiten für **1 Stunde Meeting** auf einem aktuellen i7 **ohne GPU**:

| Schritt | Größenordnung |
|---|---|
| Whisper `large-v3-turbo` int8 | ca. 30–60 min |
| Whisper `medium` int8 | ca. 20–40 min |
| pyannote Diarization + Embeddings | ca. 5–20 min |
| LLM (8 B, Q4) für ca. 8 TOPs | ca. 10–30 min |
| **Gesamt** | **ca. 45–110 min** |

Für die nachträgliche Verarbeitung ist das akzeptabel (Ergebnis am selben
Tag). **Optionales Upgrade:** Eine gebrauchte GPU mit ≥ 12 GB VRAM
(z. B. RTX 3060 12 GB) beschleunigt alle Schritte grob um den Faktor 5–10 und
ermöglicht größere LLMs.

Hinweis: Die pyannote-Modelle erfordern eine einmalige Freigabe und einen
Download über Hugging Face. Danach laufen sie vollständig offline.

---

## 8. Automatisiertes Deployment (Debian / Fedora)

Voraussetzung: Debian (ab 12) oder Fedora (aktuelle Version) ist bereits
installiert und im Netz erreichbar. Ab dort läuft alles über **ein
Installationsskript**. Das Betriebssystem selbst wird nicht angefasst.

```bash
curl -fsSLO https://<intern>/openslides-minutes/install.sh
sudo bash install.sh --openslides-url https://openslides.firma.local \
                     --allow-from 10.0.20.15
```

### 8.1 Was das Skript macht

| # | Schritt | Debian | Fedora |
|---|---|---|---|
| 1 | Distribution erkennen | `/etc/os-release` → `apt` | `/etc/os-release` → `dnf` |
| 2 | Hardware prüfen | RAM, freier Speicher (≥ 60 GB), CPU mit AVX2 | wie Debian |
| 3 | Container-Laufzeit | Docker Engine + Compose-Plugin aus dem offiziellen Docker-Repo | Docker Engine aus dem offiziellen Repo (Podman als Option, siehe 8.4) |
| 4 | Verzeichnisse | `/opt/openslides-minutes` (Compose, `.env`), `/var/lib/openslides-minutes` (Modelle, Daten) | wie Debian, zusätzlich SELinux-Kontext (`:Z` an den Volumes) |
| 5 | Konfiguration erzeugen | `.env` mit zufälligem Secret (`openssl rand`), OpenSlides-URL, Löschfristen; **Modellwahl nach RAM** (< 24 GB → 7–8 B, sonst bis 14 B) | wie Debian |
| 6 | Firewall | `nftables`/`ufw`: Port des Dienstes nur für den OpenSlides-Server | `firewalld`: eigene Zone nur für den OpenSlides-Server |
| 7 | Dienste starten | `docker compose up -d` | wie Debian |
| 8 | Autostart | systemd-Unit `openslides-minutes.service` (startet Compose beim Booten) | wie Debian |
| 9 | Warten und prüfen | Health-Check, bis alle Modelle geladen sind (Fortschrittsanzeige) | wie Debian |
| 10 | Abschluss | gibt URL und Secret für die Eintragung in OpenSlides aus | wie Debian |

Das Skript ist **idempotent**: Ein erneuter Aufruf repariert oder aktualisiert
die Installation, ohne Daten oder Stimmprofile zu löschen.

### 8.2 Container-Aufbau (`docker-compose.yml`)

| Dienst | Aufgabe | Start |
|---|---|---|
| `ollama` | stellt die LLM bereit (nur intern erreichbar) | immer, `restart: unless-stopped` |
| `model-init` | Einmal-Container: `ollama pull <modell>`, lädt Whisper- und pyannote-Modelle, prüft Prüfsummen, beendet sich | nach `ollama`, bei jedem Start (überspringt vorhandene Modelle) |
| `minutes-service` | API, Queue, Pipeline | erst wenn `model-init` erfolgreich war (`depends_on: condition: service_completed_successfully`) |

Die Modelle liegen in einem persistenten Volume. Ein Neustart oder ein
Update lädt sie nicht erneut.

### 8.3 Betrieb

- **Update:** `sudo bash install.sh --update` holt neue Images und startet neu.
- **Modellwechsel:** `LLM_MODEL` in `.env` ändern, dann `--update`.
  `model-init` lädt das neue Modell, das alte kann mit `--prune-models`
  entfernt werden.
- **Status:** `install.sh --status` bzw. die Health-Anzeige in der
  OpenSlides-Organisationsverwaltung („KI-PC bereit, Modell X geladen“).
- **Deinstallation:** `install.sh --uninstall` (fragt, ob Daten und
  Stimmprofile gelöscht werden sollen).

### 8.4 Varianten und Grenzen

- **Podman statt Docker (Fedora):** Fedora bringt Podman mit. Möglich über
  `podman compose` oder Quadlet-Units für systemd. Das bedeutet einen zweiten
  Pfad zum Testen. Deshalb ist Docker auf beiden Systemen der Standard.
- **Offline-Installation:** Falls der KI-PC schon beim Einrichten kein
  Internet haben darf, baut `build-bundle.sh` auf einem Rechner mit
  Internet ein Paket mit Images und Modellen (grob 10–20 GB). Danach
  installiert `install.sh --offline bundle.tar` ohne Netzzugriff.
- **pyannote:** Der Download erfordert einmalig ein Hugging-Face-Token mit
  akzeptierten Nutzungsbedingungen (`--hf-token` beim Aufruf). Alternativ
  kommen die Modelle über das Offline-Paket; die Lizenz ist dafür vorher zu
  prüfen.
- **Mehrere KI-PCs:** Für mehrere Maschinen kann dieselbe Logik als
  Ansible-Rolle bereitgestellt werden.
- **Aufwand:** ca. 2–4 PT, enthalten in Phase 1 (Kapitel 12).

---

## 9. Sicherheit

- Der KI-PC steht nur im internen Netz (eigenes VLAN), ist nicht aus dem
  Internet erreichbar und hat nach der Modellinstallation keinen
  Internetzugang.
- TLS zwischen Proxy und KI-PC, Upload-Tokens mit kurzer Laufzeit und Scope.
- Die Festplatte ist verschlüsselt (LUKS). Stimmprofile werden zusätzlich auf
  Anwendungsebene verschlüsselt.
- **Automatische Löschung:** Das Audio wird nach der Freigabe bzw. spätestens
  nach X Tagen (konfigurierbar) gelöscht, das Feld `audio_available` wird
  aktualisiert.
- Audit-Log: Wer hat wann aufgenommen, korrigiert und freigegeben
  (OpenSlides-History).
- Backups: Die OpenSlides-Datenbank wie gehabt. Audio wird **nicht**
  gesichert, Stimmprofile nur verschlüsselt.

---

## 10. Konfiguration

- **Organisationsweit:** URL des Minutes-Service, Secret, Standardmodelle,
  Löschfrist, Schwellwerte der Sprechererkennung.
- **Pro Meeting:** Protokoll aktiviert (ja/nein), Protokollvorlage bzw.
  Prompt, Sprache, ob ein Wortprotokoll erstellt wird, ob die Stimmerkennung
  genutzt wird.

---

## 11. Recht und Datenschutz im Arbeitskontext (vor dem Produktivbetrieb klären)

> Keine Rechtsberatung, aber diese Punkte **müssen** vor dem Einsatz mit
> Datenschutzbeauftragten und gegebenenfalls Betriebsrat geklärt werden.

- **§ 201 StGB (Vertraulichkeit des Wortes):** Die Aufnahme des nicht
  öffentlich gesprochenen Wortes ohne Einwilligung ist strafbar. Deshalb:
  Einwilligung aller Anwesenden **vor jeder Aufnahme**, eine sichtbare Anzeige
  und eine Ansage. Im Client kann eine Checkbox „Alle Teilnehmenden wurden
  informiert und haben eingewilligt“ vor dem Start Pflicht sein.
- **DSGVO Art. 9:** Stimmprofile zur Identifikation sind **biometrische
  Daten** (besondere Kategorie). Nötig sind eine ausdrückliche, freiwillige,
  jederzeit widerrufbare Einwilligung und eine Alternative ohne Nachteile
  (ohne Profil erfolgt die Zuordnung eben manuell).
- **DSGVO Art. 35:** Eine Datenschutz-Folgenabschätzung ist sehr
  wahrscheinlich erforderlich.
- **BetrVG § 87 Abs. 1 Nr. 6:** Ein technisches System, das Verhalten oder
  Leistung erfassen *kann*, ist mitbestimmungspflichtig. Vermutlich ist eine
  **Betriebsvereinbarung** nötig (Zweckbindung, keine Leistungsauswertung,
  Löschfristen).
- **Teams-Richtlinien:** Die Aufzeichnung und der Export müssen im Tenant
  erlaubt sein. Die Teams-eigenen Aufzeichnungshinweise gelten zusätzlich.
- **Datenminimierung:** Audio und Rohtranskript werden nach der Freigabe
  gelöscht, nur das freigegebene Protokoll bleibt. Das Wortprotokoll ist
  optional.

---

## 12. Umsetzungsphasen

Aufwände sind grobe Schätzungen in Personentagen (PT) für eine Person, die
OpenSlides bereits kennt.

| Phase | Inhalt | Ergebnis | Aufwand |
|---|---|---|---|
| **0 – Proof of Concept** | Standalone-Skript auf dem KI-PC: Datei → Whisper → pyannote → Stimmprofil-Abgleich → LLM → Markdown. Tests mit echten Aufnahmen (Raum + Teams). | Belastbare Aussagen zu Qualität, Laufzeit, Modellwahl und Mikrofon. **Go/No-Go.** | 5–8 PT |
| **1 – Service + Datenmodell** | `openslides-minutes-service` (API, Queue, Pipeline aus Phase 0), neue Collections/Actions/Rechte/Migration, Proxy-Route, Installationsskript für Debian/Fedora (Kapitel 8) | Datei-Upload in OpenSlides → Transkript sichtbar | 12–19 PT |
| **2 – Aufnahme-Button** | Browser-Aufnahme mit Chunk-Upload, TOP-Marker, Einwilligungsdialog | Vor-Ort-Meetings direkt aus OpenSlides | 5–8 PT |
| **3 – Stimmprofile** | Enrollment-UI, Embedding-Speicher, automatische Zuordnung, Korrektur im Review | Automatische Sprechererkennung | 5–8 PT |
| **4 – Protokoll & Review** | LLM-Schritt pro TOP, Review-Editor, Neu-Generieren, Freigabe, PDF-Export | Vollständiger Protokoll-Workflow | 10–15 PT |
| **5 – Hybrid & Feinschliff** | Teams-VTT-Import, Systemton-Aufnahme, Löschfristen, Monitoring, Doku | Produktionsreife | 5–10 PT |

**Gesamt: ca. 42–69 PT.** Phase 0 ist bewusst vorgeschaltet: Wenn die
Transkriptionsqualität mit dem vorhandenen Mikrofon oder die Laufzeit auf der
CPU nicht reicht, sollte das vor dem Integrationsaufwand feststehen.

---

## 13. Architekturentscheidungen und Risiken

**Wartung des Forks (wichtigstes Risiko).** Dieses Repository bindet
`openslides-backend`, `openslides-client`, `meta` und `openslides-proxy` als
Submodule der **Upstream-Repos** ein. Die Integration erfordert **eigene Forks**
dieser Repos und das regelmäßige Nachziehen von Upstream-Updates (Migrationen,
Datenmodell). Gegenmaßnahmen:

- Die Änderungen möglichst **additiv** halten: neue Collections, neues
  Client-Modul, keine Änderungen an bestehender Logik.
- Die gesamte KI-Logik liegt im separaten Minutes-Service. Dieser ist
  unabhängig von OpenSlides-Releases.
- Mittelfristig lohnt sich der Kontakt zum OpenSlides-Team (Intevation), ob
  ein Plugin- bzw. Erweiterungspunkt oder eine Upstream-Integration denkbar ist.

**Alternative „leichte Integration“ (falls der Fork-Aufwand zu hoch ist):**
Kein neues Datenmodell. Der Minutes-Service arbeitet als eigene kleine Web-App
mit OpenSlides-Login und schreibt das freigegebene Protokoll als PDF-Mediafile
bzw. als Text in einen Topic „Protokoll“ ins Meeting. Das bedeutet weniger
Wartung, aber keinen Button direkt in OpenSlides und keine Live-Statusanzeige
über Autoupdate.

**Weitere Risiken:**

- **Audioqualität im Raum:** ein größerer Einflussfaktor als die Modellwahl →
  ein Konferenzmikrofon einplanen.
- **CPU-Laufzeit:** Mehrere Meetings pro Tag stauen sich in der Queue →
  Priorisierung, gegebenenfalls GPU nachrüsten.
- **LLM-Halluzinationen bei Beschlüssen und Aufgaben:** Quellverweise sowie die
  Pflicht zu Review und Freigabe.
- **Browser-Aufnahme:** Tab-Schließen oder Standby → IndexedDB-Puffer,
  Wake Lock, Warnungen.
- **Akzeptanz und Mitbestimmung:** frühzeitig Betriebsrat und
  Datenschutzbeauftragte einbinden (Kapitel 11).

---

## 14. Offene Fragen

1. Wie viele Meetings pro Woche und wie lang sind sie? (Queue-Auslegung, GPU ja/nein)
2. Gibt es eine verbindliche Protokollvorlage (Aufbau, Pflichtfelder) im Unternehmen?
3. Wer darf aufnehmen, wer darf freigeben? (Mapping auf OpenSlides-Gruppen)
4. Soll das freigegebene Protokoll an die Teilnehmenden verteilt werden (E-Mail über OpenSlides)?
5. Welche Löschfristen gelten für Audio, Transkript und Protokoll?
6. Ist ein Fork von Backend und Client langfristig tragbar oder wird die leichte Integration bevorzugt?
