# ab_engine: Strata-Versionen mit ihren eigenen Defaults vergleichen, ohne die Installation zu berühren

Stand: 2026-11-16 · Repo `D:\GitHub\Strata`, Branch `feat/ab-engine-versions` (von `upstream/main` c77572806)
· Ziel: PR an Niko1221/Strata

## Absicht

Christoph will Strata-Versionen auf seinem Rechner vergleichen: zwei gegeneinander (x gegen y) oder eine Spanne
(x bis y). Jede Version läuft so, wie ein Nutzer sie bekommen hätte: **Engine und Control desselben Releases, mit
den Default-Werten, die dieses Control für Modell und Hardware wählt.** Auf Wunsch wird vor der Messung für jede
Version getunt. Daten und Config der normalen Installation bleiben unverändert. Während der Messung läuft nur der
Benchmark. Am Ende steht ein Bericht: eine HTML-Seite mit sortierbarer Vergleichstabelle und eine PNG-Karte im
Stil der Strata-Release-Bilder (Vorbild: Release-Karte „Strata 0.2.12“).

## Ziele

- `tools/ab_engine.py` bekommt einen Modus „Versionen“:
  `python tools/ab_engine.py --model-path <gguf|ordner> --engines 0.1.55,0.1.56 [--tune quick|thorough] [--rounds 3]`
  bzw. `--engines 0.1.50..0.1.56`. Die Nummer ist die Engine-Version; dazu gehört das Control-Paket gleicher
  Version aus demselben Release (v0.2.9 trägt `strata-engine-0.1.55-…` und `strata-control-0.1.55-…`).
- Je Version: Engine- und Control-Paket besorgen (installiert → nur lesen; sonst aus dem GitHub-Release laden, gegen
  `SHA256SUMS.txt` prüfen, in einen Bench-Cache entpacken).
- Je Version ein eigenes Home (`HOME`, `USERPROFILE`, `APPDATA`, `LOCALAPPDATA`, XDG-Variablen; `STRATA_*` aus der
  Umgebung entfernt), in dem genau diese Engine und dieses Control liegen. Das Control dieses Releases startet
  `strata launch --model-path … --yes --no-tui --port <p>` und erzeugt damit seine Default-Config; mit `--tune`
  vorher sein Tuning (`--tune <modus> --retune`).
- Der Checkout ist nur Dirigent und Messclient: Er startet und stoppt die Versionen und schickt die Anfragen über
  HTTP. Messlogik und Anfragen sind für alle Versionen gleich.
- Abbruch vor dem ersten Start, wenn schon ein Strata-Server läuft (`updater.serving_now()` oder Port belegt).
- Startet eine Version nicht, oder läuft eine andere Engine-Version als verlangt (z. B. weil ein Control still
  nachgeladen hat), wird sie als „nicht gelaufen“ mit Grund gemeldet; die anderen laufen weiter.
- Bericht `report.html` und `report.png` im Lauf-Ordner (siehe Bericht).
- Die Configs, die jedes Release-Control erzeugt, und besonders die getunten, werden je Version in
  `<bench-dir>/configs/<ver>/<lauf>/` gesammelt (Run-Config, Tuning-Ergebnis, Choices, Engine-Argumente) und nie
  gelöscht. Das Werkzeug löscht im Bench-Ordner überhaupt nichts; Aufräumen ist Sache des Nutzers.

## Nicht-Ziele

- Eigene Config oder Flags pro Version. Dafür bleibt der bisherige Modus (`config` + `exe=`).
- Engine und Control getrennt mischen (Engine x mit Control y).
- Die normale Instanz stoppen oder neu starten.

## Anti-Ziele

- Schreiben unter dem echten `%LOCALAPPDATA%\Strata`, `~/.strata`, dem Ordner der Live-Config oder den
  Modell-Ordnern des Nutzers (`--model-path` wird nur gelesen).
- Den bisherigen Aufruf `ab_engine.py <config> <arme…>` verändern (Upstream: neue Features opt-in, Default-Pfad
  unverändert).
- Zahlen unter falschem Namen: Läuft nicht die verlangte Engine-Version, gibt es keine Zahlen für diesen Arm.
- Ein Tuning-Ergebnis einer Version für eine andere verwenden.

