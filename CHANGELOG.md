# Changelog — Literaturverzeichnisprüfung

## [0.5.3] — 2026-07-03

Robustheits-Korrekturen an Canary-, Retry- und Gegenprobe-Mechanik; alle geänderten Muster live
gegen die Schnittstellen verifiziert.

### Behoben

- **Canary-Aufrufe mit sitzungseindeutigem Suchwort** (`canary-JJJJMMTT-hhmm` statt konstantem
  Testwort): Viele Agent-Umgebungen cachen `web_fetch`-Antworten pro URL; ein konstanter Canary
  kann aus dem Cache „gelingen" und eine aktuell nicht erreichbare Datenbank als aktiv ausweisen.
  Gilt für OpenAlex- und BASE-Canary sowie den Canary vor dem Fan-out.
- **Retry-Regel umgestellt — variieren statt wiederholen:** Ein wortgleicher Retry ist wegen des
  URL-Caches wirkungslos; leere Bodies treten zudem query-spezifisch reproduzierbar auf
  (mutmaßlich langsame Suchen mit sehr häufigen Wörtern). Retries verändern jetzt die Query in
  fester Reihenfolge: vollständigerer Titel inkl. Stoppwörter → zweiter Autor → andere
  Keyword-Mischung. ASCII-Transliteration bleibt nachgelagerter Notbehelf.

### Geändert

- **Ein einheitliches Muster für die freie K10plus-Suche:** Gegenprobe und freie Schlagwortsuche
  verwenden überall dasselbe kopierfertige Muster `query=pica.all%3DKERNTITEL+AUTOR`
  (Verifikations-Kern, Step 3, Entscheidungsbaum und api-endpoints.md waren zuvor uneinheitlich —
  eine Quelle fehlgebauter Abfragen).
- **Gültigkeitskriterium für die Gegenprobe:** Eine Gegenprobe zählt nur als ausgeführt, wenn die
  Query Autor-Nachname UND Kerntitel-Keywords enthält und eine überschaubare Trefferzahl liefert;
  unspezifische Abfragen mit dreistelliger oder höherer Trefferzahl gelten als nicht ausgeführt.

## [0.5.2] — 2026-07-02

Schnittstellen-Korrekturen für header-lose Agent-Umgebungen; alle geänderten Abfragemuster gegen
die Live-Schnittstellen geprüft.

### Behoben

- **DOI-Auflösung:** Der Crossref-Einzelwerk-Lookup mit `select=` auf der Singleton-Route
  (`/works/{DOI}?select=…`) ist defekt (leerer Body; `select` gilt nur für die Listen-Route) und
  als bekannt-defektes Muster gekennzeichnet. Neues Standardmuster ist der Lookup über die
  Listen-Route `works?filter=doi:…&select=…&rows=1` — sparsam und ohne Key nutzbar.
- **doi.org-Content-Negotiation als headerabhängig gekennzeichnet:** Ohne setzbaren
  `Accept`-Header leitet doi.org auf JS-Verlagsseiten weiter (leerer Body). Der CSL-JSON-Weg gilt
  nur noch für Umgebungen mit Header-Unterstützung; Standard ist der `filter=doi:`-Lookup.

### Geändert

- **DNB-Standardmuster auf `recordSchema=oai_dc` umgestellt** (analog zur K10plus-DC-Abfrage);
  `MARC21-xml` nur noch als Eskalationsstufe (Sparse-then-Full). Reduziert die Antwortgröße je
  DNB-Call erheblich.
- **Welle 0 (Identifier-First):** Batch-DOI-Auflösung ohne OpenAlex-Key jetzt über Crossref
  (`filter=doi:A,doi:B`, kommagetrennt, max. ~5 DOIs) statt Einzelauflösung über doi.org.
- **Kaskaden-Anpassungen:** Internationale Bücher K10plus-zuerst (Open Library nur noch für
  ISBN-Lookups); lobid in den Buch-/Kapitel-Kaskaden Canary-pflichtig; Semantic Scholar wird ohne
  `--s2-key` nicht mehr abgefragt; arXiv- und EconBiz-Erstcalls als Canary.
- **Trefferzahl-Angaben vereinheitlicht** (`rows=2–3` als Rahmen; SKILL.md und api-endpoints.md
  widersprechen sich nicht mehr).

### Hinzugefügt

