# A Wednesday on the German rail network

A 24-hour time-lapse of one real day of German rail traffic. It includes all scheduled long-distance, regional, and S-Bahn trains.
The data comes from the official nationwide timetable. The map stays dark, so the trains are the brightest items.

**Live: https://chillchamp1.github.io/github.io/**

Open `index.html` — it is self-contained, so a local double-click works as well
as GitHub Pages. No server, no build step, no network calls except the webfont.
Every push to `main` republishes the site via `.github/workflows/pages.yml`.
The page is one self-contained file of approximately 10 MB.

## What is on screen

**Wednesday 26 August 2026**, from gtfs.de feeds based on DELFI data: 40,229 trains at 6,537 stations.

| Category | Trips | Drawn as |
|---|---|---|
| **ICE / TGV / RJ** | 737 | high-speed, full-size dot |
| **IC / EC** | 409 | intercity, full-size dot |
| **RE / RB / MEX** | 23,343 | regional, small dot |
| **S-Bahn** | 15,740 | suburban rail, smallest dot |

S-Bahn is included as a separate category. U-Bahn, tram, bus, dial-a-ride, and rail-replacement buses are excluded.
Each train has a tail that shows its direction. S-Bahn trains use the shortest tail and smallest dot to limit city-level density.
Only the largest cities have labels. A faint outline of Germany and its state borders gives orientation.

The animation opens at the quietest minute of the day, found by scanning
per-minute occupancy rather than hard-coded, and runs at 4x by default: a full
day in about 90 seconds. A ring opens outward at the station where a service
begins, in that service's colour. Regional and S-Bahn rings stay short to limit visual density.
At 4x speed, the 05:00–07:00 increase shows the whole country starting service.
Terminations are not marked: one ring per service
is already dense, and two left the map permanently speckled. The strip behind
the scrubber counts the same starts across the day, in one ink because the
categorical hues stay reserved for the trains.

A compact key — swatch, code, one word — is drawn on the map itself, in the
open ground below Saxony. It costs the frame nothing and means a screen
recording carries its own legend; the panel below the map keeps the full
labels, the live counts and the note.

The clock sits on the Baltic about 30 km off the Fischland-Darß coast, where
the nearest station is far enough away that it never covers the network. Giving
it open water rather than a reserved band hands the whole stage to the map.

Hover a train for its line and destination. Space bar toggles playback.

The window itself adapts to its container: whichever axis has room to spare is widened
towards the reach of the feed's international services. A phone in portrait
gets Germany filling the screen rather than a small map marooned between two
empty bands; a wide desktop gets the neighbours. On
phones the map keeps a full screen to itself and the legend, figures and
controls sit below the fold.

The builder excludes NightJet, EuroNight, and other sleeper services.

## The data

`data/trains.json` uses the free long-distance and regional-rail feeds from [gtfs.de](https://gtfs.de/en/feeds/).
These feeds use the DELFI NeTEx dataset and a Creative Commons 4.0 license.
The regional-rail feed includes RB, RE, IRE, S-Bahn, and non-federal railways.
The free feeds cover a moving 30-day timetable period. Select a date in that period.

`data/germany.json` is the basemap: the national outline and the sixteen state
borders, from [`isellsoap/deutschlandGeoJSON`](https://github.com/isellsoap/deutschlandGeoJSON)
(Unlicense, public domain), reduced to three-decimal coordinates.

## Rebuilding

```sh
curl -o de_fv.zip "https://download.gtfs.de/germany/fv_free/latest.zip"
curl -o de_rv.zip "https://download.gtfs.de/germany/rv_free/latest.zip"
unzip -d de_fv de_fv.zip
unzip -d de_rv de_rv.zip
python3 build/build_gtfs.py de_fv de_rv 20260826 -o data/trains.json \
    --note "All categories cover the whole country, from gtfs.de feeds based on the official DELFI dataset (timetable of 26 August 2026)."
python3 build/bundle.py          # inlines the JSON back into index.html
```

`build_gtfs.py` accepts one or more GTFS feeds and a service date that they share.
It recognizes S-Bahn from route type 109 or line names such as `S1`.
It recognizes extended route types before route names. For plain type-2 feeds, it uses route names first.
The JSON stores times as whole minutes to keep the file small.

A portrait video for phones and social posts comes from the page itself:

```sh
node build/export_video.js --seconds 60 --start 00:00 --out german-rail-day.mp4
```

That gives 1080x1920 H.264. Playback is not screen-recorded -- the page is
paused and the scrubber stepped one frame at a time, so each frame lands on an
exact simulated minute however long the render takes, and the whole day fits
the requested length regardless of machine speed. Frames go out as JPEG
because PNG encoding at that size costs more per frame than the page takes to
draw. `--start HH:MM` picks the clock time the day opens on; omit it to start
where the page does, at the quietest minute of the night. Needs playwright and
ffmpeg (`pip install imageio-ffmpeg` supplies one).

The basemap only needs rebuilding if you change the geometry:

```sh
python3 build/build_geo.py outline.geo.json states.geo.json -o data/germany.json
```

## Layout

```
index.html            the whole visualisation, data inlined
data/trains.json      generated timetable extract
data/germany.json     generated basemap rings
build/build_gtfs.py   GTFS feed(s) -> JSON, merged onto one service date
build/build_geo.py    GeoJSON -> compact rings
build/bundle.py       both JSON files -> inlined into index.html
build/export_video.js index.html -> portrait MP4
```

## Colour and rendering

The map uses one near-black surface. The five category colors stay visible against this surface:

| | |
|---|---|
| high-speed | `#5aa9ff` |
| intercity | `#ff7a45` |
| regional | `#35d69a` |
| S-Bahn | `#d889ff` |

The marks are small enough that pixel geometry matters. Device pixel ratio is
honoured up to 3x, and any dot whose radius falls below about 1.3 device pixels
is snapped to the device grid and drawn as a hard square rather than a circle —
same apparent size, none of the antialiasing smudge that made the regional
trains look blurred. Trail widths have a one-device-pixel floor for the same
reason.

Sky, land and the ~6,500 station dots are rendered once onto an offscreen
canvas and blitted each frame, so the per-frame cost is the moving trains
alone: around 60 fps with 1,660 trains on screen at 3x pixel density. The day
profile is likewise drawn once per resize and blitted.

Origin rings come from a time-sorted event index — one entry per service — so
each frame binary-searches the live window instead of rescanning 40,229 trips. Ring lifetime scales with the playback multiplier, so
an event stays visible for roughly two thirds of a second at any speed.
