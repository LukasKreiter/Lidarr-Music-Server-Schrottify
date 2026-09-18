# Projektplan: Navidrome Music Uploader

**Spring-Boot-Dashboard zum Suchen, Herunterladen und Einpflegen von Musik in eine bestehende Navidrome-Bibliothek**

Stand: 18.09.2026 · Autor: Nico

---

## 1. Ziel in einem Satz

Ein selbst gehostetes Web-Dashboard, über das man — auch vom Smartphone, ohne lokale Dateien — einen Künstler/Album/Track sucht, per Klick einen Download-Job startet, und die Applikation lädt die Dateien über einen Soulseek-Client (slskd) herunter, taggt sie, legt sie korrekt benannt in der Navidrome-Bibliothek ab und stößt den Navidrome-Scan an.

---

## 2. Bestätigte Entscheidungen

| Thema | Entscheidung |
|---|---|
| Quelle | Soulseek, via **slskd** (Daemon mit REST-API + SignalR) |
| Topologie | **Alles auf demselben Ubuntu-Server**: slskd, Spring-Boot-App, Navidrome |
| Dateitransfer | Kein SSH/SFTP — direkter Dateisystemzugriff, Import per `move` innerhalb desselben Mounts |
| Trefferauswahl | **Automatisch nach Regeln** (Scoring), mit Fallback auf den nächstbesten Treffer |
| MVP-Umfang | Persistente Job-Queue (DB), Retry, Job-Historie, Live-Status via HTMX, MusicBrainz-Tagging + Cover |
| Frontend | Thymeleaf + HTMX, mobil-first, als PWA installierbar |
| Login | Durchgereicht an Navidrome (keine eigene Benutzerverwaltung) |

> Damit entfällt die frühere Annahme „App auf separatem Host, Transfer per SFTP". Das vereinfacht Fehlerbehandlung, Rechte und Performance erheblich.

---

## 3. Rechtlicher Rahmen

slskd und das Soulseek-Protokoll sind legitime Open-Source-Software; das Netz wird jedoch überwiegend zum Austausch urheberrechtlich geschützter Aufnahmen genutzt. Downloads erfolgen auf eigene Verantwortung, und je nach Rechtslage kann auch das automatische Weiter-Teilen (Upload-Slots) relevant sein. Die Architektur kapselt die Quelle deshalb hinter einem `SourceProvider`-Interface — slskd ist eine Implementierung, weitere (Jamendo, Internet Archive, eigene Nextcloud/S3, Direkt-URL) lassen sich ohne Änderung der Kernlogik ergänzen.

---

## 4. Requirements

### 4.1 Funktionale Anforderungen

| ID | Anforderung | Prio |
|---|---|---|
| FR-1 | Nutzer meldet sich mit seinen **Navidrome-Zugangsdaten** an; Validierung gegen die Navidrome-API | MUSS |
| FR-2 | Suchfeld für Freitext (`Artist – Album` oder `Artist – Track`); optional Modus „Album" / „Track" | MUSS |
| FR-3 | App fragt **MusicBrainz** ab, um die Eingabe zu einem kanonischen Release/Recording aufzulösen (Artist, Albumtitel, Tracklist, Jahr, MBIDs) | MUSS |
| FR-4 | App prüft vor dem Download, ob das Release **bereits in Navidrome existiert** (Duplikatschutz) | MUSS |
| FR-5 | App startet eine Soulseek-Suche über slskd und sammelt Treffer über ein Zeitfenster (z. B. 15 s) | MUSS |
| FR-6 | **Automatisches Scoring** der Treffer (Format, Bitrate, Vollständigkeit, freie Slots, Queue-Länge, Namensähnlichkeit); bester Kandidat wird gewählt | MUSS |
| FR-7 | Bei Fehlschlag/Timeout automatischer Fallback auf den nächstbesten Kandidaten (max. N Versuche) | MUSS |
| FR-8 | Jobs werden **persistent** gespeichert; Neustart der App verliert keinen Job | MUSS |
| FR-9 | Dashboard zeigt alle Jobs mit Status, Fortschritt (%), Geschwindigkeit, Fehlermeldung; Aktualisierung ohne Reload | MUSS |
| FR-10 | Heruntergeladene Dateien werden **getaggt** (ID3v2.4 / Vorbis) aus MusicBrainz-Daten und mit Cover aus dem Cover Art Archive versehen | MUSS |
| FR-11 | Ablage nach Schema `<library>/<AlbumArtist>/<Jahr> - <Album>/<NN> - <Titel>.<ext>` | MUSS |
| FR-12 | Nach erfolgreichem Import wird ein **Navidrome-Scan** angestoßen | MUSS |
| FR-13 | Job manuell abbrechen, wiederholen (Retry) und löschen | MUSS |
| FR-14 | Job-Historie mit Filter (Status, Zeitraum) und Detailansicht inkl. Log | SOLL |
| FR-15 | Optionaler Upload eigener Dateien über dasselbe Dashboard (gleiche Import-Pipeline) | SOLL |
| FR-16 | Als PWA installierbar, Startbildschirm-Icon, funktioniert auf Mobilgeräten | SOLL |
| FR-17 | Lyrics durch .LRC Dateien | SOLL |
| FR-18 | Einstellungsseite: Formatpräferenz, Mindest-Bitrate, Suchfenster, Scan-Trigger an/aus | SOLL |
| FR-19 | Watchlist: fehlgeschlagene Suchen periodisch erneut versuchen | KANN |