- **Übersicht „Bekannte Umgebungsausfälle"** im Fallback-Protokoll: Endpunkte, die aus geteilten
  Agent-Umgebungen typischerweise leere Bodies liefern (OpenAlex keyless, lobid, Semantic Scholar
  keyless, Open Library `search.json`, arXiv, EconBiz, doi.org ohne Header), mit der Regel:
  erster Call = Canary, bei Fehlschlag sitzungsweit überspringen, nie in parallele Wellen aufnehmen.
- **arXiv-Ausweichweg** über registrierte Preprint-DOIs (`10.48550/arXiv.…`) via Crossref/OpenAlex.
- **EconBiz-Weblink** als vorbereiteter manueller Prüflink.

## [0.5.1] — 2026-06-26

Schnittstellen-Korrekturen nach Abgleich mit der offiziellen DNB-, ZDB- und Crossref-Dokumentation;
die geänderten Abfragemuster sind gegen die Live-Schnittstellen geprüft.

### Behoben

- **DNB-SRU — korrekter Jahresindex `jhr`:** Der Jahresfilter verwendet nun `jhr` statt des
  K10plus-spezifischen PICA-Index `jah`, der auf der DNB-SRU die Diagnose „Unsupported index" auslöste
  (vergeudeter Call + Retry). `jah`/`pub` ausdrücklich als ungültig für die DNB gekennzeichnet.
- **ZDB-SRU — Relation `all` für mehrteilige Titel:** Der Zeitschriften-ISSN-Lookup nutzt
  `tit all "WORT1 WORT2 …"`. Der Standardvergleich `=` verlangt Wortreihenfolge (Phrasensuche) und
  lieferte bei kurzen Haupttiteln fälschlich 0 Treffer; die `AND`-Verknüpfung bleibt als Fallback.
- **Crossref — „403 URL exceeds maximum length" richtig eingeordnet:** Die Meldung stammt vom
  URL-Längenlimit der `web_fetch`-Umgebung, nicht von Crossref (das lange Titel akzeptiert). Reaktion:
  Titel-Keywords auf 3–4 kürzen (Trim-Reihenfolge), nicht als „nicht gefunden" werten.

### Geändert

- **Crossref-Artikelsuche auf `query.bibliographic` umgestellt** (statt des seit API-Version 66
  veralteten `query.title`); konsistent mit der bereits so gebauten Gegenprobe.
- **Deutsche Zeitschriften — ISSN zuerst über ZDB, dann Crossref-ISSN-Direktendpunkt:** Statt der bei
  kurzen deutschen Titeln versagenden `/journals?query=`-Freitextsuche wird die ISSN über ZDB ermittelt
  und Crossref über `/journals/{ISSN}` bzw. `/journals/{ISSN}/works` angesteuert.
- **OpenAlex-Canary um Mid-Session-Abbruch erweitert:** Nach vier aufeinanderfolgenden leeren Bodies
  Sitzungs-Flag setzen und auf die Crossref-primäre Kaskade umschalten; ein valider Body setzt den
  Zähler zurück.
- **DNB:** schlankes Schema `oai_dc` als Standardoption dokumentiert (analog zur K10plus-DC-Abfrage).

### Hinzugefügt

- **ISSN-Coverage-Heuristik:** Ist eine Zeitschrift in Crossref substanziell abgedeckt
  (`total-dois` ≥ 100) und der konkrete Artikel dennoch über `query.bibliographic` und `query.author`
  nicht auffindbar, stützt das valide 0-Ergebnis eine 🔴-IV-Einstufung; bei schwach indexierten
  Zeitschriften zählt ein Miss nicht Richtung IV.
- **Gegenprobe bei mehreren Autor:innen/Herausgeber:innen:** zusätzlich einen zweiten Namen als
  Suchterm einsetzen; wirkt einseitig gegen falsche 🔴-IV-Urteile.

## [0.5.0] — 2026-06-25

### Hinzugefügt

- **Erweiterte Key-Übergabe:** API-Keys können nun (1) **direkt beim Aufruf** — per Flag oder frei im
  Aufruftext genannt — oder (2) über eine **Konfigurationsdatei** übergeben werden. Neues Flag
  `--config <pfad>`; zusätzlich werden im Arbeitsverzeichnis/Upload liegende Dateien
  (`literaturpruefung-keys.txt`/`.json`/`.env`/`.literaturpruefung-keys`) automatisch erkannt.
  Tolerantes Schlüssel-Wert-Format mit Aliassen; JSON-Variante unterstützt.
