# API-Endpunkte — Kurzreferenz

> **Effizienz-Grundregeln (gelten für ALLE Datenbanken):**
> 
> 1. **Schlank abfragen.** Immer das leichteste Schema/Fieldset nutzen: K10plus → `recordSchema=dc`,
>    DNB → `recordSchema=oai_dc`, OpenAlex/Crossref → `select=`, DOI-Resolve → Crossref
>    `filter=doi:`+`select=`. Nie das volle Objekt holen, wenn ein sparse Fieldset reicht.
> 2. **Wenige Treffer.** Ergebnislisten auf **2–3** begrenzen (`per_page`/`rows`/`limit`/`size`/
>    `maximumRecords`). Bei gezielter Autor+Titel+Jahr-Abfrage steht der richtige Treffer fast immer ganz oben.
> 3. **Sparse-then-Full-Fallback.** Wenn ein schlank abgefragter Treffer ein Vergleichsfeld vermissen
>    lässt, mehrdeutig ist oder das Parsing scheitert: **genau diese eine Quelle** noch einmal voll
>    abrufen (ohne `select=` / mit MARCXML). Nie pauschal auf das fette Objekt zurückfallen.
> 4. **Identifier zuerst.** DOI/ISBN direkt auflösen, bevor eine Titel/Autor-Kaskade startet (billigster,
>    sicherster Treffer). Mehrere DOIs lassen sich bei OpenAlex in einem Call bündeln (`filter=doi:a|b|c`);
>    **ohne OpenAlex-Key** bei Crossref (`filter=doi:A,doi:B`, kommagetrennt — siehe Crossref-Abschnitt).

## Robuste Ausführung & Fallback-Protokoll (ZUERST lesen)

Die Verifikationsqualität hängt davon ab, dass ein **technischer Fehlschlag** nie als „Quelle nicht
gefunden" missgedeutet wird. Sonst entstehen falsche 🔴-IV-Urteile oder die KI weicht unkontrolliert
auf allgemeine Websuche aus (und zieht Amazon/ZVAB statt Bibliothekskatalogen heran).

**1. Technischer Fehlschlag vs. valides 0-Ergebnis — unbedingt unterscheiden:**

- **Technischer Fehlschlag** = leerer Response-Body / nur die URL wird zurückgegeben / kein/kaputtes
  JSON / HTTP 403 (`URL exceeds maximum length`) / 4xx / 5xx / Timeout / JS-Shell ohne Inhalt.
- **Valides 0-Ergebnis** = wohlgeformte Antwort, die sauber „keine Treffer" sagt
  (z. B. OpenAlex `"meta":{"count":0}`, Crossref `"total-results":0`, leeres SRU-`searchRetrieveResponse`).

**2. Reaktion auf technischen Fehlschlag (NICHT als 0 Treffer werten):**
0. **ZUERST die eigene URL gegen das dokumentierte Muster in dieser Datei prüfen** — Basis-Pfad
   und Indices Zeichen für Zeichen vergleichen. `web_fetch` zeigt **404-Fehlerseiten als leeren
   Body**: Eine erfundene URL (z. B. K10plus `/sru` statt `/opac-de-627`) ist von einem
   Datenbank-Ausfall **nicht unterscheidbar**.

1. Bei 403/Länge: URL kürzen (siehe OpenAlex-Trim-Reihenfolge) und **einmal** erneut.
2. Sonst: **einmal mit VARIIERTER Query** erneut — **nie wortgleich**. Viele Agent-Umgebungen
   **cachen `web_fetch`-Antworten pro URL**; die identische URL erneut zu senden liefert denselben
   (auch denselben leeren) Body und ist als Retry wertlos. Variieren in dieser Reihenfolge:
   (a) vollständigerer Titel inkl. Stoppwörter/Untertitel-Anfang, (b) zweiten Autor in den
   Autor-Parameter aufnehmen bzw. auf ihn wechseln, (c) andere Kerntitel-Keyword-Mischung.
   Leere Bodies sind oft **query-spezifisch reproduzierbar** (mutmaßlich langsame Suchen mit sehr
   häufigen Wörtern → Proxy-Timeout); eine variierte Query auf dasselbe Werk trifft dann sofort.
3. Scheitert es wieder: **zur nächsten Datenbank der Kaskade** weitergehen (Routing-Datei). Die
   Datenbank für den Rest der Sitzung als „nicht erreichbar" im Transparenz-Log vermerken, wenn sie
   2× am Stück technisch fehlschlägt.
4. **Niemals** aufgrund eines technischen Fehlschlags klassifizieren und **nicht** still auf WebSearch
   ausweichen.