### 4.2 Nicht-funktionale Anforderungen

| ID | Anforderung |
|---|---|
| NFR-1 | **Sicherheit**: nur die App ist von außen erreichbar (Reverse Proxy + TLS); slskd-Web-UI und Navidrome-Port bleiben auf `127.0.0.1` |
| NFR-2 | Keine Passwörter im Klartext persistieren; slskd-API-Key und Navidrome-URL aus Environment/`.env`, nie im Repo |
| NFR-3 | **Dateirechte**: Import erzeugt Dateien mit `0644`/Verzeichnisse `0755`, Eigentümer-Gruppe = Navidrome-Gruppe |
| NFR-4 | Import ist **atomar**: erst in temporäres Verzeichnis auf demselben Dateisystem, dann `rename` an den Zielpfad |
| NFR-5 | **Idempotenz**: ein wiederholter Job für dasselbe Release darf keine Duplikate erzeugen |
| NFR-6 | MusicBrainz-Rate-Limit einhalten (max. 1 Request/s, aussagekräftiger User-Agent) |
| NFR-7 | Ressourcen: läuft zusammen mit Navidrome auf kleiner Hardware — Heap-Limit gesetzt, keine In-Memory-Pufferung ganzer Dateien |
| NFR-8 | Beobachtbarkeit: strukturierte Logs, `/actuator/health`, Metriken für Job-Durchsatz und Fehlerquote |
| NFR-9 | Testbarkeit: slskd und Navidrome in Tests gemockt (WireMock), DB via Testcontainers |
| NFR-10 | Ein einziger App-Prozess; keine verteilte Koordination nötig, aber Queue-Zugriff trotzdem mit `SELECT … FOR UPDATE SKIP LOCKED` |

---

## 5. Architektur

### 5.1 Komponenten auf dem Ubuntu-Server

```
                    ┌──────────────── Ubuntu Server ────────────────┐
  Smartphone  ──►   │  Caddy/nginx  :443  (TLS, Auth-Proxy optional)│
   Browser          │        │                                      │
                    │        ▼                                      │
                    │  music-uploader (Spring Boot)  :8080          │
                    │     │            │              │             │
                    │     │ REST       │ Subsonic API │ Filesystem  │
                    │     ▼            ▼              ▼             │
                    │  slskd :5030   Navidrome :4533   /srv/music   │
                    │     │                              ▲          │
                    │     └── /srv/slskd/downloads ───────┘ (move)  │
                    │                                               │
                    │  extern: MusicBrainz WS/2, Cover Art Archive  │
                    └───────────────────────────────────────────────┘
```

Wichtig: `/srv/slskd/downloads` und `/srv/music` liegen **auf demselben Dateisystem**, damit der Import ein reines `rename()` ist (atomar, keine Kopie, kein halbfertiger Track im Scan).

### 5.2 Paketstruktur (Java)

