## Plan: Navidrome Uploader-Dashboard (Spring Boot)

Basierend auf deinen Antworten: externe Metadatensuche, App auf separatem Host mit SFTP-Transfer, Thymeleaf+HTMX, Login gegen Navidrome.

---

### 0. Scope-Abgrenzung (wichtig vorab)

Die App **lädt keine Audiodateien aus dem Internet**. „Suche in externen Quellen" heißt: Metadaten-Recherche (MusicBrainz, Cover Art Archive) um Releases zu identifizieren, Tracklisten und Cover zu holen und deine Dateien sauber zu taggen. Die Audiodateien liefert der Nutzer per Browser-Upload.

---

### 1. Requirements

**Funktional**

| ID | Anforderung | Akzeptanzkriterium |
|---|---|---|
| FR-1 | Login mit Navidrome-Zugangsdaten | Falsche Credentials → 401, keine Session; korrekte → Dashboard |
| FR-2 | Suche nach Artist / Release / Track in MusicBrainz | Trefferliste mit Artist, Titel, Jahr, Format, Land |
| FR-3 | Release-Detail mit Tracklist + Cover (CAA) | Tracknummern, Titel, Dauer, MBIDs, Coverbild |
| FR-4 | Abgleich mit Bibliothek | Pro Treffer Badge „in Bibliothek" / „fehlt" via Subsonic `search3` |
| FR-5 | Upload mehrerer Dateien (Drag & Drop) | Fortschrittsanzeige je Datei, erlaubte Formate konfigurierbar |
| FR-6 | Tag-Zuordnung vor Transfer | Datei ↔ Track-Mapping vorschlagen, manuell korrigierbar; Tags werden geschrieben |
| FR-7 | Zielpfad nach Namensschema | Vorschau des Zielpfads vor dem Transfer (Dry-Run) |
| FR-8 | SFTP-Transfer mit Staging | Upload nach `.incoming/<job-id>/`, dann atomares Rename in die Library |
| FR-9 | Scan-Trigger | Nach Job-Abschluss `startScan`, Status via `getScanStatus` pollen |
| FR-10 | Job-Historie | Liste aller Uploads mit Status (QUEUED/RUNNING/DONE/FAILED), Fehlertext, Retry |
| FR-11 | Kollisionsschutz | Existierende Zieldatei → Job stoppt und fragt (Skip / Ersetzen / Umbenennen) |

**Nicht-funktional**

- Upload-Jobs laufen asynchron und **neustart-fest** (Queue in DB, nicht im RAM)
- Dateigrößen bis ~200 MB/Datei, ~2 GB/Job (FLAC-Alben)
- Keine Klartext-Passwörter in DB oder Logs
- MusicBrainz: max. 1 Request/Sekunde + Caching (harte API-Regel) + aussagekräftiger User-Agent
- Audit: wer hat wann welche Datei wohin geschrieben
- Deployment als Fat-JAR + systemd-Unit oder Docker-Image

---

### 2. Architektur

```
Browser (Thymeleaf + HTMX)
        │ HTTPS, Session-Cookie
┌───────▼──────────────────────────────────────────┐
│ Spring Boot App (Host B)                         │
│                                                  │
│  web/        Controller, HTMX-Fragmente          │
│  security/   SubsonicAuthenticationProvider      │
│  metadata/   MusicBrainzClient, CoverArtClient   │
│  library/    SubsonicClient (search3, scan)      │
│  ingest/     TagWriter, PathResolver, JobService │
│  transfer/   SftpTransferService                 │
│  persistence/ JPA + Flyway                       │
└──┬───────────────┬────────────────────┬──────────┘
   │ HTTPS         │ HTTPS /rest/*      │ SSH/SFTP :22
   ▼               ▼                    ▼
MusicBrainz    Navidrome (Host A)   Musikordner (Host A)
```

**Upload-Sequenz**

1. Browser → Multipart-Upload → temporäres Verzeichnis der App
2. Validierung: Magic Bytes + Extension + Größe (nicht nur Content-Type vertrauen)
3. Tags lesen (jaudiotagger), Vorschlag zum Mapping auf gewählten Release
4. Nutzer bestätigt → Job wird als `QUEUED` persistiert, Request endet
5. Worker: Tags schreiben → Zielpfad berechnen → SFTP nach `<musicdir>/.incoming/<jobId>/`
6. Remote: `mkdir -p` Zielordner, dann `rename` innerhalb desselben Dateisystems → atomar, der Scanner sieht nie halbe Dateien
7. `startScan` → `getScanStatus` pollen → Job `DONE`, HTMX-Polling aktualisiert die UI

