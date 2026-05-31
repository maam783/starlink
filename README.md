# Starlink Constellation — Live 3D Tracker

Interaktive 3D-Visualisierung der **echten** Starlink-Konstellation in einer einzigen
`index.html`. Keine erfundene „Live"-Telemetrie: Es werden **echte TLE-Bahnelemente**
von Celestrak geladen und clientseitig mit **SGP4** (`satellite.js`) auf die
Simulationszeit propagiert. Alle angezeigten Kennzahlen sind aus diesen Bahndaten
berechnet.

- **Live:** https://maam783.github.io/starlink/
- **Repo:** https://github.com/maam783/starlink
- **Sprache:** Englisch (Default) / Deutsch — umschaltbar oben rechts (EN/DE), Wahl in `localStorage`.
- **Inoffiziell** — nicht mit SpaceX / Starlink verbunden oder von ihnen unterstützt.

---

## Inhaltsverzeichnis
1. [Features](#features)
2. [Wie es funktioniert (Architektur)](#wie-es-funktioniert-architektur)
3. [Mathematik / Geometrie](#mathematik--geometrie)
4. [Projektstruktur](#projektstruktur)
5. [Code-Karte (Funktionen in index.html)](#code-karte-funktionen-in-indexhtml)
6. [Konfiguration / Stellschrauben](#konfiguration--stellschrauben)
7. [Bedienung](#bedienung)
8. [Internationalisierung (i18n)](#internationalisierung-i18n)
9. [Datenquellen & Lizenzen](#datenquellen--lizenzen)
10. [Datenschutz & Recht](#datenschutz--recht)
11. [Hosting / Deployment](#hosting--deployment)
12. [TLE-Aktualisierung (GitHub Action)](#tle-aktualisierung-github-action)
13. [Lokale Entwicklung](#lokale-entwicklung)
14. [Bekannte Einschränkungen & ehrliche Caveats](#bekannte-einschränkungen--ehrliche-caveats)

---

## Features

- **3D-Globus** (three.js, WebGL) mit prozeduraler Ozean-Textur, echten Kontinent-
  Outlines & Ländergrenzen (Natural Earth), Wolken-/Atmosphären-Layer und dünnem
  Sternenfeld. Maus-Drag dreht, Rad/Pinch zoomt.
- **~10.400 Satelliten** als kleine, farbcodierte Punkte (Farbe = Schale), pro Frame
  per SGP4 propagiert (auf ~16 Hz gedrosselt, Render läuft mit voller FPS).
- **Zeitraffer**: Pause / 1× / 10× / 60× / 300× (Default **10×**), prominentes
  Geschwindigkeits-HUD oben links + Simulationszeit oben rechts.
- **Konstellations-Status** (links): getrackte Anzahl, Ø Höhe, Höhen-Spanne,
  Ø Umlaufzeit, Ø Geschwindigkeit — alles aus den TLE berechnet.
- **Orbitale Schalen** (links): 5 Schalen, per Inklination & Bahnhöhe klassifiziert,
  mit Live-Zählung & Prozent; einzeln ein-/ausblendbar (Checkboxen).
- **Start-Tranchen / „Perlenkette"**: Klick auf einen Satelliten hebt seine ganze
  Start-Tranche hervor (gruppiert über den International Designator) — leuchtende
  Punktreihe + reale Orbit-Bahn + Namens-Labels entlang der Kette; der Rest dimmt.
  Zurück per Klick auf leere Stelle, nochmaligem Klick oder **Esc**.
- **Nächste Starts** (links): kommende SpaceX/Starlink-Launches **live** von
  The Space Devs (Launch Library 2) — Schalen-Tendenz (aus Startplatz abgeleitet),
  Gruppe, Ort, Countdown, Status (Go / TBD / In Flight).
- **Flotten-Wachstum** (rechts): kumulierte Flottengröße über die Zeit. Das laufende
  Jahr sitzt an seiner fraktionalen Zeitposition (bis „jetzt") + **gestrichelte
  Hochrechnung** aufs Jahresende → konstante Steigung statt Teiljahr-Knick.
- **Sichtbar über Stadt** (rechts): für **20 Städte** die Anzahl Satelliten aktuell
  ≥ 25° über dem lokalen Horizont (echte Elevations-Geometrie). Liste **nach
  Breitengrad sortiert** (Nord→Süd) → der Abdeckungs-Gradient wird beim Runterlesen
  sichtbar (Pol/Äquator = wenige, Mittelbreiten = viele). Klick **rotiert** zur Stadt
  (Standard-Zoom), setzt einen roten Marker und zeichnet einen halbtransparenten,
  off-weißen **Sichtbarkeits-Trichter** (Kegel von der Stadt hoch zum ≥25°-Ring auf
  Bahnhöhe).
- **Responsiv** (Desktop 3-spaltig, Handy gestapelt) + **Touch** (1 Finger drehen,
  2 Finger Pinch-Zoom, Tippen = Auswahl) + Zoom-Buttons + Tastatur (`+`/`−`, `Esc`).
- **Ehrliche Status-Anzeige**: zeigt, ob die TLE **live**, aus **Cache** oder aus dem
  **Paket-Snapshot** stammen, plus die **TLE-Epoche** (Alter der Bahndaten).

---

## Wie es funktioniert (Architektur)

Eine einzige `index.html` (HTML + Tailwind-Klassen + ein `<script>`-Block). Keine
Build-Pipeline, kein Backend. Externe Libs/Assets sind selbst gehostet (siehe
[Datenschutz](#datenschutz--recht)).

### TLE-Datenfluss (`loadStarlinkData()`)
Versucht der Reihe nach (alles real, **kein** synthetischer Fallback):

1. **localStorage-Cache** (`starlink_tle_cache_v1`), falls < 12 h alt → schont
   Celestraks Rate-Limit bei jedem Reload.
2. **Celestrak live** — `gp.php?GROUP=starlink&FORMAT=tle` (CORS-fähig). Rate-Limit-/
   Fehlerantworten sind Prosa und werden über `countTLESets()` (> 500 TLE-Sätze?)
   erkannt und verworfen. Erfolg wird gecacht.
3. **Mitgeliefertes Snapshot** — `data/starlink.tle`, täglich von der GitHub Action
   aktualisiert (same-origin, immer verfügbar).

Klappt nichts (z. B. lokal per `file://` geöffnet → `fetch` blockiert), zeigt die App
einen ehrlichen Leerzustand statt erfundener Daten.

### Propagation & Render
- `parseTLEData()` baut pro Satellit ein `satrec` (`twoline2satrec`) und berechnet
  Inklination, Bahnhöhe, Periode, Geschwindigkeit (Kepler), NORAD-ID und die
  Start-Tranche (aus dem International Designator). Schalen-Klassifikation aus
  Inklination + Höhe.
- `updateSatellitePositions()` propagiert jeden Sat mit SGP4 auf `simTime`, wandelt
  ECI→Geodätik (`eciToGeodetic` mit GMST) und mappt lat/lon→Szenen-Koordinaten.
- Punktfarbe & -größe werden **einmalig** gesetzt (nicht pro Frame → GC-schonend);
  die Highlight-Logik schreibt die Farb-/Größen-Buffer nur bei Auswahländerung.

### Koordinatensystem
Die Punkte liegen in einem **erdfesten** Frame: `render = (ecef_x, ecef_z, −ecef_y)`,
wobei `ECEF = Rz(−GMST)·ECI`. Kontinent-Outlines, Stadt-Marker und Orbit-Ringe nutzen
exakt dasselbe Mapping (`latLonToVector3` mit negierter Länge), damit alles deckungs-
gleich ist. Der Erd-Mesh dreht sich nur kosmetisch.

---

## Mathematik / Geometrie

- **Sichtbarkeit über einer Stadt** (`countVisibleSatsFrom`): echter Elevationswinkel
  `sin(el) = (S−G)·û / |S−G|` mit Boden-Up-Vektor `û` und Bodenposition `G`. Gezählt
  wird `el ≥ MIN_ELEVATION_DEG (25°)`. (Zwei unabhängige Methoden geben dieselbe Zahl.)
- **Sichtbarkeits-Trichter/Ring**: geozentrischer Kappen-Halbwinkel
  `γ = arccos((R/(R+h))·cos e) − e` für die 25°-Maske bei mittlerer Schalenhöhe `h`.
  Der Ring liegt auf Bahnhöhe (`R+h`), der Trichter ist der Kegelmantel von der Stadt
  hoch zum Ring (per-Vertex-Alpha, off-weiß, transparent).
- **Tranche-Orbit** (`drawTrancheOrbit`): die **momentane Bahnebene** (nicht die
  Bodenspur!). Aus ECI-Position + -Geschwindigkeit eines Repräsentanten wird der
  Bahnkreis in seiner Ebene gesampelt und ins erdfeste Render-Frame transformiert →
  die Satelliten der Tranche liegen tatsächlich auf dem Ring (~1–2 % Abweichung).
- **Wachstums-Hochrechnung**: aktuelles Jahr `proj = cum(Vorjahr) + sats(Jahr)/Anteil`,
  `Anteil = vergangene Jahreszeit` (aus der TLE-Epoche).

---

## Projektstruktur

```
starlink/
├─ index.html                    # die komplette App (HTML + CSS + JS)
├─ README.md
├─ .gitignore                    # ignoriert die Referenz-Screenshots
├─ data/
│  ├─ starlink.tle               # TLE-Snapshot (von der GitHub Action gepflegt)
│  ├─ ne_110m_land.geojson        # Kontinent-Outlines (Natural Earth, PD)
│  └─ ne_110m_admin_0_countries.geojson  # Ländergrenzen (Natural Earth, PD)
├─ fonts/
│  ├─ inter-latin.woff2          # selbst gehostet (war Google Fonts)
│  └─ spacegrotesk-latin.woff2
├─ vendor/
│  ├─ three.min.js               # three.js r157 (war unpkg)
│  ├─ satellite.min.js           # satellite.js 4.1.4 (war unpkg)
│  ├─ tailwind.js                # Tailwind Play-CDN-Skript (war cdn.tailwindcss.com)
│  └─ fontawesome/               # Font Awesome 6.5.1 (CSS + fa-solid-900.woff2)
└─ .github/workflows/update-tle.yml   # täglicher TLE-Updater
```

---

## Code-Karte (Funktionen in `index.html`)

| Bereich | Funktionen |
|---|---|
| i18n | `STR` (Dict), `t()`, `applyI18n()`, `setLang()`, `toggleLang()`, `nf()` (Zahlen) |
| Three-Setup | `initThree()`, `createStarfield()`, `createEarth()`, `generateEarthTexture()` |
| Geo-Layer | `latLonToVector3()`, `loadContinentOutlines()`, `loadCountryBorders()` |
| Daten | `loadStarlinkData()`, `countTLESets()`, `parseTLEData()`, `getTLEEpoch()`, `showNoDataState()` |
| Render | `createSatellitePoints()`, `createGlowTexture()`, `brightenColor()`, `updateSatellitePositions()` |
| Tranchen | `buildLaunchGroups()`, `applyHighlight()`, `drawTrancheOrbit()`, `updateTrancheLabels()`, `selectTranche()`, `clearTranche()` |
| Dashboard | `computeConstellationStats()`, `updateDashboardStats()`, `drawGrowthChart()`, `renderShellsUI()`, `setDataStatus()` |
| Städte | `CITIES`, `countVisibleSatsFrom()`, `focusOnLatLon()`, `showCityMarker()`, `clearCityMarker()`, `renderCitiesList()`, `refreshCityCounts()` |
| Starts | `loadUpcomingLaunches()`, `launchSiteInfo()`, `renderLaunches()` |
| Interaktion | `onMouseDown/Move/Up`, `onWheel`, `onCanvasClick`, `onTouchStart/Move/End`, `zoomBy()`, `animateZoom()`, `onResize()`, `resetView()`, `setTimeMultiplier()` |
| Loop/Init | `animate()`, `updateFPS()`, `init()` |

---

## Konfiguration / Stellschrauben

Oben im `<script>` (Objekt `CONFIG` bzw. Konstanten):

| Konstante | Default | Bedeutung |
|---|---|---|
| `CONFIG.SAT_POINT_SIZE` | `0.23` | Punktgröße der Satelliten |
| `SAT_BRIGHTNESS` | `1.4` | Helligkeits-Boost der Punktfarben (Sichtbarkeit) |
| `MIN_ELEVATION_DEG` | `25` | Elevations-Maske für „sichtbar"/Trichter |
| `DEFAULT_VIEW_DISTANCE` | `22` | Standard-Zoom (Reset + Stadt-Fokus) |
| `TLE_CACHE_MAX_AGE_MS` | `12 h` | wie lange der Live-TLE-Cache gilt |
| `LAUNCH_CACHE_MAX_AGE_MS` | `2 h` | Cache für den Startplan (Free-Tier-Limit) |
| Default-Tempo | `10×` | `setTimeMultiplier(10)` in `init()` |
| Punkt-Deckkraft | `1.0` | `opacity` im `PointsMaterial` |

---

## Bedienung

- **Drehen:** Maus-Drag · 1-Finger-Swipe.
- **Zoom:** Mausrad · 2-Finger-Pinch · `+`/`−`-Buttons (links) · Tastatur `+`/`−`.
- **Satellit wählen:** Klick/Tipp → hebt die Start-Tranche hervor. Leere Stelle /
  nochmal / `Esc` = zurück.
- **Stadt wählen:** Klick in der Liste → **dreht** zur Stadt (kein Zoom), Marker +
  Trichter; am Handy scrollt die Seite zum Globus hoch.
- **Schalen:** Checkboxen links blenden Schalen ein/aus.
- **Zeit:** Pause / 1× / 10× / 60× / 300×. „Ansicht zurücksetzen" oben rechts.

---

## Internationalisierung (i18n)

`STR = { en: {...}, de: {...} }`, Lookup über `t(key)`. Statische Texte tragen
`data-i18n` / `data-i18n-html` / `data-i18n-title` und werden von `applyI18n()`
gesetzt; dynamisch gerenderte Texte rufen `t()` direkt. `setLang()` rendert alles
Dynamische neu. Zahlen folgen dem Gebietsschema (`nf()` → `10,396` vs `10.396`),
Städtenamen haben DE-Overrides (Tokyo/Tokio, Moscow/Moskau, …). Default **Englisch**.

---

## Datenquellen & Lizenzen

| Quelle | Nutzung | Hinweis |
|---|---|---|
| **Celestrak** | TLE-Bahnelemente (live + Snapshot) | öffentliche Daten; Attribution + Rate-Limit (gecacht) |
| **The Space Devs** (Launch Library 2) | kommende Starts | Attribution; Free-Tier **non-commercial**, ~15 Req/h (gecacht 2 h) |
| **Natural Earth** | Kontinente/Grenzen (GeoJSON) | Public Domain |
| **three.js**, **satellite.js** | 3D / SGP4 | MIT |
| **Tailwind**, **Font Awesome (Free)**, **Inter**, **Space Grotesk** | UI | MIT / SIL OFL / CC-BY |

Alle Credits stehen im Footer der App.

---

## Datenschutz & Recht

> Kein Rechtsrat — praktische Hinweise für eine öffentlich erreichbare, in DE
> betriebene Hobby-Seite.

- **Selbst gehostet:** Schriften, Tailwind, Font Awesome, three.js, satellite.js und
  die GeoJSON liegen lokal → zur Laufzeit wird **keine Besucher-IP an Dritte**
  (Google Fonts, CDNs, GitHub raw) übertragen. Einzige externe Aufrufe: **Celestrak**
  und **The Space Devs** (die Live-Daten = Kernfunktion).
- **Keine Cookies / kein Tracking / keine Analytics.** `localStorage` nur funktional
  (TLE-/Launch-Cache, Sprachwahl) → i. d. R. kein Cookie-Banner nötig.
- **Marke:** „Starlink"/„SpaceX" sind Marken von SpaceX. Der Name wird nur
  **beschreibend** verwendet; ein **Disclaimer** („Unofficial · not affiliated…")
  steht im Footer, der Untertitel wurde de-gebrandet, kein Original-Logo. **Nicht
  kommerzialisieren.**
- **Offen / empfehlenswert:** für DE ggf. ein **Impressum + kurze Datenschutz-
  erklärung** ergänzen (Hinweis auf die Verbindungen zu Celestrak/The Space Devs).

---

## Hosting / Deployment

Reines statisches HTML — kein Server/Backend. Aktuell **GitHub Pages**:
`maam783.github.io/starlink/`. Jeder Push auf `main` deployt automatisch.

Alternativen: **Vercel** (Ordner deployen oder Repo verbinden) — jede `https://`-
Origin reicht, damit der Celestrak-`fetch` (CORS) funktioniert.

---

## TLE-Aktualisierung (GitHub Action)

`.github/workflows/update-tle.yml` läuft **täglich 05:00 UTC** (+ manuell über
**Actions → Run workflow**): holt frische TLE von Celestrak (mit Supplemental-
Fallback), validiert (> 500 Sätze) und committet `data/starlink.tle` nur bei
Änderung. So ist die Seite unabhängig von Celestraks Verfügbarkeit/Limits zur
Ladezeit. Benötigt `permissions: contents: write` (gesetzt).

---

## Lokale Entwicklung

Wegen `fetch` (TLE, GeoJSON) **nicht** per Doppelklick (`file://`) öffnen, sondern
über einen lokalen Server, z. B.:

```bash
python3 -m http.server 8099
# → http://localhost:8099
```

Alles ist Vanilla JS in `index.html`; einfach editieren & neu laden.

---

## Bekannte Einschränkungen & ehrliche Caveats

- **„Sichtbar"-Zahl** zählt alle getrackten Objekte ≥ 25° (geometrische Sichtlinie),
  inkl. nicht-operativer/aufsteigender. Eine echte Antenne verbindet zu **einem**
  davon. Der **Trichter/Ring** ist ein qualitatives Symbol — die belastbare Zahl ist
  die Listenangabe.
- **Wachstums-Chart** zeigt *aktuell getrackte* Sats nach Startjahr — frühe Jahre
  (2019–2021) untertreiben, da Gen-1-Sats deorbitiert sind; die „konstante"-Aussage
  gilt für die jüngsten Jahre.
- **Schalen-Zuordnung** (5 Schalen) ist aus Inklination & Höhe **approximiert**,
  nicht offiziell.
- **Tranche-Orbit** wird aus **einem** Repräsentanten propagiert (zeigt den Pfad der
  Kette, nicht jede Einzelbahn).
- **Tailwind** läuft als Play-CDN-Skript (lokal) im Browser-JIT — für echte Produktion
  wäre ein Build-Step sauberer, funktional aber unkritisch.
- **Bewegung** ist immer propagiert/simuliert (SGP4 auf Simulationszeit) — die
  Startwerte (TLE) sind echt, aber kein Echtzeit-Stream.