```
de.nico.musicuploader
├── MusicUploaderApplication.java
├── config/            SecurityConfig, RestClientConfig, AsyncConfig, AppProperties
├── web/
│   ├── controller/    DashboardController, SearchController, JobController, SettingsController
│   ├── dto/           SearchFormDto, JobViewDto, CandidateViewDto
│   └── fragment/      (Thymeleaf-Fragmente für HTMX-Teilrenderings)
├── domain/
│   ├── job/           DownloadJob, JobItem, JobStatus, JobEvent, Repositories
│   └── settings/      AppSetting, SettingsService
├── source/            SourceProvider (Interface), Candidate, SearchQuery
│   └── slskd/         SlskdClient, SlskdSourceProvider, SlskdProperties, dto/
├── metadata/          MusicBrainzClient, CoverArtClient, ReleaseMetadata, TagWriter
├── library/           LibraryPathResolver, ImportService, DuplicateDetector, FilePermissions
├── navidrome/         NavidromeClient (Subsonic), NavidromeAuthenticator, ScanTrigger
├── orchestration/     JobQueueService, JobWorker, CandidateScorer, RetryPolicy
└── support/           Slugify, Fuzzy (Levenshtein/Jaro), AudioFileFilter
```

### 5.3 Technologie-Stack

| Ebene | Wahl | Begründung |
|---|---|---|
| Java | 21 (LTS) | Virtual Threads für die vielen blockierenden HTTP-/IO-Warteschleifen |
| Framework | Spring Boot 3.5.x | aktuell, Support für Java 21, `RestClient` |
| Build | Gradle (Kotlin DSL) | schnelle inkrementelle Builds; Maven ebenso möglich |
| Web | Spring MVC + Thymeleaf + HTMX 2 | serverseitiges Rendering, kein JS-Buildstep |
| Security | Spring Security, Form-Login mit eigenem `AuthenticationProvider` gegen Navidrome | FR-1 |
| Persistenz | Spring Data JPA + **PostgreSQL 16** (Docker) | robuste `SKIP LOCKED`-Queue; H2-File als Leichtgewicht-Alternative |
| Migration | Flyway | versionierte Schema-Änderungen |
| HTTP-Client | `RestClient` + Resilience4j (Retry, CircuitBreaker, RateLimiter) | NFR-6 |
| Tagging | **jaudiotagger** (FLAC/MP3/OGG) — ffmpeg als Fallback | Cover + Tags einbetten |
| Async | `@Async` mit Virtual-Thread-Executor + `@Scheduled`-Poller | einfache, transparente Queue |
| Live-Update | HTMX `hx-trigger="every 2s"` auf ein Fragment; später SSE | robust hinter Reverse Proxy |
| Tests | JUnit 5, Testcontainers (Postgres), WireMock (slskd/Navidrome/MusicBrainz) | NFR-9 |
| Deployment | Docker Compose (App + Postgres) neben bestehendem slskd/Navidrome, oder systemd-Unit mit Fat-JAR | Wahl nach bestehendem Setup |

---

## 6. Datenmodell

```
download_job
  id              uuid        PK
  created_at      timestamptz
  updated_at      timestamptz
  created_by      text            -- Navidrome-Username
  query_raw       text            -- was der Nutzer eingegeben hat
  mode            text            -- ALBUM | TRACK
  mb_release_id   text null       -- MusicBrainz Release-MBID
  mb_artist_id    text null
  artist          text
  album           text null
  title           text null
  year            int null
  status          text            -- siehe Statusmaschine
  attempt         int  default 0
  max_attempts    int  default 3
  error_message   text null
  target_path     text null       -- finaler Ordner in der Library
  finished_at     timestamptz null

job_item                            -- eine Datei innerhalb eines Jobs
  id              uuid        PK
  job_id          uuid        FK -> download_job
  track_number    int null
  track_title     text null
  source_user     text            -- Soulseek-Peer
  source_filename text            -- Remote-Pfad
  size_bytes      bigint
  bitrate         int null
  format          text            -- FLAC | MP3 | OGG | …
  status          text
  bytes_done      bigint default 0
  local_path      text null       -- Pfad im slskd-Downloadordner
  slskd_transfer_id text null

job_event                           -- Audit-/Log-Trail pro Job
  id              bigserial   PK
  job_id          uuid        FK
  at              timestamptz
  level           text            -- INFO | WARN | ERROR
  message         text

candidate                           -- verworfene und gewählte Treffer, für Fallback + Nachvollziehbarkeit
  id              uuid        PK
  job_id          uuid        FK
  rank            int
  username        text
  score           numeric
  file_count      int
  total_bytes     bigint
  avg_bitrate     int null
  format          text
  free_slots      bool
  queue_length    int
  upload_speed    bigint
  payload         jsonb           -- vollständige slskd-Antwort
  chosen          bool default false

app_setting
  key             text        PK
  value           text
```

