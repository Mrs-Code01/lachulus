# Video und Poster

Die Videodateien und das Poster **liegen im Projekt** und sind eingebunden.
Dieses Dokument hält fest, woraus und wie sie entstanden sind — damit sich der
Schritt wiederholen lässt, sobald ein anderer Clip vorliegt.

Alle Befehle brauchen [FFmpeg](https://ffmpeg.org/download.html).

## Was aktuell drin ist

| Datei | Auflösung | Größe | SSIM |
| --- | --- | --- | --- |
| `assets/video/lachulus-loop-v3.mp4` | 1280 × 720 | 3,65 MB | 0,9923 |
| `assets/video/lachulus-loop-mobile-v3.mp4` | 1280 × 720 | 2,21 MB | 0,9873 |
| `assets/img/poster-v2.jpg` | 1280 × 720 | 129 KB | — |

Die Dateinamen stehen im `<source>`-Markup von `index.html` (`data-src` und
`data-src-small`) sowie im `poster`-Attribut, dort zweimal — einmal im
sichtbaren Video, einmal in der `<noscript>`-Fassung.

**Warum `-v3` im Namen:** `vercel.json` lässt Videos unbefristet
zwischenspeichern (`immutable`), `.htaccess` tut dasselbe für Apache-Hosting. Eine ausgetauschte Datei braucht deshalb einen neuen Namen,
sonst sehen wiederkehrende Besucher weiter die alte Fassung. Bei der nächsten
Runde also `-v4`.

## Herkunft und Grenzen der Vorlage

Quelle war `https://youtu.be/KDCnlLkbD8o`, Kanal *Lachulus TV*, geladen mit
`yt-dlp -f 136` (Videospur ohne Ton, es wird keine gebraucht). Im Projekt liegt
sie als `lachulus-master-720p.mp4`.

- **720p ist das Maximum.** Auf YouTube liegt keine 1080p-Fassung. Hochskalieren
  würde nur Bytes kosten, keine Schärfe bringen. **Liegt die Originaldatei des
  Animators vor, bringt sie mehr als jede Einstellung hier** — der Clip wird
  auf dem Desktop formatfüllend gezeigt, also auf 1920 Pixel Breite
  hochgerechnet.
- **Es ist bereits komprimiertes Material.** Jede weitere Kodierung ist eine
  zweite Generation.
- **Der Clip ist 30,7 Sekunden lang und läuft nicht rund.** Das Briefing nennt
  ca. 20 Sekunden, nahtlos loopend. Beides wird unten hergestellt.

## 1. Nahtlose 20-Sekunden-Schleife

Der Clip zoomt langsam hinein: das letzte Bild ist deutlich näher als das
erste, ein direkter Loop springt also sichtbar. Gelöst per Ping-Pong — die
ersten zehn Sekunden vorwärts, dieselben zehn rückwärts angehängt. Der
Übergang ist damit rechnerisch nahtlos, und die Länge trifft die geforderten
20 Sekunden exakt.

```bash
ffmpeg -i lachulus-master-720p.mp4 -an \
  -filter_complex "[0:v]trim=0:10,setpts=PTS-STARTPTS,split[a][b];[b]reverse[r];[a][r]concat=n=2:v=1[out]" \
  -map "[out]" -c:v libx264 -crf 16 -preset medium -pix_fmt yuv420p \
  loop20.mp4
```

Das Ergebnis ist die Zwischendatei für alle folgenden Schritte und dient
zugleich als Referenz für die Qualitätsmessung. `-crf 16` hält sie bewusst
nahezu verlustfrei — komprimiert wird erst im nächsten Schritt.

Nebeneffekt: an den beiden Nahtstellen steht je ein Bild doppelt. Bei 24 fps
ist das nicht wahrnehmbar.

## 2. Die zwei Auslieferungsdateien

```bash
# Desktop
ffmpeg -i loop20.mp4 -an -vf "scale=1280:720:flags=lanczos,format=yuv420p" \
  -c:v libx264 -profile:v high -crf 25 -preset veryslow \
  -g 48 -keyint_min 48 -sc_threshold 0 -movflags +faststart \
  assets/video/lachulus-loop-v3.mp4

# Handy (gleiche Auflösung, nur sparsamer)
ffmpeg -i loop20.mp4 -an -vf "scale=1280:720:flags=lanczos,format=yuv420p" \
  -c:v libx264 -profile:v high -crf 30 -preset veryslow \
  -g 48 -keyint_min 48 -sc_threshold 0 -movflags +faststart \
  assets/video/lachulus-loop-mobile-v3.mp4
```

- `-an` wirft die Tonspur weg. Das Video läuft stumm, die Spur kostet nur Bytes.
- `-movflags +faststart` schiebt den Index an den Dateianfang, damit die
  Wiedergabe beginnt, bevor die Datei vollständig geladen ist. Ohne das Flag
  wartet der Browser auf das Dateiende — der häufigste Grund für ein
  Hintergrundvideo, das erst nach Sekunden erscheint.

### Warum kein WebM mehr

Früher lagen von jeder Fassung zusätzlich VP9/WebM-Dateien bereit, weil WebM
bei niedriger Qualität kleiner ausfällt. Auf der jetzt gefahrenen Stufe kehrt
sich das um. Gemessen gegen `loop20.mp4`:

| Kodierung | Größe | SSIM |
| --- | --- | --- |
| H.264 CRF 25 | 3,92 MB | 0,99266 |
| VP9 CRF 30 | 4,94 MB | 0,99266 |
| H.264 CRF 20 | 6,49 MB | 0,99574 |
| VP9 CRF 26 | 6,23 MB | 0,99359 |

Bei **gleichem SSIM** ist das MP4 rund 1 MB kleiner, bei ähnlicher Größe
deutlich besser. WebM kostet hier also Bytes, statt welche zu sparen. Dazu
kommt: H.264 wird auf jedem Gerät in Hardware dekodiert, VP9 auf iPhones
nicht — dort lief das WebM auf der CPU und das Video ruckelte.


## Warum die Datenrate begrenzt bleibt — und warum erst geladen, dann gespielt wird

Eine frühere Fassung lief mit CRF 20 und 2,7 Mbit/s. Sie war schärfer, aber sie
**ruckelte**, und zwar aus einem Grund, der nichts mit dem Zeichnen zu tun hat:

    Datenrate des Videos     2,72 Mbit/s
    gemessene Leitung        1,0 – 3,5 Mbit/s
    Ladezeit                 18,9 s bei 20 s Laufzeit

Die Wiedergabe lief dem Puffer davon. Zwei Konsequenzen daraus:

**1. Die Datenrate bleibt unter ~1,5 Mbit/s.** CRF 25 statt 20 spart 45 % der
Bytes und kostet nur 0,0034 SSIM.

**2. Das Video wird komplett geladen, bevor es startet.** Das Skript in
`index.html` holt die Datei per `fetch()` am Stück und übergibt sie als Blob an
das `<video>`. Erst dann läuft sie los — ab da liegt sie vollständig im
Speicher und kann technisch nicht mehr nachladen.

Der naheliegendere Weg funktioniert **nicht**: einfach zu warten, bis
`video.buffered` die ganze Datei abdeckt, führt nie zum Ziel, weil Browser das
Vorausladen bei einem Video, das nicht läuft, absichtlich abbrechen. Gemessen
war die Datei auch nach 40 Sekunden auf einer 3-Mbit-Leitung nie vollständig
gepuffert. Ebenso wenig taugt `readyState === 4`: das ist nur die optimistische
Schätzung des Browsers und stand hier schon nach 1,2 Sekunden auf „reicht".

Nachgemessen über einen echten HTTP-Server mit gedrosselter Leitung, je 70 s
beobachtet:

| Fall | Start | Stocker | verworfene Frames |
| --- | --- | --- | --- |
| Desktop, 3 Mbit/s | 12,4 s | 0 | 0 / 1386 |
| Desktop, 1,5 Mbit/s | 23,6 s | 0 | 0 / 1119 |
| Handy, 800 kbit/s | 27,6 s | 0 | 0 / 1022 |
| Handy, 2. Besuch | **0,5 s** | 0 | 1 / 1673 |

Der Preis ist die Wartezeit beim **ersten** Besuch. Sichtbar ist in der Zeit das
Posterbild — und das ist das erste Bild der Schleife, der Übergang also
unsichtbar. Beim zweiten Besuch entfällt das Laden ganz, weil `vercel.json` die
Videos unbefristet zwischenspeichern lässt.

### Warum das Handy dieselbe Auflösung bekommt

Die frühere Handyfassung war 854 × 480. Durch `object-fit: cover` wird das Bild
auf einem hochkant gehaltenen Telefon aber stark vergrößert, und auf einem
Display mit dreifacher Pixeldichte war das sichtbar weich. Bei praktisch
gleicher Dateigröße gewinnt mehr Auflösung klar vor mehr Datenrate:

| Kodierung | Größe | SSIM (auf 720p hochskaliert) |
| --- | --- | --- |
| 854 × 480, CRF 24 | 2,69 MB | 0,9832 |
| 1280 × 720, CRF 28 | 2,90 MB | 0,9900 |

Ausgeliefert wird 1280 × 720 bei CRF 30 (2,21 MB).
Qualität, die vorher der *Desktop* bekam.

## 3. Poster

```bash
ffmpeg -i loop20.mp4 -frames:v 1 \
  -vf "scale=1280:720:flags=lanczos" -q:v 3 \
  assets/img/poster-v2.jpg
```

Bewusst das **erste Bild der Schleife**, nicht irgendein Standbild: so zeigt
das Poster exakt das, was beim Start der Wiedergabe erscheint, und der
Übergang vom Standbild zum laufenden Video ist unsichtbar.

## 4. Kontrolle

Größe und Dauer:

```bash
ffprobe -v error -show_entries format=duration,size -of default=nw=1 \
  assets/video/lachulus-loop-v3.mp4
```

Qualität gegen die Referenz messen (höher ist besser, 1,0 wäre identisch):

```bash
ffmpeg -v error -i assets/video/lachulus-loop-v3.mp4 -i loop20.mp4 \
  -lavfi "[0:v][1:v]ssim=stats_file=-" -f null - | tail -1
```

Liegt der Index an der Dateispitze (`moov` muss **vor** `mdat` stehen):

```bash
ffprobe -v trace -i assets/video/lachulus-loop-v3.mp4 2>&1 \
  | grep -m2 -E "type:'(moov|mdat)'"
```

## Noch offen: Vorschaubild für Social Media

`assets/img/og-image.jpg` (1200 × 630) fehlt und wird in `index.html` in den
Open-Graph-Angaben referenziert. Es erscheint, wenn jemand den Link bei
WhatsApp, Facebook oder LinkedIn teilt — bei einer Kampagne, die über soziale
Netzwerke läuft, ist das die meistgesehene Grafik des Projekts. Ein Standbild
aus dem Video mit dem Schriftzug „Lachulus on Tour" wäre naheliegend.

## Hinweis zum gelieferten Clip

Der Clip ist ein **Studio-Signet**, kein ruhiger Hintergrund: er trägt den
Schriftzug „LACHULUS EDUTAINMENT STUDIOS" samt Leistungsband mitten im Bild.
Die Headline der Seite liegt damit auf Schrift. Abgefangen wird das über die
Abdunklung in `.stage__scrim`, die in den ersten drei Sekunden jedes
15-Sekunden-Takts zurückgenommen wird, damit die Zeichentrickwelt sichtbar
ist. Sobald ein ruhigerer Clip ohne Schriftzug vorliegt, wird beides einfacher.