- **Keine Doppelnachfrage:** Liegt vorab (Aufruf oder Konfigurationsdatei) mindestens ein Key vor,
  entfällt die Step-0-Rückfrage vollständig; der Skill startet ohne erneutes Fragen.
- **Sequenzieller Rückfallpfad:** Die parallele Wellen-Ausführung ist ausdrücklich als Optimierung
  gekennzeichnet. Umgebungen ohne Nebenläufigkeit arbeiten dieselbe Kaskade rein seriell ab — identisches
  Ergebnis, nur längere Laufzeit; Korrektheit hängt nie von der Parallelität ab.

### Geändert

- **Rate-Limits von Zeit- auf Handlungsregel umgestellt:** Da ein Sprachmodell keine Sekunden „abwartet",
  gilt für BASE und für HTTP 429 nun „höchstens ein Aufruf pro Antwortschritt; Wiederholung erst in einem
  späteren Schritt; nie gebündelt/parallel". Die Sekundenangaben (1,5 s, 5 s …) bleiben nur als
  Begründung des Limits stehen, nicht als ausführbare Anweisung.
- **Operative Dateien gestrafft:** Historische Herleitungen und Einzelbeispiele (OpenAlex-A/B-Diagnose,
  konkrete Beispielquellen) wurden aus SKILL.md und api-endpoints.md generalisiert bzw. in die README
  („Hintergrund & Entwicklungsnotizen") verschoben, um die Attention-Last zur Laufzeit zu senken. Die
  Trennung in kurze SKILL.md und bei Bedarf nachgeladene Referenzdateien bleibt erhalten.
- **`compatibility.md` in die README integriert** und als separate Datei entfernt (reine Informationsdatei).
- **Weitere Verschlankung der operativen Dateien:** nutzer- und entwicklerseitige Erläuterungen
  (Hintergründe, Begründungen, Kostendetails, Beispielquellen) aus SKILL.md und den Referenzdateien nach
  README verlagert; in den Skill-Dateien bleiben nur ausführungsrelevante Regeln. Zeitbasierte
  DNB-/429-Hinweise in api-endpoints.md ebenfalls auf Handlungsregeln umgestellt.

## [0.4.8] — 2026-06-18

### Hinzugefügt

- **Verifikations-Kern** ganz am Anfang der SKILL.md: kompakte Pflichtreihenfolge (DOI → Crossref/
  OpenAlex mit `select=` → K10plus DC → Gegenprobe) und der Grundsatz „jede gewöhnliche Quelle endet in
  I–IV; 🎓/⚪/‚technisch nicht prüfbar' sind eng begrenzte Ausnahmen mit definierten Vorbedingungen".
  Gegen Instruktionsüberladung bei langen Verzeichnissen.

### Geändert

- **Konsistenz-Gate Kategorie ↔ Abweichung:** 🟢 I nur bei leerem Abweichungsfeld; jede inhaltliche
  Abweichung (Jahr/Auflage/Verlag …) erzwingt mindestens 🟡 II. Reprint-Schlupfloch geschlossen — eine
  nur gefundene Neuauflage macht eine Jahresabweichung nicht trivial (🟡 II, bis das zitierte Jahr
  separat bestätigt ist).
- **⚪ ? / „technisch nicht prüfbar" eingegrenzt:** ⚪ ? nur für strukturell nicht katalogisierbare
  Sonderfälle, nie für gewöhnliche Bücher/Artikel und nie als Ersatz für API-Fehlschläge. „Technisch
  nicht prüfbar" nur bei Totalausfall **aller** zuständigen DBs; ein einzelner leerer Body (z. B.
  OpenAlex) bei sonst valider Antwort verlangt das Weiterführen der Kaskade. Gegenprobe + vollständige
  Kaskade sind vor **jedem** Nicht-Treffer-Status Pflicht.
