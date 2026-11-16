# ab_engine: Strata-Versionen vergleichen – Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `tools/ab_engine.py` vergleicht Strata-Releases (Engine + Control gleicher Version) mit ihren eigenen Defaults, optional getunt, isoliert von der Installation, und schreibt einen HTML-/PNG-Bericht.

**Architecture:** `ab_engine.py` bleibt Einstieg und Messclient. Neues Modul `tools/ab_versions.py` löst Versionen auf, besorgt Pakete, baut je Version ein isoliertes Home und startet dort das Release-Control per `strata launch`. Neues Modul `tools/ab_report.py` erzeugt aus einer `results.json` die HTML-Seite und per Headless-Browser die PNG-Karte.

**Tech Stack:** Python ≥ 3.11 Standardbibliothek (`urllib`, `tarfile`/`zipfile` über `strata_control.updater.extract`, `unittest`), vorhandene Module `strata_control.engine_component`, `strata_control.updater`; Edge/Chrome/Chromium headless für das PNG.

**Spec:** `docs/.superpowers/specs/2026-11-16-ab-engine-versions-design.md`

**Repo:** `D:\GitHub\Strata`, Branch `feat/ab-engine-versions` (von `upstream/main` c77572806). Muss in loomux beschreibbar sein.

## Global Constraints

- Der bisherige Aufruf `python tools/ab_engine.py <config> <arme…> [--rounds …]` verhält sich byte-gleich wie vorher (Ausgabe, `ab_engine.jsonl`).
- Keine Schreibzugriffe unter dem echten `%LOCALAPPDATA%\Strata`, `~/.strata`, dem Ordner der Live-Config oder `--model-path`.
- Nur Standardbibliothek und Module, die schon im Repo sind. Keine neue Abhängigkeit.
- Tests ohne GPU und ohne Netz; Ablage `tools/test_ab_*.py`, Stil `unittest`, Lauf `python -m pytest tools/test_ab_versions.py tools/test_ab_report.py tools/test_ab_engine.py`.
- Engine-Asset: `strata-engine-<ver>-<target>-<backend>.tar.gz`; Control-Asset: `strata-control-<ver>-<target>.(zip|tar.gz)`; Summen in `SHA256SUMS.txt` desselben Releases.
- Arm-Name `v<ver>`; Bench-Ordner Standard `Path.home() / "strata-ab"`; Lauf-Ordner `<bench-dir>/runs/<YYYYmmdd-HHMMSS>/`.
- Das Werkzeug löscht nichts im Bench-Ordner. Configs und Tuning je Version landen in `<bench-dir>/configs/<ver>/<YYYYmmdd-HHMMSS>/` und bleiben.
- Kartenmaß 1600 × 900; Fußzeile `github.com/Niko1221/Strata`; kein Fremdlogo; Systemschriften.
- Code, Kommentare, Meldungen englisch. Commits ohne Co-Author-Zeile.
- PR-Titel `[Benchmark] ab_engine: compare Strata versions with their own defaults, isolated from the install`.

## Review Focus

- Ein Control lädt still eine andere Engine nach → Arm `skipped`, Grund „served engine X, expected Y“ (Task 5, `test_version_mismatch_skips`).
- Abbruch mit Ctrl-C mitten in einer Runde → Server im Test-Home wird gestoppt, Ergebnisse bis dahin bleiben in `results.json` (Task 6, `test_interrupt_stops_server`).
- Abgebrochener Download → kein halbes Paket im Cache; nächster Lauf lädt neu (Task 2, `test_partial_download_not_cached`).
- Spanne oder Liste mit Version ohne passendes Asset → klare Meldung vor dem ersten Start, nicht mitten im Lauf (Task 1, `test_unknown_version_in_list_fails_early`).
- Modellpfad mit `#`, Leerzeichen oder Umlaut (`C:\…\#GIT\…`, `Übung`) → wird unverändert als Argument übergeben, nie über eine Shell (Task 5, `test_launch_args_keep_odd_paths`).

