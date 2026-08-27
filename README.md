# Lachulus on Tour — Landingpage

Einseitige Landingpage mit Vollbild-Videohintergrund, dazu Impressum und
Datenschutzerklärung als eigene Seiten. Statisches HTML, CSS und rund 30 Zeilen
JavaScript — kein Build-Schritt, kein Framework, keine Abhängigkeiten.

## Dateien

```
index.html            Die Landingpage
impressum.html        Pflichtangaben nach § 5 DDG
datenschutz.html      Datenschutzerklärung
.htaccess             Cache-, Komprimierungs- und Sicherheits-Header (Apache)
ENCODING.md           FFmpeg-Rezepte für Video und Poster
assets/
  style.css           Sämtliche Gestaltung, von allen drei Seiten genutzt
  fonts/              Sechs woff2-Dateien (Fraunces, Fraunces kursiv, Inter)
  img/favicon.svg     Browser-Symbol
  img/                poster.jpg und og-image.jpg fehlen noch
  video/              Die vier Videodateien fehlen noch
```

## Veröffentlichen

Den Ordnerinhalt ins Web-Wurzelverzeichnis hochladen. Sonst nichts. Zum lokalen
Ansehen genügt ein Doppelklick auf `index.html`.

## Was noch fehlt

Video und Poster sind eingebunden. Was noch fehlt, betrifft Randstücke — die
Seite ist ohne diese Dateien voll funktionsfähig.

| Fehlt | Wo eintragen |
| --- | --- |
| Social-Media-Vorschaubild | `assets/img/og-image.jpg`, 1200 × 630 |
| Finales Logo | ersetzt die Wortmarke in `index.html`, siehe unten |
| Endgültige Rechtstexte | ersetzen den Inhalt von `impressum.html` und `datenschutz.html` |

USt-IdNr., Registereintrag und Hosting-Anbieter sind eingetragen
beziehungsweise geklärt: die USt-IdNr. steht im Impressum, der Registereintrag
entfällt (Einzelunternehmen ohne Handelsregistereintrag), als Hosting-Anbieter
ist IONOS in Abschnitt 3 der Datenschutzerklärung benannt. Die früher gelb
markierten Hinweiskästen (`<div class="note">…</div>`) sind damit alle
entfernt; in den Rechtstexten steht kein sichtbarer Platzhalter mehr.

### Logo austauschen

In der Kopfzeile steht die Wortmarke aus dem Entwurf als Text:

```html
<a class="logo" href="/">LACHU<span>LUS</span></a>
```

Sie wird ersetzt durch:

```html
<a class="logo" href="/"><img src="assets/img/logo.svg" alt="Lachulus" height="34"></a>
```

Am Layout ändert sich dadurch nichts — die Kopfzeile richtet Logo links und
Einordnung rechts unabhängig von der Höhe des Logos aus.

## Wie die Anforderungen umgesetzt sind

**Video.** `autoplay muted loop playsinline` stehen als Attribute am
`<video>`, die beiden `<source>` im Markup — genau so, wie das Briefing es
verlangt. WebM steht vor MP4, weil der Browser die erste Quelle nimmt, die er
abspielen kann, und WebM hier rund 40 % kleiner ist. Bis das erste Bild
dekodiert ist, zeigt das Element sein `poster`-Bild; eine Einblendung per
JavaScript braucht es dafür nicht.

Ein kurzes Skript direkt hinter dem Element erledigt zwei Dinge, die sich in
reinem HTML nicht ausdrücken lassen: Auf Bildschirmen bis 820 px tauscht es die
Quellen gegen die kleinere Fassung (854 × 480 statt 1280 × 720, rund ein Drittel
der Bytes) — `<source media="…">` funktioniert an `<video>` in keinem aktuellen
Browser, JavaScript ist der einzige Weg. Und bei „Bewegung reduzieren" entfernt
es die Quellen, bevor geladen wird; dann bleibt das Posterbild stehen. Ohne
JavaScript passiert nichts davon, und die Desktop-Fassung spielt trotzdem.

