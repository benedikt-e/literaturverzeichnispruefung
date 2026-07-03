# Kategorienregeln — Detailreferenz

## Kategoriendefinitionen

### 🟢 I — Fehlerfrei (nahezu perfekt)

Die Quelle existiert exakt wie zitiert. Akzeptable triviale Abweichungen:

- fehlende oder zusätzliche Zeichensetzung (Komma, Punkt, Doppelpunkt nach dem Titel),
- geringe Groß-/Kleinschreibungs-Unterschiede (z. B. „und" vs. „Und"),
- fehlendes „and"/„und" zwischen den letzten beiden Autoren einer langen Autorenliste,
- DOI oder URL im Original vorhanden, im Datensatz nicht (oder umgekehrt),
- geringe Leerzeichen-Unterschiede.

**NICHT als I einordnen, wenn:**

- das Jahr abweicht — **auch dann nicht**, wenn der Treffer „nur eine Neuauflage" ist. Eine gefundene
  neuere Auflage macht die Jahresabweichung **nicht** trivial: Solange das zitierte (ältere) Jahr nicht
  **separat als eigene Ausgabe** nachgewiesen wurde, ist die Quelle **🟡 II** mit Hinweis „nur Neuauflage
  [Jahr] nachgewiesen". 🟢 I nur, wenn genau die zitierte Ausgabe/das zitierte Jahr im Katalog bestätigt ist.
- der Haupttitel wesentlich abweicht,
- ein Autoren-Nachname falsch geschrieben ist,
- der Verlag falsch ist.

**Konsistenzregel:** 🟢 I und ein nicht-leeres Abweichungsfeld schließen sich aus. Steht in „Art der
Abweichungen" irgendeine Abweichung, ist die Kategorie mindestens 🟡 II.

---

### 🟡 II — Kleine Fehler

Die Quelle existiert im zitierten Publikationsformat, hat aber geringe Abweichungen:

- **fehlender Untertitel** (Haupttitel passt; Untertitel fehlt oder weicht ab),
- **Jahr um 1 daneben**, plausibel durch Neuauflage oder Nachdruck erklärbar — explizit vermerken,
- **fehlende Auflagenangabe** (z. B. „3. Aufl." nicht genannt),
- **fehlende „Hrsg."/„(Ed.)"-Kennzeichnung** für als Autoren zitierte Herausgeber,
- **fehlende Seitenangabe** bei einem Buchkapitel,
- **abgekürzter vs. vollständiger Zeitschriftenname** (KZSS vs. Kölner Zeitschrift für Soziologie und Sozialpsychologie),
- **verifizierter Herausgeberband** für ein nicht indexiertes Buchkapitel (siehe Sammelbände-Regel unten),
- **geringe Verlagsnamen-Variante** (z. B. „Springer VS" vs. „VS Verlag für Sozialwissenschaften").

---

### 🟠 III — Gravierende Fehler

Die Quelle existiert im richtigen Publikationsformat, aber wesentliche Metadaten sind klar und deutlich falsch:

- **Jahr um mehr als 1 daneben** (nicht durch Nachdruck/Neuauflage erklärbar),
- **Verlag völlig falsch** (z. B. Springer statt Campus zitiert),
- **Zeitschriftenname falsch** (andere Zeitschrift, nicht bloß eine Abkürzung),
- **Seitenzahlen grob falsch** (um mehr als ~10 % abweichend oder völlig anderer Bereich),
- **Erscheinungsort falsch** bei einem Buch,
- **falscher Band/falsches Heft** bei einem Zeitschriftenartikel.

---

### 🔴 IV — Möglicherweise halluziniert

**Der Wortlaut zählt:** Diese Kategorie ist eine **Warnung, kein Urteil mit Gewissheit**. Die
Abwesenheit aus Datenbanken macht eine erfundene Quelle *wahrscheinlich*, beweist sie aber nicht — eine
echte Quelle kann schlicht in den abgefragten Katalogen fehlen. Den Befund stets als „möglicherweise
halluziniert" / „kein Nachweis gefunden" formulieren, nie als definitives „ist erfunden".

**Vorbedingung (Klassifikations-Gate):** 🔴 IV ist nur zulässig, nachdem (a) mindestens eine Datenbank
ein *valides* 0-Ergebnis auf eine feldsemantisch korrekte Query lieferte und (b) die obligatorische
Gegenprobe (freie K10plus-Schlüsselwortsuche) ebenfalls 0 Treffer ergab. Ohne valide DB-Antwort →
„technisch nicht prüfbar", nicht IV. Nicht gefundenes FernUni-Material → 🎓 FU, nicht IV.

🔴 IV vergeben, wenn keine Quelle der Kombination entspricht aus:

- Autor(en) mit diesen Nachnamen
  **UND**
- diesem Haupttitel (oder einer sehr nahen Variante)
  **UND**
- diesem Publikationstyp (Artikel vs. Buch vs. Kapitel).

**Nie über „Plausibilität" erreichbar:** „Zeitschrift/Verlag existiert", „Jahrgang plausibel",
„Klassiker" usw. stufen ein nicht gefundenes Werk **nicht** auf 🟡 II herab — eine nicht gefundene
reguläre Publikation ist 🔴 IV.

**ISSN-Coverage-Gegenprobe bei Zeitschriftenartikeln (stützt 🔴 IV):** Existiert die Zeitschrift und
ist sie in Crossref **substanziell abgedeckt** (`/journals/{ISSN}` → `counts.total-dois` ≥ 100 bzw.
≥ 100 Treffer beim reinen `filter=issn:`-Lauf), bleibt aber der konkrete Artikel über
`query.bibliographic` **und** `query.author` unauffindbar, dann ist die Nicht-Auffindbarkeit ein
**valides 0-Ergebnis für das Werk** und stützt 🔴 IV (Hinweis: Artikel über Verlags-Direktzugang
prüfen). **Nur bei positiver Coverage** — bei schwach indexierten Zeitschriften (< 100) zählt ein
Crossref-Miss **nicht** Richtung IV; dann genügt der ZDB-Existenznachweis der Zeitschrift, der Artikel
bleibt mit Hinweis offen. (Technik: api-endpoints.md → Crossref → ISSN-Coverage-Heuristik.)

**Wichtige Nuancen:**

- Existiert ein *Buch* mit diesem Titel, das Zitat formatiert es aber als *Zeitschriftenartikel* → trotzdem **IV**, aber den Near-Match vermerken.
- Existieren die Autor(en) und haben sie im zitierten Jahr in/bei der zitierten Zeitschrift/dem Verlag publiziert, aber zu einem anderen Thema → **IV**, aber vermerken: *„Autor:innen haben in [Jahr] andere Publikationen in/bei [Quelle] veröffentlicht"*.
- Findest du ein Werk mit gleichem Titel, aber anderen Autoren → **IV**, Near-Match vermerken.

**Bei IV:** Den nächstliegenden Near-Match (falls vorhanden) mit kanonischer URL in „Gefundene Quelle" schreiben. Wurde gar nichts gefunden, in „Gefundene Quelle" „Nicht gefunden in: …" mit den tatsächlich durchsuchten Datenbanken vermerken (vollständige Linkliste im „Belege & Nachschlagen"-Block).