---

### Task 1: Releases lesen und Versionen auflösen

**Files:**
- Create: `tools/ab_versions.py`
- Test: `tools/test_ab_versions.py`

**Interfaces:**
- Produces:
  - `@dataclass(frozen=True) class ReleasePair: version: str; tag: str; engine_name: str; engine_url: str; control_name: str; control_url: str; sums_url: str`
  - `def pairs_from_releases(releases: list[dict], target: str, backend: str) -> dict[str, ReleasePair]` – Eingabe ist die JSON-Liste von `GET /repos/Niko1221/Strata/releases`; erstes Vorkommen einer Version (neuestes Release zuerst) gewinnt; nur Versionen, für die Engine **und** Control da sind.
  - `def resolve_versions(spec: str, available: dict[str, ReleasePair]) -> list[str]` – `"a,b"` oder `"a..b"`, aufsteigend nach Versionstupel sortiert; `ValueError` mit Liste der fehlenden Versionen.
  - `def list_releases(repo: str = "Niko1221/Strata", opener=urllib.request.urlopen) -> list[dict]` – paginiert, ohne Token.

- [ ] **Step 1: Write the failing tests**

```python
FIXTURE = [  # newest first, as the API returns them
    {"tag_name": "v0.2.10", "assets": [a("strata-engine-0.1.56-windows-x86_64-cuda-cuda13.tar.gz"),
                                       a("strata-control-0.1.56-windows-x86_64.zip"), a("SHA256SUMS.txt")]},
    {"tag_name": "v0.2.9",  "assets": [a("strata-engine-0.1.55-windows-x86_64-cuda-cuda13.tar.gz"),
                                       a("strata-control-0.1.55-windows-x86_64.zip"), a("SHA256SUMS.txt")]},
    {"tag_name": "v0.2.8",  "assets": [a("strata-engine-0.1.54-windows-x86_64-cuda-cuda12.tar.gz"),
                                       a("strata-control-0.1.54-windows-x86_64.zip"), a("SHA256SUMS.txt")]},
]

def test_pairs_need_engine_and_control_for_backend(self):
    p = pairs_from_releases(FIXTURE, "windows-x86_64", "cuda-cuda13")
    self.assertEqual(sorted(p), ["0.1.55", "0.1.56"])
    self.assertEqual(p["0.1.55"].tag, "v0.2.9")

def test_range_is_inclusive_and_sorted(self):
    p = pairs_from_releases(FIXTURE, "windows-x86_64", "cuda-cuda13")
    self.assertEqual(resolve_versions("0.1.50..0.1.56", p), ["0.1.55", "0.1.56"])
    self.assertEqual(resolve_versions("0.1.56,0.1.55", p), ["0.1.55", "0.1.56"])

def test_unknown_version_in_list_fails_early(self):
    with self.assertRaisesRegex(ValueError, "0.1.54"):
        resolve_versions("0.1.54,0.1.55", pairs_from_releases(FIXTURE, "windows-x86_64", "cuda-cuda13"))

def test_empty_range_fails(self):
    with self.assertRaises(ValueError):
        resolve_versions("0.1.60..0.1.70", pairs_from_releases(FIXTURE, "windows-x86_64", "cuda-cuda13"))
```

- [ ] **Step 2:** `python -m pytest tools/test_ab_versions.py -v` → FAIL (`ImportError`).
- [ ] **Step 3:** Implement the four functions in `tools/ab_versions.py`. Version comparison via `tuple(int(x) for x in v.split("."))`. Asset name parsed with one regex per kind; `browser_download_url` used as URL.
- [ ] **Step 4:** Tests → PASS.
- [ ] **Step 5:** `git commit -m "feat(bench): resolve Strata release pairs for ab_engine"`

### Task 2: Pakete laden, prüfen, cachen

**Files:**
- Modify: `tools/ab_versions.py`
- Test: `tools/test_ab_versions.py`