Indizes: `download_job(status, created_at)`, `job_item(job_id)`, `candidate(job_id, rank)`, Unique auf `download_job(mb_release_id)` für `status IN (IMPORTED, DONE)` (Duplikatschutz, FR-4/NFR-5).

---

## 7. Statusmaschine eines Jobs

```
QUEUED
  └─► RESOLVING_METADATA      (MusicBrainz: Query → Release + Tracklist)
        └─► DUPLICATE_CHECK   (Navidrome search3)
              ├─► SKIPPED_DUPLICATE  (Endzustand)
              └─► SEARCHING          (slskd-Suche, Sammelfenster)
                    ├─► NO_RESULTS   (Endzustand / Watchlist)
                    └─► DOWNLOADING  (bester Kandidat enqueued)
                          ├─► DOWNLOAD_FAILED ─► SEARCHING (nächster Kandidat, attempt++)
                          └─► IMPORTING        (Dateien verifizieren, in temp verschieben)
                                └─► TAGGING    (Tags + Cover schreiben)
                                      └─► PUBLISHING  (rename in Library, Rechte setzen)
                                            └─► SCANNING (Navidrome-Scan anstoßen)
                                                  └─► DONE
Jeder Zustand ─► FAILED (bei erschöpften Versuchen)   |   ─► CANCELLED (durch Nutzer)
```

Regeln:
- Retry nur aus `DOWNLOAD_FAILED`, `SEARCHING`, `SCANNING` — nie mitten im Dateisystem-Schreiben.
- `attempt` zählt Kandidaten-Fallbacks, nicht Netzwerk-Retries (die macht Resilience4j intern).
- Ein Worker hält einen Job per `SKIP LOCKED`; ein Watchdog setzt Jobs, die länger als X Minuten in einem Nicht-Endzustand hängen, auf `FAILED` zurück (Stale-Job-Recovery nach Crash).

---

## 8. Scoring der Soulseek-Treffer (FR-6)

Für jeden Peer-Treffer (`candidate`) wird ein gewichteter Score berechnet. Startwerte, später über die Einstellungsseite justierbar:

| Kriterium | Gewicht | Berechnung |
|---|---|---|
| Format | 30 | FLAC = 1,0 · MP3 ≥ 320 = 0,8 · MP3 ≥ 256 = 0,6 · MP3 ≥ 192 = 0,4 · darunter = 0,1 · exotisch = 0 |
| Vollständigkeit (Album-Modus) | 25 | gefundene Tracks / erwartete Tracks laut MusicBrainz; unter 0,9 harter Malus |
| Namensähnlichkeit | 15 | Jaro-Winkler von normalisiertem `Artist Album Track` gegen Dateipfad |
| Freier Upload-Slot | 12 | `freeUploadSlots > 0` → 1,0, sonst 0,2 |
| Queue-Länge | 8 | `1 / (1 + queueLength/10)` |
| Upload-Geschwindigkeit | 6 | logarithmisch normalisiert |
| Konsistenz | 4 | alle Dateien gleiches Format/gleicher Ordner → 1,0 |

Harte Filter **vor** dem Scoring:
- nur Audio-Endungen (`.flac .mp3 .m4a .ogg .opus .wav`), alles andere (`.cue`, `.log`, `.nfo`, `.jpg`) fließt nicht in die Bewertung ein, wird aber bei FLAC-Alben optional mitgeladen;
- unplausible Dateigrößen (z. B. < 500 KB pro Track) verwerfen — typische Fake-/Werbedateien;
- Mindest-Bitrate aus den Einstellungen.

Die Top-N (z. B. 5) Kandidaten werden gespeichert, damit der Fallback bei Abbruch sofort greift, ohne neu zu suchen.