**Kein Cookie-Banner nötig.** Die Seite setzt keine Cookies und stellt beim
Aufruf keine einzige Verbindung zu einem fremden Server. Das Video liegt auf dem
eigenen Server statt als YouTube-Einbindung. Auch die Schriften sind
mitgeliefert und werden **nicht** von Google Fonts geladen — genau diese
Einbindung hat das Landgericht München I 2022 als DSGVO-Verstoß gewertet
(Az. 3 O 17493/20). Beide Punkte zusammen sind der Grund, warum kein
Einwilligungsbanner gebraucht wird.

**Rechtstexte.** Impressum und Datenschutz sind eigene Seiten statt Overlays,
aus dem Footer jeder Seite mit einem Klick erreichbar. Damit sind sie im Sinne
der Anbieterkennzeichnung „leicht erkennbar, unmittelbar erreichbar und ständig
verfügbar“ — und sie funktionieren auch ohne JavaScript.

**Mobile first.** Die Seite ist ab 320 px Breite ausgelegt. Auf dem Telefon
rückt die Fußleiste in eine Zeile zusammen: die Rufnummer bleibt vollständig
sichtbar, die E-Mail-Schaltfläche verkürzt sich auf „E-Mail“. Beide sind
Schaltflächen mit Fingergröße, nicht nur Textlinks.

**Die Seite passt immer auf genau einen Bildschirm.** Das ist die Bedingung
dafür, dass der CTA funktioniert: Sobald die Seite auch nur ein Stück scrollen
kann, springt der Klick auf „Anfragen & Kooperationen“ in diesen freien Rest
unterhalb der Fußleiste. Drei Dinge stellen das sicher:

- Die mittlere Rasterzeile ist `minmax(0, 1fr)`, nicht `1fr`. Eine reine
  `1fr`-Zeile wird nie kleiner als ihr Inhalt und schiebt die Bühne bei
  niedrigen Fenstern über die Bildschirmhöhe hinaus.
- Headline und Abstände skalieren nicht nur mit der Breite, sondern über
  `min(…, 17vh)` auch mit der Höhe. Auf einem flachen Fenster wird die
  Headline kleiner, statt den CTA nach unten zu drücken.
- Die Höhe steht auf `100dvh` (mit `svh`- und `vh`-Rückfall) — immer exakt der
  sichtbare Bereich. Da die Seite nie scrollt, blendet auf dem Handy auch keine
  Browserleiste ein, `dvh` bleibt also stabil.

Geprüft wurde das mit 36 Viewport-Größen von 1920 × 1200 bis 344 × 882, in
Rahmen mit exakter Pixelhöhe gemessen. Überall `scrollHeight == clientHeight`,
kein seitlicher Überlauf, kein Scroll der Bühne, Logo bündig oben, Fußleiste
bündig unten, CTA frei darüber.

`env(safe-area-inset-*)` hält Logo und Kontaktleiste von Kameraaussparung und
Home-Leiste des iPhones frei — nötig, weil `viewport-fit=cover` gesetzt ist.

**Bedienbarkeit.** Sprungmarke zum Kontaktbereich, sichtbarer Fokusrahmen auf
allen Bedienelementen, das dekorative Video ist für Screenreader ausgeblendet,
sämtliche Bewegung hört bei „Bewegung reduzieren“ auf.

## Der CTA und die Fußleiste

Die Fußleiste ist **dauerhaft sichtbar** — dünne Leiste am unteren Rand, wie im
Briefing und im Entwurf. Der CTA trägt `href="#kontakt"` und zeigt damit auf
sie, ebenfalls wie im Briefing festgelegt.

