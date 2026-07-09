---
name: literaturverzeichnispruefung
description: Verify academic references and bibliographies for citation errors and AI hallucinations using live database APIs (OpenAlex, OpenAIRE, K10plus, lobid, DNB, Crossref, Semantic Scholar, ZBW EconBiz, Open Library, arXiv, Library of Congress, GESIS). Trigger whenever a user wants to check, verify, or validate ANY bibliography, reference list, or Literaturverzeichnis — their own or someone else's (a student's thesis, a colleague's paper, an AI-generated draft). Trigger on phrasings like "prüfe mein/dieses/das Literaturverzeichnis", "kannst du dieses Quellenverzeichnis checken", "check this/these references", "verify this bibliography", "sind diese Quellen echt / halluziniert", or whenever they paste or upload a list of references for checking. Also trigger when the user suspects fabricated sources, or asks whether a specific book, paper, or article actually exists. Use for any literature verification task, even a single source.
license: GPL-3.0-or-later
metadata:
  author: Benedikt Engelmeier
  version: "0.6.0"
  hermes:
    tags: [research, citations, bibliography, hallucination-detection, academic, verification, literatur, literaturverzeichnis]
    category: research
    requires_toolsets: [web]
---

# Literaturprüfung — Skill zur Zitations- und Literaturverzeichnisprüfung

Akademische Quellenangaben systematisch gegen live abgefragte bibliografische Datenbanken prüfen, um
Zitierfehler und KI-halluzinierte Quellen aufzudecken. Jeder Befund braucht eine kanonische
Datenbank-URL als maschinell überprüfbaren Beleg — keine Behauptung, einen Link.

**Grundregel:** Niemals aus dem Gedächtnis oder aus Trainingswissen verifizieren. Jede Klassifikation
erfordert eine Live-Abfrage mit anklickbarer Ergebnis-URL. Ohne diese URL gilt die Quelle als ungeprüft.

## ⭐ Verifikations-Kern (immer zuerst lesen, gilt für JEDE Quelle)

1. **Pflichtreihenfolge je Quelle:** DOI vorhanden → direkt auflösen (Standard: Crossref
   `works?filter=doi:…&select=…`, siehe api-endpoints.md). Sonst nach Typ:
   Artikel → **OpenAlex → Crossref** (beide mit `select=` und kleiner Trefferzahl, `rows=2–3`); ohne
   OpenAlex-Key bzw. nach gescheitertem Canary **Crossref-primär**. Deutsches Buch → **K10plus (DC)**
   → OpenAlex → DNB (lobid nur nach bestandenem Canary, siehe Routing).
   Kaskade **vollständig** durchlaufen; ein einzelner Fehlschlag einer DB ist kein Endergebnis.
