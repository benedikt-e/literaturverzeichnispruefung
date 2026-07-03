# Datenbank-Routing — Fachgebiet & Typ-Kaskaden

Diese Datei steuert, **welche Datenbanken für welche Quelle abgefragt werden**. Sie ist bewusst leicht
editierbar: eine Kaskaden-Zeile ändern, eine Prioritätsliste umsortieren oder ein Fachgebiet ergänzen,
ohne die Hauptlogik der Skill anzufassen.

Zwei Dinge bestimmen die Kaskade für eine Quelle:
1. **Publikationstyp** (aus dem BibTeX-Eintragstyp / der Einordnung in Step 1) — der primäre Schalter.
2. **Fachgebiet** — verfeinert die Kaskade und ergänzt fachspezifische Datenbanken.

**Rate-Limit-Prinzip:** Datenbanken innerhalb einer Kaskade stehen **in Prioritätsreihenfolge**. Von
oben nach unten abfragen und **beim ersten bestätigten Treffer** mit kanonischer URL **stoppen** (gemäß
SKILL.md Step 3). Erst bei einem Fehlschlag oder unsicheren Treffer weiter nach unten. Das hält
API-Calls — und Rate-Limit-Druck — minimal.

**Ordnungsregel — günstig-und-wahrscheinlich zuerst, verbose-und-throttled zuletzt.** Jede Kaskade
stellt die Datenbanken nach vorn, die (a) **günstig abzufragen** sind — JSON, keine Chunk-Reads,
großzügige Limits — **und** (b) für diesen Quellentyp **häufig treffen**. Die verbosen oder
throttle-anfälligen rücken nach hinten:

- **K10plus** mit der Dublin-Core-Standardabfrage (`recordSchema=dc&maximumRecords=1`, siehe
  api-endpoints.md) ist token-günstig **und** hat die höchste Trefferquote für deutsche Bücher → bleibt
  **vorne** in den Deutsch-Buch-Kaskaden. (OpenAlex davorzusetzen hilft nicht: seine Deutsch-Buch-
  Abdeckung ist lückenhafter, es würde oft verfehlen und der K10plus-Call käme ohnehin — zwei statt
  einem Call.)
- **DNB** ist MARCXML-verbose **und** throttelt aktiv → rückt **hinter** die günstigen JSON-Kataloge
  (lobid, OpenAlex) und dient nur als **später Fallback**, obwohl die Abdeckung hoch ist.
- **Semantic Scholar** (striktes Limit 100/5 min) bleibt nach OpenAlex/Crossref.
- **DOI vorhanden?** Den DOI immer zuerst direkt auflösen (billigste Bestätigung), vor jeder Kaskade —
  in header-losen Umgebungen über Crossref `filter=doi:` (auch als Batch), siehe api-endpoints.md → Crossref.

---

## Teil A — Fachgebiets-Erkennung & Nutzer-Override

### Auto-Erkennung (Standard)