---

## 9. Integrationen im Detail

### 9.1 slskd

- Läuft als Docker-Container oder systemd-Dienst, Web-API auf `127.0.0.1:5030`.
- Authentifizierung über **API-Key** (Header `X-API-Key`) — in slskd unter `web.authentication.api_keys` konfigurieren, an die App per Environment-Variable übergeben.
- Ablauf: Suche starten → Ergebnisse pollen bzw. über den SignalR-Hub empfangen → Download für den gewählten Peer + Dateiliste einreihen → Transferstatus pollen bis `Completed`/`Errored`.
- slskd legt fertige Dateien in seinem Download-Verzeichnis ab; die App liest dort und verschiebt in die Library.
- **Vor der Implementierung**: die exakten Endpunktpfade gegen die laufende Instanz verifizieren (slskd bringt eine Swagger-/OpenAPI-Oberfläche mit). Der Adapter kapselt sie, sodass eine Abweichung nur `SlskdClient` betrifft.
- Sinnvolle slskd-Einstellungen: eigenes Download-Verzeichnis, `remote_file_management` aus, Shares bewusst konfigurieren.

### 9.2 Navidrome

- **Login-Durchreichung (FR-1)**: eigener `AuthenticationProvider` ruft die Subsonic-Ping-Operation mit `u` + Salt/Token-Verfahren auf; HTTP 200 mit `status="ok"` ⇒ Anmeldung gültig. Das Navidrome-Passwort wird nur für diesen Aufruf gehalten und nicht persistiert; die Session merkt sich lediglich den Benutzernamen.
- **Duplikatprüfung (FR-4)**: Subsonic-Suche (`search3`) nach Artist + Album; zusätzlich Abgleich über die MusicBrainz-ID, falls Navidrome sie liefert.
- **Scan-Trigger (FR-12)**: Subsonic-Operation `startScan`, Fortschritt über `getScanStatus`. Alternativ auf Navidromes eigenen Datei-Watcher verlassen und den Trigger in den Einstellungen abschaltbar machen.
- Schreibrechte: die App muss in die Library schreiben dürfen — dedizierter Systembenutzer `musicuploader` in der Gruppe von Navidrome, Library-Verzeichnis `g+ws`.

### 9.3 MusicBrainz / Cover Art Archive

- Aufruf mit eigenem User-Agent (`music-uploader/1.0 ( kontakt )`), Rate-Limiter auf 1 Request/s, Ergebnisse in der DB cachen.
- Release-Auswahl: bevorzugt offizielle Alben, primärer Release-Typ, passendes Land/Jahr; die Tracklist liefert die Soll-Struktur für Vollständigkeitsprüfung und Dateinamen.
- Cover in bis zu 1000 px laden, in die Dateien einbetten **und** zusätzlich als `cover.jpg` im Albumordner ablegen (Navidrome nutzt beides).

---

## 10. Import-Pipeline

1. **Verifizieren** — Datei existiert, Größe stimmt mit dem angekündigten Wert überein, Audio-Header lesbar.
2. **In Arbeitsverzeichnis verschieben** — `/srv/music/.incoming/<jobId>/` (auf demselben Dateisystem wie die Library, damit alles `rename` bleibt).
3. **Tracks zuordnen** — Dateireihenfolge gegen die MusicBrainz-Tracklist matchen (Tracknummer aus Dateinamen, sonst Titelähnlichkeit, sonst Sortierreihenfolge).
4. **Taggen** — Artist, AlbumArtist, Album, Titel, Tracknummer/-gesamt, Discnummer, Jahr, Genre, MBIDs; Cover einbetten.
5. **Umbenennen** — `<AlbumArtist>/<Jahr> - <Album>/<NN> - <Titel>.<ext>`, Sonderzeichen ersetzen, Länge auf 255 Bytes begrenzen, Unicode-Normalisierung NFC.
6. **Rechte setzen** — Dateien `0644`, Ordner `0755`, Gruppe = Navidrome-Gruppe.
7. **Veröffentlichen** — Albumordner per `rename` an den Zielpfad; existiert er bereits, Kollisionsstrategie aus den Einstellungen (überspringen / `(2)` anhängen / ersetzen).
8. **Aufräumen** — Arbeitsverzeichnis löschen, slskd-Downloadordner leeren.
9. **Scan anstoßen** und Job auf `DONE` setzen.