**Interfaces:**
- Consumes: `ReleasePair` (Task 1); `strata_control.updater.extract(archive, destination, fmt, installed_bytes)`.
- Produces:
  - `@dataclass(frozen=True) class Packages: version: str; engine_dir: Path; control_dir: Path`
  - `def parse_sums(text: str) -> dict[str, str]` – Dateiname → sha256.
  - `def ensure_packages(pair: ReleasePair, cache: Path, opener=urllib.request.urlopen) -> Packages` – Ziel `cache/<asset-stem>/`; vorhandener Ordner mit Marker `.ok` = Cache-Treffer ohne Netz; Download nach `<name>.part`, Summenprüfung, dann Entpacken nach `<stem>.tmp` und Umbenennen; Marker zuletzt.

- [ ] **Step 1: Write the failing tests** (Fake-Opener liefert Bytes aus einem Dict URL → bytes; Archive werden im Test mit `tarfile`/`zipfile` erzeugt)

```python
def test_checksum_mismatch_raises_and_leaves_no_cache(self): ...
    # sums list a wrong hash -> ValueError("sha256 mismatch ..."); cache dir has no <stem>/ and no .part
def test_cache_hit_needs_no_network(self): ...
    # second call with an opener that raises -> same Packages, no exception
def test_partial_download_not_cached(self): ...
    # opener raises mid-stream on the engine archive -> exception; next call with a good opener downloads again
def test_parse_sums_handles_star_and_spaces(self):
    self.assertEqual(parse_sums("abc *x.tar.gz\ndef  y.zip\n"), {"x.tar.gz": "abc", "y.zip": "def"})
```

- [ ] **Step 2:** Tests → FAIL.
- [ ] **Step 3:** Implement. Installierte Komponenten der echten Installation werden **nicht** benutzt (Spec: „nur lesen“ ist erfüllt, indem gar nicht darauf zugegriffen wird; einfacher und ohne Risiko). Vermerk im Docstring.
- [ ] **Step 4:** Tests → PASS.
- [ ] **Step 5:** `git commit -m "feat(bench): download and verify release packages into a bench cache"`

### Task 3: Isoliertes Home

**Files:**
- Modify: `tools/ab_versions.py`
- Test: `tools/test_ab_versions.py`

**Interfaces:**
- Consumes: `Packages` (Task 2); `strata_control.engine_component.components_dir(environ)`, `scan(root)`.
- Produces:
  - `def isolated_env(home: Path, base: Mapping[str, str]) -> dict[str, str]` – setzt `HOME`, `USERPROFILE`, `APPDATA`, `LOCALAPPDATA`, `XDG_STATE_HOME`, `XDG_DATA_HOME`, `XDG_CONFIG_HOME`, `XDG_CACHE_HOME` unter `home`; entfernt jeden Schlüssel mit Präfix `STRATA_`; setzt danach `STRATA_NO_ACQUIRE=1`.
  - `def provision_home(home: Path, packages: Packages) -> Path` – legt die Engine unter `components_dir(env)/engine/<engine_dir.name>` und das Control unter `components_dir(env)/control/<version>` samt `active.json` (`{"version", "folder"}`) ab; Kopie per `shutil.copytree`, Rückgabe Pfad zum Control.

- [ ] **Step 1: Write the failing tests**

```python
def test_env_points_inside_home_and_scrubs_strata(self):
    env = isolated_env(home, {"LOCALAPPDATA": r"C:\real", "STRATA_COMPONENTS_DIR": "x", "PATH": "p"})
    self.assertTrue(Path(env["LOCALAPPDATA"]).is_relative_to(home))
    self.assertNotIn("STRATA_COMPONENTS_DIR", env)
    self.assertEqual(env["STRATA_NO_ACQUIRE"], "1")
    self.assertEqual(env["PATH"], "p")

def test_components_dir_follows_env(self):
    self.assertTrue(components_dir(isolated_env(home, {})).is_relative_to(home))

def test_provisioned_engine_is_the_only_one_scan_sees(self):
    provision_home(home, fake_packages("0.1.55"))   # fake engine dir with a minimal manifest read_manifest accepts
    found, _ = scan(components_dir(isolated_env(home, {})) / "engine")
    self.assertEqual([c.version for c in found], ["0.1.55"])
```