---

## Sonderregeln

### Sammelbände / Buchkapitel (In-Collection-Regel)

Bibliografische Datenbanken indexieren routinemäßig den Host-Band (das herausgegebene Buch), aber nicht die einzelnen Kapitel darin. Das ist eine Datenbank-Beschränkung, kein Zitierfehler.

**Diese Regel anwenden, wenn ALLES Folgende zutrifft:**

1. Die zitierte Quelle ist ein Kapitel in einem Herausgeberband (Format: *„In: [Herausgeber] (Hrsg.): [Bandtitel]. [Ort]: [Verlag], S. [Seiten]"*).
2. Das spezifische Kapitel ist nicht als eigenständiger Datensatz auffindbar.
3. Der Host-Band ist verifizierbar: Herausgeber, Bandtitel, Verlag und Jahr sind alle in einer Datenbank bestätigt.

**→ Als 🟡 II einordnen**

In „Art der Abweichungen" schreiben:

> Herausgeberband verifiziert ([Herausgeber, Jahr]), Einzelkapitel maschinell nicht indexiert.

In „Gefundene Quelle" die Daten des verifizierten Host-Bandes schreiben (nicht die Kapiteldaten).

**NICHT als IV einordnen**, nur weil das Kapitel nicht als eigenständiger Datensatz erscheint.

Lässt sich nicht einmal der Host-Band verifizieren → zur vollen IV-Prüfung übergehen.

---

### DOI-Reverse-Verifikation

Enthält das Original einen DOI, ist das der zuverlässigste Verifikationspfad — und das zuverlässigste Halluzinationssignal.

**Einen zitierten DOI immer auflösen** (Standard, funktioniert ohne Request-Header und ohne Key):

```
GET https://api.crossref.org/works?filter=doi:[DOI]&select=DOI,title,author,published,container-title,publisher,type&rows=1
```

Nur wenn das Fetch-Tool Request-Header setzen kann, alternativ `GET https://doi.org/[DOI]` mit
`Accept: application/vnd.citationstyles.csl+json` (Fallback `Accept: application/json`). Ein bei
Crossref unbekannter DOI (kein Treffer im `filter=doi:`-Ergebnis) wird vor dem Urteil „löst nicht
auf" über OpenAlex geprüft (DataCite-DOIs). Details: api-endpoints.md → Crossref.

**Den zurückgegebenen Titel und die Autoren mit dem Originalzitat vergleichen.**

- Passen sie → starke Bestätigung, weiter zu 🟢 I oder 🟡 II.
- Passen sie nicht (gleicher DOI, anderes Werk) → **🔴 IV** — der DOI ist halluziniert oder neu vergeben, selbst wenn das Werk unter einem anderen DOI existiert.
- Löst der DOI nicht auf (HTTP 404) → vermerken, weitere Datenbanken durchsuchen.

---

### Near-Matches

Wenn du etwas *fast* Passendes findest — gleicher Autor, anderer Titel, oder gleicher Titel, anderer Autor — sorgfältig dokumentieren:

- Trotzdem als **🔴 IV** einordnen.
- In „Gefundene Quelle": den Near-Match mit seiner kanonischen URL.
- In „Art der Abweichungen": die konkrete Diskrepanz beschreiben, z. B. *„Autor Müller hat 2018 ein ähnliches Werk veröffentlicht: [tatsächlicher Titel]"*.

Das hilft dem Leser zu erkennen, ob die Quelle mit einer Korrektur rettbar ist.

---

## Studienbriefe und Lerneinheiten (FernUni-Kontext)

*Automatisch ausgelöst, sobald ein FernUni-Signal erkannt wird (Verlag/Ort „Hagen: FernUniversität",
Studienbrief-/Lerneinheitsformat); `--kontext fernuni` nur als expliziter Override.*

### Was sie sind

Studienbriefe bzw. Lerneinheiten sind akademische Kursmaterialien der Fernuniversität. Sie sind häufig **nicht** in akademischen Standarddatenbanken (DNB, OpenAlex, Crossref) katalogisiert. Das ist eine strukturelle Beschränkung — Abwesenheit aus Datenbanken bedeutet **nicht**, dass die Quelle halluziniert ist.

**Logik:** FernUni-Material wird **nicht** vorab auf eine Kategorie gesetzt. Eine Überprüfung wird
**immer versucht** (Prozedur unten). Erst das Ergebnis entscheidet: gefunden → regulär 🟢/🟡/🟠; trotz
Suchversuch nicht auffindbar → **🎓 FU** (FernUni-Lernmaterial, nicht überprüfbar), **niemals 🔴 IV**.

Typische Zitierformate:

- als eigenständiger Studienbrief: `[Autor]. ([Jahr]). [Titel]. Hagen: FernUniversität in Hagen.`
- als Kapitel innerhalb eines Studienbriefs: `[Autor]. ([Jahr]). [Kapiteltitel]. In: [Herausgeber] (Hrsg.): [Studienbrieftitel]. Hagen: FernUniversität, S. [Seiten].`

### Wichtig: das „Doppeljahr"-Problem

FernUni-Kapitel haben oft zwei relevante Jahre:

1. das **Originalerscheinungsjahr** des Textes (Erstveröffentlichung anderswo bzw. Abfassung),
2. das **Studienbrief-Auflagenjahr** (erster Einsatz in der Lehre).

KI-Schreibwerkzeuge halluzinieren häufig eines oder beide dieser Jahre. Immer prüfen:

- Passt das zitierte Jahr zu einem plausiblen Originalerscheinungsjahr für diesen Autor und dieses Thema?
- Passt das zitierte Jahr zu einem plausiblen Studienbrief-Auflagenjahr?
- Passt keines → als Fehler markieren, auch wenn der Quellentyp selbst unprüfbar ist.

### Verifikationsprozedur

1. **K10plus und DNB SRU** nach dem Studienbrief per Titel + Verlag „FernUniversität" durchsuchen. Varianten probieren.
2. **Suche nach einer kommerziell verlegten Buchausgabe (PFLICHT).** FernUni-Lerneinheiten werden sehr
   oft **zusätzlich als reguläres Lehrbuch verlegt** (UTB, Nomos, Springer VS, Kohlhammer …), häufig
   unter leicht anderem Titel und anderem Jahr/Auflage. **Immer** eine freie Schlagwortsuche
   `pica.all=KERNTITEL_KEYWORDS + AUTOR_NACHNAME` in K10plus (und OpenAlex/Crossref) **ohne** den Zusatz
   „FernUniversität" absetzen. Das ist zugleich die obligatorische Gegenprobe (siehe unten). Der Titel
   der verlegten Buchausgabe weicht dabei häufig vom Lerneinheits-Titel ab — daher breit über
   Kern-Keywords + Autor suchen, nicht über den exakten Lerneinheits-Titel. Wird die Buchausgabe
   gefunden, **existiert** die Quelle → nicht 🔴 IV.
3. **GESIS** per Websuche durchsuchen: `site:search.gesis.org "TITEL_KEYWORDS" FernUniversität`.
4. **Nach dem Originaltext suchen**, falls das Kapitel ein Nachdruck ist: Autor + wahrscheinliches Originaljahr in OpenAlex/Crossref probieren. Viele Studienbrief-Kapitel sind Nachdrucke früherer Veröffentlichungen — findest du das Original, kannst du prüfen, ob das zitierte Jahr plausibel ist.
5. **Erst wenn ALLES Obige — inkl. der freien Buchausgaben-Gegenprobe (Schritt 2) — nichts liefert:**
   als **🎓 FU** (FernUni-Lernmaterial, nicht überprüfbar) einordnen — **nicht 🔴 IV**. Die tatsächlich
   versuchten Suchen (mit Links) im „Belege & Nachschlagen"-Block festhalten.

### Klassifikation

| Situation                                                                                                             | Kategorie                                                                       |
| --------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Lerneinheit in DNB/K10plus gefunden, Jahr und Details passen                                                          | 🟢 I oder 🟡 II                                                                 |
| Kommerziell verlegte Buchausgabe gefunden (ggf. anderer Titel/anderes Jahr)                                           | 🟢 I / 🟡 II / 🟠 III je nach Metadatenabgleich; Ausgabe vermerken. NICHT 🎓 FU |
| Studienbrief gefunden, Jahr klar falsch                                                                               | 🟠 III                                                                          |
| NICHT in einer Datenbank UND keine verlegte Ausgabe gefunden, aber Originalveröffentlichung mit anderem Jahr gefunden | 🎓 FU mit Hinweis „Hinweis auf Fehler: Originaltext erschienen [Jahr]"          |
| NICHT in einer Datenbank, keine verlegte Ausgabe, keine sonstige Evidenz (alle Suchen inkl. Gegenprobe versucht)      | 🎓 FU — Suche versucht, nicht maschinell verifizierbar                          |
| Studienbrief-Kapitel ist ein Nachdruck und das Jahr passt zur gefundenen Originalveröffentlichung                     | 🟢 I oder 🟡 II                                                                 |

---

## Graue Literatur (Webseiten, Behörden-/Regierungs-/Parlamentsdokumente, Pressemitteilungen, Nachrichten, Koalitionsverträge, Projekt-/Forschungsberichte)

**Grundsatz:** Graue Literatur wird **normal geprüft** und erhält eine Kategorie **I–IV**. Sie wird
**nicht** als 🎓 FU eingeordnet (das ist allein für echtes FernUni-Lernmaterial reserviert) und
**nicht** pauschal als ⚪ ? abgetan, nur weil sie in Katalog-DBs fehlt. Das Fehlen in K10plus/OpenAlex
ist hier erwartbar und **kein** Befund.

### Verifikationsprozedur

1. **Mit zitierter URL:** URL mit `web_fetch` auflösen; prüfen, ob die Seite lädt und herausgebende
   Stelle, Titel/Gegenstand und (wo zitiert) Datum passen. JS-Shell/leere Seite → Claude-in-Chrome.
   Tote, aber über Web-Archiv/Herausgeber auffindbare Seite zählt als Beleg (mit Hinweis).
2. **Ohne URL** (nur Herausgeber + Titel, z. B. Koalitionsvertrag, Behördenbericht): über die Seite der
   herausgebenden Stelle bzw. gezielte Websuche nach Herausgeber + Titel + Jahr bestätigen.
3. **Mit DOI/ISBN/Handle** (Berichte, Working Paper): DOI-Resolve bzw. Repositorium/Katalog (econstor,
   SSRN, Projektseite) — wie bei regulären Werken.

### Klassifikation

| Situation                                                                                                                | Kategorie                                            |
| ------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------- |
| URL lädt, Herausgeber/Titel/Datum passen                                                                                 | 🟢 I                                                 |
| Seite vorhanden, aber Abweichung (falsches Datum, geänderter Titel, abweichender Herausgeber, tote-aber-archivierte URL) | 🟡 II / 🟠 III                                       |
| URL tot **und** auch über Herausgeber/Titelsuche nicht bestätigbar                                                       | 🔴 IV (möglicherweise erfunden / falsche Fundstelle) |
| Keine URL/Fundstelle vorhanden und Existenz strukturell nicht prüfbar                                                    | ⚪ ? (Ausnahmefall)                                   |