Fehler in Schritt 4–7 rollen das Arbeitsverzeichnis zurück; die Library wird nie in einem Zwischenzustand sichtbar.

---

## 11. UI / Dashboard

| Seite | Inhalt |
|---|---|
| `/login` | Navidrome-Benutzername + Passwort |
| `/` | Großes Suchfeld (mobil-first), darunter „Aktive Jobs" als HTMX-Fragment mit Auto-Refresh |
| `/search` | Auflösung der Eingabe über MusicBrainz: Trefferliste mit Cover, Artist, Album, Jahr, Trackzahl → Button „Auf Server laden" |
| `/jobs` | Historie mit Statusfilter, Suche, Paginierung |
| `/jobs/{id}` | Detail: Kandidaten mit Score, Dateiliste, Fortschritt, Event-Log, Aktionen (Abbrechen, Retry, Löschen) |
| `/settings` | Formatpräferenz, Mindest-Bitrate, Suchfenster, max. Kandidaten, Scan-Trigger, Kollisionsstrategie, Pfade |
| `/upload` | Optionaler Datei-Upload (FR-15), landet in derselben Pipeline |

HTMX-Muster: Fortschrittstabelle als eigenes Fragment, das per `hx-get="/jobs/active" hx-trigger="every 2s"` aktualisiert wird — kein JS-Framework, funktioniert zuverlässig auf Mobilfunk. PWA-Manifest + Service Worker nur fürs Icon und einen Offline-Hinweis; keine Offline-Funktionalität nötig.

---

## 12. Umsetzungsphasen

### Phase 0 — Server-Inventur (½ Tag)
- Navidrome-Version, Library-Pfad, Benutzer/Gruppe, Datei-Watcher an/aus prüfen.
- slskd installieren bzw. Version prüfen, API-Key anlegen, Download-Verzeichnis auf dasselbe Dateisystem wie die Library legen.
- Manueller Rauchtest: Suche und Download über die slskd-Web-UI, Datei per Hand in die Library legen, Scan auslösen. **Erst wenn das von Hand funktioniert, lohnt Code.**
- Endpunkte von slskd und Navidrome per `curl` dokumentieren → Grundlage für die Adapter.

### Phase 1 — Projekt-Setup (½ Tag)
- Spring Initializr: Web, Thymeleaf, Security, Data JPA, Validation, Flyway, Actuator, Testcontainers.
- Git-Repo, `.gitignore`, `application.yml` + `application-local.yml`, Secrets ausschließlich über Environment.
- Docker Compose: App + Postgres; Healthchecks.
- CI (GitHub Actions oder lokal): Build + Tests.
- **Ergebnis**: „Hello Dashboard" läuft unter `:8080`.

### Phase 2 — Auth + Navidrome-Adapter (1 Tag)
- `NavidromeClient` (Ping, search3, startScan, getScanStatus) mit WireMock-Tests.
- `AuthenticationProvider` gegen Navidrome, Form-Login, Session, CSRF.
- **Ergebnis**: Login mit Navidrome-Account funktioniert, Scan lässt sich per Button auslösen.

### Phase 3 — Metadaten (1 Tag)
- `MusicBrainzClient` mit Rate-Limiter + Cache, `CoverArtClient`.
- Such-Seite: Eingabe → Release-Kandidaten mit Cover.
- **Ergebnis**: Ein Album lässt sich eindeutig auflösen; Tracklist liegt vor.

### Phase 4 — slskd-Adapter (1–2 Tage)
- `SlskdClient`: Suche starten, Ergebnisse einsammeln, Download einreihen, Transferstatus abfragen, Transfer abbrechen.
- `CandidateScorer` inkl. Unit-Tests mit echten, abgespeicherten Antwort-Payloads.
- **Ergebnis**: Konsolennah (Integrationstest) lässt sich ein Album auf die Platte holen.

### Phase 5 — Job-Queue + Orchestrierung (1–2 Tage)
- Flyway-Migrationen, Entities, Repositories.
- `JobQueueService` (`SKIP LOCKED`), `JobWorker` mit der Statusmaschine, Retry-/Fallback-Logik, Stale-Job-Watchdog.
- **Ergebnis**: Job überlebt Neustart, läuft bis `DOWNLOADING` durch.