Warum `.incoming`: Navidrome ignoriert Ordner mit führendem Punkt. Das Staging-Verzeichnis **muss auf demselben Dateisystem** liegen wie die Library, sonst ist das Rename kein atomarer Syscall mehr.

---

### 3. Tech-Stack

| Baustein | Wahl | Begründung |
|---|---|---|
| Java | 21 (LTS) | breite Tool-Unterstützung; 25 möglich |
| Spring Boot | 3.5.x | aktuell 3.5.16, kommerzieller Support bis 2032, größtes Ökosystem. Alternativ 4.1.x (seit Juni 2026 stabil) wenn du auf dem neuesten Stand bauen willst |
| View | Thymeleaf + HTMX 2 (WebJar) + Bootstrap/Pico.css | kein Node-Build |
| Persistenz | Spring Data JPA + H2 (file) oder PostgreSQL | H2 reicht für Single-User; Postgres wenn mehrere parallel arbeiten |
| Migration | Flyway | |
| SFTP | **sshj** (hierynomus) oder Apache MINA SSHD | JSch-Original ist tot — falls JSch, dann den `com.github.mwiede`-Fork |
| Audio-Tags | jaudiotagger | FLAC/MP3/M4A/Ogg |
| HTTP-Clients | `RestClient` + Resilience4j (RateLimiter, Retry, CircuitBreaker) | MusicBrainz-Limit |
| Async | `@Async` + DB-Queue, oder Spring Batch bei mehr Komplexität | |
| Tests | JUnit 5, Testcontainers (`atmoz/sftp`), WireMock, Mockito | SFTP echt testbar |

---

### 4. Projekt-Setup (Schritt 1 der Umsetzung)

```bash
curl https://start.spring.io/starter.zip \
  -d type=maven-project -d language=java -d javaVersion=21 \
  -d bootVersion=3.5.16 \
  -d groupId=de.nico -d artifactId=navidrome-uploader \
  -d name=navidrome-uploader -d packageName=de.nico.uploader \
  -d dependencies=web,thymeleaf,security,validation,data-jpa,flyway,actuator,h2,devtools \
  -o navidrome-uploader.zip
```

Paketstruktur:

```
de.nico.uploader
├── config/        SecurityConfig, SftpProperties, MusicBrainzProperties
├── security/      SubsonicAuthenticationProvider, SubsonicUserDetails
├── metadata/      MusicBrainzClient, CoverArtClient, dto/
├── library/       NavidromeClient (search3, ping, getUser, startScan)
├── ingest/        UploadController, StagingService, TagWriter, PathResolver
├── transfer/      SftpTransferService, TransferWorker
├── job/           UploadJob, UploadFile, JobRepository, JobService
└── web/           DashboardController, fragments/
```

---

### 5. Datenmodell

```
upload_job(id, created_by, created_at, status, release_mbid, 
           target_dir, error_message, scan_triggered_at)
upload_file(id, job_id, original_filename, size_bytes, sha256,
            track_mbid, target_path, status, error_message)
metadata_cache(query_hash, payload_json, fetched_at)   -- MB-Rate-Limit schonen
audit_log(id, ts, username, action, detail)
```

`sha256` dient der Duplikaterkennung und dem Wiederaufsetzen nach Abbruch.

---

### 6. Sicherheit

**Login gegen Navidrome**
Eigener `AuthenticationProvider`, der `GET /rest/ping.view` mit Token-Auth aufruft:
`u=<user>&t=md5(password+salt)&s=<zufälliges salt>&v=1.16.1&c=navidrome-uploader&f=json`
Bei `status="ok"` folgt `getUser.view` → `adminRole` bestimmt, ob `ROLE_ADMIN` (darf hochladen) oder `ROLE_USER` (darf nur suchen).

**Wichtig:** `startScan` verlangt in aktuellen Navidrome-Versionen Admin-Rechte. Entweder du erlaubst Upload nur Admins, oder du hinterlegst einen Service-Account in der Config nur für den Scan-Trigger.

Weitere Punkte:
- Passwort nicht persistieren; für Folge-Calls entweder pro Request neu anfordern oder verschlüsselt in der Server-Session halten
- CSRF aktiv lassen (HTMX sendet den Token per `hx-headers`)
- `spring.servlet.multipart.max-file-size` / `max-request-size` setzen, sonst OOM-Risiko
- Zielpfad strikt sanitizen (Path Traversal über Tag-Werte wie `../` ist ein realer Angriffsvektor)
- SFTP: dedizierter User + Public-Key-Auth, **kein** `StrictHostKeyChecking=no` — Host-Key im known_hosts der App pinnen
- App nur hinter TLS (Caddy/nginx als Reverse Proxy)