2. **Vor jedem Nicht-Treffer-Urteil** (🔴 IV **oder** „nicht prüfbar"): die **Gegenprobe** ausführen —
   freie K10plus-Schlüsselwortsuche `query=pica.all%3DKERNTITEL_KEYWORDS+AUTOR_NACHNAME` (Bücher;
   dieses eine Muster, siehe api-endpoints.md → K10plus) bzw. Crossref
   `query.bibliographic=…&query.author=…&select=…` (Artikel). Bei **mehreren Autoren/Herausgebern**
   auch einen **zweiten Namen** als Suchterm probieren (`pica.all%3DZWEITER_NAME+KERNTITEL`).
   **Gültigkeit:** Eine Gegenprobe zählt nur als ausgeführt, wenn die Query Autor-Nachname UND
   Kerntitel-Keywords enthält; liefert sie eine dreistellige oder höhere Trefferzahl, war sie zu
   unspezifisch und gilt als NICHT ausgeführt (enger neu formulieren). Leerer
   Body → Retry mit **variierter** Query (siehe Step 3), dann nächste DB — **nicht** vorzeitig aufgeben.
3. **Festlegungspflicht:** Jede **gewöhnliche** Quelle (Buch, Artikel, Sammelbandkapitel) endet
   zwingend in **🟢 I / 🟡 II / 🟠 III / 🔴 IV**. Die Ausweich-Status sind eng begrenzt:
   
   - **🎓 FU** nur für echtes FernUni-Lernmaterial (nach voller Prozedur inkl. Buchausgaben-Suche),
   
   - **⚪ ?** nur für strukturell nicht katalogisierbare Sonderfälle (kein gewöhnliches Buch/Artikel),
   
   - **„technisch nicht prüfbar"** nur, wenn **ALLE** zuständigen DBs **technisch** fehlschlugen
     (kein valides Ergebnis von irgendeiner DB). Ein einzelner leerer Body (z. B. OpenAlex) bei sonst
     funktionierender DB ist **kein** Gesamt-Fehlschlag.
4. **Konsistenz:** Die Kategorie muss zur Abweichungsbeschreibung passen — 🟢 I nur bei „keine
   wesentlichen Abweichungen" (siehe Klassifikations-Gate, Step 5).

Alles Weitere (Spezialfälle, Routing-Details, Datenbank-Syntax) konkretisiert nur diese vier Punkte.

---

## Sprache

Erkenne die Sprache der vorherigen Nutzernachrichten und verwende sie durchgängig — für
Tabellenüberschriften, Hinweise, Fehlerbeschreibungen und den Abschlussbericht. Die Sprache der
Quellen selbst beeinflusst dies nicht.

---

## Eingabe

Kläre, wie das Literaturverzeichnis übergeben wurde:

- **Datei hochgeladen** (.pdf, .docx, .txt, .md): Datei lesen und den Literatur-/Quellenteil extrahieren.
- **Text eingefügt**: direkt verwenden.

Ist die Eingabe ein vollständiges Dokument, nur die Literaturliste extrahieren. Sollte der Umfang
**wirklich unklar** sein (z. B. mehrere Literaturabschnitte, oder es ist mehrdeutig, welcher Teil zu
prüfen ist), **keinen separaten Bestätigungs-Turn öffnen** — diese eine Umfangsfrage **in die
Step-0-Frage falten**, sodass der Nutzer Keys und Umfang in einem Zug beantwortet. Ist der Umfang
eindeutig, die Umfangsfrage ganz weglassen. Bei weniger als 3 Quellen alle auf einmal verarbeiten.
Bei längeren Listen alle Quellen in einem Durchgang prüfen — keine manuelle Bestätigung zwischen
Stapeln —, aber Rate-Limits abfangen (siehe api-endpoints.md).

---

## Optionen / Flags

Alle Flags sind optional. Ohne Angabe gilt das Standardverhalten.

- **`--fachgebiet <name>`** — Fachgebiet manuell setzen statt Auto-Erkennung. Werte:
  `sozialwissenschaften | wirtschaft | recht | stem | medizin | geistes | allgemein`.
  Siehe `references/database-routing.md`.
- **`--kontext fernuni`** — FernUniversität-Hagen-Handling erzwingen (siehe unten).
- **`--bibtex`** — Das intern erzeugte BibTeX des Verzeichnisses zusätzlich als herunterladbare
  `.bib`-Datei speichern (siehe Step 1b). Standardmäßig aus.
- **`--openalex-key <key>`** — Vom Nutzer bereitgestellter OpenAlex-API-Key. Wird als `&api_key=` an
  jede OpenAlex-Anfrage angehängt. **Ohne Key ist OpenAlex in vielen Agent-Umgebungen faktisch unbrauchbar**
  (schlüssellose Anfragen liefern einen leeren Body); die Sitzung fährt dann die Crossref-primäre Kaskade.
  **Sicherheit (harte Regeln):** Key niemals in eine gespeicherte Datei, einen Bericht, das
  Transparenz-Log, eine Zwischenstand-Datei oder eine angezeigte URL schreiben — `api_key=…` vor jedem
  Zitieren einer Query-URL entfernen.
- **`--s2-key <key>`** — Vom Nutzer bereitgestellter Semantic-Scholar-API-Key. Wird als HTTP-Header
  `x-api-key: <key>` bei jeder Semantic-Scholar-Anfrage mitgesendet; hebt das strenge Limit
  (100 Calls / 5 min) an. Optional. Gleiche Sicherheitsregeln wie oben (nur sitzungsweise, nie
  gespeichert, nie angezeigt).
- **`--base-key <key>`** — Vom Nutzer bereitgestellter BASE-API-Key (Bielefeld Academic Search Engine).
  Wird als `&apikey=` angehängt. **BASE ist OHNE Key nicht nutzbar** und wird dann gar nicht abgefragt.
  **Hartes Rate-Limit (1 Anfrage/Sekunde): höchstens EIN BASE-Aufruf pro Antwortschritt, niemals
  gebündelt oder parallel** (siehe Step 3 → „BASE — hartes Rate-Limit"). BASE ist ein OA-/Repositorien-
  Aggregator (graue Literatur, Hochschulschriften, Working Paper), **kein** Buch-/Verlagskatalog —
  Rolle und Grenzen siehe api-endpoints.md → BASE. Gleiche Sicherheitsregeln wie oben (nur sitzungsweise,
  nie gespeichert, nie angezeigt).
- **`--config <pfad>`** — Pfad zu einer **Konfigurationsdatei**, aus der API-Keys gelesen werden (statt
  sie einzeln per Flag oder in der Step-0-Frage zu nennen). Format und Auto-Erkennung siehe Step 0 →
  „Konfigurationsdatei". Der Skill **liest** die Datei nur und schreibt **niemals** Keys hinein.

**Drei Wege, Keys zu übergeben** (Reihenfolge = Priorität): (1) **direkt beim Aufruf** — als Flag oben
**oder** frei im Aufruftext genannt; (2) **Konfigurationsdatei** (`--config` oder eine auto-erkannte
Datei, siehe Step 0); (3) die **Step-0-Rückfrage**. Liegt über Weg 1 oder 2 bereits mindestens ein Key
vor, **entfällt die Step-0-Frage vollständig** — der Skill startet ohne erneutes Nachfragen.

---

## FernUni-Kontext (automatisch ausgelöst bei Erkennung)

Das FernUni-Handling wird **automatisch ausgelöst**, sobald eine Quelle ein FernUni-Signal trägt
(Studienbrief-/Lerneinheitsformat, Verlag/Ort „Hagen: FernUniversität"). Das Flag `--kontext fernuni`
ist nicht mehr nötig — es dient nur noch als expliziter Override, falls die Auto-Erkennung ein Signal
verfehlt.

Bei vorhandenem FernUni-Signal (automatisch oder per Flag):

- **Überprüfung versuchen** (die Quelle **nicht** vorab klassifizieren) — Prozedur in
  `references/category-rules.md` → „Studienbriefe und Lerneinheiten" anwenden.
- „Hagen: FernUniversität" als Verlagssignal behandeln.
- Wird das Material gefunden → regulär klassifizieren (🟢/🟡/🟠). Wird es trotz Versuch nicht gefunden →
  **🎓 FU** (FernUni-Lernmaterial, nicht überprüfbar) — niemals 🔴 IV.

Keine Bestätigungsfrage pro Quelle nötig; die Erkennung leitet die Quelle still durch die
FernUni-Verifikationsprozedur.

---

## Step 0 — API-Keys ermitteln (VOR der ersten Suche)

API-Keys können bei der Prüfung helfen (OpenAlex, Semantic Scholar, BASE). Sie sind **optional**. Bevor
irgendeine Datenbank abgefragt wird, **zuerst die Key-Quellen in dieser Reihenfolge prüfen** und erst
fragen, wenn keine davon einen Key liefert:

### Step 0a — Vorhandene Keys ermitteln (keine Rückfrage, wenn schon vorhanden)

1. **Direkt beim Aufruf übergeben?** Ein per Flag (`--openalex-key`/`--s2-key`/`--base-key`) **oder**
   frei im Aufruftext genannter Key (z. B. „mein OpenAlex-Key ist …", „hier mein BASE-Key: …") wird
   übernommen. Den frei genannten Key dem passenden Dienst zuordnen (Schlüsselwörter „OpenAlex",
   „Semantic Scholar"/„S2", „BASE").
2. **Konfigurationsdatei vorhanden?** Wenn `--config <pfad>` gesetzt ist, **oder** im Arbeitsverzeichnis
   bzw. unter den hochgeladenen Dateien eine Datei namens `literaturpruefung-keys.txt`,
   `literaturpruefung-keys.json`, `literaturpruefung.config.txt`/`.json` oder `.literaturpruefung-keys`
   liegt, **diese Datei lesen** und die Keys daraus übernehmen (Format siehe unten).
3. **Sonst:** die Step-0b-Rückfrage stellen.

**HARTE REGEL — keine erneute Rückfrage:** Liefert Quelle 1 **oder** 2 mindestens einen Key, **entfällt
die Step-0b-Frage vollständig**. Der Skill übernimmt die gefundenen Keys still und **beginnt sofort mit
Step 1** — kein „Hast du noch weitere Keys?", kein „Soll ich starten?". Fehlende Keys (z. B. nur
OpenAlex vorhanden, kein BASE) werden **nicht** nacherfragt; die betroffene Datenbank läuft dann ohne Key
bzw. inaktiv (BASE).

#### Konfigurationsdatei — Format

Die Datei wird **nur gelesen, nie geschrieben**. Einfaches, fehlertolerantes Schlüssel-Wert-Format
(eine Zeile je Key), z. B. als `literaturpruefung-keys.txt`:

```
# Literaturprüfung — API-Keys (optional). Diese Datei privat halten, nicht versionieren/teilen.
openalex = DEIN_OPENALEX_KEY
semanticscholar = DEIN_S2_KEY
base = DEIN_BASE_KEY
```

Parsing-Regeln (tolerant): Zeilen mit `#` oder `;` am Anfang sind Kommentare; leere Zeilen ignorieren;
Trenner ist `=` oder `:`; Groß-/Kleinschreibung der Schlüssel egal; führende/folgende Leerzeichen und
einfache/doppelte Anführungszeichen um den Wert entfernen. **Akzeptierte Schlüssel-Aliase:**
`openalex` (auch `openalex_key`, `openalex-api-key`); `semanticscholar` (auch `s2`, `semantic_scholar`,
`s2_key`); `base` (auch `base_key`, `base-api-key`). Endet die Datei auf `.json`, stattdessen ein
JSON-Objekt mit denselben Schlüsseln erwarten (`{"openalex": "...", "base": "..."}`). Ein
`.env`-Stil (`OPENALEX_KEY=...`) wird durch dieselben Aliase mitabgedeckt.

**Sicherheit:** Ein aus der Konfigurationsdatei gelesener Key wird **genauso** behandelt wie ein
sitzungsweise genannter Key — nur im Arbeitsspeicher gehalten, nie in Berichte, Logs, Zwischenstände
oder angezeigte URLs geschrieben. Der Pfad der Konfigurationsdatei darf genannt werden, ihr **Inhalt
(der Key) niemals**.

### Step 0b — Nur falls kein Key vorab vorlag: einmalig fragen

**Nur** ausführen, wenn weder Aufruf noch Konfigurationsdatei einen Key lieferten. **Einmal** (nicht pro
Quelle) fragen, kurz halten, dann mit der vorhandenen Antwort weiterarbeiten:

> Bevor ich mit der Prüfung beginne: Hast du API-Keys für eine der folgenden Datenbanken? Sie sind
> optional, verbessern aber Trefferquote und Tempo.
> 
> - **OpenAlex** — in vielen Agent-Umgebungen faktisch nötig; ohne Key liefern Anfragen oft einen leeren Body, mit Key
>   sofort valides JSON. (Kostenlos unter `openalex.org` → Settings → API.)
> - **Semantic Scholar** (optional) — hebt das strenge Rate-Limit (100 Calls / 5 min) an.
> - **BASE** (optional) — ergänzt graue Literatur, Hochschulschriften, Working Paper und Open-Access-
>   Inhalte. Nur mit Key nutzbar (kostenlose, nicht-kommerzielle Registrierung). Hinweis: Aus manchen
>   Agent-Umgebungen (geteilte Proxy-IP/User-Agent) blockt BASE die Anfragen; dann bleibt es trotz Key
>   inaktiv (der Canary erkennt das).
> 
> Du kannst den/die Key(s) hier nennen oder mit „nein/weiter" ohne Key fortfahren.

Verarbeitung der Antwort:

- Nennt der Nutzer einen OpenAlex-Key → wie `--openalex-key` behandeln (an jede OpenAlex-URL
  `&api_key=` anhängen, `mailto=` entfällt).
- Nennt der Nutzer einen Semantic-Scholar-Key → bei jeder Semantic-Scholar-Abfrage als Header
  `x-api-key: <key>` mitsenden.
- Nennt der Nutzer einen BASE-Key → wie `--base-key` behandeln (an jede BASE-URL `&apikey=` anhängen);
  BASE-Canary in Step 3 ausführen, hartes 1,5-s-Limit und serielle Ausführung einhalten.
- Kein BASE-Key → BASE **gar nicht** abfragen (auch keine prepared-link-Pflicht entfällt: der manuelle
  BASE-Suchlink darf weiter im Bericht stehen).
- Kein Key / „weiter" → ohne Key fortfahren; OpenAlex läuft dann über den Sitzungs-Canary (Step 3),
  bei Fehlschlag Crossref-primäre Kaskade. **Nicht** pro Quelle erneut nachfragen.

**Auto-Start (HARTE REGEL — gegen doppeltes Nachfragen):** Diese Key-Frage ist das **einzige und
letzte** Gate vor der Recherche. Sobald die Antwort vorliegt (Key **oder** „weiter"), **unmittelbar
mit Step 1 fortfahren** — in derselben Antwort, ohne weitere Rückfrage. **Verboten** ist jede zweite
Bestätigungsphase wie „Soll ich jetzt beginnen?", „Bitte um Geduld", „Ich starte gleich …" oder eine
erneute Scope-/Bereitschaftsfrage. Die Key-Antwort **ist** das Startsignal; ein zusätzliches „Okay,
leg los" des Nutzers darf **nicht** abgewartet werden. Wurde ein Key bereits **vorab** ermittelt
(Flag, frei im Aufruftext genannt **oder** aus einer Konfigurationsdatei, siehe Step 0a), entfällt die
Frage ganz und die Recherche beginnt direkt. Die Scope-Klärung (welcher Abschnitt geprüft wird) gehört,
falls überhaupt nötig, **in dieselbe** Step-0-Frage — nie in einen separaten Turn.

**Sicherheit (harte Regeln, gelten für ALLE Keys):** Keys werden nur für die laufende Sitzung im
Arbeitsspeicher gehalten. Niemals in eine gespeicherte Datei, einen Bericht, das Transparenz-Log,
eine Zwischenstand-Datei oder eine angezeigte URL schreiben — `api_key=…` bzw. den Header-Wert vor
jedem Zitieren einer Query-URL entfernen. Keys werden auch nicht in der dauerhaften Memory abgelegt.

---

## Step 1 — Jede Quelle parsen und einordnen

Für jede Quelle extrahieren:

- **Autor(en):** Nachname(n) für die Suche.
- **Jahr:** Erscheinungsjahr.
- **Titel:** nur Haupttitel (Untertitel für die Erstsuche weglassen — senkt Falsch-Negative).
- **Publikationstyp:** `journal-article` | `book` | `book-chapter` | `grey-literature`.
- **Identifikatoren:** DOI, ISBN, ISSN, falls im Original vorhanden.

Der Publikationstyp bestimmt, welche Datenbank-Kaskade in Step 2 verwendet wird.

### Step 1a — FernUni-Material erkennen und markieren (ohne Vorab-Verurteilung)

**Gilt nur für ECHTES FernUni-Lernmaterial** (Studienbrief, Lerneinheit/Kurseinheit; Ort/Publisher
„Hagen: FernUniversität"). Graue Literatur (Behörden-/Regierungs-/Parlamentsseiten, Pressemitteilungen,
Nachrichten, Koalitionsverträge, Projektberichte) fällt **nicht** hierunter, auch wenn sie nicht in
Bibliothekskatalogen steht — sie läuft über die Graue-Literatur-Prüfung (Step 2) und erhält I–IV.

Wenn Publisher/Ort einer Quelle das Muster `FernUniversität` enthält (`Hagen: FernUniversität`,
`FernUniversität in Hagen`, `FernUniversität Hagen`), die Quelle als **FernUni-Lernmaterial-Kandidat**
**markieren** — aber **nicht** vorab klassifizieren. Eine Überprüfung wird **immer versucht**
(FernUni-Verifikationsprozedur in `references/category-rules.md` → „Studienbriefe und Lerneinheiten"):

1. K10plus/DNB nach Titel + „FernUniversität", GESIS.
2. **Suche nach einer kommerziell verlegten Buchausgabe (PFLICHT):** FernUni-Lerneinheiten werden
   **häufig zusätzlich als reguläres Lehrbuch verlegt** (UTB, Nomos, Springer VS, Kohlhammer …),
   oft mit leicht anderem Titel und anderem Jahr/Auflage. Daher **immer** eine freie Schlagwortsuche
   `pica.all=KERNTITEL_KEYWORDS + AUTOR_NACHNAME` in K10plus (und OpenAlex/Crossref) absetzen — ohne
   den Zusatz „FernUniversität". Der zitierte Lerneinheits-Titel weicht dabei häufig vom Titel der
   verlegten Buchausgabe ab; deshalb breit über Kern-Keywords + Autor suchen, nicht über den exakten
   Lerneinheits-Titel.
3. Suche nach dem ggf. anderswo erschienenen Originaltext.

Auswertung der Überprüfung:

- **Gefunden** (Lerneinheit, Buchausgabe **oder** Originaltext in einem Katalog nachweisbar) → **regulär
  klassifizieren** (🟢 I / 🟡 II / 🟠 III je nach Metadatenabgleich; abweichender Titel/Jahr der
  Buchausgabe → 🟡 II/🟠 III mit Hinweis), mit kanonischem Link. Eine gefundene Buchausgabe ist **kein**
  🎓 FU.
- **Erst wenn ALLE drei Schritte — inkl. der freien Buchausgaben-Gegenprobe — erfolglos bleiben** →
  **🎓 FU** (FernUni-Lernmaterial, nicht überprüfbar), **niemals 🔴 IV**. Im „Belege & Nachschlagen"-
  Block die tatsächlich versuchten Suchen samt Links ausweisen, damit transparent ist, dass geprüft wurde.

Wenn der **Sammelband-Host** einer Quelle bereits geprüft und als 🎓 FU eingestuft wurde, erben alle
weiteren Kapitel aus demselben Host automatisch 🎓 FU (Sammelband-Cache) — keine erneute Suche.

`--kontext fernuni` und FernUni-Erkennung sind gleichwertig: Sobald ein FernUni-Signal erkannt wird,
greift diese Prozedur (Routing Teil D) — unabhängig davon, ob das Flag gesetzt
ist. Das Flag dient nur noch als expliziter Schalter, falls die Erkennung das Signal nicht aufgreift.

### Step 1b — Internes BibTeX aufbauen (Strukturierungsschritt)

Die geparsten Felder jeder Quelle in einen BibTeX-Eintrag serialisieren. Das standardisiert die Daten
für die API-Abfragen und lässt den **Eintragstyp das Routing steuern** (`@article` → Artikel-Kaskade,
`@book` → Buch-Kaskade, `@incollection` → Kapitel/Sammelbände, `@inproceedings` → Konferenz,
`@misc`/`@techreport` → graue Literatur). Siehe `references/database-routing.md` → Teil B.

**Harte Regel — verbatim, keine Korrekturen:** Das BibTeX **rein mechanisch aus dem Originaltext**
aufbauen. **Nichts** korrigieren, ergänzen, normalisieren oder „bereinigen" (nicht das Jahr, nicht
einen fehlenden Untertitel, nicht den Verlag). Der wörtliche Originalstring bleibt unverändert — er
ist die Spalte „Originalquelle". Verglichen wird stets **Original gegen Datenbank**, nie BibTeX gegen
Datenbank. Das BibTeX ist ein Strukturierungshilfsmittel, keine korrigierte Fassung.

Jedem Eintrag einen stabilen Citekey geben (z. B. `gergs2020`) für den internen Bezug.

**Flag `--bibtex`:** Wenn gesetzt, das BibTeX zusätzlich als `literaturverzeichnis_YYYY-MM-DD.bib` im
Arbeitsverzeichnis speichern und dem Nutzer den Pfad nennen. Ohne das Flag bleibt das BibTeX intern
(wird nicht gespeichert).

### Step 1c — Fachgebiet bestimmen (einmal für die ganze Liste)

Das Fachgebiet **einmal für das gesamte Verzeichnis** erkennen, oder aus `--fachgebiet` übernehmen,
falls angegeben. `references/database-routing.md` → Teil A folgen. Das erkannte Fachgebiet und die
resultierenden Datenbanken **ein einziges Mal** nennen und fortfahren; nur bei wirklich gemischten
Signalen für eine Bestätigung innehalten. **Niemals pro Quelle fragen.**

---

## Step 2 — Suchstrategie: welche Datenbanken, in welcher Reihenfolge

Niemals nach dem vollständigen Zitations-String verbatim suchen. In Autor + Jahr + 2–3 Kern-Titel-
Keywords zerlegen. Das vermeidet Falsch-Negative durch Zeichensetzung, weggelassene Untertitel oder
Encoding-Probleme.

**Feld-Semantik (HARTE REGEL):** Titel-Keywords stammen **ausschließlich aus dem `title`-Feld des
BibTeX** (Step 1b); der Autor gehört **ausschließlich** in den Autor-Index (`pica.per`,
`query.author`, lobid `contribution…`) bzw. bei OpenAlex mit in `search=`. **Niemals den Autornamen
in einen Titel-Index schreiben.** `pica.tit=Crozier Zwänge` liefert ein formal valides, aber
bedeutungsloses 0-Ergebnis (der Autor steht nicht im Buchtitel). Richtig:
`pica.tit=Zwänge kollektiven Handelns and pica.per=Crozier`.

**Die Kaskade ist in `references/database-routing.md` definiert** — für die exakte Datenbankreihenfolge
laden. Die Kaskade je Quelle ergibt sich aus:

1. **Publikationstyp** (BibTeX-Eintragstyp aus Step 1b) — der primäre Schalter (Routing Teil B).
2. **Fachgebiet** (aus Step 1c) — verfeinert und ergänzt Fach-Datenbanken (Routing Teil C).

**Rate-Limit-Prinzip:** Datenbanken in einer Kaskade stehen in **Prioritätsreihenfolge**. Von oben nach
unten abfragen und **beim ersten bestätigten Treffer stoppen** (Step 3). Die genaue Reihenfolge je Typ
und Fachgebiet steht in `references/database-routing.md`.

**Sitzungs-Caches (einmal suchen, mehrfach nutzen):** Führe während der Prüfung interne Listen, die vor
**jeder** Abfrage geprüft werden — bei Treffer wird der API-Call übersprungen:

- **Sammelband/Host-Volume-Cache:** Key = Autor + Jahr + Kurztitel, Value = Status
  (`confirmed`/`not_found`), PPN/DOI, Notiz. So wird ein von mehreren Kapiteln geteilter Sammelband
  (typisch in sozialwiss. Hausarbeiten) nur **einmal** gesucht.
- **Journal-ISSN-Cache:** Key = Zeitschriftenname, Value = ISSN(s) aus dem Crossref-Journals-Lookup.
  Mehrere Artikel derselben Zeitschrift lösen den ISSN-Lookup so nur **einmal** aus.
- **Negativ-Cache:** Bereits erfolglos gesuchte Werke/Muster nicht erneut abfragen.

**Dedupe vor der Suche:** Identische Einträge (gleicher Citekey/DOI/Titel, z. B. doppelt zitiert) vor
Welle 1 zusammenfassen — nur **einmal** prüfen, das Ergebnis auf alle Vorkommen mappen.

Schnellorientierung (volles Detail in der Routing-Datei):

- **Zeitschriftenartikel:** OpenAlex → Crossref → OpenAIRE (keyless) → (Semantic Scholar, nur mit Key) → Fach-DB (EconBiz/GESIS)
- **Deutsche Bücher / Sammelbände:** K10plus (DC) → OpenAlex → lobid (Canary) → DNB → (Open Library, nur ISBN)
- **Internationale Bücher:** K10plus → Open Library (nur ISBN) → OpenAlex → (Library of Congress)
- **Buchkapitel:** Host-Volume über K10plus → lobid (Canary) → DNB (zuerst Sitzungs-Cache prüfen; Sammelbände-Regel anwenden)
- **Preprints / STEM:** arXiv (Canary) → OpenAlex → Crossref
- **Deutsche Sozialwissenschaften:** **GESIS** per Websuche ergänzen: `site:search.gesis.org TITEL_KEYWORDS AUTOR_NACHNAME`
- **Graue Literatur (mit URL/Herausgeber):** Graue-Literatur-Prüfung (siehe unten) — **nicht** der Buchkaskade zuführen, **nicht** als ⚪/🎓 abtun.

### Graue-Literatur-Prüfung (Webseiten, Behörden-/Regierungs-/Parlamentsdokumente, Pressemitteilungen, Nachrichten, Koalitionsverträge, Projekt-/Forschungsberichte)

Graue Literatur wird **normal geprüft** — nur über einen anderen Kanal als Katalog-DBs, weil sie dort
strukturell nicht steht. Das Fehlen in K10plus/OpenAlex ist **kein** Befund.

1. **Hat die Quelle eine zitierte URL?**
   
   - **Ja:** Die URL mit `web_fetch` auflösen und prüfen, ob (a) die Seite lädt und (b) herausgebende
     Stelle, Titel/Gegenstand und — wo zitiert — das Datum zur Quelle passen. JS-Shell/leere Seite →
     Claude-in-Chrome statt `web_fetch` (siehe api-endpoints.md). Tote, aber via Web-Archiv/Herausgeber
     auffindbare Seite zählt als Beleg (mit Hinweis).
   
   - **Nein** (nur Herausgeber + Titel, z. B. „Koalitionsvertrag … Berlin"): über die Seite der
     herausgebenden Stelle bzw. eine gezielte Websuche nach Herausgeber + Titel + Jahr bestätigen.
2. **Berichte/Working Paper mit DOI/ISBN/Handle:** zusätzlich DOI-Resolve bzw. Katalog/Repositorium
   (z. B. econstor, SSRN, Projektseite) — wie bei regulären Werken.
3. **Klassifikation:** lädt/auffindbar und passend → 🟢 I (🟡/🟠 bei Abweichung wie falschem Datum,
   geändertem Seitentitel, abweichendem Herausgeber). URL tot **und** auch über die herausgebende
   Stelle/Titelsuche nicht bestätigbar → 🔴 IV (möglicherweise erfunden / falsche Fundstelle). **Nie**
   🎓 FU; ⚪ ? nur, wenn es gar keine URL/Fundstelle gibt und die Existenz strukturell nicht prüfbar ist.

---

## Step 3 — Abfrage ausführen

### URL-Disziplin (HARTE REGEL — häufigste Fehlerquelle)

Query-URLs werden **ausschließlich durch Kopieren der kopierfertigen Muster aus
`references/api-endpoints.md`** aufgebaut — **niemals aus dem Gedächtnis**. Vor der **ersten**
Abfrage an eine Datenbank in einer Sitzung MUSS deren Abschnitt in api-endpoints.md gelesen worden
sein. Hintergrund: Werden K10plus-URLs frei erfunden (Pfad `/sru` statt `/opac-de-627`, Indices
`tit=`/`aut=` statt `pica.tit`/`pica.per`), erscheint das 404 in `web_fetch` als **leerer Body** und
sieht exakt wie ein Datenbank-Ausfall aus. Folge: ganze Kaskade kollabiert, Fehlklassifikationen.

**Basis-URL-Kurzreferenz (nur diese Pfade existieren — keine anderen verwenden):**

| Datenbank           | Basis-URL                                                                         |
| ------------------- | --------------------------------------------------------------------------------- |
| OpenAlex            | `https://api.openalex.org/works`                                                  |
| OpenAIRE Graph      | `https://api.openaire.eu/graph/v1/researchProducts` (keyless; nur `pid`/`mainTitle`/`authorFullName`/`search` — falscher Param = leerer Body) |
| Crossref            | `https://api.crossref.org/works` · DOI-Resolve: `https://doi.org/`                |
| K10plus             | `https://sru.k10plus.de/opac-de-627` (⚠️ **NICHT** `/sru` — existiert nicht, 404) |
| DNB                 | `https://services.dnb.de/sru/dnb`                                                 |
| ZDB                 | `https://services.dnb.de/sru/zdb` (⚠️ **NIE** `zdb-katalog.de` — bot-geschützt)   |
| lobid               | `https://lobid.org/resources/search`                                              |
| Open Library        | `https://openlibrary.org/search.json`                                             |
| Semantic Scholar    | `https://api.semanticscholar.org/graph/v1/paper/search`                           |
| arXiv               | `http://export.arxiv.org/api/query`                                               |
| Library of Congress | `http://lx2.loc.gov:210/lcdb`                                                     |
| ZBW EconBiz         | `https://api.econbiz.de/api/v1/search`                                            |
| BASE (nur mit Key)  | `https://api.base-search.net/cgi-bin/BaseHttpSearchInterface.fcgi`                |

Query-Syntax (Indices, Parameter, Schemas) steht **nur** in api-endpoints.md — die Tabelle hier
ersetzt das Lesen nicht.

Für jede Datenbank der zutreffenden Kaskade:

1. Die Query-URL durch **Kopieren des dokumentierten Musters** aus `references/api-endpoints.md` bauen (siehe URL-Disziplin oben).
2. Antwort abrufen.
3. Parsen: Titel, Autor(en), Jahr, Verlag/Zeitschrift extrahieren und — entscheidend — die **datenbankspezifische persistente ID**.
4. Aus dieser ID die kanonische Proof-of-Work-URL bauen.

**Eine verifizierte Datenbank genügt**, um eine Quelle zu klassifizieren. Die Kaskade stoppt, sobald
ein bestätigter Treffer mit kanonischer URL vorliegt. Erst zur nächsten Datenbank weitergehen, wenn die
aktuelle keinen Treffer oder einen plausibel-aber-unsicheren Treffer liefert.

**Die kanonische URL muss auf das Werk selbst zeigen.** Eine URL zu einem Werk, das das Zielwerk
*zitiert*, ist kein gültiger Beleg. Prüfen, dass Titel und Autoren im aufgelösten Datensatz zur
geprüften Quelle passen.
**Das gilt ausdrücklich auch für Rezensionen und Sekundärquellen:** Eine Buchrezension, ein
zitierendes Werk oder eine Erwähnung in Sekundärliteratur ist **kein** Nachweis und darf **keine**
🟢/🟡-Klassifikation stützen. Ebenso unzulässig: 🟡 II als „existiert wahrscheinlich, aber nicht
indexiert" — gewöhnliche Bücher/Artikel müssen in einem Katalog **gefunden** werden (🟢/🟡/🟠)
oder sind 🔴 IV; bei rein technischen Ausfällen „technisch nicht prüfbar" (siehe
Klassifikations-Sperre unten).

**Rate-Limits (Handlungsregel, nicht Zeitvorgabe):** Ein Sprachmodell kann keine Sekunden „abwarten" —
maßgeblich ist daher **was** getan wird, nicht eine Wartezeit. Liefert eine Datenbank HTTP 429, sie in
diesem Schritt **nicht** sofort erneut anfragen, sondern **einmal in einem späteren Schritt** (nachdem
andere Quellen/Calls bearbeitet wurden) wiederholen. Schlägt es erneut fehl, im Transparenz-Log
vermerken und weitergehen.

### Robuste Ausführung — technischer Fehlschlag ≠ „nicht gefunden" (KRITISCH)

Dies ist die wichtigste Regel für verlässliche Ergebnisse (siehe api-endpoints.md → „Robuste Ausführung
& Fallback-Protokoll"):

- **Unterscheide** einen **technischen Fehlschlag** (leerer Body / nur URL zurück / kein JSON / HTTP 403
  `URL exceeds maximum length` / 4xx / 5xx / Timeout / JS-Shell) von einem **validen 0-Ergebnis**
  (wohlgeformte Antwort, die echte null Treffer meldet).
- **Bei technischem Fehlschlag:** bei 403/Länge die URL kürzen (OpenAlex: Autor-Filter raus → in
  `search=`, ggf. `mailto=` weg) und **einmal** erneut; sonst **einmal mit VARIIERTER Query** erneut
  (siehe Retry-Regel unten); dann **zur nächsten
  Datenbank der Kaskade** weitergehen. **Nicht** klassifizieren und **nicht** still auf WebSearch ausweichen.
- **Retry heißt variieren, niemals wortgleich wiederholen (HARTE REGEL):** Viele Agent-Umgebungen
  cachen `web_fetch`-Antworten pro URL — die **identische** URL erneut zu senden liefert denselben
  (auch denselben leeren) Body zurück und ist als Retry wertlos. Ein Retry MUSS die Query verändern,
  in dieser Reihenfolge: (1) **vollständigerer Titel** inkl. Stoppwörter/Untertitel-Anfang,
  (2) **zweiten Autor** in den Autor-Parameter aufnehmen bzw. auf ihn wechseln, (3) andere
  Kerntitel-Keyword-Mischung. Leere Bodies treten query-spezifisch reproduzierbar auf (mutmaßlich
  langsame Suchen mit sehr häufigen Wörtern); eine variierte Query auf dasselbe Werk trifft dann oft
  sofort.
- **Klassifikations-Sperre:** 🔴 IV oder ⚪ ? („nicht gefunden") nur, wenn **mindestens eine** Datenbank
  ein **valides** Ergebnis lieferte. „Technisch nicht prüfbar" gilt **nur, wenn ALLE für den Typ
  zuständigen DBs technisch fehlschlugen** — ein leerer Body **einer** DB (z. B. OpenAlex) bei sonst
  valider Antwort (Crossref/K10plus) ist **kein** Gesamt-Fehlschlag, sondern verlangt, die Kaskade
  weiter abzuarbeiten. Lagen nur technische Fehlschläge vor, im Bericht offen als „technisch nicht
  prüfbar – APIs nicht erreichbar" ausweisen (NICHT als ⚪ ?, NICHT als Halluzination).
  **Zusatz:** Ein 0-Ergebnis zählt nur, wenn die Query **feldsemantisch korrekt** war (Titel-Keywords
  aus dem Titelfeld, Autor im Autor-Index — siehe Feld-Semantik-Regel in Step 2). Ein 0-Ergebnis
  auf eine fehlgebaute Query ist wertlos.
  **BASE-Zusatz:** Ein **BASE-Nulltreffer für eine Monografie oder einen Zeitschriftenartikel zählt
  NICHT** als valides „nicht gefunden" — BASE deckt diese Typen nicht zuverlässig ab (OA-/Repositorien-
  Aggregator). Ein BASE-Treffer kann eine Quelle **bestätigen** (🟢/🟡), ein BASE-Miss allein darf
  **nie** Richtung 🔴 IV zählen. Maßgeblich für IV bleiben die Katalog-/DOI-Quellen (K10plus, DNB,
  Crossref, OpenAlex, lobid).
- **Gegenprobe-Pflicht vor 🔴 IV:** Bevor ein Buch oder Artikel 🔴 IV erhält, **eine** letzte
  Kontrollabfrage senden, die gegen Feldsemantik-Fehler robust ist: bei **Büchern** die K10plus **freie
  Schlüsselwortsuche** `query=pica.all%3DKERNTITEL_KEYWORDS+AUTOR_NACHNAME` (exakt dieses Muster,
  siehe api-endpoints.md → K10plus → freie Suche); bei **Artikeln** Crossref
  `query.bibliographic=KERNTITEL&query.author=NACHNAME&select=DOI,title,author,published,container-title&rows=2`.
  **Gültigkeitskriterium:** Die Gegenprobe muss Autor-Nachname UND Kerntitel-Keywords enthalten und
  eine überschaubare Trefferzahl liefern; ein „0 passende von 1000+ Treffern" auf eine unspezifische
  Query (nur ein Allerweltswort, nur ein Jahr, ohne Autor) ist KEINE ausgeführte Gegenprobe.
  Erst wenn eine so gebaute Abfrage 0 Treffer liefert, ist 🔴 IV zulässig. Ein „nicht gefunden" bei einem breit
  rezipierten Standardwerk zeigt fast immer eine fehlgebaute oder abgebrochene Abfrage an, keine
  fehlende Quelle.
- **Überdimensionierte Antwort = vergessenes `select=`:** Meldet eine Crossref-/OpenAlex-Antwort „zu
  groß"/viele KB und wird abgebrochen, war die Query **ohne** `select=`+`rows=2` gebaut. **Kein** Grund
  für „nicht prüfbar" — die **gleiche** Quelle mit `select=`+`rows=2` erneut abfragen.
- **Leerer Crossref-Body:** Retry-Protokoll ausführen — 1× mit **variierter** Query (vollerer Titel /
  Zweitautor, nie wortgleich), notfalls 1× ASCII —, dann OpenAlex/nächste
  DB — **nicht** vorzeitig „nicht prüfbar" vergeben.
- **Reprint/Container-Treffer = Existenznachweis:** Liefert die DB statt der Originalausgabe eine
  Nachdruck-/Sammelband-Version mit passendem Titel + Autor(en), gilt die Quelle als **existent**
  (🟢/🟡) — Originalvenue/-jahr ggf. per gezielter Query/DOI nachziehen, aber nicht als „nicht gefunden" werten.
- **Fällt OpenAlex aus, aktiv Crossref ansteuern** (kurze URLs) — nicht überspringen.

**Ist im Original ein DOI vorhanden:** Immer direkt auflösen — Standard ist der Crossref-Lookup
`GET https://api.crossref.org/works?filter=doi:[DOI]&select=…&rows=1` (funktioniert ohne
Request-Header und ohne Key; Muster in api-endpoints.md). Nur wenn das Fetch-Tool der Umgebung
Request-Header setzen kann, alternativ `GET https://doi.org/[DOI]` mit
`Accept: application/vnd.citationstyles.csl+json`. Prüfen, ob zurückgegebener Titel und Autoren
passen. Ein DOI, der zu einem anderen Werk auflöst, ist ein starkes Halluzinationssignal; ein bei
Crossref unbekannter DOI wird über OpenAlex nachgeprüft (DataCite-DOIs), bevor er als nicht
auflösbar gilt.

**Schlank abfragen — immer das leichteste Fieldset (siehe api-endpoints.md, Effizienz-Grundregeln):**
K10plus/DNB → `recordSchema=dc`/`oai_dc`, OpenAlex/Crossref → `select=`, DOI-Resolve → Crossref
`filter=doi:`+`select=` (bzw. CSL-JSON, nur mit Header-Unterstützung), Ergebnislisten auf
2–3 begrenzt. **Sparse-then-Full-Fallback:** Lässt ein schlank abgefragter Treffer ein Vergleichsfeld
vermissen, ist er mehrdeutig oder scheitert das Parsing, **genau diese eine Quelle** noch einmal voll
abrufen (ohne `select=` / mit MARCXML). Nie pauschal auf das fette Objekt zurückfallen.

### Parallele Ausführung in Wellen

**Parallelität ist eine Optimierung, keine Voraussetzung.** Unterstützt die Agent-Umgebung keine
nebenläufigen `web_fetch`-Calls aus einem Schritt (manche Backends serialisieren sie oder greifen mit
einem Ausführungszeitlimit ein), **dieselbe Kaskade rein sequenziell** abarbeiten — Quelle für Quelle,
Datenbank für Datenbank. Das **Ergebnis ist identisch**, nur die Laufzeit länger; Korrektheit und
Klassifikation hängen **nie** von der Nebenläufigkeit ab. Die folgenden Wellen sind also als
Beschleunigung zu lesen, wo sie möglich ist. (BASE bleibt in **jedem** Fall streng seriell, siehe unten.)

Sequentielle Einzelabfragen sind sonst der größte Zeitfaktor. Unabhängige Quellen teilen keine Daten und
können — sofern die Umgebung es zulässt — **gleichzeitig** abgefragt werden, mehrere `web_fetch`-Calls
in **einem** Schritt:

0. **Sitzungs-Canary (allererster Call, vor Welle 0):** Einen minimalen OpenAlex-Test-Call senden —
   **mit `api_key=`, falls `--openalex-key` gesetzt** — und zwar mit einem **sitzungseindeutigen
   Suchwort** statt eines konstanten Testworts:
   `works?search=canary-JJJJMMTT-hhmm&per_page=1[&api_key=…]` (Datum/Uhrzeit einsetzen).
   **Grund (HART):** Viele Agent-Umgebungen cachen `web_fetch`-Antworten pro URL; ein Canary mit
   immer gleicher URL kann aus dem Cache „gelingen", obwohl die Datenbank aktuell nicht erreichbar
   ist — und weist damit eine tote DB als lebendig aus. Ein einzigartiges Suchwort erzwingt einen
   echten Live-Call (das Suchwort darf 0 Treffer liefern; entscheidend ist wohlgeformtes JSON mit
   `meta`/`results`, nicht die Trefferzahl).
   Kommt kein wohlgeformtes JSON: Ohne Key ist das der **Normalfall in geteilten Agent-Umgebungen**
   (Keyless-Anfragen liefern leeren Body, siehe api-endpoints.md → OpenAlex). Die Key-Frage wurde bereits in **Step 0**
   gestellt — hier **nicht erneut** nachfragen: liegt kein Key vor, OpenAlex für die **gesamte Sitzung**
   deaktivieren und alle Artikel-Kaskaden Crossref-primär fahren. Das spart pro Quelle 1–2 tote Calls.
   **BASE-Canary (nur wenn `--base-key`/Step-0-Key vorliegt):** zusätzlich **einen** BASE-Test-Call
   `func=PerformSearch&query=canary-JJJJMMTT-hhmm&hits=1&format=json&apikey=…` senden (sitzungseindeutiges
   Suchwort aus demselben Grund; siehe api-endpoints.md → BASE).
   Kommt ein leerer Body (in geteilten Agent-Umgebungen der Normalfall — UA/IP-Block), BASE für die
   **gesamte Sitzung deaktivieren**, ohne erneute Nachfrage. Dieser eine Canary zählt bereits zum
   1,5-s-BASE-Takt.
1. **Welle 0 — Identifier-First (billigster, sicherster Treffer zuerst):** Alle Quellen **mit DOI oder
   ISBN** direkt auflösen, bevor irgendeine Titel/Autor-Kaskade startet. DOI → Crossref
   `filter=doi:`-Lookup; mehrere DOIs in **einem** Call bündeln — Crossref kommagetrennt
   (`filter=doi:A,doi:B`, ohne Key) oder OpenAlex (`filter=doi:a|b|c`, mit Key); **max. 5 DOIs** pro
   Call wegen URL-Limit. ISBN → K10plus/Open Library (`/isbn/`). Bestätigte Quellen fallen aus den
   teuren Wellen heraus.
2. **Welle 1 — gruppieren:** Die **verbleibenden** Quellen (ohne Identifier) parsen und nach Typ
   sortieren (Artikel, deutsche Monografien, internationale Monografien, Buchkapitel/Sammelbände,
   FernUni/Graue Literatur).
3. **Canary vor dem Fan-out:** Für jede **neue** Query-Form/Datenbank zuerst **EINEN** Test-Call senden
   und prüfen, dass wohlgeformtes JSON/XML kommt (auch hier keine in dieser Sitzung schon einmal
   gesendete URL wiederverwenden — Cache-Gefahr). Erst dann auffächern. Scheitert der Canary technisch,
   erst das Muster reparieren (URL kürzen) bzw. die DB für diese Welle überspringen — so verbrennt nicht
   eine ganze Welle an einem kaputten Muster.
4. **Welle 2 — parallele Erstabfragen:** Pro Welle 4–6 unabhängige Queries gleichzeitig (schlankes Fieldset).
5. **Welle 3 — parallele Nachfolgeabfragen:** Quellen ohne Treffer in der ersten Runde gesammelt mit
   erweiterten Parametern (nächste Datenbank der Kaskade / ohne Jahresfilter) nachsuchen.

**Rate-Limit-Regel:** Höchstens **3 gleichzeitige Calls pro einzelner Datenbank** (K10plus, DNB,
Crossref je getrennt zählen). Verschiedene Datenbanken dürfen parallel laufen. DNB besonders schonen
(siehe api-endpoints.md): DNB-Calls über mehrere Wellen strecken bzw. mit Calls anderer DBs abwechseln.

### ⛔ BASE — hartes Rate-Limit (KRITISCH, zwingend einzuhalten)

BASE erlaubt **maximal 1 Anfrage pro Sekunde**; bei Verstoß droht laut BASE der **Entzug des Keys ohne
Vorwarnung**. Da ein Sprachmodell keine Sekunden „abwarten" kann, wird das Limit **über die Handlung**
abgesichert, nicht über eine Zeitvorgabe — die folgenden Regeln übersteuern alle obigen Parallel-Regeln:

- **Höchstens EIN BASE-Aufruf pro Antwortschritt.** Der nächste BASE-Aufruf erst in einem **späteren
  Schritt**, nachdem das Ergebnis des vorigen vorliegt. So kann zwischen zwei BASE-Aufrufen technisch
  gar keine Sekunde unterschritten werden — unabhängig davon, wie schnell das Modell Token erzeugt.
- **NIEMALS bündeln oder parallelisieren.** Nie zwei BASE-Calls im selben Schritt, nie BASE in eine
  „Welle" mit anderen Calls aufnehmen. Der BASE-Canary ist bereits der erste solche Einzelschritt.
- **Sparsam einsetzen.** BASE nur für die Quellen abfragen, für die es laut Routing zuständig ist
  (graue Literatur, Hochschulschriften, Working Paper, OA-Inhalte) — nicht pauschal für jede Quelle.
- Diese Regeln gelten **zusätzlich** zur Key-Pflicht und zum Canary: ohne erfolgreichen BASE-Canary
  wird BASE gar nicht abgefragt.

### Zwischenstand-Speicherung (alle 10 Quellen)

Nach Abschluss **jeder 10. verifizierten Quelle** den aktuellen Stand als Datei speichern:
`literaturpruefung_zwischenstand_YYYY-MM-DD_NvonM.md` im Arbeitsverzeichnis. Inhalt:

1. alle bisher klassifizierten Quellen mit Kategorie + Begründung,
2. alle gefundenen PPNs/DOIs/Canonical-URLs,
3. der Sammelband-Sitzungscache,
4. die noch offenen Quellen mit ihren BibTeX-Citekeys.

**⛔ KEINE KEYS in die Zwischenstand-Datei (oder irgendeine gespeicherte Datei):** Ein „Sitzungs-Status"-
Block darf den Aktiv/Inaktiv-Zustand einer Datenbank vermerken, aber **niemals den Key-Wert selbst**
(kein `api_key=`/`apikey=`/Header-Wert, kein OpenAlex-/Semantic-Scholar-/BASE-Key). Nur „OpenAlex: aktiv
(mit Key)" o. ä. — nie der Schlüssel. Das gilt für Zwischenstand, Bericht, Transparenz-Log und jede
angezeigte URL gleichermaßen.

So lässt sich die Prüfung nach Kontextüberschreitung oder Sitzungsunterbrechung durch Einlesen der
Zwischenstands-Datei nahtlos fortsetzen — ohne Rekonstruktionsaufwand. Bei sehr langen Bibliografien
(40–60 Quellen, typisch für Master-/Doktorarbeiten) ist das die Absicherung gegen Zwischenstand-Verlust.

---

## Step 4 — Vergleichen und einordnen

Die Datenbank-Metadaten gegen das Original vergleichen. Fokus auf:

- Autoren-Nachnamen (Schreibung, Vollständigkeit),
- Erscheinungsjahr,
- Haupttitel (muss passen; Untertitel-Abweichungen sind geringfügig),
- Publikationsorgan (Zeitschriftenname, Verlag, Reihe),
- Auflage oder Seitenzahlen (bei Büchern und Kapiteln).

Das Kategoriensystem und die Sonderregeln aus `references/category-rules.md` anwenden.

---

## Step 5 — Ausgabe

**Schluss-Selbstcheck (HART — vor Abgabe des Berichts):**

1. Jede 🟢-I-, 🟡-II- und 🟠-III-Zeile MUSS (a) auf einem **positiven, kanonischen DB-Treffer** für
   das Werk beruhen (Klassifikations-Gate, siehe Kategoriensystem) und (b) sowohl **inline in der
   Tabellenspalte „Gefundene Quelle"** als auch im „Belege & Nachschlagen"-Block **mindestens einen
   kanonischen Datenbank-Link** (PPN, DOI, IDN, OpenAlex-W-ID, lobid-ID …) tragen.
2. Begründungen wie „Klassiker", „etabliert", „bekanntes Grundlagenwerk", „Verlag/Zeitschrift
   existiert/plausibel", „Jahrgang plausibel", „Verifikation erforderlich" sind **verboten** — das ist
   Klassifikation aus Trainingswissen und verstößt gegen die Grundregel. Eine so begründete 🟡-II-Zeile
   ist falsch: ohne positiven Treffer gilt 🔴 IV / 🎓 FU / ⚪ ? (Entscheidungsbaum).
3. Zeilen ohne kanonischen Treffer-Link gehen **zurück in die Prüfung** (Gegenprobe-Pflicht, Step 3),
   werden als 🔴 IV / 🎓 FU / ⚪ ? klassifiziert, oder ehrlich als „technisch nicht prüfbar" ausgewiesen.
4. **Konsistenz Kategorie ↔ Abweichung (HART):** 🟢 I ist **nur** zulässig, wenn das Feld „Art der
   Abweichungen" leer bzw. „keine wesentlichen Abweichungen" ist. Beschreibt die Zeile **irgendeine**
   Abweichung (anderes Jahr, andere/​neuere Auflage, anderer Verlag, fehlender Untertitel …), ist die
   Mindestkategorie **🟡 II** — eine 🟢-I-Zeile mit Abweichungstext ist ein Widerspruch und falsch.
   Sonderfall Neuauflage: Wurde nur eine **neuere** Auflage gefunden, das zitierte (ältere) Jahr aber
   **nicht separat** bestätigt, gilt **🟡 II** mit Hinweis „nur Neuauflage [Jahr] nachgewiesen; zitiertes
   Jahr [Zitatjahr] nicht separat verifiziert" (optional vorher gezielt mit `pica.jah=Zitatjahr` suchen).
5. **Kein Nicht-Treffer-Status ohne erfüllte Vorbedingungen:** Eine gewöhnliche Quelle (Buch/Artikel/
   Kapitel) darf **nur dann** „technisch nicht prüfbar"/⚪ erhalten, wenn **alle** zuständigen DBs
   **technisch** fehlschlugen (siehe Verifikations-Kern Pkt. 3 + Klassifikations-Sperre, Step 3). Gab
   mindestens eine DB ein **valides** Ergebnis und wurde die Gegenprobe ausgeführt, ist 🔴 IV zu vergeben.

**Pflicht-Reminder gegen Kontextdrift:** Unmittelbar **bevor** der Abschlussbericht geschrieben wird,
den Abschnitt „Ausgabeformat" dieser Datei **erneut lesen** (separater Read-Call auf SKILL.md). Bei
langen Prüfungen (200+ Tool-Calls) ist das Ausgabeformat sonst aus dem Kontext gedriftet. Der Bericht
**muss** die zweiteilige Struktur haben (Bewertungstabelle + „Belege & Nachschlagen"); Fließtext
ersetzt die Tabelle nicht. Quellen ohne Treffer (⚪ ?/🔴 IV/🎓 FU) erhalten in der Treffer-Zeile
explizit „Nicht gefunden in: …" — leere Felder sind kein Grund, auf Prosa auszuweichen.

Die Bewertungstabelle aufbauen, im Chat ausgeben und als `.md`-Datei speichern.

---

## Ausgabeformat

### Bewertungstabelle

Im Chat ausgeben UND als `literaturverzeichnispruefung_YYYY-MM-DD.md` im Arbeitsverzeichnis speichern.
Dem Nutzer nach dem Speichern den Dateipfad nennen.

**Die Ausgabe muss immer mit diesem Kopfblock beginnen — keine Varianten:**

```
# Literaturprüfung

**Geprüfte Datei:** [exakter Dateiname wie übergeben, oder „Texteingabe" bei eingefügtem Text]
**Prüfdatum:** YYYY-MM-DD
```

Die Ausgabe hat **zwei Teile**: die Bewertungstabelle — die nun **den einen wichtigsten Link pro Zeile
inline** trägt, sodass jede Zeile für sich überprüfbar ist — gefolgt von einem „Belege & Nachschlagen"-
Block, der **alle** abgefragten Datenbanken und die vorbereiteten manuellen Suchlinks enthält.

**Teil 1 — Bewertungstabelle:**

| #   | Bewertung | Originalquelle       | Gefundene Quelle                                               | Art der Abweichungen       |
| --- | --------- | -------------------- | -------------------------------------------------------------- | -------------------------- |
| 1   | 🟢 I      | Exakter Originaltext | [Korrigierter Eintrag, **fett** = Korrekturen](kanonische-URL) | Präzise Fehlerbeschreibung |

**Spaltenregeln:**

- **Bewertung** — Emoji + römische Ziffer (siehe Kategoriensystem unten).
- **Originalquelle** — exakt wie gegeben übernehmen. Nicht korrigieren oder umformatieren.
- **Gefundene Quelle** — die Metadaten der Datenbank verwenden, **Korrekturen fett markieren** und
  **stets den einen wichtigsten kanonischen Link für die Klassifikation als Markdown-Link auf diesen
  Text legen** (die Fundstelle, die ein Leser zur Bestätigung öffnen würde). Welcher Link:
  - 🟢 I / 🟡 II / 🟠 III → die kanonische Treffer-URL des Werks (K10plus PPN, DOI, OpenAlex-W-ID, lobid, …).
  - 🔴 IV → der **Near-Match**-Link, falls vorhanden (z. B. *„Near-Match: [Häusling 2020](…)"*); wurde
    gar nichts gefunden, den vorbereiteten **Google-Scholar**-Suchlink legen, damit der Nutzer
    nachprüfen kann (z. B. *„Kein Treffer — [🔍 Scholar](…)"*).
  - 🎓 FU → der versuchte Katalog-**Such**-Link (z. B. *„[🔍 K10plus](…)"*).
  - ⚪ ? → der versuchte vorbereitete Suchlink.
    Die vollständige Liste aller abgefragten Datenbanken steht weiterhin in Teil 2 — Teil 1 trägt nur den
    **einen** entscheidenden Link.
- **Art der Abweichungen** — konkret und spezifisch, z. B. *„Jahr falsch: 2018 statt 2016"* oder
  *„Untertitel fehlt"*. Bei 🟢 I: *„Keine wesentlichen Abweichungen"*.

**Teil 2 — Belege & Nachschlagen** (unmittelbar nach der Tabelle):

Für **jede** Quelle — nicht nur die problematischen — einen Block listen. **Alle** für diese Quelle
tatsächlich abgefragten Datenbanken zeigen (damit kein Wissen verloren geht und der Nutzer alles
nachprüfen kann), plus die vorbereiteten manuellen Suchlinks. Zwei Zeilen pro Quelle:

- **Treffer:** jede gefundene kanonische Treffer-URL als anklickbarer Link, getrennt durch ` · ` (z. B.
  K10plus, OpenAlex, DNB). Wurde eine Datenbank abgefragt, lieferte aber nichts, mit *(kein Treffer)*
  markieren. Bei 🔴 IV oder 🎓 FU ohne Treffer irgendwo: die durchsuchten Datenbanken auflisten, z. B.
  *„Nicht gefunden in: K10plus, OpenAlex, DNB, Crossref"* (bei 🎓 FU zusätzlich GESIS
  nennen, falls versucht). Neutral formulieren — es dokumentiert den Prüfversuch, ist keine Empfehlung.
- **Suche:** vorbereitete Suchlinks, die der Nutzer anklicken kann (keine vom Skill ausgeführte Abfrage):
  immer Google Scholar; WorldCat / BASE ergänzen, wo sinnvoll (Link-Muster in
  `references/api-endpoints.md`).

Format:

```
**#1 — gergs2020** (Gergs/Lakeit 2020)
- Treffer: [K10plus PPN 161165839X](https://opac.k10plus.de/DB=2.1/PPNSET?PPN=161165839X) · [OpenAlex W123](https://openalex.org/works/W123) · DNB *(kein Treffer)*
- Suche: [🔍 Google Scholar](https://scholar.google.com/scholar?q=Gergs+Lakeit+Organisationsentwicklung+2020) · [WorldCat](https://search.worldcat.org/search?q=Gergs+Lakeit+Organisationsentwicklung+2020)
```

Um lange Listen übersichtlich zu halten, **darf** der Block jeder Quelle in
`<details><summary>#1 — gergs2020</summary> … </details>` eingeklappt werden. Haben DNB und OpenAlex
beide das Werk, **beide** Treffer-Links angeben — OpenAlex ist die throttle-freie Reserve für DNB.

### Kategoriensystem

**Klassifikations-Gate (HARTE REGEL — zuerst lesen, vor der Tabelle):** Die Kategorien 🟢 I, 🟡 II
und 🟠 III sind **ausschließlich** zulässig, wenn für **das Werk selbst** ein **positiver, kanonischer
Datenbank-Treffer** vorliegt (PPN, DOI, IDN, OpenAlex-W-ID, lobid-ID …), dessen Titel **und** Autor:in
mit der Quelle übereinstimmen. Liegt **kein** solcher Treffer vor, ist **nur** 🔴 IV, 🎓 FU oder ⚪ ?
erlaubt — niemals II/III. Folge dem Entscheidungsbaum:

```
1. Positiver DB-Treffer für das Werk (Titel + Autor stimmen)?
   JA  → Metadaten vergleichen → 🟢 I / 🟡 II / 🟠 III (je nach Abweichung)
   NEIN ↓
2. Nur technische Fehlschläge, keine valide DB-Antwort? (Klassifikations-Sperre, Step 3)
   JA  → "technisch nicht prüfbar", NICHT klassifizieren
   NEIN ↓
3. Graue Literatur mit URL bzw. herausgebender Stelle? (Webseite, Pressemitteilung, Behörden-,
   Parlaments- oder Regierungsdokument, Nachrichtenartikel, Koalitionsvertrag, Projekt-/
   Forschungsbericht, Working Paper)
   JA  → GRAUE-LITERATUR-PRÜFUNG (normale Prüfung, siehe Step 2 + category-rules.md):
         URL auflösen und Herausgeber/Titel/Datum abgleichen.
         · lädt und passt        → 🟢 I  (🟡/🟠 bei Abweichung, z. B. Datum/Titel/tote-aber-archivierte URL)
         · tot UND auch über Herausgeber-/Titelsuche nicht bestätigt → 🔴 IV
         NIE 🎓 FU und NICHT automatisch ⚪ für graue Literatur.
   NEIN ↓
4. Gegenprobe GÜLTIG AUSGEFÜHRT? (freie K10plus-Suche `query=pica.all%3DKERNTITEL+AUTOR` = 0 Treffer;
   Query enthält Autor UND Kerntitel; keine dreistellige+ Trefferzahl — Step 3)
   NEIN → Gegenprobe ZWINGEND nachholen, dann erneut ab 1. prüfen
   JA  ↓
5. Echtes FernUni-Lernmaterial (Studienbrief/Lerneinheit, Ort/Publisher „Hagen: FernUniversität")
   UND FernUni-Prozedur inkl. Suche nach kommerziell verlegter Buchausgabe (UTB/Nomos/Springer …)
   lief erfolglos?
   JA  → 🎓 FU  (NICHT 🔴 IV; NICHT für graue Literatur/Behördenquellen)
   NEIN ↓
6. Strukturell nicht katalogisierbarer Sonderfall OHNE URL (internes Dokument ohne Fundstelle)?
   JA  → ⚪ ?
   NEIN → 🔴 IV  (möglicherweise halluziniert)
```

**Verbotene Begründungen für I/II/III** (das ist Klassifikation aus Trainingswissen, nicht aus Daten):
„Zeitschrift/Verlag existiert", „Jahrgang/Bandnummer plausibel", „Klassiker", „etabliert",
„bekanntes Grundlagenwerk", „Thema/Jahr stimmig", „Verifikation erforderlich". Ein noch **nicht
aufgelöster** Artikel oder ein nicht gefundenes Buch ist **kein** 🟡 II — er ist 🔴 IV (oder 🎓 FU/⚪ ?
nach Entscheidungsbaum). „Plausibel, aber unauffindbar" ist als 🟡 II **ausdrücklich unzulässig**.

| Symbol | Kategorie                                    | Wann verwenden                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| ------ | -------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 🟢 I   | **Fehlerfrei**                               | Positiver DB-Treffer; existiert wie zitiert. **Nur** triviale Formatierungsabweichungen (Zeichensetzung, Groß-/Kleinschreibung). Abweichungsfeld = „keine wesentlichen Abweichungen". **Jede** inhaltliche Abweichung (Jahr/Auflage/Verlag/…) ⇒ mindestens 🟡 II.                                                                                                                                                                                                                                                                      |
| 🟡 II  | **Kleine Fehler**                            | Positiver DB-Treffer; Werk existiert, kleine Abweichung: fehlender Untertitel, Jahr ±1 (Neuauflage), fehlende Auflagenangabe, oder verifizierter Herausgeberband für ein nicht indexiertes Kapitel.                                                                                                                                                                                                                                                                                                                                    |
| 🟠 III | **Gravierende Fehler**                       | Positiver DB-Treffer im richtigen Publikationsformat, aber wesentliche Metadaten klar falsch: völlig falsches Jahr, falscher Verlag, grob falsche Seitenzahlen.                                                                                                                                                                                                                                                                                                                                                                        |
| 🔴 IV  | **Möglicherweise halluziniert**              | Kein DB-Treffer für Autor:in(nen) + Haupttitel + Publikationstyp gefunden — trotz Gegenprobe. Kein Existenzbeweis, daher **Warnung, keine Gewissheit**. Near-Match (falls vorhanden) in „Gefundene Quelle" angeben.                                                                                                                                                                                                                                                                                                                    |
| 🎓 FU  | **FernUni-Lernmaterial (nicht überprüfbar)** | **NUR** für echtes FernUni-Lernmaterial (Studienbrief/Lerneinheit, Ort/Publisher „Hagen: FernUniversität"), bei dem die volle FernUni-Prozedur **inkl. Suche nach einer kommerziell verlegten Buchausgabe und ausgeführter Gegenprobe** ergebnislos blieb. **Ausdrücklich KEINE Halluzination.** **NICHT** für graue Literatur: Webseiten, Pressemitteilungen, Behörden-/Parlaments-/Regierungsdokumente, Nachrichten, Koalitionsverträge, NGO-/Projektberichte → die werden normal geprüft (graue-Literatur-Prüfung, Kategorie I–IV). |
| ⚪ ?    | **Nicht prüfbar**                            | **NUR** für strukturell nicht katalogisierbare Quellentypen (unveröffentlichte interne Dokumente o. Ä. ohne Fundstelle). **NICHT** für gewöhnliche Bücher/Artikel — die müssen gefunden oder als 🔴 IV klassifiziert werden. ⚪ ? ist **kein** Ersatz für „API hat nicht geklappt": ein technischer Fehlschlag wird offen als „technisch nicht prüfbar" ausgewiesen (nur bei Totalausfall ALLER zuständigen DBs), nicht als ⚪ ?.                                                                                                        |

**Kritische Abgrenzungen:**

- **Graue Literatur** (Webseiten, Pressemitteilungen, Behörden-/Parlaments-/Regierungsdokumente, Nachrichten, Koalitionsverträge, Projekt-/Forschungsberichte) wird **normal geprüft** und erhält eine Kategorie **I–IV** über die URL-/Herausgeber-Prüfung — **nicht** 🎓 FU, **nicht** pauschal ⚪ ?. Eine geladene, inhaltlich passende Behörden-/Regierungsseite ist 🟢 I, keine „unprüfbare" Quelle.
- 🔴 IV: gewöhnliches Buch/Artikel, das/der in Standardkatalogen sein **sollte**, aber **trotz ausgeführter Gegenprobe** nicht gefunden wird; oder graue Literatur, deren URL tot ist und die auch über die herausgebende Stelle nicht bestätigt werden kann.
- 🎓 FU: **ausschließlich** echtes FernUni-Lernmaterial nach voller Prüfung (inkl. Buchausgaben-Suche + Gegenprobe) ohne Fund — nie für reguläre Verlagspublikationen und nie für graue Literatur. FernUni-Lerneinheiten erscheinen häufig **zusätzlich** als verlegtes Lehrbuch (UTB/Nomos/Springer); diese Ausgabe muss gesucht werden, bevor 🎓 FU vergeben wird.
- ⚪ ?: nur der Sonderfall eines strukturell nicht katalogisierbaren Quellentyps **ohne URL** (z. B. internes Dokument ohne Fundstelle), außerhalb FernUni und außerhalb prüfbarer grauer Literatur.
- Deutet anderweitige Evidenz auf falsche Metadaten hin (z. B. abweichendes Originaljahr), „Hinweis auf Fehler:" in „Art der Abweichungen" ergänzen — auch bei 🎓 FU / ⚪ ?.

### Abschlussbericht

Nach der Tabelle anhängen — **rein sachlich, keine Ratschläge**:

1. **Zusammenfassung:** Anzahl je Kategorie (🟢 / 🟡 / 🟠 / 🔴 / 🎓 / ⚪) und Gesamtzahl der geprüften
   Quellen. Duplikate als neutrale Zählung nennen (z. B. *„3 Doppeleinträge im Verzeichnis: …"*).
2. **Transparenz-Log:** jede in dieser Sitzung abgefragte Datenbank auflisten und ob sie brauchbare
   Ergebnisse lieferte.
3. **DNB-Hinweis** (immer einfügen, wenn DNB abgefragt wurde): *„DNB-Links (d-nb.info/...) können nach
   intensiver Nutzung vorübergehend nicht erreichbar sein — das ist ein bekanntes technisches Problem
   der DNB, kein defekter Link. Bitte einige Minuten warten und erneut aufrufen."*

**Keine Empfehlungen / kein redaktioneller Rat (HARTE REGEL):** Der Bericht berichtet **nur** Befunde
und Belege. Verboten sind handlungsleitende Zusätze wie „Empfehlung: …", „sollte vor Abgabe bereinigt
werden", „direkte Abfrage in ZDB empfohlen", „manuelle Prüfung empfohlen" als Aufforderung, bewertende
Gesamturteile über die Recherchequalität („solide Recherchebasis") oder stilistische Ratschläge. Der
Nutzer entscheidet selbst, was mit den Befunden geschieht. Erlaubt sind ausschließlich: die Kategorie,
die konkrete Abweichung, der/die Beleg-Link(s) und — wo eine Prüfung technisch unvollständig blieb —
der **neutrale Tatsachenhinweis**, welche Datenbanken erfolglos/nicht erreichbar waren (ohne Imperativ).
Auch in der Spalte „Art der Abweichungen" und im „Belege & Nachschlagen"-Block keine „Empfehlung:"-Zeilen.

---

## Referenzdateien

Bei Bedarf laden, nicht alle vorab laden:

- **`references/database-routing.md`** — welche Datenbanken für welchen Publikationstyp und welches
  Fachgebiet, in Prioritätsreihenfolge; Fachgebiets-Erkennung und `--fachgebiet`-Override; FernUni-
  Add-on; Muster für vorbereitete Suchlinks. Bei Step 1c / Step 2 laden.
- **`references/api-endpoints.md`** — vollständige Query-Syntax, Parameter, Rate-Limits und kanonische
  URL-Muster aller Datenbanken. Laden, wenn eine Query für eine in dieser Sitzung noch nicht genutzte
  Datenbank gebaut wird.
- **`references/category-rules.md`** — detaillierte Klassifikationsregeln, Sonderfälle, Sammelbände-
  Regel, DOI-Reverse-Check und FernUni-Studienbrief-Regeln. Laden, wenn eine Einordnung unklar ist oder
  ein Sonderfall greift.