### Phase 6 — Import-Pipeline (1–2 Tage)
- Verifikation, Track-Matching, `TagWriter`, `LibraryPathResolver`, Rechte, atomares `rename`, Kollisionsstrategie, Cleanup.
- Tests auf einem temporären Verzeichnisbaum (kein Mock-Dateisystem — echte Semantik von `rename` und Rechten ist der Punkt).
- **Ergebnis**: Ein Job läuft vollständig bis `DONE`, das Album taucht in Navidrome korrekt getaggt auf.

### Phase 7 — UI ausbauen (1–2 Tage)
- Dashboard, Job-Detail mit Event-Log, Historie mit Filter, Einstellungsseite, HTMX-Polling, mobiles Layout, PWA-Manifest.
- **Ergebnis**: vom Smartphone bedienbar, Status live sichtbar.

### Phase 8 — Härtung & Deployment (1 Tag)
- Reverse Proxy mit TLS, Rate-Limit auf `/login`, Ports von slskd/Navidrome auf localhost binden.
- systemd-Unit bzw. Compose-Service mit Restart-Policy, Logrotation, Backup der Postgres-DB.
- Actuator-Health, Metriken, Alarm bei Fehlerquote.
- Runbook: Was tun, wenn ein Job hängt / slskd offline ist / die Library voll ist.

### Phase 9 — Optional
Watchlist für erfolglose Suchen · Duplikaterkennung über akustische Fingerprints (AcoustID) · zweiter `SourceProvider` (Internet Archive, eigene Nextcloud) · Album-Batch aus einer Wunschliste · Web-Push bei fertigem Job.

**Grobe Gesamtdauer**: rund 8–12 Arbeitstage bis zu einer belastbaren Version; das lauffähige Skelett (Phase 1–5) steht typischerweise nach 4–5 Tagen.

---

## 13. Risiken & Gegenmaßnahmen

| Risiko | Auswirkung | Gegenmaßnahme |
|---|---|---|
| Soulseek-Peer geht offline / Queue ewig lang | Job hängt | Timeout pro Kandidat, automatischer Fallback auf Rang 2–5, Watchdog |
| Falsch benannte oder gefälschte Dateien | Müll in der Library | Harte Filter, Größen-Plausibilität, Header-Prüfung, Vollständigkeitsabgleich gegen MusicBrainz |
| slskd-API ändert sich zwischen Versionen | Adapter bricht | Alles in `SlskdClient` gekapselt, Contract-Tests mit gespeicherten Payloads, slskd-Version pinnen |
| MusicBrainz-Rate-Limit / Ausfall | Jobs stocken | Rate-Limiter, Cache, Degradationspfad: ohne Metadaten importieren und Tagging später nachholen |
| Navidrome scannt halbfertige Dateien | Kaputte Einträge | Import ausschließlich per `rename` aus `.incoming/` auf demselben Dateisystem |
| Dateirechte falsch | Navidrome sieht nichts | Fester umask im Service, Rechte im Import explizit setzen, Smoke-Test im Runbook |
| Dashboard öffentlich erreichbar | Fremdzugriff | Navidrome-Login erzwungen, TLS, Login-Rate-Limit, optional zusätzlich VPN/Tailscale statt Portfreigabe |
| Platte läuft voll | Import schlägt mitten drin fehl | Vor dem Enqueue freien Speicher prüfen, Schwellwert-Warnung im Dashboard |

---

## 14. Nächste konkrete Schritte

1. Phase 0 durchziehen: slskd-Version, API-Key, Pfade, Navidrome-Version und Library-Pfad festhalten — und einmal **von Hand** ein Album durchschleusen.
2. Die tatsächlichen slskd-Endpunkte per `curl` mitschneiden und als Testfixtures ablegen.
3. Entscheidung Postgres vs. H2-File treffen (Postgres empfohlen, wenn ohnehin Docker läuft).
4. Repository anlegen und mit Phase 1 starten.

Wenn Phase 0 steht, kann ich das Projektgerüst inklusive Adapter, Flyway-Migrationen und Statusmaschine direkt ausbauen.