## Rahmenbedingungen

- Upstream-Regeln (`CONTRIBUTING.md`, `AGENTS.md`): Titel `[Benchmark] …`, ein Problem pro PR, Validierung getrennt
  nach bestanden / gescheitert / nicht gelaufen, nur selbst erzeugte Messwerte. Für ein Feature vorher Issue.
- Target/Backend von `engine_component.detect_backend()`, überschreibbar per `--backend`.
- `components_dir()` folgt `LOCALAPPDATA`/`XDG_DATA_HOME`, solange `STRATA_COMPONENTS_DIR` nicht gesetzt ist
  (engine_component.py:178-188); Status-, Profil- und Laufzeit-Ordner folgen ebenfalls (paths.py, profiles.py,
  runtime.py). `launch --stop` stoppt alle `launch-*`-Profile im umgebogenen Home (cmd.py:515-531). Das gilt für
  den aktuellen Stand; ältere Controls prüft der Builder an ihrem eigenen Code.
- Ein Control kann bei zu alter oder unpassender Engine `strata update --no-control` auslösen (runner.py:685-776)
  und mit `--yes` Downloads zulassen. Darum: Downloads im Lauf blockieren, wo das Control es erlaubt, und in jedem
  Fall nach dem Start prüfen, welche Engine-Version wirklich dient.
- Versionen, die ein Release-Control selbst ablehnt (z. B. unter seinem `MIN_ENGINE`), kommen nicht vor, weil Engine
  und Control aus demselben Release stammen.
- Liegt das GGUF in einem bestehenden Strata-Datenordner, liest `launch` dort vorbereitete Packs
  (discovery.py:453-465). Das ist erlaubt; der Nachweis am Ende deckt diesen Ordner mit ab.
- Starts sind lang (SC117 IQ3_S: bis 1008 s bis „bereit“), Pack-Bau und Tuning kommen dazu. Das Werkzeug nennt
  vorher die Zahl der Starts und Tunings.
- Tests ohne GPU und ohne Netz.

## Ansätze

1. **`ab_engine.py` erweitern** (gewählt): ein Werkzeug, auf das `COMMUNITY_BENCHMARKS.md` für A/B schon verweist.
2. Neues `tools/engine_sweep.py`: mehr Dateien für dieselbe Logik.
3. Nur lokal installierte Versionen: kein Download, praktisch keine Spannen.

Defaults: Christoph hat „Defaults der jeweiligen Version“ gewählt (Engine + Control des Releases), nicht „heutige
Defaults auf jeder Engine“. Damit misst ein Vergleich, was Nutzer der Versionen bekommen hätten, nicht die Engine
allein.

Bericht-PNG: Christoph hat „HTML als einzige Quelle, PNG per Headless-Browser“ gewählt, nicht „Pillow zeichnet
das PNG“ (zweites Layout, das auseinanderläuft).

## Entwurf

- **Versionen auflösen.** `a,b,c` → genau diese. `a..b` → alle Engine-Versionen zwischen a und b (einschließlich),
  zu denen ein Release Engine- und Control-Paket für Target/Backend trägt; nach Version sortiert. Arm-Name `v<ver>`.
- **Pakete besorgen.** Erst die echte Installation (`engine_component.scan()`, Control-Ordner; nur lesen), dann
  Bench-Cache `<bench-dir>/packages/<name>/`, sonst Download + Summe + `updater.extract()`.
- **Home je Version.** `<bench-dir>/homes/<ver>/`; Engine und Control werden darin so bereitgestellt, wie der
  Installer es täte (Verknüpfung oder Kopie, je nachdem was `scan()`/`resolve()` des Release-Controls akzeptiert).
- **Einrichten und Starten.** Im Home mit dem Control des Releases: `strata launch --model-path <p>
  --data-dir <bench-dir>/data --yes --no-tui --port <p>` plus `--tune <modus> --retune`, wenn `--tune` gesetzt.
  Drafting bleibt beim Default; ein Drafter-Download landet einmal in `<bench-dir>/data` und wird geteilt.
