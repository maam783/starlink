# Starlink Konstellations-Visualisierung

Single-file 3D-Visualisierung (`index.html`) der echten Starlink-Konstellation.
Keine echte „Live"-Telemetrie — sondern **echte TLE-Bahnelemente** von Celestrak,
clientseitig mit SGP4 (`satellite.js`) auf die Simulationszeit propagiert.

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
- **Start-Tranchen ("Perlenkette")**: Satelliten gruppiert nach echtem Launch
  (aus dem International Designator). Hover/Klick auf eine Tranche hebt die ganze
  Kette als leuchtende Punktreihe hervor, zeichnet ihre Orbit-Bahn und blendet
  Namen entlang der Kette ein; der Rest dimmt. Klick auf einen Satelliten im
  Globus wählt dessen Tranche. Liste rechts, neueste Starts zuerst.
- **Sichtbar über Stadt**: Anzahl Satelliten aktuell ≥ 25° über dem lokalen Horizont
  (echte Elevationswinkel-Geometrie) für 11 Städte inkl. Hochbreiten wie Oslo &
  Reykjavík — macht den Abdeckungs-Gradient sichtbar (Pol = wenige, Mittelbreiten = viele).
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

## Bekannte Einschränkungen

- Tailwind & Font-Awesome kommen per CDN (Play-CDN) — für „echte" Produktion ein
  Build-Step sinnvoll, aber funktional unkritisch.
- Steuerung ist Maus/Trackpad-basiert (kein Touch); Layout ist auf Desktop-Breite
  ausgelegt. Zoom per +/- Buttons, Mausrad oder Tastatur (+/-).
- Die Orbit-Bahn einer Tranche wird aus **einem** repräsentativen Mitglied
  propagiert (eine Umlaufzeit) — sie zeigt den Pfad der Kette, nicht jede
  Einzelbahn separat.