- [ ] **Step 2:** Tests → FAIL.
- [ ] **Step 3:** Implement. Das minimale Manifest für den Test aus einer echten Engine-Paket-Struktur ableiten (`engine_component.read_manifest`); als Fixture unter `tools/` nur, wenn es klein bleibt, sonst im Test erzeugen.
- [ ] **Step 4:** Tests → PASS.
- [ ] **Step 5:** `git commit -m "feat(bench): build an isolated Strata home per version"`

### Task 4: Abbruch-Prüfung und Nachweis

**Files:**
- Modify: `tools/ab_versions.py`
- Test: `tools/test_ab_versions.py`

**Interfaces:**
- Produces:
  - `class Busy(RuntimeError)`
  - `def assert_idle(ports: Iterable[int], serving=updater.serving_now) -> None` – `Busy`, wenn `serving()` nicht leer ist oder ein Port auf `127.0.0.1` eine Verbindung annimmt.
  - `def snapshot(roots: Iterable[Path]) -> dict[str, tuple[int, int]]` – relativer Pfad mit Wurzel-Präfix → `(size, mtime_ns)`; fehlende Wurzel = leer.
  - `def changed(before: dict, after: dict) -> list[str]` – sortiert: neu, weg, geändert.
  - `def live_roots(env: Mapping[str, str], model_path: Path) -> list[Path]` – echte Strata-Ordner aus `components_dir(env)`-Elternordner (`…/Strata`), `~/.strata`, plus `model_path`s Strata-Datenordner, wenn `model_path` unter einem `models/`-Ordner liegt.

- [ ] **Step 1: Write the failing tests**

```python
def test_busy_when_port_open(self): ...           # bind a socket on a free port -> Busy
def test_busy_when_strata_serving(self):
    with self.assertRaises(Busy): assert_idle([], serving=lambda: [{"pid": 1}])
def test_idle_passes(self): assert_idle([free_port()], serving=lambda: [])
def test_changed_lists_new_removed_modified(self):
    self.assertEqual(changed({"a": (1, 1), "b": (1, 1)}, {"a": (2, 1), "c": (1, 1)}), ["+c", "-b", "~a"])
```

- [ ] **Step 2:** Tests → FAIL. **Step 3:** Implement. **Step 4:** Tests → PASS.
- [ ] **Step 5:** `git commit -m "feat(bench): refuse to run beside a live server and prove the install is untouched"`

### Task 5: Release-Control starten, stoppen, Version prüfen

**Files:**
- Modify: `tools/ab_versions.py`
- Test: `tools/test_ab_versions.py`

**Interfaces:**
- Consumes: `Packages`, `isolated_env`, `provision_home`.
- Produces:
  - `@dataclass class LaunchResult: ok: bool; seconds: float; reason: str = ""`
  - `def control_command(control_dir: Path) -> list[str]` – Windows `[control_dir/"bin"/"strata.cmd"]`, sonst `[control_dir/"bin"/"strata"]`.
  - `def launch(control_dir: Path, env: dict, model_path: Path, data_dir: Path, port: int, tune: str | None, timeout: float = 7200) -> LaunchResult` – Argumente: `launch --model-path <m> --data-dir <d> --port <p> --yes --no-tui --output json` und, wenn `tune`, `--tune <tune> --retune`; Liste an `subprocess.run`, nie `shell=True`; `ok` = Exit 0 und `GET /health` liefert `status == "ok"`; sonst `reason` = letzte nicht leere Zeile aus stdout/stderr oder Engine-Log.
  - `def stop(control_dir: Path, env: dict) -> None` – `launch --stop`, Fehler werden geloggt, nicht geworfen.
  - `def served_engine(home: Path) -> Path | None` – neueste Zeile `engine started: <exe> --serve` aus `home/**/*.engine.log`; Rückgabe `Path(exe).parent`.
  - `def version_problem(home: Path, packages: Packages) -> str | None` – `None`, wenn `served_engine(home).name == packages.engine_dir.name`, sonst `"served engine <x>, expected <y>"`.
  - `CONFIG_GLOBS = ("**/Strata/launch/*.json", "**/.strata/data/native-tune/**/*", "**/Strata/tuning/**/*")`
  - `def collect_configs(home: Path, dest: Path) -> list[Path]` – kopiert jede Datei aus `CONFIG_GLOBS` (ohne `*.log`) mit relativem Pfad nach `dest`; überschreibt nie (vorhandene Zieldatei → Suffix `.1`, `.2`); Kopie per `shutil.copy2`, Inhalt unverändert.