**3. Klassifikations-Sperre:** Ein Werk darf nur dann als 🔴 IV oder ⚪ ? („nicht gefunden") eingestuft
werden, wenn **mindestens eine** Datenbank ein **valides** Ergebnis (echtes 0-Ergebnis) geliefert hat —
nicht, wenn nur technische Fehlschläge vorlagen. Andernfalls: weitersuchen oder im Bericht offen als
„technisch nicht prüfbar – APIs nicht erreichbar" ausweisen.
**Feldsemantik-Vorbehalt:** Ein 0-Ergebnis zählt nur, wenn die Query korrekt gebaut war — Titel-Keywords
ausschließlich aus dem Titelfeld, Autor ausschließlich im Autor-Index. `pica.tit=Crozier Zwänge` (Autor
im Titel-Index) erzeugt ein formal valides, aber bedeutungsloses 0-Ergebnis. **Vor jedem 🔴 IV:** eine
Gegenprobe per freier K10plus-Suche — **exakt** das Muster
`query=pica.all%3DKERNTITEL_KEYWORDS+AUTOR_NACHNAME` (kopierfertig im K10plus-Abschnitt → freie
Suche) — robust gegen Feldsemantik-Fehler. **Gültigkeit:** Die Gegenprobe muss Autor-Nachname UND
Kerntitel-Keywords enthalten und eine überschaubare Trefferzahl liefern; eine unspezifische Query
(nur ein Allerweltswort oder nur ein Jahr, ohne Autor) mit dreistelliger+ Trefferzahl ist KEINE
ausgeführte Gegenprobe. **Bei mehreren Autoren/Herausgebern** zusätzlich **mindestens einen zweiten Namen**
als Suchterm einsetzen (z. B. `pica.all%3DZWEITER_NAME+KERNTITEL`); der Erstautor ist nicht immer der am
besten katalogisierte. Diese Erweiterung kann nur zusätzliche Treffer erzeugen, nie einen bestehenden
entwerten — sie wirkt allein gegen falsche 🔴-IV-Urteile.

**4. Canary vor dem Fan-out:** Bevor eine **neue** Query-Form/Endpoint parallel in einer Welle (4–6
Calls) abgefeuert wird, **erst EINEN** Test-Call senden und prüfen, dass wohlgeformtes JSON zurückkommt.
**Cache-Vorbehalt:** Als Canary nie eine in dieser Sitzung (oder als konstantes Testwort in jeder
Sitzung) schon verwendete URL senden — gecachte Antworten können eine tote DB als lebendig ausweisen;
ein sitzungseindeutiges Suchwort erzwingt den Live-Call. Erst dann auffächern. Scheitert der Canary technisch, erst das Muster reparieren (kürzen) bzw. die DB für
diese Welle überspringen — so verbrennt nicht eine ganze Welle an einem kaputten Muster.

**5. Resiliente Reihenfolge bei Artikeln:** Fällt OpenAlex technisch aus, ist **Crossref** der nächste
Schritt — dessen URLs sind kurz (kein langer Filter). Crossref also **aktiv** ansteuern, nicht überspringen.

**6. WebSearch ist nur letztes Mittel** — Regeln im Abschnitt „Vorbereitete Suchlinks" und in SKILL.md.
JS-gerenderte Seiten gehen an Claude-in-Chrome, nicht an `web_fetch` (Abschnitt „Nicht per web_fetch
abrufbare Verlagsseiten").

**7. Bekannte Umgebungsausfälle (geteilte Agent-Umgebungen → leerer Body).** Die folgenden Endpunkte
liefern aus vielen geteilten Agent-Umgebungen (Proxy-IP/fehlende Header) reproduzierbar einen leeren
Body, obwohl dieselben URLs vom eigenen Rechner funktionieren. **Regel:** Der **erste** Call einer
Sitzung an einen dieser Endpunkte ist zugleich sein **Canary** — scheitert er, den Endpunkt
**sitzungsweit überspringen** (Transparenz-Log) und **nie** in parallele Wellen aufnehmen; kein
Retry-Sturm. Ein leerer Body dieser Endpunkte ist immer ein technischer Fehlschlag, nie ein 0-Ergebnis.

| Endpunkt | Typisches Verhalten | Ausweichweg |
|---|---|---|
| OpenAlex (keyless) | leerer Body | `--openalex-key` oder Crossref-primäre Kaskade |
| lobid `/resources/search` | leerer Body | K10plus (DC) / DNB; lobid-Weblink als manueller Prüflink |
| Semantic Scholar (keyless) | leerer Body | ohne `--s2-key` gar nicht abfragen |
| Open Library `search.json` | leerer Body | nur `/isbn/`-Lookup nutzen (funktioniert); Titel/Autor → K10plus |
| arXiv `export.arxiv.org` | leerer Body | arXiv-DOIs (`10.48550/arXiv.…`) via Crossref/OpenAlex auflösen |
| ZBW EconBiz | leerer Body | OpenAlex/Crossref/K10plus; EconBiz-Weblink als Prüflink |
| doi.org ohne Accept-Header | leerer Body (JS-Verlage) | Crossref `filter=doi:` (siehe Crossref) |

---

## OpenAlex

**Zweck:** Breite Abdeckung von Zeitschriften, Büchern, Preprints. Stark für Sozialwissenschaften und
Wirtschaft. Gute deutschsprachige Abdeckung.

### ⚠️ KRITISCH — OpenAlex braucht in vielen Agent-Umgebungen faktisch einen API-Key (Keyless = leerer Body)

**Operativ:** Ohne `api_key=` liefern OpenAlex-Anfragen in geteilten Agent-Umgebungen oft einen leeren
Body (nur URL-Echo, kein JSON); **mit** Key sofort valides JSON (Antwort enthält `cost_usd` pro Call).
Das ist **keine** Netzwerk-/Sandbox-Blockade, sondern die IP-gebundene Keyless-Kontingentlogik — vom
eigenen Browser/Rechner funktioniert dieselbe URL keyless. Konsequenz für den Skill: Canary + Key- bzw.
Crossref-Fallback (siehe unten). Die ausführliche Herleitung (Live-A/B-Test) steht in der README →
„Hintergrund & Entwicklungsnotizen" und wird zur Laufzeit nicht benötigt.

**Key-Verwendung (Flag `--openalex-key`, siehe SKILL.md):**

- An jede OpenAlex-URL `&api_key=USER_KEY` anhängen. Der Parameter ist kurz (kein URL-Längenrisiko);
  `mailto=` ist bei gesetztem Key redundant und entfällt.
- **Sicherheits-Pflichtregeln:** Der Key stammt ausschließlich vom Nutzer und darf **niemals** in
  gespeicherte Dateien gelangen — nicht in den Prüfbericht, nicht ins Transparenz-Log, nicht in
  Zwischenstands-Dateien, nicht in „Belege & Nachschlagen"-Links. Vor jeder Wiedergabe einer
  Abfrage-URL `api_key=…` entfernen. Canonical-URLs (`openalex.org/works/W…`) enthalten nie einen Key.
- Jeder Call kostet (metered) → Effizienz-Grundregeln strikt: Batch-DOI bündeln, `select=`,
  `per_page=2–3`, Caches, keine Wiederholungs-Calls.

**Konsequenz — OpenAlex-Canary ganz an den Anfang (vor Welle 0):**

1. **Key vorhanden:** Canary **mit** Key senden — mit **sitzungseindeutigem Suchwort**
   (`GET https://api.openalex.org/works?search=canary-JJJJMMTT-hhmm&per_page=1&api_key=…`,
   Datum/Uhrzeit einsetzen). **Nie** ein konstantes Testwort wie `search=test` verwenden: Viele
   Agent-Umgebungen cachen `web_fetch`-Antworten pro URL, ein konstanter Canary kann aus dem Cache
   „gelingen", obwohl OpenAlex aktuell leer antwortet. 0 Treffer sind okay — entscheidend ist
   wohlgeformtes JSON (`meta`/`results`). Scheitert der einzigartige Canary
   technisch → OpenAlex sitzungsweit deaktivieren.
2. **Kein Key:** Keyless-Canary senden. Kommt kein wohlgeformtes JSON (der Normalfall in geteilten Agent-Umgebungen), wurde
   die Key-Frage bereits in Step 0 gestellt — hier nicht erneut fragen. Liegt kein Key vor: OpenAlex für
   die **gesamte Sitzung** als „nicht nutzbar (kein API-Key)" markieren (Transparenz-Log) und auf die
   **Crossref-primäre Artikel-Kaskade** umschalten (Crossref → Semantic Scholar → Fach-DB). Keine
   weiteren OpenAlex-Versuche pro Quelle.
3. Ohne OpenAlex laufen Batch-DOI-Lookups über Crossref (`filter=doi:A,doi:B`, kommagetrennt —
   siehe Crossref-Abschnitt).

**Mid-Session-Abbruch (Canary gelang, OpenAlex fällt später aus):** Der Canary kann zu Sitzungsbeginn
gelingen, danach aber (z. B. nach Kontext-Komprimierung oder bei aufgebrauchtem Tageskontingent)
durchgehend leere Bodies liefern. Daher mitzählen: Nach **vier aufeinanderfolgenden leeren
OpenAlex-Bodies** das Sitzungs-Flag `openalex_blocked` setzen, sofort auf die **Crossref-primäre
Artikel-Kaskade** umschalten und **keine weiteren OpenAlex-Versuche** mehr senden. Die Schwelle 4 ist
bewusst konservativ gewählt, damit einzelne transiente Leerbodies OpenAlex nicht vorschnell
deaktivieren; ein zwischenzeitlich valider Body setzt den Zähler zurück.

### ⚠️ KRITISCH — OpenAlex-URLs KURZ halten (`web_fetch`-Längenlimit)

Manche `web_fetch`-Implementierungen haben ein **maximales URL-Längenlimit**. Wird es überschritten,
kommt **HTTP 403 `"URL exceeds maximum length"`** — kein Treffer, sondern ein technischer Fehlschlag.
Der Hauptverursacher ist der lange Autor-Filter `filter=authorships.author.display_name.search:…`
**in Kombination mit** `select=`. **Daher: den Autor-Filter NICHT verwenden** — Autor stattdessen mit
in `search=` schreiben. Das hält die URL klein und `select=` (Token-Sparen) bleibt nutzbar.

**Trim-Reihenfolge, falls eine OpenAlex-URL trotzdem mit 403 (Länge) quittiert wird:**

1. Autor-`filter=` entfernen → Autor in `search=` (fast immer schon der Fix).
2. `mailto=` weglassen (optional, nur Polite Pool).
3. Titel-Keywords auf 2 kürzen.
   `select=` bleibt drin — es ist kurz genug und spart Antwort-Token.

### ⭐ Standardabfrage — `search=` (Titel + Autor) mit kurzem `select=`

Standard-Feldliste (knapp halten): `select=id,doi,title,publication_year,authorships,primary_location`

**Titel + Autor suchen (kopierfertig, KEIN author-filter):**

```
GET https://api.openalex.org/works?search=TITLE_KEYWORDS+AUTHOR_LASTNAME&select=id,doi,title,publication_year,authorships,primary_location&per_page=3&mailto=user@example.com
```

OpenAlex' `search=` deckt Titel **und** Autor/Abstract ab — der Nachname im `search=` genügt zur
Eingrenzung und vermeidet den langen Filter.

**Nur nach Titel suchen:**

```
GET https://api.openalex.org/works?search=TITLE_KEYWORDS&select=id,doi,title,publication_year,authorships,primary_location&per_page=3&mailto=user@example.com
```

**Direkter DOI-Lookup (Einzelwerk, kürzeste & sicherste Abfrage):**

```
GET https://api.openalex.org/works/https://doi.org/10.xxxx/xxxx?select=id,doi,title,publication_year,authorships,primary_location&mailto=user@example.com
```

**Batch mehrerer DOIs in EINEM Call** — **max. 5 DOIs pro Batch** (mehr sprengt das URL-Limit):

```
GET https://api.openalex.org/works?filter=doi:10.1007/xxx|10.1086/yyy|10.2307/zzz&select=id,doi,title,publication_year,authorships,primary_location&per_page=5&mailto=user@example.com
```

(Der `doi:`-Filter ist kurz und unkritisch; nur die Anzahl der DOIs begrenzen.)

**⤵ Sparse-then-Full-Fallback:** Wirkt ein Treffer unvollständig/mehrdeutig (Vergleichsfeld fehlt,
Parsing scheitert), **dieselbe eine Quelle ohne `select=` erneut abrufen**. Nur die betroffene Quelle.

### HTTP-Statuscodes richtig deuten (OpenAlex-spezifisch)

OpenAlex nutzt Standard-HTTP-Codes mit einem aussagekräftigen `message`-Feld im Fehler-JSON
(z. B. `{"error":"Invalid filter","message":"Unknown filter field: author_name. Did you mean:
authorships.author.id?"}`). **Das `message`-Feld lesen** — es nennt oft direkt den Fix. Zuordnung:

| Code  | Bedeutung                                                        | Reaktion im Skill                                                                                                                                                                                                                                                  |
| ----- | ---------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `200` | OK                                                               | Treffer auswerten. `meta.count:0` = **valides 0-Ergebnis** (echtes „nicht gefunden").                                                                                                                                                                              |
| `301` | Entität wurde **zusammengeführt** (merged)                       | **Kein Fehler, kein 0-Ergebnis.** Redirect folgen — die Quelle **existiert**, zählt als gefunden. Tritt bei Singleton-`/works/W…`- oder DOI-Lookups auf.                                                                                                           |
| `400` | Ungültige Filter-/Parametersyntax                                | **Query-Bug, kein** „nicht gefunden". `message` lesen, Syntax fixen (z. B. Name→ID, Integer-Feld), **einmal** korrigiert wiederholen.                                                                                                                              |
| `403` | **Zwei Ursachen unterscheiden!**                                 | (a) `web_fetch`-Wrapper meldet `"URL exceeds maximum length"` → URL trimmen (Reihenfolge oben). (b) OpenAlex selbst (`message` nennt Rate/Forbidden) → kurz warten, **einmal** erneut; sonst Crossref. Beides ist ein **technischer Fehlschlag**, kein 0-Ergebnis. |
| `404` | Entität existiert nicht                                          | Bei **Singleton-Lookup** (`/works/W…`, DOI-Direktauflösung) ein **valides** „nicht gefunden" → darf 🔴 IV stützen. Vorher prüfen, ob nicht in Wahrheit ein `301`-Merge vorlag.                                                                                     |
| `429` | **Tages**-Limit erschöpft (oft = keyless $0,01/Tag aufgebraucht) | Technischer Fehlschlag. 3 s warten, **einmal** erneut; bleibt es → OpenAlex für die Sitzung als „nicht erreichbar" markieren, auf Crossref umschalten. Key behebt die Hauptursache.                                                                                |
| `500` | Vorübergehender Serverfehler                                     | Technischer Fehlschlag. Exponentiell zurückweichen (1 s, 2 s, 4 s), bis zu 2× erneut; sonst Crossref.                                                                                                                                                              |

**Merksatz:** Nur `200` mit `meta.count:0` und `404` beim Singleton-Lookup sind **valide
0-Ergebnisse**. `301` heißt **gefunden**. Leerer Body / nur URL-Echo / kein JSON / `400`/`403`/`429`/
`5xx` sind **technische Fehlschläge** → Fallback-Protokoll (Abschnitt **„Robuste Ausführung"** oben),
**nie** als „nicht gefunden" klassifizieren.

**Tipps:**

- 2–3 Keywords aus dem Haupttitel nutzen, nicht den vollen Titel-String.
- Leerzeichen als `%20` (oder `+`) und Sonderzeichen URL-codieren.
- Ist `--openalex-key` gesetzt: in **allen** obigen Mustern `&api_key=USER_KEY` anhängen und
  `mailto=` weglassen (redundant). Ohne Key: `mailto=` optional (Polite Pool).

**Rate-Limits:** Keyless-Freikontingent ist in geteilten Agent-Umgebungen praktisch sofort wirkungslos
(leerer Body statt Antwort, siehe Canary-Abschnitt oben); mit kostenlosem Key valide Antworten (Feld
`cost_usd` pro Call). Technisches Limit ~10 Requests/Sekunde. Kontingent-/Kostendetails: README →
„Hintergrund & Entwicklungsnotizen".

**Aufbau der kanonischen URL:**

- Antwort enthält `"id": "https://openalex.org/W2503015745"`.
- Kanonische Web-URL: `https://openalex.org/works/W2503015745`.
- Link-Format: `[OpenAlex W2503015745](https://openalex.org/works/W2503015745)`.

**Wichtige Antwortfelder:** `title`, `authorships[].author.display_name`, `publication_year`, `primary_location.source.display_name`, `doi`, `id`, `type`

---

## Crossref

**Zweck:** Das umfassendste DOI-Register. Am besten für Zeitschriftenartikel mit DOI und für die direkte
DOI-Verifikation.

### ⭐ Standardabfrage — IMMER mit `select=` (sparse fieldset)

Crossref-Werke enthalten per Default `reference[]` (oft hunderte Einträge), `abstract`, Lizenz- und
Funder-Blöcke. **Immer `select=` setzen** und `rows=3`.

Standard-Feldliste: `select=DOI,title,author,published,container-title,publisher,ISBN,type`

**Titel + Autor suchen (kopierfertig) — `query.bibliographic`, max. 3–4 Titelwörter:**

```
GET https://api.crossref.org/works?query.bibliographic=TITLE_KEYWORDS&query.author=AUTHOR_LASTNAME&select=DOI,title,author,published,container-title,publisher,ISBN,type&rows=3&mailto=user@example.com
```

> **Parameter `query.bibliographic` statt `query.title`.** Crossref hat `query.title` mit API-Version
> 66 zugunsten von `query.bibliographic` (Titel + Autor + ISSN + Jahr) als veraltet markiert;
> `query.bibliographic` ist der empfohlene Parameter für die Zitationssuche und wird hier durchgängig
> verwendet (auch in der Gegenprobe).
>
> **403 „URL exceeds maximum length" ist ein Umgebungslimit, NICHT Crossref.** Crossref selbst nimmt
> auch lange Titel an (live: 9-Wort-`query.bibliographic` → HTTP 200, 1.047 Treffer; ein echtes
> Crossref-Längenlimit wäre HTTP 414). Die 403-Meldung stammt vom `web_fetch`-URL-Längenlimit der
> Umgebung — wie in der OpenAlex-Sektion. **Reaktion:** Titel-Keywords auf 3–4 kürzen, nötigenfalls
> `mailto=` weglassen (Trim-Reihenfolge wie bei OpenAlex). Ein verkürzter Treffer bedeutet **nicht**,
> dass Crossref den vollen Titel abgelehnt hätte.

**⭐ Direkter DOI-Lookup — über die Listen-Route mit `filter=doi:` (sparse, live verifiziert):**

```
GET https://api.crossref.org/works?filter=doi:10.xxxx/xxxx&select=DOI,title,author,published,container-title,publisher,ISBN,type&rows=1&mailto=user@example.com
```

**Batch mehrerer DOIs in EINEM Call** (kommagetrennt, **max. ~5 DOIs** wegen URL-Länge) — funktioniert
**ohne Key** und ist damit der Standard-Batch-Weg der Welle 0, wenn kein OpenAlex-Key vorliegt:

```
GET https://api.crossref.org/works?filter=doi:10.xxxx/aaa,doi:10.yyyy/bbb&select=DOI,title,author,published,container-title&rows=5&mailto=user@example.com
```

Ein DOI, der nicht im Ergebnis auftaucht (`total-results` kleiner als die Anzahl der angefragten
DOIs), ist bei Crossref nicht registriert → für diesen DOI OpenAlex-Lookup bzw. reguläre Kaskade.

> **⚠️ `select=` funktioniert NUR auf der Listen-Route `/works?…`.** Auf der Singleton-Route
> (`/works/10.xxxx/xxxx?select=…`) liefert `select=` einen Fehler, der in `web_fetch` als **leerer
> Body** erscheint (live verifiziert, deterministisch) — nicht als Ausfall deuten. Der
> Singleton-Lookup **ohne** `select=` funktioniert, liefert aber das volle Objekt inkl. `abstract`
> und `reference[]` — nur als Sparse-then-Full-Fallback verwenden.

**DOI auflösen via doi.org (CSL-JSON) — NUR wenn das Fetch-Tool Request-Header setzen kann:**

```
GET https://doi.org/10.xxxx/xxxx
Headers: Accept: application/vnd.citationstyles.csl+json
```

Content-Negotiation liefert ein schlankes CSL-Objekt (Titel, Autoren, Jahr, Container, Verlag) statt der
vollen Crossref-Antwort. Felder: `title`, `author[].family/given`, `issued.date-parts[0][0]`,
`container-title`, `publisher`, `DOI`. Der zurückgegebene `title`/`author` **muss** zur Zitation passen —
sonst halluzinierter DOI → 🔴 IV. (Fallback, falls ein Server CSL nicht liefert: `Accept: application/json`.)

> **⚠️ Header-lose Fetch-Tools:** Kann das Fetch-Tool der Umgebung **keine** eigenen Request-Header
> setzen (viele Agent-Umgebungen), leitet doi.org ohne Accept-Header auf die Verlagsseite weiter —
> bei JS-Verlagen kommt ein **leerer Body** zurück (live verifiziert). Dann **nicht** doi.org
> verwenden, sondern den `filter=doi:`-Lookup oben (deckt denselben Verifikationszweck ab; DOIs
> außerhalb Crossrefs — z. B. DataCite — über OpenAlex-DOI-Lookup prüfen).

**⤵ Fallback bei Problem:** Fehlt einem `select=`-Treffer ein Vergleichsfeld oder ist der Match unsicher,
**dieselbe Quelle ohne `select=` erneut abrufen** (volles Objekt). Nur für die betroffene Quelle.

### ⚠️ Leerer Response-Body bei Crossref — Retry-Protokoll (Umlaute NICHT pauschal entfernen)

Vereinzelt liefert Crossref über `web_fetch` einen **leeren Body** (nur URL-Echo), auch bei
UTF-8-Umlauten (`%C3%A4`) — das ist ein **transientes Sandbox-/Netzwerkproblem**, kein
Umlaut-Problem (UTF-8-Umlaute funktionieren in Crossref-Queries grundsätzlich).

**Daher kein pauschales ASCII:** Eine ASCII-Transliteration (`ä→a`) als Standard **senkt die
Trefferquote** (z. B. `Agilitat` findet das Buch nicht, `Agilität` schon).

**Retry-Protokoll bei leerem Body (nie wortgleich wiederholen — Cache!):**

1. **Einmal mit VARIIERTER Query** wiederholen (UTF-8 beibehalten): vollständigerer Titel inkl.
   Stoppwörter, zweiter Autor in `query.author`, oder andere Kerntitel-Keyword-Mischung. Die
   identische URL erneut zu senden ist wertlos — gecachte leere Bodies kommen identisch zurück,
   und leere Bodies sind oft query-spezifisch reproduzierbar (mutmaßlich langsame Suchen mit sehr
   häufigen Wörtern); eine variierte Query auf dasselbe Werk trifft dann sofort.
2. Bleibt der Body leer: **einmal** mit ASCII-Transliteration (`ä→a`, `ö→o`, `ü→u`, `ß→ss`) wiederholen —
   als Diagnose-/Notbehelf, nicht als Standard.
3. Liefert auch das nichts Wohlgeformtes: technischer Fehlschlag → nächste Datenbank der Kaskade
   (Fallback-Protokoll oben). Ein ASCII-0-Treffer ist **kein** Beleg für „nicht gefunden", wenn die
   UTF-8-Query nie valide beantwortet wurde.

### ⚠️ Bekannt-defekte Crossref-Muster (nicht verwenden)

- **`filter=container-doi:` für Bücher:** Funktioniert nur, wenn das Werk als Container registriert
  ist — bei normalen Buch-DOIs (z. B. Schäffer-Poeschel) kommt ein leerer Body. **Nicht** zur
  Kapitel-Enumeration von Büchern verwenden. Stattdessen: Kapitel direkt per
  `query.bibliographic=KAPITELTITEL&query.author=AUTOR` suchen (DOI-Suffixe wie `…-45` zeigen Kapitel an).
- **Transform-Endpoint `api.crossref.org/works/DOI/transform/...`:** Liefert über `web_fetch` oft
  Binärdaten (Content-Type-Verhandlung scheitert). Stattdessen den `filter=doi:`-Lookup (oben) bzw. —
  nur mit Header-Unterstützung — Content-Negotiation über `doi.org`.
- **Singleton-Route mit `select=` (`/works/10.xxxx/xxxx?select=…`):** liefert einen leeren Body
  (live verifiziert) — `select=` gehört nur auf die Listen-Route `/works?…` (siehe DOI-Lookup oben).

**Rate-Limits:** Großzügig im Polite Pool (`mailto=` ergänzen). Kein Key erforderlich.

### ISSN-Lookup vor der Artikelsuche (dynamisch, keine statische Liste)

Wenn ein Zeitschriftenartikel **keinen DOI** hat und in der zitierten Quelle **keine ISSN** angegeben
ist, die ISSN **frisch über die Crossref Journals API** holen — nicht raten und keine fest hinterlegte
ISSN-Liste pflegen (der Skill prüft auch nicht-sozialwissenschaftliche Verzeichnisse, eine fachspezifische
Liste wäre dort wertlos und veraltet schnell):

```
GET https://api.crossref.org/journals?query=ZEITSCHRIFTENNAME&rows=1&mailto=user@example.com
```

- ISSN aus `message.items[0].ISSN` entnehmen (Array: i.d.R. `[0]` = Print, `[1]` = Electronic).
- Den Zeitschriftennamen **präzise und vollständig** angeben — bei kurzen/mehrdeutigen Titeln
  (z. B. „Arbeit") liefert eine knappe Query sonst die falsche Zeitschrift.
- Anschließend Artikelsuche mit der gefundenen ISSN: `query.bibliographic=...&filter=issn:XXXX-XXXX`.

**Beispiel:** `?query=Zeitschrift+für+Soziologie&rows=1` → `ISSN: ["0340-1804","2366-0325"]`,
Publisher „Walter de Gruyter GmbH".

> **⚠️ Deutsche Zeitschriften: ISSN zuerst über ZDB, nicht über `/journals?query=`.** Die
> Crossref-`/journals`-Freitextsuche versagt bei kurzen deutschen Titeln (live: `?query=Arbeit
> Zeitschrift Arbeitsforschung` → `total-results: 0`, obwohl 1.280 Artikel der Zeitschrift indexiert
> sind). Für deutschsprachige Zeitschriften (Heuristik: deutscher Titel/Verlag) daher die **ISSN
> zuerst über ZDB SRU** ermitteln (`tit all`, siehe unten) und Crossref dann über den **direkten
> ISSN-Endpunkt** ansteuern:
>
> ```
> GET https://api.crossref.org/journals/ISSN?mailto=user@example.com          # Zeitschriften-Metadaten + Coverage
> GET https://api.crossref.org/journals/ISSN/works?filter=...&select=...&rows=3 # Artikel der Zeitschrift
> ```
>
> Der direkte ISSN-Endpunkt ist deterministisch (live: `/journals/0941-5025` → Verlag, beide ISSN,
> `counts.total-dois`, `breakdowns.dois-by-issued-year`) und umgeht die schwache Freitextsuche. Für
> internationale Zeitschriften bleibt `/journals?query=` als Einstieg zulässig.

> **ISSN-Coverage-Heuristik (stützt 🔴 IV bei Artikeln).** Liefert der ISSN-Bestand eine
> **substanzielle Abdeckung** (Richtwert `counts.total-dois` ≥ 100 bzw. ≥ 100 Treffer beim reinen
> `filter=issn:`-Lauf) und bleibt der gesuchte Artikel dennoch über `query.bibliographic` **und**
> `query.author` unauffindbar, dann ist das ein **valides 0-Ergebnis für das Werk** und **stützt eine
> 🔴-IV-Einstufung** (Hinweis an den Nutzer: Artikel über Verlags-Direktzugang prüfen). Die Heuristik
> greift **nur** bei positiver Coverage — bei schlecht indexierten Zeitschriften (< 100) zählt ein
> Crossref-Miss **nicht** Richtung IV (dann ZDB-Existenznachweis + ⚪/Hinweis).

### Fallback für deutsche Zeitschriften: ZDB SRU

**Zweck im Skill:** Die ZDB (Zeitschriftendatenbank) ist hier **kein** Werk-Nachweis, sondern der
**ISSN-/Existenz-Lookup-Fallback für Zeitschriften, die Crossref nicht indexiert** (typisch: kleinere
deutsche Zeitschriften, z. B. *Arbeit*, *Industrielle Beziehungen*). Sie beantwortet nur „existiert
diese Zeitschrift, welche ISSN, welcher Verlag?" — nie „existiert dieser Artikel?".

**⚠️ NUR den SRU-Endpoint `services.dnb.de/sru/zdb` verwenden — NIEMALS den Web-Katalog
`zdb-katalog.de`.** Der Web-Katalog ist durch Anubis (Proof-of-Work-Bot-Schutz) für maschinelle
Abrufe gesperrt („Oh noes! / Access Denied"). Der SRU-Endpoint läuft über die DNB-Infrastruktur
und ist frei zugänglich.

**Funktionierende Indices (live getestet):** `tit` (Titel), `tst` (vollständiger Titel inkl.
Untertitel), `iss` (ISSN), `zdbid` (ZDB-ID), `idn` (DNB-Identifier).
**Nicht verwenden:** `dc.title=` mit mehreren Stichworten — liefert fälschlich 0 Treffer.
**Schema:** `oai_dc` bevorzugen (kompakt, < 25 Zeilen, enthält ISSN + ZDB-ID); MARC21-xml nur bei Bedarf.

**⚠️ Relation Qualifier beachten — `all` statt `=` für mehrere Titelwörter.** Der Standardvergleich
`=` bedeutet in der ZDB-CQL „Reihenfolge der Suchbegriffe muss übereinstimmen" und wirkt damit wie
eine **Phrasensuche**: `tit=Arbeit Zeitschrift Arbeitsforschung` scheitert (0 Treffer), weil der reale
Titel „Arbeit : Zeitschrift *für* Arbeitsforschung …" das eingeschobene „für" enthält und die
Wortfolge bricht. Die Relation **`all`** („Reihenfolge muss nicht übereinstimmen") trifft hingegen.

**Titel → ISSN (kopierfertig, live verifiziert):**

```
GET https://services.dnb.de/sru/zdb?version=1.1&operation=searchRetrieve&query=tit%20all%20%22ZEITSCHRIFTENTITEL_KEYWORDS%22&maximumRecords=3&recordSchema=oai_dc
```

Beispiel: `tit%20all%20%22Arbeit%20Arbeitsforschung%20Arbeitspolitik%22` → 2 Treffer mit ISSN
0941-5025 (Print, ZDB-ID 1106284-8) + 2365-984X (Online, ZDB-ID 2064442-5), Verlag De Gruyter
Oldenbourg. UTF-8-Umlaute funktionieren. (Hinweis: `tst=` mit denselben Wörtern ergab 0 Treffer —
`tit all` ist vorzuziehen.)

**Fallback (gleichwertig):** zwei `tit=`-Klauseln per `AND` verknüpfen, je ein charakteristisches
Wort aus Haupt- und Untertitel: `query=tit%3DArbeit%20and%20tit%3DArbeitspolitik`.

**ISSN → Zeitschrift (Gegenprobe):**

```
GET https://services.dnb.de/sru/zdb?version=1.1&operation=searchRetrieve&query=iss%3DXXXX-XXXX&maximumRecords=2&recordSchema=oai_dc
```

**Relevante Antwortfelder (oai_dc):** `dc:title`, `dc:publisher`, `dc:identifier` mit
`tel:ISSN` (ISSN), `dnb:IDN`, `dnb:ZDBID`. Mehrere Stichworte beim `tit=`-Index verwenden —
ein Ein-Wort-Titel wie „Arbeit" allein ist zu unspezifisch.

**Kanonische URL:** `https://d-nb.info/IDN` (aus `dnb:IDN`) — **nicht** auf zdb-katalog.de verlinken,
da der Nutzer dort zwar manuell hinkommt, der Skill den Link aber nie verifizieren kann.

**Kanonische URL:** Der DOI selbst ist der kanonische persistente Identifikator.

- Kanonische URL: `https://doi.org/10.1007/978-3-531-19982-1`
- Link-Format: `[DOI 10.1007/978-3-531-19982-1](https://doi.org/10.1007/978-3-531-19982-1)`

**Wichtige Antwortfelder:** `message.title`, `message.author[].family`, `message.author[].given`, `message.published.date-parts[0][0]`, `message.publisher`, `message.container-title`, `message.DOI`, `message.ISBN`

---

## Semantic Scholar

**Zweck:** Gut für englischsprachige akademische Paper, besonders interdisziplinäre und neuere Literatur.
Als Sekundärcheck nützlich.

> **⚠️ Keyless in geteilten Agent-Umgebungen unbrauchbar (leerer Body, live verifiziert).** Ohne
> `--s2-key` Semantic Scholar **gar nicht** abfragen — der Fehlversuch kostet nur Calls. Mit Key
> (Header `x-api-key:`) normal nutzbar.

**Suche:**

```
GET https://api.semanticscholar.org/graph/v1/paper/search?query=TITLE_KEYWORDS+AUTHOR_LASTNAME&fields=title,authors,year,externalIds,venue,publicationTypes&limit=3
```

**Direkter DOI-Lookup:**

```
GET https://api.semanticscholar.org/graph/v1/paper/DOI:10.xxxx/xxxx?fields=title,authors,year,externalIds,venue
```

**Rate-Limits:** 100 Requests pro 5 Minuten. Bei HTTP 429: 10 Sekunden warten,
einmal erneut. (Mit `--s2-key` als Header `x-api-key:` wird das Limit angehoben; ohne Key in
Agent-Umgebungen nicht nutzbar, siehe oben.)

**Aufbau der kanonischen URL:**

- Antwort enthält `"paperId": "abc123def456..."`.
- Ist ein DOI in `externalIds.DOI` vorhanden, die DOI-Kanonik-URL bevorzugen.
- Sonst: `https://www.semanticscholar.org/paper/[paperId]`.
- Link-Format: `[Semantic Scholar](https://www.semanticscholar.org/paper/abc123def456...)`.

**Wichtige Antwortfelder:** `title`, `authors[].name`, `year`, `venue`, `externalIds.DOI`, `paperId`

---

## DNB SRU (Deutsche Nationalbibliothek)

**Zweck:** Die maßgebliche Quelle für deutschsprachige Monografien, Sammelbände und Dissertationen.
Kostenlos, keine Registrierung nötig.

### ⭐ Standardabfragen — IMMER `recordSchema=oai_dc` (analog K10plus DC)

Das schlanke `oai_dc` liefert alle verifikationsrelevanten Felder (`dc:title`, `dc:creator`,
`dc:publisher`, `dc:date`, `dc:identifier` mit **ISBN + IDN**) in < 20 Zeilen je Treffer — live
verifiziert. Das verbose `MARC21-xml` **nur als Eskalation** (Sparse-then-Full), wenn ein Feld
gebraucht wird, das DC nicht enthält.

**Titel + Autor suchen:**

```
GET https://services.dnb.de/sru/dnb?version=1.1&operation=searchRetrieve&query=tit%3DTITLE_KEYWORD%20and%20per%3DAUTHOR_LASTNAME&recordSchema=oai_dc&maximumRecords=3
```

**Nur nach Titel suchen:**

```
GET https://services.dnb.de/sru/dnb?version=1.1&operation=searchRetrieve&query=tit%3DTITLE_KEYWORD&recordSchema=oai_dc&maximumRecords=3
```

**Nach ISBN suchen:**

```
GET https://services.dnb.de/sru/dnb?version=1.1&operation=searchRetrieve&query=isbn%3DISBN&recordSchema=oai_dc&maximumRecords=2
```

**Titel + Autor + Jahr suchen (Jahresindex `jhr`):**

```
GET https://services.dnb.de/sru/dnb?version=1.1&operation=searchRetrieve&query=tit%3DTITLE_KEYWORD%20and%20per%3DAUTHOR_LASTNAME%20and%20jhr%3DYEAR&recordSchema=oai_dc&maximumRecords=3
```

> **⚠️ Jahresindex der DNB ist `jhr`, NICHT `jah` und NICHT `pub`.** Der Index `jah` ist
> K10plus-spezifisch (PICA); auf der DNB-SRU löst er die Diagnose
> `info:srw/diagnostic/1/16 Unsupported index` aus (vergeudeter Call + Retry). Die korrekte
> DNB-CQL-Syntax ist live verifiziert: `jhr%3D2020` liefert valide Treffer, `jah%3D2020` die
> Diagnose. DNB-Indizes (Auswahl): `tit` (Titel), `per` (Person), `woe` (alle Begriffe,
> Standardindex), `jhr` (Erscheinungsjahr), `isbn`. Groß-/Kleinschreibung der Kürzel ist
> unerheblich. **Sicherheitsnetz:** Liefert die jahresgefilterte Abfrage unerwartet 0 Treffer,
> einmal ohne `jhr` wiederholen und im Ergebnis-Set manuell nach Jahr filtern (Auflagenjahr kann
> abweichen).

> **Schema-Hinweis:** `oai_dc` ist der Standard (siehe oben). Die IDN steht im DC-Schema in
> `dc:identifier` mit `xsi:type="dnb:IDN"`.

**URL-Encoding für deutsche Sonderzeichen:**
| Zeichen | Encoded |
|-----------|---------|
| ä | `%C3%A4` |
| ö | `%C3%B6` |
| ü | `%C3%BC` |
| Ä | `%C3%84` |
| Ö | `%C3%96` |
| Ü | `%C3%9C` |
| ß | `%C3%9F` |
| Leerzeichen | `%20` |
| = | `%3D` |

**Antwort (nur bei MARC21-xml-Eskalation):** Diese Felder parsen:

- `<controlfield tag="001">` → **IDN** (der persistente Identifikator, z. B. `993634095`)
- `<datafield tag="245">` → Titel
- `<datafield tag="100">` oder `<tag="700">` → Autor(en)
- `<datafield tag="264">` → Verlag und Jahr
- `<datafield tag="020">` → ISBN
- `<datafield tag="300">` → Seiten

**Rate-Limits:** Nicht dokumentiert. Bei Fehler: 2 Sekunden warten, einmal erneut.

**Aufbau der kanonischen URL:**

- Die IDN aus `dc:identifier` (`xsi:type="dnb:IDN"`, oai_dc) bzw. `<controlfield tag="001">` (MARC) extrahieren.
- Kanonische URL: `https://d-nb.info/[IDN]`.
- Beispiel: `https://d-nb.info/993634095`.
- Link-Format: `[DNB 993634095](https://d-nb.info/993634095)`.

---

## K10plus SRU (GBV/SWB-Verbundkatalog)

**Zweck:** Verbundkatalog von GBV + SWB — Bestände nahezu aller deutschen wissenschaftlichen
Bibliotheken. Breiter als die DNB (inkl. fremdsprachiger und älterer Titel) und **nicht vom
DNB-Throttling betroffen**. Primärquelle für deutschsprachige Bücher und Sammelbände; starke
Sekundärquelle für alles andere.

**Basis-URL:** `https://sru.k10plus.de/opac-de-627` — **exakt dieser Pfad, nichts anderes.**

> **⚠️ Erfundene Pfade = leerer Body:** `https://sru.k10plus.de/sru` existiert **NICHT**
> (YAZ-404 „Not Found", im Browser sichtbar, via `web_fetch` nur als leerer Body). Die URL **immer**
> aus dieser Datei kopieren, nie rekonstruieren.

### ⭐ Standardabfrage — IMMER zuerst: Dublin Core, maximumRecords=1

MARCXML-Antworten sind extrem verbose (3.000–6.000 Zeichen je Treffer wegen Holdings, Provenienz,
Klassifikationen) und erzwingen Datei-Speichern + mehrere Chunk-Reads — der größte einzelne
Token-/Zeit-Treiber bei Monografie-Recherche. **Beginne jede K10plus-Suche mit dem leichten
Dublin-Core-Schema und nur einem Treffer:**

```
GET https://sru.k10plus.de/opac-de-627?version=1.1&operation=searchRetrieve&query=pica.tit%3DTITLE_KEYWORD%20and%20pica.per%3DAUTHOR_LASTNAME%20and%20pica.jah%3DYEAR&maximumRecords=1&recordSchema=dc
```

Dublin Core liefert alle verifikationsrelevanten Felder in < 20 Zeilen — ohne Holdings/Klassifikation:
`dc:title`, `dc:creator`, `dc:date`, `dc:publisher`, `dc:identifier` (enthält ISSN/ISBN/PPN).

> **⚠️ Feldsemantik — Autor NIE in `pica.tit`:** `pica.tit` durchsucht nur das Titelfeld. Der
> Autorname gehört ausschließlich in `pica.per`. **Falsch:** `pica.tit=Crozier Zwänge` (0 Treffer,
> Buch existiert aber). **Richtig:** `pica.tit=Zwänge kollektiven Handelns and pica.per=Crozier`
> (1. Treffer: PPN 1944372954). 0-Treffer-Fehler entstehen durch Autornamen im Titel-Index, nicht
> durch Lücken in K10plus.

**Eskalationsstufen — nur wenn nötig:**

1. **Erstcheck:** `recordSchema=dc&maximumRecords=1` mit `pica.tit + pica.per + pica.jah` (siehe oben).
2. **Bei 0 Treffern / unklarer Metadatenlage:** zweite Abfrage `recordSchema=dc&maximumRecords=3`
   **ohne** Jahresfilter (`pica.jah` weglassen).
3. **MARCXML nur für Spezialfälle:** `recordSchema=marcxml` ausschließlich, wenn das Inhaltsverzeichnis
   (Feld 520) oder die Auflagenbeschreibung (Feld 250) für eine Sammelbandprüfung explizit gebraucht wird.

Die PPN steht im DC-Schema im `dc:identifier`-Feld (Präfix `ppn:` bzw. `gvk-ppn:`).

### Weitere Abfragemuster (nur bei Bedarf, MARCXML)

**Alle Felder durchsuchen (Eskalation):**

```
GET https://sru.k10plus.de/opac-de-627?version=1.1&operation=searchRetrieve&query=pica.all%3DKEYWORDS&maximumRecords=3&recordSchema=dc
```

**CQL-Index-Schlüssel:** `pica.tit` (Titel) · `pica.per` (Person/Autor) · `pica.jah` (Jahr) ·
`pica.all` (alle Felder). Sortieren mit `&sortKeys=year,,1`. Deutsche Sonderzeichen URL-codiert wie in der DNB-Tabelle oben.

> **⚠️ Nicht unterstützte Indices (führen zu „Query feature unsupported" / „Unsupported index",
> kosten einen Fehl-Call + Retry):**
> 
> - **Nackte CQL-Präfixe `tit=` und `per=` (OHNE `pica.`) sind NICHT unterstützt** → SRU-Diagnostic
>   `info:srw/diagnostic/1/16 Unsupported index`. Der `pica.`-Präfix ist **Pflicht**. Die korrekte
>   Syntax `pica.tit%3D… and pica.per%3D…` ist live verifiziert.
>   Niemals die DNB-Syntax (`tit=`/`per=`) auf K10plus übertragen — die Indices heißen dort anders.
> - `pica.isbn`, `pica.isb`, `pica.mtr` (Medientyp), `pica.edi` (Auflage), `dc.type`, `dc.description`.
>   Ersatz: ISBN über `pica.all%3DISBN` oder Titelsuche; Auflage über `pica.jah` (Erscheinungsjahr der
>   Auflage); Medientyp nicht filterbar (manuell in Ergebnissen prüfen).
>   → Bei Index-Fehlermeldung: Abfrage auf `pica.tit + pica.per + pica.jah` vereinfachen; als letzte
>   Stufe freie Schlüsselwortsuche ganz ohne Index (`query=TITEL_KEYWORDS+AUTOR`) — sie funktioniert
>   immer, ist aber unschärfer.

**Antwort:** Dublin Core (Standard) oder MARCXML (Eskalation, gleiche Parsing-Logik wie DNB). Die Record-ID ist die **PPN**
(MARCXML: `<controlfield tag="001">`; DC: `dc:identifier` mit `ppn:`-Präfix).

**Rate-Limits:** Nicht dokumentiert; in der Praxis robust, **kein DNB-Throttling**. Bei Fehler: 2 Sekunden warten, einmal erneut.

**Aufbau der kanonischen URL:**

- Die PPN aus `<controlfield tag="001">` extrahieren.
- Kanonische URL: `https://opac.k10plus.de/DB=2.1/PPNSET?PPN=PPN`.
- Link-Format: `[K10plus PPN 1234567890](https://opac.k10plus.de/DB=2.1/PPNSET?PPN=1234567890)`.

---

## lobid (hbz-Verbundkatalog)

**Zweck:** Katalog des hbz (NRW-Hochschulbibliotheken). JSON-LD-API, schlüsselfrei. Stark für
deutschsprachige Bücher; auch nützlich zur **Autoren-Disambiguierung via GND**.
Gute Sekundärquelle für deutsche Bücher neben K10plus.

> **⚠️ In geteilten Agent-Umgebungen typischerweise nicht erreichbar:** Die lobid-Daten-API liefert
> dort für **jede** Abfrage einen leeren Body (kein Block, sondern eine Transport-/Encoding-
> Inkompatibilität; vom eigenen Browser funktioniert dieselbe URL). **Erster lobid-Call = Canary:**
> scheitert er, lobid sitzungsweit überspringen — der leere Body ist ein **technischer Fehlschlag**,
> nie ein 0-Ergebnis, und K10plus deckt denselben Buchraum mit höherer Trefferquote ab. Der
> lobid-Weblink bleibt als manueller Prüflink nutzbar.

**Titel + Autor suchen (Lucene-Query-Syntax):**

```
GET https://lobid.org/resources/search?q=title%3ATITLE_KEYWORDS%20AND%20contribution.agent.label%3AAUTHOR_LASTNAME&format=json&size=3
```

**Alle Felder durchsuchen:**

```
GET https://lobid.org/resources/search?q=KEYWORDS&format=json&size=3
```

**Nach ISBN suchen:**

```
GET https://lobid.org/resources/search?q=isbn%3AISBN&format=json&size=3
```

**Antwort:** JSON-LD. `member[]` jeweils mit `id` (der kanonischen URL), `title`,
`contribution[].agent.label` (Autor), `publication[].startDate` (Jahr), `isbn[]`.

**Rate-Limits:** Großzügig, kein Key. Bei Fehler: 2 Sekunden warten, einmal erneut.

**Aufbau der kanonischen URL:**

- Das `id`-Feld ist bereits die kanonische URL, z. B. `https://lobid.org/resources/HT018787636`.
- Link-Format: `[lobid HT018787636](https://lobid.org/resources/HT018787636)`.

---

## Open Library

**Zweck:** Buchkatalog des Internet Archive. Stark für **ISBN-Lookups** internationaler /
englischsprachiger Bücher. Kostenlos. Sekundärcheck, wenn K10plus/DNB ein nicht-deutsches Buch verfehlen.

> **⚠️ Nur der ISBN-Lookup ist in geteilten Agent-Umgebungen zuverlässig.** Die Suche
> `search.json` liefert dort reproduzierbar einen leeren Body (live verifiziert, mehrere
> Query-Varianten); `/isbn/{ISBN}.json` funktioniert (Redirect auf `/books/…` mit validem JSON).
> **Regel:** Open Library nur mit ISBN ansteuern; Titel/Autor-Suchen internationaler Bücher laufen
> über K10plus (DC) — es enthält auch fremdsprachige Bestände. Erster `search.json`-Call = Canary.

**Nach ISBN suchen (⭐ Standard):**

```
GET https://openlibrary.org/isbn/ISBN.json
```

**Titel + Autor suchen (nur nach bestandenem Canary):**

```
GET https://openlibrary.org/search.json?title=TITLE_KEYWORDS&author=AUTHOR_LASTNAME&limit=3&fields=title,author_name,first_publish_year,key,isbn,publisher
```

**Antwort:** JSON. Suche: `docs[]` jeweils mit `title`, `author_name[]`, `first_publish_year`, `key`
(z. B. `/works/OL12345W`), `isbn[]`, `publisher[]`. ISBN-Lookup: Edition-Objekt mit `title`,
`publishers[]`, `publish_date`, `key` (z. B. `/books/OL12345M`), `works[].key`.

**Rate-Limits:** Großzügig, kein Key. Bei Fehler: 2 Sekunden warten, einmal erneut.

**Aufbau der kanonischen URL:**

- Das `key`-Feld nutzen → `https://openlibrary.org` + key.
- Link-Format: `[Open Library OL12345W](https://openlibrary.org/works/OL12345W)`.

---

## arXiv

**Zweck:** Preprints in Physik, Informatik, Mathematik, Statistik, quantitativer Biologie
**und Wirtschaft** (`econ.*`). Für STEM-Kaskaden und jede Quelle, die nach Preprint aussieht.

> **⚠️ In geteilten Agent-Umgebungen typischerweise nicht erreichbar** (leerer Body, live über
> http **und** https verifiziert). Erster arXiv-Call = Canary; scheitert er, arXiv sitzungsweit
> überspringen. **Ausweichweg:** arXiv-Preprints haben durchgängig registrierte DOIs
> (`10.48550/arXiv.<ID>`) — über Crossref `filter=doi:` bzw. OpenAlex auflösen; auch die
> Titel/Autor-Suche nach Preprints deckt OpenAlex/Crossref ab.

**Titel + Autor suchen:**

```
GET http://export.arxiv.org/api/query?search_query=ti:TITLE_KEYWORDS+AND+au:AUTHOR_LASTNAME&max_results=3
```

Boolesche Operatoren (`AND`, `OR`, `ANDNOT`) **müssen großgeschrieben** sein. Feldpräfixe: `ti:` Titel, `au:` Autor, `all:` alle.

**Direkter ID-Lookup:**

```
GET http://export.arxiv.org/api/query?id_list=ARXIV_ID
```

**Antwort:** Atom-XML. `<entry>` parsen: `<title>`, `<author><name>`, `<published>` (Jahr),
`<id>` (die abs-URL), `<arxiv:doi>` falls vorhanden.

**Rate-Limits:** **Mindestens 3 Sekunden Abstand zwischen Requests** (erzwungen). Bei Fehler: 3 Sekunden warten, einmal erneut.

**Aufbau der kanonischen URL:**

- Das `<id>`-Element ist bereits die kanonische URL: `https://arxiv.org/abs/2401.01234`.
- Link-Format: `[arXiv 2401.01234](https://arxiv.org/abs/2401.01234)`.

---

## Library of Congress (SRU-Katalog)

**Zweck:** US-/englischsprachige Monografien. Nachrangig — nutzen, wenn Open Library verfehlt.

**Wichtig:** Die bequeme JSON-API (`loc.gov/search/?q=...&fo=json`) liefert **nur digitalisierte
Objekte, keine Katalogdaten**. Für die Buchprüfung den **SRU-Katalog-Endpoint** nutzen:

```
GET http://lx2.loc.gov:210/lcdb?version=1.1&operation=searchRetrieve&query=bath.title%3DTITLE_KEYWORDS%20and%20bath.author%3DAUTHOR_LASTNAME&maximumRecords=3&recordSchema=mods
```

**CQL-Indices:** `bath.title`, `bath.author`, `bath.isbn`, `dc.title`. Record-Schema `mods` oder `marcxml`.

**Antwort:** MODS/MARC-XML. Die **LCCN** extrahieren (z. B. `2004012345`).

**Rate-Limits:** Erzwungen, aber undokumentiert; sparsam abfragen. Bei Fehler: 3 Sekunden warten, einmal erneut.
Experimenteller behandeln als die anderen Kataloge — schlägt es zweimal fehl, auf einen vorbereiteten
LoC-Suchlink zurückfallen und weitergehen.

**Aufbau der kanonischen URL:**

- Kanonische URL über LCCN: `https://lccn.loc.gov/LCCN`.
- Link-Format: `[LoC 2004012345](https://lccn.loc.gov/2004012345)`.

---

## Nicht per web_fetch abrufbare Verlagsseiten (JavaScript-gerendert)

Wissenschaftliche Verlage rendern ihre Journal- und Artikelseiten client-seitig per JavaScript.
`web_fetch` liefert dann nur einen leeren HTML-Shell — kein Inhalt, aber ein vergeudeter Call plus
Verzögerung. **Kein Direktabruf-Versuch bei diesen Domains.** Stattdessen direkt zu Crossref
eskalieren (DOI-Direktauflösung oder ISSN-Filter, siehe Crossref-Abschnitt):

| Verlag           | Domain                  | Alternative                       |
| ---------------- | ----------------------- | --------------------------------- |
| De Gruyter       | degruyter.com           | Crossref mit DOI oder ISSN-Filter |
| Wiley            | onlinelibrary.wiley.com | Crossref / DOI-Auflösung          |
| Taylor & Francis | tandfonline.com         | Crossref / DOI-Auflösung          |
| SAGE             | journals.sagepub.com    | Crossref / DOI-Auflösung          |
| Elsevier         | sciencedirect.com       | Crossref / DOI-Auflösung          |
| Springer         | link.springer.com       | Crossref / DOI-Auflösung          |

**Ausnahme Springer:** `link.springer.com` liefert für **einzelne Artikel-DOIs**
(`/article/10.1007/...`) oft brauchbare Metadaten auch ohne JavaScript. Zeitschriften-Inhaltsseiten
(Heft-/Bandübersichten) sind aber auch hier JS-gerendert → Crossref nutzen.

**Eskalation, falls eine JS-Seite wirklich der einzige Weg ist** (selten, z. B. eine Heft-Inhaltsseite
zur Prüfung eines Artikels ohne DOI/ISSN-Treffer): **nicht** `web_fetch` (liefert nur den leeren Shell),
sondern die **Claude-in-Chrome-Tools** (`navigate` + `get_page_text`), die JavaScript rendern. Davor
aber immer erst den ISSN-Lookup → Crossref-Weg ausschöpfen.

---

## BASE (Bielefeld Academic Search Engine) — NUR mit API-Key

**Zweck & Rolle:** OA-/Repositorien-Aggregator (470 Mio. Dokumente aus 12.000+ Quellen, via OAI-PMH
geerntet). Stark für **graue Literatur, Hochschulschriften/Dissertationen, Working Paper, Konferenz-
und Open-Access-Inhalte**. **Kein Buch-/Verlagskatalog:** klassische Monografien und
Verlags-Zeitschriftenartikel fehlen oft (Live-Beleg: ein real existierendes LIT-Verlag-Buch war nicht
auffindbar). Daher **ergänzend** im Grey-Literature-/Thesis-/OA-Zweig einsetzen, **nicht** als Ersatz
für K10plus/DNB/Crossref/OpenAlex.

### ⛔ Zugang & hartes Rate-Limit (zwingend)

- **Nur mit API-Key** (`apikey=`-Parameter; kostenlose, nicht-kommerzielle Registrierung). **Ohne Key
  gar nicht abfragen.**
- **Max. 1 Anfrage/Sekunde** — bei Verstoß droht **Key-Entzug ohne Vorwarnung**. Da ein Modell keine
  Sekunden abwartet, gilt im Skill die Handlungsregel: **höchstens EIN BASE-Aufruf pro Antwortschritt,
  strikt seriell, nie gebündelt/parallel** (der Mindestabstand von ~1,5 s ist nur die Begründung des
  Limits; siehe SKILL.md → „BASE — hartes Rate-Limit").
- **Zugangshürde in Agent-Umgebungen:** Aus geteilten Proxy-Umgebungen (z. B. die hiesige
  `web_fetch`-Sandbox) blockt BASE die Anfragen (UA/IP-Filter, Fehlermeldung „Access denied for IP
  address … and user agent …") → **leerer Body**. Vom eigenen Rechner/Browser (normale IP + UA)
  funktioniert derselbe Key. Der **BASE-Canary** fängt das ab: leerer Body → BASE sitzungsweit aus.
- **Sicherheit:** `apikey=` ist ein Geheimnis — nie in Berichte, Logs, Zwischenstands-Dateien oder
  angezeigte URLs schreiben; vor jeder URL-Wiedergabe `apikey=…` entfernen.

**Basis-URL:** `https://api.base-search.net/cgi-bin/BaseHttpSearchInterface.fcgi`
(⚠️ **`api.`**-Host, **nicht** `www.base-search.net` — Letzterer ist die Web-Oberfläche, keine API.)

### BASE-Canary (Step 0 / vor erstem BASE-Call)

```
GET https://api.base-search.net/cgi-bin/BaseHttpSearchInterface.fcgi?func=PerformSearch&query=canary-JJJJMMTT-hhmm&hits=1&format=json&apikey=USER_KEY
```

Sitzungseindeutiges Suchwort (Datum/Uhrzeit) statt eines konstanten Testworts — gecachte Antworten
könnten sonst einen Block verdecken. Wohlgeformtes JSON mit `response`-Objekt (`numFound` darf 0
sein) → BASE aktiv. Leerer Body / `<error>` → BASE für die Sitzung deaktivieren.

### Suche (PerformSearch, JSON)

**Titel + Autor (feldgenau, empfohlen):**

```
GET https://api.base-search.net/cgi-bin/BaseHttpSearchInterface.fcgi?func=PerformSearch&query=dctitle:TITEL_KEYWORDS+AND+dccreator:AUTOR_NACHNAME&hits=3&format=json&apikey=USER_KEY
```

**Freie Stichwortsuche (Fallback):**

```
GET https://api.base-search.net/cgi-bin/BaseHttpSearchInterface.fcgi?func=PerformSearch&query=TITEL_KEYWORDS+AUTOR_NACHNAME&hits=3&format=json&apikey=USER_KEY
```

- **Suchfelder:** `dctitle` (Titel), `dccreator` (Autor), `dcyear`/`dcdate`. Boolesche Operatoren
  `AND`/`OR`/`NOT` großschreiben; Leerzeichen als `+`.
- **⚠️ KEINE ISBN-Suche:** BASE indexiert ISBNs nicht durchsuchbar (Live-Test `query=9783825883248`
  → 0 Treffer, obwohl das Buch existiert). ISBN nicht als BASE-Query verwenden.
- `hits` = 2–3 (max. 120), `format=json`. Standardmäßig UND-Verknüpfung aller Begriffe.

### Antwort (JSON) parsen

Struktur (Solr-Stil): `response.numFound` (Trefferzahl) und `response.docs[]`. Relevante Doc-Felder:

- `dctitle` — Titel · `dccreator` (Array) — Autor(en) · `dcyear` / `dcdate` — Jahr
- `dcpublisher` (Array) — Verlag/Institution · `dctypenorm` (Array) — Dokumenttyp-Code
  (11 Buch, 111 Buchteil, 121 Artikel, 13 Konferenz, 14 Report, 16 Kursmaterial, 18/181/182/183 Thesis)
- `dcdoi` — DOI (falls vorhanden) · `dcoa` — Open-Access-Flag · `dcdocid` — interne ID
- `dclink` — bevorzugte Quell-URL · `dcidentifier` (Array) — alle URLs/Identifier

**Abgleich:** `dctitle` + `dccreator` müssen zur Zitation passen; `dcyear` mit dem zitierten Jahr
vergleichen. `numFound:0` ist ein **valides 0-Ergebnis** nur im Rahmen der BASE-Rolle (Grey/Thesis/OA)
— für Monografien/Artikel zählt ein BASE-Miss **nicht** Richtung 🔴 IV (siehe SKILL.md → Klassifikations-Sperre).

### Aufbau der kanonischen URL

- `dclink` ist die bevorzugte, anklickbare Quell-URL (z. B. Repositoriums-Seite).
- Link-Format: `[BASE: <Repositorium>](DCLINK)` — alternativ die erste `dcidentifier`-URL.
- (Eine interne BASE-Detailseite je `dcdocid` gibt es nicht stabil öffentlich; daher `dclink` nutzen.)

**Rate-Limits:** **1 qps (hart)** — im Skill: höchstens ein Aufruf pro Antwortschritt, strikt seriell,
nie parallel (1,5 s nur als Begründung). Bei leerem Body: kein Retry-Sturm; als
Zugangsblock werten (Canary deaktiviert BASE bereits sitzungsweit).

---

## Vorbereitete Suchlinks (keine Live-Abfrage — an den Nutzer übergeben)

Diese werden **nicht von der Skill abgerufen**. Sie werden als anklickfertige Links für den
„Belege & Nachschlagen"-Block gebaut, damit der Nutzer manuell nachprüfen kann. Den Query-String aus
`Autor-Nachname(n) + 2–3 Titel-Keywords + Jahr` bauen, Leerzeichen als `+`.

| Dienst             | Link-Muster                                                |
| ------------------ | ---------------------------------------------------------- |
| **Google Scholar** | `https://scholar.google.com/scholar?q=QUERY`               |
| **WorldCat**       | `https://search.worldcat.org/search?q=QUERY`               |
| **BASE**           | `https://www.base-search.net/Search/Results?lookfor=QUERY` |

> **Hinweis zu BASE:** BASE wird **live abgefragt, sobald ein API-Key vorliegt** (siehe Abschnitt
> „BASE" oben — key-gated, Canary, hartes 1,5-s-Limit). Der hier gelistete `www.base-search.net`-Link
> ist die **Web-Oberfläche** und bleibt zusätzlich als manueller Suchlink im Bericht — nützlich, wenn
> kein Key vorliegt oder BASE in der jeweiligen Umgebung geblockt ist.

### WebSearch — nur letztes Mittel, strikt eingegrenzt

`WebSearch` ist **keine** kuratierte Bibliotheksdatenbank und kein regulärer Kaskadenschritt. Einsatz
nur, wenn die API-Kaskade **valide** ausgeschöpft ist (echte 0-Ergebnisse, kein technischer Fehlschlag)
oder alle erreichbaren APIs technisch ausgefallen sind. Dann mit Disziplin:

- **`allowed_domains` auf akademische/katalogische Quellen beschränken**, z. B.:
  `doi.org, openalex.org, api.crossref.org, d-nb.info, portal.dnb.de, k10plus.de, opac.k10plus.de,
  lobid.org, search.gesis.org, econbiz.de, base-search.net, worldcat.org, scholar.google.com,
  semanticscholar.org, jstor.org, ssrn.com` sowie die Verlags-DOI-Hosts
  (springer.com, tandfonline.com, sagepub.com, degruyter.com, wiley.com, nomos-elibrary.de).
- **Nie alleiniger Beleg.** WebSearch-Treffer dürfen 🟢/🟡/🟠 **nicht** allein stützen — dafür ist immer
  ein Katalog-/DOI-Nachweis (DNB, K10plus, Crossref/DOI, OpenAlex, lobid …) nötig. WebSearch darf nur
  **hinführen** oder **bestätigen**.
- **Kommerz/Buchhandel und Social-Networks sind kein Nachweis:** Amazon, ZVAB, booklooker, AbeBooks,
  Eurobuch, ResearchGate, Academia.edu — **nicht** als Beleg verwenden (höchstens als Existenz-Indiz,
  nie als Quelle in der Ergebnistabelle). Per `allowed_domains` faktisch ausschließen.

---

## ZBW EconBiz

**Zweck:** Spezialisiert auf Wirtschafts- und Managementliteratur. Starke Abdeckung deutschsprachiger
wirtschaftswissenschaftlicher Publikationen und Working Paper.

> **⚠️ In geteilten Agent-Umgebungen typischerweise nicht erreichbar** (leerer Body, live
> verifiziert). Erster EconBiz-Call = Canary; scheitert er, sitzungsweit überspringen und die
> Wirtschafts-Kaskade über OpenAlex/Crossref/K10plus fahren. Als manueller Prüflink:
> `https://www.econbiz.de/Search/Results?lookfor=QUERY`.

**Suche:**

```
GET https://api.econbiz.de/api/v1/search?q=TITLE_KEYWORDS+AUTHOR_LASTNAME&hits=3
```

**Wirtschaftsspezifisch suchen:**

```
GET https://api.econbiz.de/api/v1/search?q=TITLE_KEYWORDS&type=article&hits=3
```

**Rate-Limits:** Nicht dokumentiert. Kein Key erforderlich. Bei Fehler: 2 Sekunden warten, einmal erneut.

**Aufbau der kanonischen URL:**

- Antwort enthält ein `"link"`-Feld mit der Record-URL, z. B. `https://www.econbiz.de/Record/10011234567`.
- Link-Format: `[EconBiz 10011234567](https://www.econbiz.de/Record/10011234567)`.

**Wichtige Antwortfelder:** `title`, `creators[].person.name`, `date`, `publisher`, `source`, `link`, `id`

---

## GESIS (Sozialwissenschaften)

**Zweck:** Spezialisiert auf deutschsprachige sozialwissenschaftliche Literatur (Soziologie,
Politikwissenschaft). Deckt ältere und graue Literatur ab, die nicht in OpenAlex steht.

**Keine öffentliche Such-API.** Stattdessen Websuche nutzen:

**Websuche-Ansatz:**

```
Web search: site:search.gesis.org "TITLE_KEYWORDS" AUTHOR_LASTNAME
```

oder

```
Web search: GESIS SOLIS "TITLE_KEYWORDS" AUTHOR_LASTNAME
```

**Kanonisches URL-Muster:** `https://search.gesis.org/publication/gesis-solis-XXXXXXXX`

- Die vollständige URL aus den Websuche-Ergebnissen extrahieren.
- Diese URL ist durch Anklicken direkt überprüfbar.
- Link-Format: `[GESIS gesis-solis-00178524](https://search.gesis.org/publication/gesis-solis-00178524)`.

**Hinweis:** GESIS ist langsamer zu finden als API-basierte Datenbanken. Als Fallback für deutsche
sozialwissenschaftliche Artikel nutzen, die OpenAlex nicht indexiert, besonders Literatur vor 2000.

---

## Rate-Limit-Übersicht

| Datenbank                 | Dokumentiertes Limit                                  | Praktisches Verhalten                                                            | Warten bei 429                                    |
| ------------------------- | ----------------------------------------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------- |
| OpenAlex                  | Keyless ~$0,01/Tag (klein) · mit Key $1/Tag · ~10/sec | Keyless-Limit kann in langen Listen erschöpfen → leerer Body/429; Key behebt das | 3 Sekunden (429: Tageslimit, ggf. Crossref)       |
| Crossref                  | Nicht dokumentiert (Polite Pool)                      | Großzügig                                                                        | 2 Sekunden                                        |
| Semantic Scholar          | 100 / 5 min                                           | Strikt; keyless in Agent-Umgebungen leerer Body → nur mit `--s2-key`            | 10 Sekunden                                       |
| DNB SRU                   | **Nicht dokumentiert**                                | Throttelt nach ~30–50 schnellen Calls; temporäres IP-Blocking                    | **3 Sekunden Mindestabstand zwischen Calls**      |
| K10plus SRU               | Nicht dokumentiert                                    | Robust, selten Throttle                                                          | 2 Sekunden                                        |
| Open Library              | Großzügig                                             | `/isbn/` robust; `search.json` in Agent-Umgebungen leerer Body (Canary)         | 2 Sekunden                                        |
| arXiv                     | **Mind. 3 Sek. zwischen Calls**                       | In Agent-Umgebungen oft leerer Body (Canary; DOI-Weg via Crossref)              | 3 Sekunden                                        |
| Library of Congress SRU   | Nicht dokumentiert                                    | Empfindlich; sparsam abfragen                                                    | 3 Sekunden                                        |
| ZBW EconBiz               | Nicht dokumentiert                                    | In Agent-Umgebungen oft leerer Body (Canary)                                    | 2 Sekunden                                        |
| ZDB SRU (services.dnb.de) | Nicht dokumentiert                                    | Wie DNB SRU behandeln (gleiche Infrastruktur)                                    | 3 Sekunden Mindestabstand                         |
| GESIS                     | Via Websuche                                          | Hängt von Suchmaschine ab                                                        | —                                                 |
| BASE (nur mit Key)        | **1 qps (hart!)**                                     | Key-Entzug bei Verstoß; in Agent-Umgebungen oft UA/IP-geblockt (leerer Body)     | **ein Aufruf/Schritt, strikt seriell, nie parallel** (1,5 s = Begründung) |

Liefert eine Datenbank dauerhaft Fehler (3+ Fehlschläge), sie im Transparenz-Log als nicht erreichbar
vermerken und für den Rest der Sitzung überspringen.

## DNB: Besondere Hinweise zu Rate-Limits und Proof-of-Work-Links

Die DNB SRU API drosselt in der Praxis IPs, die viele schnelle Anfragen stellen.

**Throttling vermeiden (Handlungsregeln):**

- DNB-Calls über mehrere Antwortschritte strecken, nicht bündeln; mit Calls anderer DBs abwechseln.
- Bei langen Listen (15+ Quellen) DNB nur dort einsetzen, wo die günstigeren Kataloge verfehlen.
- Bei HTTP 429: nicht sofort, sondern erst in einem späteren Schritt einmal erneut.

**Dual-URL-Regel (operativ):** Wird eine Quelle in DNB **und** OpenAlex gefunden, beide kanonischen URLs
in den Beleg aufnehmen — OpenAlex-Links unterliegen keinem DNB-Throttling. Format:
`[DNB 993634095](https://d-nb.info/993634095) · [OpenAlex W...](https://openalex.org/works/W...)`