---

### 7. Vorbereitung auf dem Ubuntu-Server

```bash
sudo adduser --system --group nd-uploader
sudo usermod -aG navidrome nd-uploader
sudo mkdir -p /srv/music/.incoming
sudo chown -R :navidrome /srv/music
sudo chmod -R g+rws /srv/music          # setgid: neue Dateien erben die Gruppe
# SSH-Key der App in /home/nd-uploader/.ssh/authorized_keys
```

Optional in der Navidrome-Config `ScanSchedule` deaktivieren, wenn nur noch die App Scans auslöst.

---

### 8. Umsetzung in Meilensteinen

| M | Inhalt | Ergebnis | Aufwand |
|---|---|---|---|
| **M0** | Projekt-Skeleton, Flyway, Actuator, Docker/systemd | App startet, `/actuator/health` grün | 0,5 d |
| **M1** | `NavidromeClient` + Login | Login gegen echten Server funktioniert, Rollen gemappt | 1 d |
| **M2** | Dashboard-Shell (Thymeleaf, HTMX, Layout, Navigation) | Eingeloggte Startseite | 0,5 d |
| **M3** | MusicBrainz-Suche + Cover + Rate-Limiting + Cache | Suchen, Release öffnen, Tracklist sehen | 1,5 d |
| **M4** | Bibliotheks-Abgleich via `search3` | „vorhanden/fehlt"-Badges | 0,5 d |
| **M5** | Upload + Validierung + Tag-Mapping-UI | Dateien liegen im Staging der App, Tags korrekt | 2 d |
| **M6** | SFTP-Transfer, Staging, atomares Rename, Kollisionslogik | Datei landet korrekt in der Library | 1,5 d |
| **M7** | Job-Queue, Worker, Retry, Historie, Scan-Trigger + Polling | Vollständiger Flow inkl. UI-Status | 1,5 d |
| **M8** | Härtung: Tests (Testcontainers/WireMock), Logging, Audit, Deployment | Produktionsreif | 1,5 d |

Realistisch **10–11 Personentage**. M0–M2 ergeben schon einen sinnvollen Zwischenstand zum Anschauen.

---

### 9. Risiken

| Risiko | Gegenmaßnahme |
|---|---|
| Navidrome indexiert halb übertragene Dateien | Staging in `.incoming` + Rename im selben Dateisystem |
| MusicBrainz sperrt bei zu vielen Requests | RateLimiter 1 rps, Cache, korrekter User-Agent mit Kontakt |
| Falsche Dateirechte → Navidrome kann nicht lesen | setgid-Bit + umask beim SFTP-User, nach Transfer `chmod` setzen |
| Subsonic-Token-Auth erfordert reversibel gespeicherte Passwörter in Navidrome | vorab an einem Testuser verifizieren; Fallback: Passwort-Parameter über HTTPS |
| Große Uploads blockieren Request-Threads | Streaming auf Platte, dann asynchroner Worker |
| SSH-Verbindung bricht mitten im Transfer | Job resumable über `sha256` + Status pro Datei |

---

### 10. Noch offene Punkte für die Umsetzung

1. **Namensschema** — Vorschlag: `AlbumArtist/Album (Jahr)/01 - Titel.flac`. Passt das zu deiner bestehenden Struktur? (Wichtig: das Schema muss zur schon vorhandenen Bibliothek passen.)
2. **Erlaubte Formate** — nur FLAC/MP3/M4A, oder alles? Soll die App Nicht-Audio blockieren?
3. **DB** — H2-Datei reicht, oder hast du schon einen Postgres laufen?
4. **Mehrere Nutzer** oder nur du? Das entscheidet, wie streng die Job-Isolation sein muss.
5. **ZIP-/Ordner-Upload** gewünscht, oder reicht Mehrfachauswahl von Einzeldateien?

Sag mir zu 1–3 kurz was, dann kann ich direkt mit M0 anfangen und das Skeleton bauen. Wenn du den Plan als lebendes Dokument willst, mach ich daraus ein Doc, an dem wir während der Umsetzung weiterarbeiten.

Sources: [Navidrome Subsonic API Compatibility](https://www.navidrome.org/docs/developers/subsonic-api/), [Spring Boot EOL / Versionen](https://endoflife.date/spring-boot), [Navidrome PR #5510 – Admin-Middleware für Subsonic-Management-Endpoints](https://github.com/navidrome/navidrome/pull/5510)