- **Crossref-/OpenAlex-Ausführung gehärtet:** `select=`+`rows=2` sind nicht verhandelbar
  (überdimensionierte Antwort = vergessenes `select=` → erneut, nicht „nicht prüfbar"); leerer Body →
  Retry-Protokoll + Fallback statt Abbruch; Artikel-Gegenprobe per `query.bibliographic`+`query.author`;
  Reprint-/Container-Treffer gelten als Existenznachweis.

### Behoben

- **Key-Leak in Zwischenstand-Dateien:** Ein „Sitzungs-Status"-Block darf nur Aktiv/Inaktiv vermerken,
  **niemals** den Key-Wert selbst. Verbot von Key-Werten in jeder gespeicherten Datei (Zwischenstand,
  Bericht, Transparenz-Log, angezeigte URLs) ausdrücklich verankert.

### Sonstiges

- README: Hinweis zur Modellwahl (stärkeres Modell erhöht Trefferquote/Regeltreue bei langen Listen).

## [0.4.7] — 2026-06-17

### Hinzugefügt

- **BASE (Bielefeld Academic Search Engine) als optionale, key-gebundene Datenbank.** Neues Flag
  `--base-key`; in derselben Step-0-Frage wie OpenAlex/Semantic Scholar abgefragt. BASE wird **nur mit
  Key** abgefragt und ergänzt gezielt graue Literatur, Hochschulschriften, Working Paper und
  Open-Access-Inhalte — **kein** Ersatz für Buch-/Artikel-Kataloge.
- **BASE-Canary** (analog OpenAlex): minimaler Testaufruf zu Sitzungsbeginn; bei leerem Body (Block
  durch IP/User-Agent in geteilten Agent-Umgebungen) wird BASE sitzungsweit deaktiviert.
- **Hartes BASE-Rate-Limit fest verankert:** max. 1 Anfrage/Sekunde, im Skill **≥ 1,5 s Abstand,
  strikt seriell, nie in paralleler Welle** (Verstoß ⇒ Key-Entzug).
- Voller BASE-Abschnitt in `api-endpoints.md` (Endpoint `api.base-search.net`, `PerformSearch`,
  Felder `dctitle`/`dccreator`, JSON-Struktur `response.numFound`/`docs[]`, Canonical-URL via `dclink`),
  Routing-Einbindung (Grey-/Thesis-/OA-Zweig) und README-Hinweis zu Zugangshürde und Rolle.

### Geändert

- **Klassifikations-Sperre:** Ein BASE-Nulltreffer für eine Monografie oder einen Zeitschriftenartikel
  zählt **nicht** als „nicht gefunden" und darf nie Richtung 🔴 IV wirken (BASE deckt diese Typen nicht
  zuverlässig ab). ISBN ist als BASE-Suchfeld ausgeschlossen (nicht durchsuchbar).

## [0.4.6] — 2026-06-17

### Geändert

- Sprache vereinheitlicht: `SKILL.md` sowie alle `references/*.md` (api-endpoints, category-rules,
  database-routing) durchgängig auf **Deutsch** umgestellt — passend zum deutschen akademischen
  Einsatzkontext, zur deutschen Ausgabesprache und zur Nutzerkommunikation. CHANGELOG und README waren
  bereits deutsch. Technische Tokens (URLs, Flags, Query-Syntax, JSON, Feldnamen, Emojis, Dateinamen)
  und die zweisprachige YAML-`description` (Trigger-Erkennung) blieben unverändert; keine inhaltliche
  Regeländerung.

## [0.4.5] — 2026-06-17

### Geändert

- 🎓 FU hart auf echtes FernUni-Lernmaterial begrenzt; explizite Negativliste (Webseiten,
  Pressemitteilungen, Behörden-/Parlaments-/Regierungsdokumente, Nachrichten, Koalitionsverträge,
  Projektberichte sind **nie** 🎓 FU). 🎓 FU erst nach voller FernUni-Prozedur inkl. Buchausgaben-Suche
  und ausgeführter Gegenprobe. Entscheidungsbaum, Kategorientabelle und Abgrenzungen geschärft.