- [ ] **Step 1: Write the failing tests** (Fake-Control: `bin/strata(.cmd)` ruft ein kleines Python-Skript, das Argumente nach `args.json` schreibt, eine Engine-Log-Zeile schreibt und Exit-Code aus einer Datei nimmt; `/health` per Fake-Opener)

```python
def test_launch_passes_tune_and_retune(self): ...      # args.json contains "--tune","quick","--retune"
def test_launch_without_tune_has_no_tune_flags(self): ...
def test_launch_args_keep_odd_paths(self): ...          # model path r"C:\x\#GIT\Übung dir\m.gguf" arrives unchanged
def test_failed_launch_reports_last_line(self): ...     # exit 3, last stderr line "unknown flag --vram-elastic"
def test_version_mismatch_skips(self): ...              # log names 0.1.56 folder, packages 0.1.55 -> message
def test_served_engine_takes_newest_line(self): ...
def test_collect_configs_keeps_tuned_files_and_never_overwrites(self): ...
    # home with launch/x.json, launch/x.tune.json, launch/x.engine.log -> dest has both json, no log;
    # second call into the same dest -> x.tune.json.1 appears, original unchanged
```

- [ ] **Step 2:** Tests → FAIL. **Step 3:** Implement. **Step 4:** Tests → PASS.
- [ ] **Step 5:** `git commit -m "feat(bench): start a release's own control in the isolated home and check the served engine"`

### Task 6: Modus „Versionen“ in ab_engine.py

**Files:**
- Modify: `tools/ab_engine.py`
- Test: `tools/test_ab_engine.py` (neu)

**Interfaces:**
- Consumes: alles aus Task 1–5.
- Produces:
  - CLI: `config` wird optional (`nargs="?"`), `arms` `nargs="*"`; neue Optionen `--model-path`, `--engines`, `--tune {quick,thorough}`, `--bench-dir`, `--backend`. Genau einer der Modi: `config`+`arms` **oder** `--model-path`+`--engines`, sonst `ap.error`.
  - `def run_requests(port: int, reqs: list[tuple[str, str]], max_new: int, rnd: int, arm: str, out) -> list[dict]` – aus der bestehenden Schleife herausgezogen; der alte Modus ruft sie und druckt wie bisher.
  - `def run_versions(a: argparse.Namespace) -> int` – Ablauf: Releases → `resolve_versions` → Zahl der Starts/Tunings ausgeben → `assert_idle` → `snapshot` → je Version `ensure_packages`, `provision_home`; Runde 0 mit `tune`, spätere ohne; Reihenfolge abwechselnd wie heute; je Start `launch` → `collect_configs(home, <bench-dir>/configs/<ver>/<lauf>)` (auch wenn `launch` scheiterte) → `version_problem` → `run_requests` → `stop` in `finally`; nichts im Bench-Ordner wird gelöscht; Versionen mit Problem werden `skipped` und in späteren Runden ausgelassen → `snapshot` + `changed` → `results.json` → `ab_report.write_report`. Exit 1, wenn `changed` nicht leer.
  - `results.json` (Eingang für Task 7):
    `{"created": iso, "model": str, "gpu": str, "tune": str|null, "rounds": int, "requests": [tag…], "arms": [{"version": str, "status": "ran"|"skipped", "reason": str, "setup_s": float|null, "start_s": [float…], "rows": [{"round", "req", "decode", "prompt", "prompt_n", "n"}]}], "untouched": bool, "changed": [str…]}`; `gpu` aus `nvidia-smi --query-gpu=name,memory.total`, sonst `""`.