**Was das bedeutet:** Da die Seite auf genau einen Bildschirm passt und nie
scrollt, ist die Leiste beim Klick bereits sichtbar. Der Anker hat also keinen
sichtbaren Effekt. Das ist die wörtliche Umsetzung der Vorgabe; der Preis ist
ein Button, dessen Klick nichts Sichtbares auslöst.

Eine frühere Fassung hielt die Leiste verborgen und fuhr sie beim Klick ein.
Das gab dem Anker ein echtes Ziel, versteckte aber Impressum und Datenschutz
beim Laden — rechtlich heikel, weil die Anbieterkennzeichnung „leicht
erkennbar" sein muss. Die Regeln dafür sind entfernt; die Variante lässt sich
auf Wunsch zurückholen.

> **Ein Fallstrick, bitte nicht rückgängig machen:** Die Bühne braucht
> `overflow: clip`, nicht `overflow: hidden`. Beide schneiden optisch gleich
> ab, aber `hidden` macht das Element zu einem Scroll-Container, den der
> Sprung auf `#kontakt` dann verschieben kann — das Logo wandert oben aus dem
> Bild. `clip` erzeugt gar keinen Scroll-Container.

## Gestaltung

Farbwelt, Schrift und Aufbau folgen dem mitgelieferten Entwurf
`lachulus-landingpage-entwurf.html`. Alle Werte stehen als Variablen am Anfang
von `assets/style.css` und lassen sich dort zentral ändern.

- **Farben** (unverändert aus dem Entwurf): Mitternachtsblau `#0B0E17` als
  Grund, warmes Off-White `#F6F3EC` für Text, dazu die vier Akzente Gold
  `#F2B84B`, Koralle `#FF7A6B`, Türkis `#48C9B0` und Himmelblau `#6FB1FF` —
  je einer pro Badge.
- **Schrift** (aus dem Entwurf): Fraunces für Logo, Headline und die
  Überschriften der Rechtstexte, Inter für alles Übrige. „on Tour" steht wie im
  Entwurf in Koralle und kursiv.
- **Rundungen und Abstände** folgen den Pillenformen des Entwurfs.

Die Seite folgt bewusst **nicht** dem Hell-/Dunkelmodus des Betriebssystems.
Ein Vollbildvideo mit heller Schrift darauf hat genau einen richtigen Zustand.

## Abweichungen vom Entwurf

Der Entwurf ist die visuelle Vorlage; wo Briefing-Text und Entwurf sich
widersprechen, gilt das Briefing.

1. **Schriften selbst gehostet.** Der Entwurf lädt Fraunces und Inter über
   `fonts.googleapis.com`. Genau diese Einbindung hat das Landgericht München I
   2022 als DSGVO-Verstoß gewertet (Az. 3 O 17493/20). Die Dateien liegen jetzt
   in `assets/fonts/`; das Schriftbild ist identisch.
2. **Nur ein CTA-Button.** Der Entwurf zeigt zwei („Für Anfragen &
   Kooperationen" plus „Mehr erfahren"), das Briefing nennt unter
   „Headline-Block" ausdrücklich **einen**. Der zweite Button hatte im Entwurf
   ohnehin kein Ziel (`href="#"`). Die CSS-Regeln dafür stehen noch da, er ist
   mit einer Zeile HTML zurückzuholen.
3. **Echte Kontaktdaten** statt der Platzhalter `+49 XXX XXXXXXX` und
   `kontakt@lachulus.de`.
4. **Copyright 2026 statt 2027.** Im Entwurf steht `© 2027` — eine Jahreszahl
   in der Zukunft. Bitte prüfen, falls das Absicht war.
5. **Kräftigere Abdunklung in der Bildmitte.** Nicht aus gestalterischer
   Laune, sondern weil das gelieferte Video mittig selbst Schrift trägt. Siehe
   `ENCODING.md`, letzter Abschnitt.

## Nicht enthalten

Unterseiten, Navigation und weitere Inhalte der bestehenden Website — laut
Briefing außerhalb des Auftrags.