- FernUni-Prozedur: verpflichtender Schritt „Suche nach kommerziell verlegter Buchausgabe"
  (UTB/Nomos/Springer; freie Schlagwortsuche ohne „FernUniversität"), zugleich die Gegenprobe.
- ⚪ ? enggeführt: nur noch Sonderfall ohne URL/Fundstelle; nicht mehr Default für graue Literatur.

### Hinzugefügt

- Graue-Literatur-Prüfung (Step 2, category-rules.md, Routing Part B): Quellen mit URL bzw.
  herausgebender Stelle werden regulär geprüft — URL auflösen, Herausgeber/Titel/Datum abgleichen →
  🟢 I (🟡/🟠 bei Abweichung); tote, nicht bestätigbare URL → 🔴 IV. Solche Quellen laufen nicht mehr
  durch die Katalog-Kaskade.

## [0.4.4] — 2026-06-17

### Geändert

- Step 0 — Auto-Start-Regel: Die API-Key-Frage ist das einzige und letzte Gate; sobald die Antwort
  (Key oder „weiter") vorliegt, beginnt die Recherche unmittelbar in derselben Antwort. Keine zweite
  Bestätigungs-/Bereitschaftsfrage. Liegt ein Key per Flag vor, entfällt die Frage.
- Input Handling: Scope-Klärung öffnet keinen eigenen Turn mehr — sie wird, falls überhaupt nötig, in
  die Step-0-Frage gefaltet; bei eindeutigem Umfang entfällt sie.

## [0.4.3] — 2026-06-17

### Hinzugefügt

- Klassifikations-Gate (direkt vor der Kategorientabelle): 🟢 I / 🟡 II / 🟠 III sind nur bei positivem,
  kanonischem DB-Treffer für das Werk selbst zulässig; ohne Treffer bleiben nur 🔴 IV / 🎓 FU / ⚪ ?.
  Expliziter Entscheidungsbaum und Liste verbotener Begründungen am Entscheidungspunkt.
- Step 0 — einmalige Rückfrage nach API-Keys vor der Suche (OpenAlex; Semantic Scholar optional).
  Neues Flag `--s2-key`. Keys sitzungsweise, nie gespeichert/angezeigt.
- Neue Kategorie 🎓 FU „FernUni-Lernmaterial (nicht überprüfbar)" für nicht auffindbares FernUni-
  Material; ausdrücklich keine Halluzination.

### Geändert

- Kategorie IV umbenannt: „Halluziniert" → „Möglicherweise halluziniert" (Warnung, keine Gewissheit —
  Abwesenheit aus Katalogen beweist keine Fälschung).
- FernUni-Logik: keine Vorab-Einstufung ohne Abfrage mehr; Erkennung löst immer einen Prüfversuch aus
  (K10plus/DNB, GESIS, Originaltext). Auto-getriggert bei FernUni-Signal,
  `--kontext fernuni` nur noch als Override.
- Bewertungstabelle trägt den wichtigsten Beleg-Link inline in der Spalte „Gefundene Quelle";
  vollständige DB-Liste bleibt in Teil 2.

### Entfernt

- Empfehlungen / redaktioneller Rat im Bericht (harte Regel): keine „Empfehlung: …"-,
  „sollte bereinigt werden"- oder Qualitätsurteils-Zusätze; nur Befund, Abweichung und Beleg-Links.
  Duplikate als neutrale Zählung.

## [0.4.2] — 2026-06-10

### Hinzugefügt

- Feld-Semantik-Regel: Titel-Keywords nur aus dem Titelfeld, Autor nur in den Autor-Index;
  Autorname im Titel-Index verboten.
- Klassifikations-Sperre erweitert: 0-Ergebnisse zählen nur bei feldsemantisch korrekter Query.
- Gegenprobe-Pflicht vor 🔴 IV: freie K10plus-Schlüsselwortsuche ohne Index, erst dann ist IV zulässig.
- Schluss-Selbstcheck (Step 5): jede 🟢/🟡/🟠-Zeile braucht einen kanonischen DB-Link; Begründungen
  wie „Klassiker/etabliert/plausibel/Verlag existiert" sind verboten.

### Geändert

- Versionsnummer 0.4.1 → 0.4.2.

## [0.4.1] — 2026-06-10

### Hinzugefügt

- URL-Disziplin (harte Regel, Step 3): Query-URLs nur durch Kopieren der Muster aus api-endpoints.md,
  nie aus dem Gedächtnis; Pflicht-Read des DB-Abschnitts vor der ersten Abfrage; Basis-URL-Kurzreferenz
  aller Datenbanken in SKILL.md (mit Markierung bekannter Falsch-Pfade).
- Fallback-Protokoll Schritt 0: bei leerem Body zuerst die eigene URL gegen das dokumentierte Muster
  prüfen — eine erfundene URL und ein Endpoint-Ausfall sehen identisch aus.
- K10plus: explizite Warnung, dass der Pfad `/sru` nicht existiert (404 → leerer Body); korrekt ist
  `/opac-de-627`.
- Klassifikationsregeln verschärft (Step 3): Rezensionen/Sekundärquellen sind kein Nachweis und stützen
  keine 🟢/🟡-Einstufung; 🟡 II als „existiert wahrscheinlich, nicht indexiert" ist unzulässig.

### Geändert

- Versionsnummer 0.4.0 → 0.4.1.

## [0.4.0] — 2026-06-10

### Hinzugefügt

- Flag `--openalex-key <key>`: Nutzer-Key wird als `&api_key=` an OpenAlex-Anfragen angehängt
  (`mailto=` entfällt dann). Ohne Key fragt der Skill nach gescheitertem Canary einmal nach;
  sonst Crossref-primäre Kaskade.
- Key-Sicherheitsregeln (hart): Key nie in Berichte, Transparenz-Logs, Zwischenstands-Dateien oder
  angezeigte URLs; `api_key=…` vor jeder URL-Wiedergabe entfernen. Canonical-URLs sind keyfrei.
- Kostenhinweis: Mit Key ist jeder Call kostenpflichtig → Effizienz-Grundregeln (Batch, select, Caches)
  gelten verschärft.

### Geändert

- OpenAlex-Abschnitt (api-endpoints.md), README und Sitzungs-Canary auf den schlüsselbasierten Zugriff
  umgestellt (schlüssellos = leerer Body, mit Key valides JSON).
- Versionsnummer 0.3.4 → 0.4.0.

## [0.3.4] — 2026-06-10

### Geändert

- Aufräum-Release ohne Funktionsänderung: rein erklärende/historische Hintergrundinfos aus
  `references/api-endpoints.md` in die README verschoben. `api-endpoints.md` enthält jetzt nur noch
  operative Regeln (Canary-Protokoll, Retry-Schritte, Rate-Limit-Werte).

## [0.3.3] — 2026-06-09

### Behoben

- K10plus „Unsupported index": nackte CQL-Präfixe `tit=`/`per=` (DNB-Syntax) sind bei K10plus nicht
  unterstützt; der `pica.`-Präfix ist Pflicht (`pica.tit + pica.per`). Letzte Eskalationsstufe: freie
  Schlüsselwortsuche ohne Index.
- ZDB-Abfrage: funktionierende Indices `tit=` (Titel→ISSN) und `iss=` (ISSN→Zeitschrift) mit
  `recordSchema=oai_dc`; nur SRU-Endpoint `services.dnb.de/sru/zdb` (Web-Katalog `zdb-katalog.de` ist
  bot-geschützt). Zweck: ISSN-/Existenz-Lookup für nicht in Crossref indexierte dt. Zeitschriften.
- Crossref leerer Body ist kein Umlaut-Problem: UTF-8-Umlaute funktionieren; pauschale
  ASCII-Transliteration ist nicht Standard. Retry-Protokoll: 1× unverändert → 1× ASCII als Notbehelf →
  technischer Fehlschlag, nächste DB.
- Bekannt-defekte Crossref-Muster dokumentiert: `filter=container-doi:` für Bücher und der
  Transform-Endpoint; Ersatz: Kapitelsuche per `query.title`, DOI-Resolve via doi.org-Content-Negotiation.

### Hinzugefügt

- OpenAlex-Sitzungs-Canary als erster Call (Schritt 0): scheitert er, wird OpenAlex sitzungsweit
  deaktiviert und die Crossref-primäre Kaskade gefahren.
- Ausgabeformat-Reminder (Step 5): vor der Berichtserstellung wird der Abschnitt „Output Format" per
  separatem Read-Call erneut gelesen (gegen Kontextdrift). ⚪ ?/🔴 IV ohne Canonical-URL erhalten
  „Nicht gefunden in: …".
- ZDB-Zeile in der Rate-Limit-Tabelle.

### Geändert

- Versionsnummer 0.3.2 → 0.3.3.

## [0.3.2] — 2026-06-09

### Behoben

- Regression OpenAlex HTTP 403 „URL exceeds maximum length": ein langer Autor-Filter zusammen mit der
  `select=`-Liste überschritt das URL-Längenlimit. Fix: Autor-Filter entfällt, Nachname wandert in
  `search=`; `select=`-Liste gestrafft; `mailto=` optional; dokumentierte Trim-Reihenfolge bei 403;
  Batch-DOI-Cap auf max. 5 gesenkt.

### Hinzugefügt

- Fallback-Protokoll „Robuste Ausführung" (api-endpoints.md + Step-3-Block):
  - Technischer Fehlschlag ≠ 0 Treffer (leerer Body / kein JSON / 403 / 5xx / Timeout / JS-Shell gelten
    nicht als „nicht gefunden").
  - Retry-dann-nächste-DB; eine DB nach 2 Fehlschlägen für die Sitzung als „nicht erreichbar" markieren.
  - Klassifikations-Sperre: 🔴 IV / ⚪ ? nur nach mindestens einem validen DB-Ergebnis, sonst „technisch
    nicht prüfbar".
  - Canary vor dem Fan-out: neue Query-Form erst mit einem Test-Call prüfen, dann parallelisieren.
  - Crossref aktiv ansteuern, wenn OpenAlex ausfällt.
- WebSearch-Disziplin: nur letztes Mittel nach valider Kaskadenerschöpfung; `allowed_domains` auf
  akademische/katalogische Quellen beschränkt; nie alleiniger Beleg für 🟢/🟡/🟠; Amazon, ZVAB,
  booklooker, AbeBooks, ResearchGate u. ä. als Nachweis ausgeschlossen.
- JS-Seiten → Claude-in-Chrome (`navigate` + `get_page_text` statt `web_fetch`).

### Geändert

- Versionsnummer 0.3.1 → 0.3.2.

## [0.3.1] — 2026-06-09

Zweites Effizienz-Release (Token/Zeit senken, ohne Qualitätsverlust); rein mechanische Maßnahmen —
sie ändern, *welche* Beifelder die APIs mitschicken, nicht *welche* Felder verglichen werden.

### Hinzugefügt

- Sparse Fieldsets: OpenAlex und Crossref werden immer mit `select=` abgefragt (nur prüfrelevante
  Felder); Open Library mit `&fields=`.
- CSL-JSON für DOI-Resolve (`Accept: application/vnd.citationstyles.csl+json`; Fallback
  `application/json`).
- Identifier-First-Triage (Welle 0): Quellen mit DOI/ISBN werden vor allen Titel/Autor-Kaskaden direkt
  aufgelöst; mehrere DOIs bei OpenAlex in einem Call bündeln.
- Journal-ISSN-Cache, Negativ-Cache und Dedupe.
- Globale Effizienz-Grundregeln als Kasten oben in api-endpoints.md.

### Geändert

- Sparse-then-Full-Fallback: fehlt einem schlank abgefragten Treffer ein Vergleichsfeld, ist er
  mehrdeutig oder scheitert das Parsing, wird genau diese eine Quelle voll abgefragt — kein pauschaler
  Rückfall auf das volle Objekt.
- Trefferzahlen 5 → 3 (`per_page`/`rows`/`limit`/`maximumRecords`).
- Versionsnummer 0.3.0 → 0.3.1.

## [0.3.0] — 2026-06-09

Effizienz-Release; reine Effizienzmaßnahmen ohne Präzisionsverlust.

### Hinzugefügt

- K10plus Dublin-Core-Standardabfrage: K10plus startet immer mit `recordSchema=dc&maximumRecords=1`
  (+ `pica.jah`); MARCXML nur bei Bedarf (Feld 520 Inhaltsverzeichnis / 250 Auflage).
- Dynamischer ISSN-Lookup: fehlt einem Artikel DOI und ISSN, wird die ISSN frisch über die Crossref
  Journals API geholt, Fallback ZDB SRU; bewusst keine statische ISSN-Liste.
- FernUni-Vorsortierung (Step 1a): Quellen mit Verlags-/Ortsmuster „FernUniversität" ohne API-Call ⚪ ?
  (in 0.4.3 durch einen aktiven Prüfversuch ersetzt).
- Sitzungs-Cache für Sammelbände: geteilte Host-Volumes werden nur einmal gesucht.
- Parallele Ausführung in Wellen (4–6 gleichzeitige Calls, max. 3 pro Datenbank).
- Zwischenstand-Speicherung nach jeder 10. Quelle.
- Liste JS-gerenderter Verlagsseiten (De Gruyter, Wiley, Taylor & Francis, SAGE, Elsevier) → direkt zu
  Crossref.
- Nicht unterstützte K10plus-SRU-Indices dokumentiert.

### Geändert

- Kaskaden-Reihenfolge nach dem Prinzip „cheap-and-likely first, verbose-and-throttled last": K10plus
  bleibt vorne bei dt. Büchern, DNB rückt hinter die günstigen JSON-Kataloge (lobid, OpenAlex) und
  dient nur noch als später Fallback.
- Versionsnummer 0.2.1 → 0.3.0.

## [0.2.1] — 2026-06-09

### Hinzugefügt

- lobid (hbz) als weitere schlüsselfreie Live-Datenbank (JSON-LD): dt. Verbundkatalog NRW, Bücher +
  GND-Autorabgleich. In Buch- und Kapitel-Kaskaden eingereiht (nach K10plus/DNB).

### Geändert

- Triggererkennung erweitert: Der Skill aktiviert auch bei „prüfe dieses/das Literaturverzeichnis",
  „check this bibliography" usw. — nicht nur beim eigenen Verzeichnis. Klarstellung in der
  `description`, dass typischerweise fremde Verzeichnisse geprüft werden.

## [0.2.0] — 2026-06-09

Große Funktionserweiterung.

### Hinzugefügt

- 
- Neue Live-Datenbanken:
  - K10plus (SRU) — GBV/SWB-Verbundkatalog; Primärquelle für deutschsprachige Bücher, breiter als die
    DNB und ohne deren Throttling. CQL über `pica.tit/per/all/isb`, MARCXML, Beleg via PPN.
  - Open Library — internationale/englischsprachige Bücher und ISBN-Lookups (JSON).
  - arXiv — Preprints (Physik, Informatik, Mathematik, Statistik, Ökonomie).
  - Library of Congress (SRU) — US-/englischsprachige Monografien (nachrangig).
- Fachgebiets-Routing in neuer, editierbarer Datei `references/database-routing.md`:
  - Auto-Erkennung des Fachgebiets, einmalige Nennung/Rückfrage für das ganze Verzeichnis.
  - Manuelles Übersteuern mit Flag `--fachgebiet` (sozialwissenschaften, wirtschaft, recht, stem,
    medizin, geistes, allgemein).
  - Kaskaden nach Publikationstyp und Fachgebiet, in Prioritätsreihenfolge zur Schonung der Rate Limits.
- Vorbereitete Such-Links pro Quelle (keine Live-Abfrage): Google Scholar (immer), WorldCat, BASE.
- FernUni-Erweiterung: `--kontext fernuni` behandelt „Hagen: FernUniversität" als Verlagssignal.

### Geändert

- Neues Ausgabe-Layout: schlanke Bewertungstabelle (`# | Bewertung | Originalquelle | Gefundene Quelle |
  Art der Abweichungen`); alle Links in einem Block „Belege & Nachschlagen" mit allen tatsächlich
  abgefragten Datenbanken plus vorbereiteten Such-Links, optional je Quelle in `<details>` einklappbar.
- `description` und Versionsnummer (0.1.0 → 0.2.0) aktualisiert.
- README: Datenbankenliste, Flags, BibTeX- und Scholar/WorldCat/BASE-Hinweise ergänzt.

### Hinweise / bewusste Entscheidungen

- BASE wird als vorbereiteter Such-Link statt als Live-Abfrage eingebunden, weil das BASE-API Schlüssel,
  IP-Whitelisting und festen User-Agent verlangt — inkompatibel mit dem schlüsselfreien Skill.
  (In 0.4.7 als optionale, key-gebundene Live-Datenbank nachgerüstet.)
- Library of Congress: die JSON-API liefert nur digitalisierte Objekte, keine Katalogdaten; für die
  Buchprüfung wird die SRU-Katalogschnittstelle genutzt (nachrangig).

## [0.1.0] — vor 2026-06-09

- Ausgangsstand: OpenAlex, Crossref/DOI, DNB SRU, Semantic Scholar, ZBW EconBiz, GESIS.
- Kategoriensystem 🟢 I / 🟡 II / 🟠 III / 🔴 IV / ⚪ ?, Sammelbände-Regel, DOI-Reverse-Check,
  FernUni-Studienbrief-Regeln (`--kontext fernuni`), Proof-of-Work-URLs, Transparenz-Log.