- **Messung.** Wie heute: Aufwärmen, gleiche Anfragen, Runden mit abwechselnder Reihenfolge, Mediane. Getunt wird
  einmal je Version vor der ersten Runde; spätere Runden starten nur.
- **Zeiten.** Je Version: „Einrichtung“ = Wanduhr des ersten `launch` (Pack, Drafter, Tuning, Start zusammen),
  „Start“ = Median der reinen Starts ab Runde 2.
- **Versionsprüfung.** Nach jedem Start die tatsächlich dienende Engine-Version lesen; weicht sie ab → Arm
  „nicht gelaufen“.
- **Nachweis.** Vor und nach dem Lauf eine Liste (Name, Größe, Änderungszeit) der echten Strata-Ordner und eines
  Datenordners, in dem `--model-path` liegt. Abweichung = Fehler mit Liste der geänderten Dateien.

## Bericht

- **`report.html`**, eine Datei ohne externe Abhängigkeiten:
  - Oben die Karte, 1600 × 900, im Stil der Release-Bilder: heller Grund mit grünem Schimmer oben rechts; großer
    Titel „Strata 0.1.55 → 0.1.56“ mit grüner Versionsnummer; Untertitel mit GPU, Modell, „default settings“,
    Tuning-Modus. Links eine dunkle Hauptkarte: größte Verbesserung bei den Antworten als große grüne Zahl, darunter
    je Anfrage ein Balkenpaar (grau = Basis, grün = Vergleich) mit tok/s. Rechts weiße Kacheln mit großem Wert:
    langer Prompt (Prompt tok/s), Einrichtung, Start. Unten Chips (Runden, Anfragen, „nicht gelaufen: …“) und
    `github.com/Niko1221/Strata`. Kein Qwen-Logo.
  - Bei einer Spanne vergleicht die Karte die älteste mit der neuesten Version, die gelaufen ist.
  - Darunter die Tabelle: eine Zeile je Version und Anfrage (Decode, Prompt, Einrichtung, Start, getunt, Status),
    Delta zur Basis; Klick auf einen Spaltenkopf sortiert, ein Auswahlfeld setzt die Basisversion.
- **`report.png`**: die Karte allein, aus derselben HTML per Edge/Chrome/Chromium `--headless --screenshot`
  (Fenster 1600 × 900). Kein Browser gefunden → nur HTML, mit Hinweis.

## Dem Builder überlassen

- Release-Liste lesen (GitHub-API ohne Token oder Feed), solange ohne Konto und mit Summenprüfung.
- Ob die Download-Hilfen aus `strata_control.updater` direkt passen.
- Wie das Release-Control aufgerufen wird (`bin/strata.cmd`, eigenes Python) und wie Downloads blockiert werden.
- Was genau im Home als Config/Tuning zählt (bei heutigem Control u. a. `…/Strata/launch/*.json`,
  `*.tune.json`, `*.choices.json`, `~/.strata/data/native-tune/`); bei älteren Controls am Code prüfen.

`--bench-dir` liegt standardmäßig dauerhaft unter `~/strata-ab` (nicht im Temp-Ordner, damit gesammelte Configs
einen Neustart überleben, und nicht unter den Strata-Ordnern der Installation).
- Schriften der Karte: Systemschriften mit sauberem Rückfall, keine eingebetteten Fremdschriften.
- Tests: Versions-Auflösung aus fester Asset-Liste, Umgebungs-Umbiegung, Abbruch-Prüfung, Versionsprüfung,
  „nicht gelaufen“ mit Fake-Server, HTML-Erzeugung aus festen Messwerten; Ablage nach Upstream-Muster.

## PR-Ablauf

1. Issue `[Benchmark] ab_engine: compare Strata versions with their own defaults, isolated from the install`.
2. Branch `feat/ab-engine-versions` in xidus90/Strata, PR gegen `Niko1221/Strata:main`, Issue verlinkt.
3. Validierung: Tests; ein echter Lauf auf RTX 3090 / Ryzen 7 9800X3D mit 0.1.55 gegen 0.1.56 (SC117 IQ3_S), einmal
   ohne, einmal mit `--tune quick`; Nachweis der echten Ordner vorher/nachher; Bericht als Anhang.

Diese Datei geht nicht in den PR.