- [ ] **Step 1: Write the failing tests** (Funktionen aus `ab_versions` per `unittest.mock.patch` ersetzt; `run_requests` gegen einen Fake)

```python
def test_old_mode_argv_unchanged(self): ...      # parse ["cfg.json", "A:exe=x", "--rounds", "2"] like before
def test_modes_are_exclusive(self): ...          # cfg + --engines -> SystemExit
def test_skipped_arm_is_reported_not_dropped(self): ...  # version_problem for 0.1.55 -> results arms[0].status == "skipped", reason kept
def test_tune_only_in_first_round(self): ...     # launch called with tune="quick" once per version, then tune=None
def test_interrupt_stops_server(self): ...       # run_requests raises KeyboardInterrupt -> stop called, results.json written
def test_changed_install_fails_run(self): ...    # changed() returns ["~x"] -> exit 1, results["untouched"] is False
def test_configs_collected_per_version_even_on_failed_launch(self): ...  # collect_configs called for each launch, dest under configs/<ver>/
def test_nothing_deleted_in_bench_dir(self): ... # file planted in bench-dir before the run still exists afterwards
```

- [ ] **Step 2:** Tests → FAIL. **Step 3:** Implement. **Step 4:** Tests → PASS, dazu der alte Modus einmal gegen einen Fake-Server: Ausgabe und `ab_engine.jsonl` gleich wie vor der Änderung (Vergleich mit `git stash`-Stand).
- [ ] **Step 5:** `git commit -m "feat(bench): ab_engine compares Strata versions with their own defaults"`

### Task 7: Bericht HTML + PNG

**Files:**
- Create: `tools/ab_report.py`
- Test: `tools/test_ab_report.py`

**Interfaces:**
- Consumes: `results.json`-Schema aus Task 6.
- Produces:
  - `def card_pair(results: dict) -> tuple[dict, dict] | None` – älteste und neueste Version mit `status == "ran"`; `None`, wenn weniger als zwei liefen.
  - `def build_html(results: dict) -> str` – eine Datei, CSS und JS inline, keine externen URLs.
  - `def find_browser() -> str | None` – in dieser Reihenfolge: `msedge`, `chrome`, `chromium`, `chromium-browser`, `google-chrome` über `shutil.which`, unter Windows zusätzlich die Standardpfade von Edge und Chrome unter `Program Files (x86)`/`Program Files`.
  - `def render_png(browser: str, html: Path, png: Path) -> bool` – `--headless=new --disable-gpu --hide-scrollbars --window-size=1600,900 --screenshot=<png> <html-url>#card`.
  - `def write_report(results: dict, out_dir: Path) -> tuple[Path, Path | None]`

Karteninhalt (feste Texte englisch, Werte aus `results`):
- Titel `Strata <alt> → <neu>`, Versionsteil grün (`#16a34a`-Ton wie Vorbild), Untertitel `<gpu> · <model> · default settings · tuned <modus>|not tuned`.
- Dunkle Hauptkarte: „Answers, up to“ + größte relative Verbesserung der Decode-Mediane über die Anfragen; je Anfrage Balkenpaar grau (alt) / grün (neu) mit Zahl.
- Kacheln: „Long prompt“ (Prompt tok/s, Anfrage `long`, relative Änderung), „Setup“ (`setup_s` alt → neu), „Start“ (Median `start_s` alt → neu). Verschlechterung wird als negativer Wert gezeigt, nicht versteckt.
- Chips: `<rounds> rounds`, `<n> requests`, je übersprungener Version `<ver> did not run`.
- Fußzeile `github.com/Niko1221/Strata`.
- Tabelle unter der Karte: Spalten Version, Request, Decode, Prompt, Δ Decode, Setup, Start, Status; Klick auf Kopf sortiert (numerisch, wo Zahl); `<select>` setzt Basisversion, Δ wird neu berechnet. Hell/Dunkel per `prefers-color-scheme` nur für die Tabelle; die Karte bleibt hell wie im Vorbild.

