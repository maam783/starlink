# Starlink Konstellations-Visualisierung

Single-file 3D-Visualisierung (`index.html`) der echten Starlink-Konstellation.
Keine echte „Live"-Telemetrie — sondern **echte TLE-Bahnelemente** von Celestrak,
clientseitig mit SGP4 (`satellite.js`) auf die Simulationszeit propagiert.

**Sprache:** Default Englisch, umschaltbar auf Deutsch über den Sprach-Button (EN/DE)
in der Navigationsleiste (Wahl wird in `localStorage` gemerkt). Alle Texte laufen
über eine kleine i18n-Schicht (`STR`/`t()`/`data-i18n`); Zahlen folgen dem Gebietsschema.

## Datenfluss

`loadStarlinkData()` versucht in dieser Reihenfolge (alles real, kein Fake-Fallback):

1. **localStorage-Cache** — falls < 12 h alt (schont Celestrak bei jedem Reload).
2. **Celestrak live** — `gp.php?GROUP=starlink` (CORS-fähig). Rate-Limit-/Fehlertexte
   werden über `countTLESets()` erkannt und verworfen. Erfolg wird gecacht.
3. **Mitgeliefertes Snapshot** — `data/starlink.tle`, täglich von der GitHub Action
   aktualisiert. Greift, wenn live nicht verfügbar ist (offline, rate-limited).

Wenn nichts davon klappt (z. B. lokal per `file://` geöffnet → `fetch` blockiert),
zeigt die App einen ehrlichen Leerzustand statt erfundener Daten.

## Was angezeigt wird (alles aus den TLE berechnet)

- Getrackte Satelliten, Ø Höhe, Ø Umlaufzeit, Ø Geschwindigkeit
- Schalen-Verteilung (approximiert aus Inklination & Bahnhöhe)
- **Start-Tranchen ("Perlenkette")**: Klick auf einen Satelliten im Globus hebt
  seine ganze Start-Tranche hervor (gruppiert über den International Designator) —
  als leuchtende Punktreihe mit Orbit-Bahn und Namen entlang der Kette, Rest dimmt.
  Klick auf leere Stelle, nochmal denselben, oder **Esc** = zurück zur Vollansicht.
- **Sichtbar über Stadt**: Anzahl Satelliten aktuell ≥ 25° über dem lokalen Horizont
  (echte Elevationswinkel-Geometrie) für 12 Städte inkl. Hochbreiten wie Oslo &
  Reykjavík — macht den Abdeckungs-Gradient sichtbar (Pol = wenige, Mittelbreiten = viele).
  Klick zentriert die Stadt, setzt einen Marker und zeichnet den **Horizont-Radius**
  (Boden-Kappe, innerhalb derer ein Sat in mittlerer Schalenhöhe ≥ 25° steht).
- **Nächste Starts**: kommende SpaceX/Starlink-Launches live von The Space Devs
  (Launch Library 2). Zeigt die Schalen-Tendenz (aus dem Startplatz abgeleitet:
  Florida → ≈53°, Vandenberg → hohe Inkl. / polar), Gruppe, Ort, Countdown und
  Status (Go / TBD / In Flight). localStorage-gecacht (2 h, Free-Tier ~15 Req/h).
- **Wachstums-Diagramm**: kumulierte Flottengröße nach Startjahr (aus den
  Launch-Jahren der aktuell getrackten Sats — frühe Jahre untertreiben, da
  Gen-1-Sats deorbitiert sind).
- TLE-Quelle (live / cache / paket) + TLE-Epoche (wie alt die Bahndaten sind)

## Hosting

Reines statisches HTML — kein Server/Backend nötig:

- **Vercel**: Ordner deployen (`vercel`) oder Repo verbinden.
- **GitHub Pages**: Pages auf den Branch/`/root` zeigen lassen.

Beide liefern eine `https://`-Origin, womit das Celestrak-`fetch` funktioniert.

## TLE aktuell halten (`.github/workflows/update-tle.yml`)

Die GitHub Action lädt täglich (05:00 UTC) frische TLE von Celestrak und committet
`data/starlink.tle`, falls geändert. So ist die Seite unabhängig von Celestraks
Verfügbarkeit/Limits zur Ladezeit. Manuell auslösbar über den **Actions**-Tab
(„Run workflow"). Benötigt `contents: write` (ist im Workflow gesetzt).

## Assets & Datenschutz

Alle statischen Abhängigkeiten sind **selbst gehostet** (in `vendor/`, `fonts/`,
`data/`) — Schriften (Inter/Space Grotesk woff2), Tailwind, Font Awesome, three.js,
satellite.js und die Natural-Earth-GeoJSON. Dadurch wird zur Laufzeit **keine
Besucher-IP an Dritte** (Google Fonts, CDNs, GitHub raw) übertragen. Die einzigen
externen Aufrufe sind die Live-Datenquellen **Celestrak** (TLE) und **The Space
Devs** (Starts) — die Kernfunktion. Keine Cookies/Tracking; `localStorage` nur
funktional (TLE-/Launch-Cache, Sprache).

## Bekannte Einschränkungen

- Tailwind läuft als Play-CDN-Skript (lokal gehostet) im Browser-JIT — für „echte"
  Produktion ein Build-Step sinnvoll, aber funktional unkritisch.
- Responsiv: Desktop = 3 Spalten; unter 1024px gestapelt (Globus als Hero oben,
  Panels darunter scrollbar). Steuerung: Maus (Drag/Rad), Touch (1 Finger drehen,
  2 Finger Pinch-Zoom, Tippen wählt Tranche), Zoom-Buttons, Tastatur (+/-).
- Die Orbit-Bahn einer Tranche wird aus **einem** repräsentativen Mitglied
  propagiert (eine Umlaufzeit) — sie zeigt den Pfad der Kette, nicht jede
  Einzelbahn separat.