Das Fachgebiet **einmal für das gesamte Verzeichnis** aus diesen Signalen ableiten:
- **Zeitschriften-/Reihennamen** (z. B. „Kölner Zeitschrift für Soziologie" → Sozialwissenschaften;
  „Journal of Finance" → Wirtschaft),
- **Verlage** (z. B. Campus, VS/Springer VS → Sozialwissenschaften; Nomos, C.H. Beck → Recht),
- **Titel-Keywords** und **Sprache**,
- **Identifier-Muster** (arXiv-IDs → STEM; PubMed-IDs → Medizin).

Eine einzige beste Schätzung mit grober Konfidenz bilden.

### Einmalige Bestätigung (kein Mikromanagement)

Das erkannte Fachgebiet und die resultierenden Datenbanken **einmal** für die ganze Liste nennen — nie
pro Quelle. Beispiel:

> Erkanntes Fachgebiet: **Sozialwissenschaften**. Abgefragte Datenbanken: K10plus, OpenAlex,
> GESIS, DNB (+ Crossref für Artikel). Stimmt das, oder soll ich ein anderes Fachgebiet nehmen?

Ist die Konfidenz hoch, fortfahren und die Annahme einfach im Abschlussbericht vermerken. Nur bei
wirklich gemischten oder schwachen Signalen für eine Bestätigung innehalten. **Höchstens einmal fragen.**

### Manueller Override

Der Nutzer kann das Fachgebiet explizit setzen; das übersteuert die Auto-Erkennung:

```
--fachgebiet sozialwissenschaften | wirtschaft | recht | stem | medizin | geistes | allgemein
```

### Gemischte Verzeichnisse

Das globale Fachgebiet setzt die **Standard-Kaskade**. Einzelne Quellen können dennoch nach
**Publikationstyp** abweichen (Teil B) — z. B. ein einzelner Statistik-Artikel in einer
soziologischen Arbeit läuft weiterhin durch die Artikel-Kaskade (+ arXiv, falls er nach Preprint
aussieht). Den Nutzer zu diesen Abweichungen pro Quelle nicht erneut fragen; still handhaben.

---

## Teil B — Typ-Kaskaden (primärer Schalter)

| Publikationstyp | BibTeX-Typ | Kaskade (Prioritätsreihenfolge) |
|---|---|---|
| Zeitschriftenartikel | `@article` | OpenAlex → Crossref → (Semantic Scholar, nur mit Key) → Fach-DB |
| Deutsche Monografie / Sammelband | `@book` | K10plus (DC) → OpenAlex → lobid* → DNB → (Open Library, nur ISBN) |
| Nicht-deutsches / internationales Buch | `@book` | K10plus (DC) → Open Library (nur ISBN) → OpenAlex → (Library of Congress) |
| Buchkapitel | `@incollection` | Host-Volume über K10plus (DC) → lobid* → DNB (Sammelbände-Regel anwenden) |
| Konferenzbeitrag | `@inproceedings` | OpenAlex → Crossref → K10plus |
| Preprint | `@misc` / `@article` | arXiv* → OpenAlex → Crossref (arXiv-DOIs `10.48550/…` direkt via Crossref/OpenAlex) |
| Report / Working Paper **mit DOI/ISBN/Handle** | `@techreport` | DOI-Resolve / Repositorium (econstor, SSRN) → OpenAlex → K10plus → lobid → [BASE, falls Key] |
| Hochschulschrift / Dissertation / Preprint / OA-Repositoriumsinhalt | `@phdthesis`/`@misc` | OpenAlex → [BASE, falls Key] → K10plus/DNB (Diss.) → arXiv (Preprint) |
| Graue Literatur **mit URL/Herausgeber** (Webseite, Behörden-/Regierungs-/Parlamentsdokument, Pressemitteilung, Nachricht, Koalitionsvertrag) | `@misc` / `@online` | **Graue-Literatur-Prüfung** (SKILL.md Step 2 / category-rules.md): URL auflösen + Herausgeber/Titel/Datum abgleichen → I–IV. **Keine** Katalog-Kaskade, **kein** 🎓/⚪ als Default. |

`( )` = nur, wenn die höher priorisierten Datenbanken verfehlen oder unsicher sind.
`*` = **Canary-pflichtig:** Endpunkt ist in geteilten Agent-Umgebungen typischerweise nicht erreichbar
(leerer Body — lobid, arXiv, Open Library `search.json`, EconBiz, Semantic Scholar keyless; siehe
api-endpoints.md → „Bekannte Umgebungsausfälle"). Erster Call = Canary; scheitert er, den Endpunkt
sitzungsweit überspringen — die Kaskade läuft mit den übrigen Datenbanken weiter.

**BASE (nur mit `--base-key`):** ergänzt die Grey-Literature-/Hochschulschriften-/OA-Zweige, **nicht**
die Monografie- oder Artikel-Kaskaden (BASE ist OA-/Repositorien-Aggregator, kein Buch-/Verlagskatalog —
ein BASE-Miss für ein Buch/einen Artikel ist **kein** Befund, siehe SKILL.md → Klassifikations-Sperre).
Strikte BASE-Regeln: Key-Pflicht + Canary, **ein Aufruf pro Antwortschritt, strikt seriell, nie parallel** (1,5 s = Begründung; api-endpoints.md → BASE).

**OpenAlex-Ausfall (kein API-Key / Canary gescheitert):** Scheitert der Sitzungs-Canary (SKILL.md,
Step 0) — ohne `--openalex-key` in vielen Agent-Umgebungen der Normalfall —, gilt für die **gesamte Sitzung** die
Crossref-primäre Artikel-Kaskade:
`Crossref → (Semantic Scholar) → Fach-DB → ZDB SRU (nur Zeitschriften-Existenz/ISSN)`.
OpenAlex dann auch aus den Buch-Kaskaden streichen (dort übernehmen lobid/DNB).
**K10plus (DC)** = immer mit der Dublin-Core-Standardabfrage beginnen (günstig); erst auf MARCXML
eskalieren, wenn Feld 520/250 gebraucht wird (siehe api-endpoints.md). **DNB zuletzt** in
Buch-Kaskaden: verbose + throttled, nur abfragen, wenn die günstigeren Kataloge alle verfehlen.

**Niemals Verlags-Zeitschriftenseiten direkt abrufen.** Bei Artikeln von De Gruyter (degruyter.com),
Wiley, Taylor & Francis, SAGE oder Elsevier liefert `web_fetch` nur eine leere JS-Shell — den
Direktversuch überspringen und **direkt zu Crossref** (DOI-Direkt-Resolve oder ISSN-Filter nach dem
ISSN-Lookup). Vollständige Liste in api-endpoints.md → „Nicht per web_fetch abrufbare Verlagsseiten".

**ISSN-Lookup deutscher Zeitschriften — ZDB zuerst, dann Crossref über ISSN-Direktendpunkt.** Bei
deutschsprachigen Zeitschriften ohne DOI/ISSN (Heuristik: deutscher Titel/Verlag) **zuerst die ISSN
über ZDB SRU** ermitteln (`tit all`, api-endpoints.md → ZDB), weil die Crossref-`/journals`-
Freitextsuche bei kurzen deutschen Titeln versagt; danach Crossref über den **direkten ISSN-Endpunkt**
`/journals/{ISSN}` bzw. `/journals/{ISSN}/works` ansteuern. Für internationale Zeitschriften bleibt
`/journals?query=` der reguläre Einstieg. Details + ISSN-Coverage-Heuristik in api-endpoints.md → Crossref.

---

## Teil C — Fachgebiets-Kaskaden (Verfeinerung)

Diese ergänzen oder repriorisieren Datenbanken über Teil B hinaus. Frei editierbar.

### sozialwissenschaften (Soziologie, Politik, Erziehung)
Primär: **K10plus, OpenAlex, GESIS, DNB**  ·  Artikel zusätzlich: **Crossref**  ·  Ergänzend: lobid*

### wirtschaft (VWL, BWL, Management)
Primär: **OpenAlex, Crossref, ZBW EconBiz***  ·  Bücher: **K10plus (DC), lobid*** → Fallback DNB  ·  Ergänzend: arXiv* (`econ.*`), DOAJ

### recht
Primär: **K10plus (DC), OpenAlex, lobid***  ·  Fallback: DNB
(Hinweis: Viel juristische Literatur ist von offenen APIs schlecht abgedeckt — mit mehr ⚪ ? und vorbereiteten Links rechnen.)

### stem (Physik, Informatik, Mathematik, Technik, Naturwiss.)
Primär: **OpenAlex, Crossref, arXiv***  ·  Ergänzend: Semantic Scholar (nur mit Key), DataCite

### medizin / life sciences
Primär: **OpenAlex, Crossref**  ·  Ergänzend: Europe PMC / PubMed (noch nicht als Live-API implementiert — vorbereiteter Link)

### geistes (Geschichte, Philologie, Philosophie, …)
Primär: **K10plus (DC), OpenAlex, lobid***  ·  Bücher: **Open Library (nur ISBN), Library of Congress**  ·  Fallback: DNB

### allgemein / unsicher
Rein nach Publikationstyp routen (Teil B). Standard-Buch-Kaskade: **K10plus → DNB → OpenAlex → Open Library**.
Standard-Artikel-Kaskade: **OpenAlex → Crossref**.

---

## Teil D — FernUni-Add-on (automatisch ausgelöst bei Erkennung)

Automatisch ausgelöst, sobald ein FernUni-Signal erkannt wird (Verlag/Ort „Hagen: FernUniversität",
Studienbrief-/Lerneinheitsformat); `--kontext fernuni` nur als expliziter Override (siehe SKILL.md +
category-rules.md). Die Überprüfung wird **immer versucht**; wird nichts gefunden, wird die Quelle
**🎓 FU**, niemals 🔴 IV. Ergänzt die jeweils zutreffende Kaskade:
- „Hagen: FernUniversität" als Verlagssignal behandeln → gezielte K10plus/DNB-Abfrage mit diesem Zusatz.

---

## Teil E — Vorbereitete Suchlinks (keine Live-Abfrage)

Diese werden **nicht von der Skill abgefragt** — sie werden dem Nutzer als anklickfertige Links im
„Belege & Nachschlagen"-Block übergeben (siehe SKILL.md → Ausgabeformat). Für Datenbanken ohne freie
API und als allgemeine manuelle Prüfhilfe:

- **Google Scholar:** `https://scholar.google.com/scholar?q=` + Autor(en) + 2–3 Titel-Keywords + Jahr
- **WorldCat:** `https://search.worldcat.org/search?q=` + gleiche Query (keine freie API; institutionell/kostenpflichtig)
- **BASE:** `https://www.base-search.net/Search/Results?lookfor=` + gleiche Query — die Web-Oberfläche
  als manueller Suchlink. **Live-Abfrage:** BASE wird mit `--base-key` automatisch abgefragt (key-gated,
  Canary, hartes 1,5-s-Limit; siehe api-endpoints.md → BASE). Ohne Key oder bei Block in der Umgebung
  bleibt nur dieser Web-Link.
- **EconBiz:** `https://www.econbiz.de/Search/Results?lookfor=` + gleiche Query — als manueller
  Prüflink, wenn die EconBiz-API in der Umgebung nicht erreichbar ist (Canary gescheitert).