- [ ] **Step 1: Write the failing tests**

```python
def test_card_compares_oldest_and_newest_that_ran(self):  # 0.1.54 skipped, 0.1.55, 0.1.56, 0.1.57 ran -> (0.1.55, 0.1.57)
def test_card_none_with_one_version(self): ...
def test_html_has_title_rows_and_no_external_urls(self):
    h = build_html(RESULTS)
    self.assertIn("Strata 0.1.55 → 0.1.56", h)
    self.assertEqual(h.count('<tr data-version='), 8)          # 2 versions x 4 requests
    self.assertNotRegex(h, r'(src|href)="https?://')
def test_skipped_version_appears_as_chip_and_row(self): ...
def test_regression_shown_negative(self): ...                # newer slower -> "-" sign in card
def test_no_browser_writes_html_only(self): ...              # find_browser patched to None -> (html, None)
```

- [ ] **Step 2:** Tests → FAIL. **Step 3:** Implement. **Step 4:** Tests → PASS.
- [ ] **Step 5:** Sichtprüfung: `write_report` mit `RESULTS` aus dem Test in einen Temp-Ordner, PNG mit Edge erzeugen, gegen das Vorbild ansehen (Abstände, Zahlengröße, Balken). Abweichungen korrigieren.
- [ ] **Step 6:** `git commit -m "feat(bench): ab_engine writes a sortable HTML report and a release-style PNG card"`

### Task 8: Doku, echter Lauf, Issue und PR

**Files:**
- Modify: `tools/ab_engine.py` (Docstring: neuer Modus mit Beispiel), `docs/COMMUNITY_BENCHMARKS.md` (bei „for A/B tests of two builds use `tools/ab_engine.py`“ ein Satz und ein Beispiel für den Versionsmodus).

- [ ] **Step 1:** Docstring und Doku ergänzen, Stil von `AGENTS.md` (kurze Sätze, nichts ohne Messung behaupten). Commit `docs(bench): describe ab_engine's version mode`.
- [ ] **Step 2:** Alle Tests: `python -m pytest tools/test_ab_versions.py tools/test_ab_report.py tools/test_ab_engine.py tests/terminal -q` → alles grün.
- [ ] **Step 3:** Echter Lauf auf RTX 3090 (normale Instanz vorher beenden): `python tools/ab_engine.py --model-path D:\Strata\models\sc117-abliterated\IQ3_S\Qwen3.8-Flash-Next-GSQ-RCO-abliterated-IQ3_S-00001-of-00002.gguf --engines 0.1.55,0.1.56 --rounds 1`, dann dasselbe mit `--tune quick`. Erwartet: `untouched: true`, `report.html` und `report.png`. Ergebnisse und Ausfälle festhalten, nichts schönen.
- [ ] **Step 4:** Issue anlegen (nach Freigabe durch Christoph): Titel siehe Global Constraints; Problem, Verhalten, Abwägungen.
- [ ] **Step 5a:** Den Commit mit Spec und Plan (`docs/.superpowers/`) vor dem Push aus dem PR-Branch entfernen (`git rebase --onto <commit> <commit>^` ohne `-i`, oder einen PR-Branch ohne ihn neu aufbauen). Die Dateien bleiben lokal erhalten.
- [ ] **Step 5:** Push des Branches nach `origin` (xidus90/Strata), PR gegen `Niko1221/Strata:main` (nach Freigabe): Problem und Ergebnis zuerst, dann Ansatz, dann Validierung getrennt nach bestanden / gescheitert / nicht gelaufen, dann Grenzen; Issue verlinkt; `report.png` als Bild.
