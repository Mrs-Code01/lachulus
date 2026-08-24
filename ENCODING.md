# Video und Poster

Die vier Videodateien und das Poster **liegen im Projekt** und sind eingebunden.
Dieses Dokument hält fest, woraus und wie sie entstanden sind — damit sich der
Schritt wiederholen lässt, sobald ein anderer Clip vorliegt.

Alle Befehle brauchen [FFmpeg](https://ffmpeg.org/download.html).

## Was aktuell drin ist

| Datei | Auflösung | Größe | Ziel |
| --- | --- | --- | --- |
| `assets/video/lachulus-loop.webm` | 1280 × 720 | 2,38 MB | ≤ 3 MB |
| `assets/video/lachulus-loop.mp4` | 1280 × 720 | 3,92 MB | ≤ 4 MB |
| `assets/video/lachulus-loop-mobile.webm` | 854 × 480 | 1,05 MB | ≤ 1,2 MB |
| `assets/video/lachulus-loop-mobile.mp4` | 854 × 480 | 1,30 MB | ≤ 1,5 MB |
| `assets/img/poster.jpg` | 1280 × 720 | 89 KB | ≤ 180 KB |

Die Dateinamen stehen so im `<source>`-Markup von `index.html`. Wer sie ändert,
ändert sie dort mit — an zwei Stellen im Markup und an zwei im Skript darunter.

## Herkunft und Grenzen der Vorlage

Quelle war `https://youtu.be/KDCnlLkbD8o`, Kanal *Lachulus TV*, geladen mit
`yt-dlp -f 136` (Videospur ohne Ton, es wird keine gebraucht).

Drei Dinge, die dabei zu beachten sind:

- **720p ist das Maximum.** Auf YouTube liegt keine 1080p-Fassung. Deshalb ist
  die Desktop-Datei 1280 × 720 statt 1920 × 1080 — Hochskalieren würde nur
  Bytes kosten, keine Schärfe bringen.
- **Es ist bereits komprimiertes Material.** Jede weitere Kodierung ist eine
  zweite Generation. Liegt die Originaldatei des Animators vor, bitte daraus
  neu encodieren.
- **Der Clip ist 30,7 Sekunden lang und läuft nicht rund.** Das Briefing nennt
  ca. 20 Sekunden, nahtlos loopend. Beides wurde hergestellt (siehe unten).

## 1. Nahtlose 20-Sekunden-Schleife

Der Clip zoomt langsam hinein: das letzte Bild ist deutlich näher als das
erste, ein direkter Loop springt also sichtbar. Gelöst per Ping-Pong — die
ersten zehn Sekunden vorwärts, dieselben zehn rückwärts angehängt. Der
Übergang ist damit rechnerisch nahtlos, und die Länge trifft die geforderten
20 Sekunden exakt.

```bash
ffmpeg -i master.mp4 -an \
  -filter_complex "[0:v]trim=0:10,setpts=PTS-STARTPTS,split[a][b];[b]reverse[r];[a][r]concat=n=2:v=1[out]" \
  -map "[out]" -c:v libx264 -crf 16 -preset medium -pix_fmt yuv420p \
  loop20.mp4
```

Das Ergebnis ist die Zwischendatei für alle folgenden Schritte. `-crf 16` hält
sie bewusst nahezu verlustfrei — komprimiert wird erst im nächsten Schritt.

Nebeneffekt: an den beiden Nahtstellen steht je ein Bild doppelt. Bei 24 fps
ist das nicht wahrnehmbar.

## 2. Die vier Auslieferungsdateien

```bash
# Desktop MP4 (H.264)
ffmpeg -i loop20.mp4 -an -vf "scale=1280:720:flags=lanczos,format=yuv420p" \
  -c:v libx264 -profile:v high -crf 25 -preset slow \
  -g 48 -keyint_min 48 -sc_threshold 0 -movflags +faststart \
  assets/video/lachulus-loop.mp4

# Desktop WebM (VP9)
ffmpeg -i loop20.mp4 -an -vf "scale=1280:720:flags=lanczos,format=yuv420p" \
  -c:v libvpx-vp9 -crf 41 -b:v 0 -row-mt 1 -tile-columns 2 \
  -deadline good -cpu-used 2 -g 48 \
  assets/video/lachulus-loop.webm

# Mobil MP4
ffmpeg -i loop20.mp4 -an -vf "scale=854:480:flags=lanczos,format=yuv420p" \
  -c:v libx264 -profile:v main -crf 31 -preset slow \
  -g 48 -keyint_min 48 -sc_threshold 0 -movflags +faststart \
  assets/video/lachulus-loop-mobile.mp4

# Mobil WebM
ffmpeg -i loop20.mp4 -an -vf "scale=854:480:flags=lanczos,format=yuv420p" \
  -c:v libvpx-vp9 -crf 47 -b:v 0 -row-mt 1 -tile-columns 2 \
  -deadline good -cpu-used 2 -g 48 \
  assets/video/lachulus-loop-mobile.webm
```

- `-an` wirft die Tonspur weg. Das Video läuft stumm, die Spur kostet nur Bytes.
- `-movflags +faststart` schiebt den Index an den Dateianfang, damit die
  Wiedergabe beginnt, bevor die Datei vollständig geladen ist. Ohne das Flag
  wartet der Browser auf das Dateiende — der häufigste Grund für ein
  Hintergrundvideo, das erst nach Sekunden erscheint.

**Zu den VP9-Werten:** die CRF-Skalen von x264 und VP9 sind nicht vergleichbar.
Mit VP9-CRF 36 wurde die WebM-Datei *größer* als das MP4, was den Zweck
verfehlt — der Browser nimmt WebM zuerst. Erst bei 41 beziehungsweise 47 liegt
sie deutlich darunter. Der Sternenhimmel mit seinen tausenden Funkelpunkten ist
für beide Codecs teuer; bei ruhigerem Material greifen niedrigere Werte.

## 3. Poster

```bash
ffmpeg -i loop20.mp4 -frames:v 1 \
  -vf "scale=1280:720:flags=lanczos" -q:v 6 \
  assets/img/poster.jpg
```

Bewusst das **erste Bild der Schleife**, nicht irgendein Standbild: so zeigt
das Poster exakt das, was beim Start der Wiedergabe erscheint, und der
Übergang vom Standbild zum laufenden Video ist unsichtbar.

## 4. Kontrolle

```bash
ffprobe -v error -show_entries format=duration,size -of default=nw=1 \
  assets/video/lachulus-loop.mp4
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
Die Headline der Seite liegt damit auf Schrift. Abgefangen wird das über eine
kräftig gesetzte Abdunklung in `.stage__scrim` — sichtbar dokumentiert im
Stylesheet. Sobald ein ruhigerer Clip vorliegt, sollten die dortigen Werte
zurückgenommen werden, damit das Bild wieder durchkommt.
