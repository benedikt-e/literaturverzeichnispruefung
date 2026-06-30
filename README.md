# Literaturverzeichnisprüfung — Skill

Ein KI-Skill zur systematischen Prüfung akademischer Literaturverzeichnisse auf Fehler und halluzinierte Quellen. Jede Verifikation erfolgt durch Live-Abfragen bei bibliografischen Datenbanken — nicht aus dem Gedächtnis der KI.

---

## Entstehung

Der Skill ist aus einem Experiment mit KI-generierten Hausarbeiten entstanden. Die Überprüfung der
Literaturverzeichnisse solcher Arbeiten ist von Hand aufwendig; daraus ergab sich die Frage, ob sich
ausgerechnet mit KI eine Entlastung für diesen Arbeitsschritt erreichen lässt. Der Skill ist dabei bewusst so ausgelegt, dass er möglichst universell mit verschiedenen
Modellen, Setups und Agentensystemen funktioniert — was zugleich einige Limitationen mit sich bringt (siehe weiter unten).

Entwickelt habe ich den Skill in erster Linie für meinen eigenen Bedarf; daher die Ausrichtung auf die Sozialwissenschaften und den Kontext der FernUniversität in Hagen. Vielleicht kann er aber auch für andere nützlich sein, die mit Literaturverzeichnissen zu tun haben, in denen sich möglicherweise halluzinierte Quellen verbergen.

Bei der Auswahl der bibliografischen Datenbanken konnte ich von der Expertise von **Achim Baecker** (Fachreferent an der Universitätsbibliothek der FernUniversität in Hagen) profitieren und habe durch ihn große Unterstützung bei der Entwicklung dieses Skills, insbesondere bei der Recherche zu den verwendeten Schnittstellen (DNB-SRU, ZDB-SRU, K10plus, Crossref u. a.) erhalten.

Der Skill selbst wurde mit Unterstützung von KI (Claude Cowork) erstellt — von der Konzeption über die Formulierung der Regeln bis zur Auswahl und Anbindung der Schnittstellen.

---

## Limitationen im Überblick

Der Skill ist **Work in Progress**! Die wichtigsten Einschränkungen vorab; die technischen Details
stehen weiter unten unter [Bekannte Limitationen](#bekannte-limitationen).

- **Umgebungsabhängiger Datenbankzugang:** Ein Teil der Datenbanken drosselt oder blockiert Anfragen anhand der IP-Adresse. In geteilten Agentenumgebungen sind einzelne Datenbanken deshalb gar nicht oder nur eingeschränkt erreichbar, während dieselbe Schnittstellenabfrage vom eigenen Rechner funktioniert.
- **Fachlicher Schwerpunkt Sozialwissenschaften:** Die Implementierung und die bisher eingebundenen Datenbanken sind vor allem auf die Sozialwissenschaften ausgerichtet.
- **Nur offen zugängliche Datenbanken:** Der Skill setzt bewusst ausschließlich auf frei zugängliche Datenbanken.
- **Risiko von Halluzination in der Prüfung selbst:** Der Skill verifiziert per Live-Abfrage und nicht aus dem Trainingswissen des Modells. Es können jedoch bei der Klassifizierung Fehler auftreten und doch auf das Trainingsmaterial zurückgegriffen werden — also genau die Halluzination reproduzieren, die der Skill verhindern soll. 
- **Hoher Tokenverbrauch:** Die Prüfung erzeugt und wertet sehr viele Suchanfragen aus und verbraucht entsprechend viele Tokens.
- **Indikator, kein Urteil:** Die Einordnung durch den Skill ist ein Indikator, der einer eigenen Überprüfung unterzogen werden muss — keine abschließende Bewertung. Bei umfangreichen Literaturverzeichnissen kann der Skill den Prüfaufwand trotz dieser Einschränkungen erheblich verringern.

---

## Was der Skill macht

Der Skill nimmt ein Literaturverzeichnis entgegen (als Datei-Upload oder eingetippten Text) und prüft jede Quelle einzeln gegen öffentliche Bibliotheksdatenbanken. Das Ergebnis besteht aus zwei Teilen. Erstens eine Bewertungstabelle mit den Spalten:

- **Bewertung** — Kategorie I–IV (bzw. 🎓 FU / ⚪ ?)
- **Originalquelle** — die Angabe exakt wie eingegeben
- **Gefundene Quelle** — die korrigierten Metadaten laut Datenbank (Korrekturen fett hervorgehoben), hinterlegt mit dem wichtigsten kanonischen Datenbank-Link (Proof of Work)
- **Art der Abweichungen** — konkrete Fehlerbeschreibung

Zweitens ein Block „Belege & Nachschlagen", der pro Quelle alle tatsächlich abgefragten Datenbanken samt Fundstellen-Links sowie vorbereitete manuelle Suchlinks (Google Scholar, WorldCat) auflistet.

Zusätzlich wird eine `.md`-Datei mit dem vollständigen Prüfbericht gespeichert, die Dateiname und Prüfdatum als Nachweis enthält.

---

## Kategorien

| Symbol | Kategorie                                | Bedeutung                                                                                                                                                                                                                      |
| ------ | ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 🟢 I   | Fehlerfrei                               | Quelle existiert wie zitiert. Nur triviale Formatunterschiede.                                                                                                                                                                 |
| 🟡 II  | Kleine Fehler                            | Quelle existiert. Kleinere Abweichungen: fehlender Untertitel, Auflage, oder Sammelband-Regel (Kapitel in Herausgeberband nicht einzeln indexiert).                                                                            |
| 🟠 III | Gravierende Fehler                       | Quelle existiert, aber wesentliche Angaben stimmen nicht: falsches Jahr, falscher Verlag, völlig falsche Seitenzahlen.                                                                                                         |
| 🔴 IV  | Möglicherweise halluziniert              | Trotz Gegenprobe keine Quelle auffindbar, die Autor(en) + Titel + Publikationstyp vereint. Warnung, keine Gewissheit. Nur bei vorausgegangenem positivem DB-Treffer ausgeschlossen (Klassifikations-Gate).                     |
| 🎓 FU  | FernUni-Lernmaterial (nicht überprüfbar) | **Nur** echtes FernUni-Studienbrief-/Lerneinheits-Material, bei dem auch die Suche nach einer verlegten Buchausgabe und die Gegenprobe erfolglos blieben. Ausdrücklich **keine** Halluzination. **Nicht** für graue Literatur. |
| ⚪ ?    | Nicht prüfbar                            | Sonderfall: Quellentyp ohne URL/Fundstelle, dessen Existenz strukturell nicht prüfbar ist (z. B. internes Dokument). Nicht für graue Literatur mit URL und nicht für FernUni-Material.                                         |

**Graue Literatur** (Webseiten, Behörden-/Regierungs-/Parlamentsdokumente, Pressemitteilungen, Nachrichten, Koalitionsverträge, Projektberichte) wird **normal geprüft** und erhält eine Kategorie **I–IV**: Die zitierte URL wird aufgelöst und Herausgeber/Titel/Datum abgeglichen; sie wird nicht als 🎓 FU oder ⚪ ? abgetan.

---

## Verwendete Datenbanken

| Datenbank                     | Einsatz                                                                 | Zugang                                                                                               |
| ----------------------------- | ----------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| **OpenAlex**                  | Zeitschriften, Monografien, Preprints                                   | **API-Key empfohlen** (`--openalex-key`; ohne Key in vielen Agent-Umgebungen faktisch nicht nutzbar) |
| **K10plus (SRU)**             | Dt. Verbundkatalog (GBV/SWB) — Bücher, breiter als DNB                  | Offen, kein Schlüssel nötig                                                                          |
| **lobid (hbz)**               | Dt. Verbundkatalog NRW; Bücher + GND-Autorabgleich                      | Offen, kein Schlüssel nötig                                                                          |
| **Crossref / DOI.org**        | DOI-Verifikation, Zeitschriften                                         | Offen, kein Schlüssel nötig                                                                          |
| **DNB SRU**                   | Deutschsprachige Bücher und Sammelbände                                 | Offen, kein Schlüssel nötig                                                                          |
| **Semantic Scholar**          | Englischsprachige Wissenschaft, KI/ML                                   | Offen, kein Schlüssel nötig                                                                          |
| **Open Library**              | Internationale/englischsprachige Bücher, ISBN                           | Offen, kein Schlüssel nötig                                                                          |
| **arXiv**                     | Preprints (Physik, Informatik, Mathe, Statistik, econ)                  | Offen, kein Schlüssel nötig                                                                          |
| **Library of Congress (SRU)** | US-/englischsprachige Monografien (nachrangig)                          | Offen, kein Schlüssel nötig                                                                          |
| **ZBW EconBiz**               | Wirtschaftswissenschaften                                               | Offen, kein Schlüssel nötig                                                                          |
| **GESIS Search**              | Deutschsprachige Sozialwissenschaften                                   | Websuche (kein direktes API)                                                                         |
| **BASE**                      | Graue Literatur, Hochschulschriften, Working Paper, Open-Access-Inhalte | **Nur mit API-Key** (`--base-key`); wird nur dann live abgefragt                                     |

**Vorbereitete Such-Links (keine Live-Abfrage, nur zum Anklicken):** Google Scholar, WorldCat, BASE
(Web-Oberfläche). Der BASE-Web-Link bleibt zusätzlich verfügbar, wenn kein Key vorliegt.

Welche Datenbank für welchen Quellentyp und welches Fachgebiet in welcher Reihenfolge abgefragt
wird, steht in [`references/database-routing.md`](references/database-routing.md) und ist dort
leicht editierbar.

---

## Möglicherweise kompatible Systeme

Erprobt und im Einsatz ist der Skill bislang nur unter **Hermes** und **Claude Cowork**. Da er dem
offenen [agentskills.io](https://agentskills.io/specification)-Standard (SKILL.md-Format) folgt, könnte
er grundsätzlich auch in anderen Systemen laufen, die diesen Standard unterstützen. Geprüft habe ich das
**nicht**.

---

## Verwendung

### Aufruf durch natürliche Sprache

Der Skill wird automatisch aktiviert, wenn die Anfrage auf eine Literaturprüfung hindeutet:

> *„Prüf mein Literaturverzeichnis auf Halluzinationen."*  
> *„Check my references for citation errors."*  
> *„Gibt es diese Quelle wirklich?"*

### Slash-Command

```
/literaturverzeichnispruefung
```

### Input-Formate

- **Datei hochladen**: `.pdf`, `.docx`, `.txt`, `.md` — der Skill extrahiert das Literaturverzeichnis automatisch
- **Text einfügen**: Literaturverzeichnis direkt in den Chat kopieren

### Optionen / Flags

Alle optional; ohne Angabe gilt das Standardverhalten.

| Flag                   | Wirkung                                                                                                                                                                                                                                                                               |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--fachgebiet <name>`  | Fachgebiet manuell setzen (`sozialwissenschaften`, `wirtschaft`, `recht`, `stem`, `medizin`, `geistes`, `allgemein`) statt automatischer Erkennung.                                                                                                                                   |
| `--kontext fernuni`    | FernUni-Handling explizit erzwingen (Override). Wird ohnehin automatisch ausgelöst, sobald ein FernUni-Signal erkannt wird.                                                                                                                                                           |
| `--bibtex`             | Das intern erzeugte BibTeX zusätzlich als `.bib`-Datei speichern.                                                                                                                                                                                                                     |
| `--openalex-key <key>` | OpenAlex-API-Key bereitstellen (in vielen Agent-Umgebungen faktisch nötig). Alternativ über die Step-0-Rückfrage angebbar. Nur sitzungsweise, wird nie gespeichert.                                                                                                                   |
| `--s2-key <key>`       | Semantic-Scholar-API-Key (optional; hebt das strenge Rate-Limit an). Nur sitzungsweise, wird nie gespeichert.                                                                                                                                                                         |
| `--base-key <key>`     | BASE-API-Key (optional). Nur damit wird BASE überhaupt abgefragt — für graue Literatur, Hochschulschriften, Working Paper, OA-Inhalte. Hartes Limit 1 Anfrage/Sekunde → höchstens ein BASE-Aufruf pro Schritt, strikt seriell, nie parallel. Nur sitzungsweise, wird nie gespeichert. |
| `--config <pfad>`      | Pfad zu einer Konfigurationsdatei, aus der API-Keys gelesen werden (statt Flags/Step-0-Rückfrage). Siehe Abschnitt „API-Keys übergeben". Der Skill liest die Datei nur, schreibt nie hinein.                                                                                          |

### API-Keys übergeben — drei Wege

API-Keys (OpenAlex, Semantic Scholar, BASE) sind **optional**, verbessern aber Trefferquote und Tempo.
Sie lassen sich auf drei Wegen bereitstellen; der Skill prüft sie **in dieser Reihenfolge**:

1. **Direkt beim Aufruf** — als Flag (`--openalex-key`, `--s2-key`, `--base-key`) **oder** einfach frei
   im Text genannt („mein OpenAlex-Key ist …"). Der Skill ordnet den Key automatisch dem richtigen Dienst zu.

2. **Konfigurationsdatei** — per `--config <pfad>` angegeben **oder** automatisch erkannt, wenn im
   Arbeitsordner (bzw. unter den hochgeladenen Dateien) eine Datei `literaturpruefung-keys.txt`
   (auch `.json`, `.env` oder `.literaturpruefung-keys`) liegt. Beispielinhalt:
   
   ```
   # API-Keys (optional). Diese Datei privat halten — nicht teilen, nicht ins Repository committen.
   openalex = DEIN_OPENALEX_KEY
   semanticscholar = DEIN_S2_KEY
   base = DEIN_BASE_KEY
   ```
   
   Tolerant gelesen: `#`/`;`-Kommentare und Leerzeilen werden ignoriert, Trenner `=` oder `:`,
   Schlüssel ohne Beachtung der Groß-/Kleinschreibung (Aliase wie `openalex_key`, `s2`,
   `semantic_scholar`, `base_key`). Eine `.json`-Datei darf stattdessen ein Objekt
   `{"openalex": "...", "base": "..."}` enthalten.

3. **Rückfrage (Fallback)** — nur wenn weder Aufruf noch Konfigurationsdatei einen Key liefern, fragt
   der Skill **einmalig** vor der ersten Suche nach.

**Wichtig:** Liegt über Weg 1 oder 2 schon ein Key vor, **fragt der Skill nicht noch einmal nach** und
startet sofort. In **allen** Fällen werden Keys nur für die laufende Sitzung im Arbeitsspeicher gehalten;
sie landen **nie** in Berichten, Logs, Zwischenständen, angezeigten URLs oder der Memory. Die
Konfigurationsdatei wird ausschließlich **gelesen** — der Skill schreibt niemals Keys in eine Datei.
Halte die Konfigurationsdatei privat (z. B. via `.gitignore`).

**Fachgebiet:** Standardmäßig erkennt der Skill das Fachgebiet automatisch aus dem Verzeichnis und
nennt es einmal; bei Unklarheit fragt er **einmal** nach (nicht pro Quelle). Mit `--fachgebiet`
kann das übersteuert werden. Das Mapping Fachgebiet → Datenbanken steht in
[`references/database-routing.md`](references/database-routing.md).

**BibTeX:** Vor den Abfragen wird intern ein BibTeX des Verzeichnisses erzeugt — **verbatim, ohne
Korrekturen** —, das die Feldtrennung und das Routing (Eintragstyp → Datenbankkaskade) steuert. Mit
`--bibtex` wird es zusätzlich als Download gespeichert.

### FernUni Hagen Kontext

Das FernUni-Handling wird **automatisch ausgelöst**, sobald eine Quelle ein FernUni-Signal trägt
(Publisher/Ort „Hagen: FernUniversität", Studienbrief-/Lerneinheitsformat). Das Flag ist nur noch ein
expliziter Override, falls die Erkennung ein Signal verfehlt:

```
/literaturverzeichnispruefung --kontext fernuni
```

FernUni-Material wird **nicht** ungeprüft eingestuft. Der Skill **versucht immer
eine Überprüfung** (K10plus/DNB + GESIS + ggf. Originaltext). Wird die
Quelle gefunden, klassifiziert er regulär (🟢/🟡/🟠). Lässt sie sich trotz Suchversuch nicht
nachweisen, erhält sie die Kategorie **🎓 FU** („FernUni-Lernmaterial, nicht überprüfbar") — **nicht**
🔴 IV.

---

## Bekannte Limitationen

**DNB-Drosselung:** Die Deutsche Nationalbibliothek drosselt IPs nach vielen schnellen Abfragen temporär. Betroffen sind die `d-nb.info`-Links im Prüfbericht. Das ist ein bekanntes technisches Problem der DNB (kein defekter Link). Nach einigen Minuten Wartezeit öffnen sich die Links wieder. Deshalb enthält der Bericht für DNB-Treffer immer auch einen OpenAlex-Link als sofort erreichbare Alternative.

**Google Scholar:** Hat kein öffentliches API; Scraping verstößt gegen die Nutzungsbedingungen. Der Skill fragt Scholar daher nicht selbst ab, stellt aber im Bericht pro Quelle einen **fertigen Google-Scholar-Such-Link** bereit, den der/die Nutzende anklicken kann. Für die maschinelle Verifikation dienen OpenAlex und Semantic Scholar als vollwertiger Ersatz.

**WorldCat / JSTOR:** Erfordern kostenpflichtige bzw. institutionsgebundene Zugänge und werden nicht live abgefragt; WorldCat wird — wie Google Scholar — als vorbereiteter Such-Link im Bericht angeboten.

**BASE — nur mit Key, und nicht in jeder Umgebung erreichbar:** BASE (Bielefeld Academic Search Engine) wird **nur abgefragt, wenn ein API-Key vorliegt** (`--base-key`; kostenlose, nicht-kommerzielle Registrierung). Zwei Dinge sind wichtig, damit man sich nicht wundert, wenn BASE „nichts tut":

1. **Zugang je nach Umgebung blockiert.** BASE filtert nach IP und User-Agent. Aus geteilten Agent-/Proxy-Umgebungen (z. B. der `web_fetch`-Sandbox dieses Tools) liefert BASE selbst mit gültigem Key einen **leeren Body** („Access denied for IP address … and user agent …"). Vom eigenen Rechner/Browser (eigene IP, normaler User-Agent) funktioniert derselbe Key problemlos. Der Skill prüft das zu Sitzungsbeginn per **BASE-Canary** und schaltet BASE bei leerem Body still ab — es entstehen also keine falschen Befunde, BASE bleibt dann einfach inaktiv.
2. **BASE ist kein Buch-/Verlagskatalog.** Es aggregiert Open-Access-Repositorien (Hochschulschriften, Working Paper, Preprints, graue Literatur). Klassische Monografien und Verlags-Zeitschriftenartikel fehlen dort oft — ein BASE-Nulltreffer für ein Buch oder einen Artikel bedeutet daher **nichts** und wird vom Skill **nie** als Halluzinations-Hinweis gewertet. BASE ergänzt gezielt den Bereich graue Literatur / Hochschulschriften / OA.

**Striktes Rate-Limit:** BASE erlaubt **maximal 1 Anfrage pro Sekunde**; bei Verstoß droht der Entzug des Keys. Da eine KI keine Sekunden „abwarten" kann, sichert der Skill das über die Handlung ab: **höchstens ein BASE-Aufruf pro Schritt, strikt nacheinander, nie gebündelt oder parallel** (der ~1,5-Sekunden-Abstand ist dabei nur die Begründung des Limits, keine ausführbare Zeitvorgabe).

**FernUni-Studienbriefe:** Sind häufig nicht in der DNB oder anderen Standarddatenbanken indexiert. Der Skill versucht trotzdem eine Überprüfung und klassifiziert sie bei erfolglosem Suchversuch als 🎓 FU (nicht 🔴 IV), wobei der Suchversuch dokumentiert wird. Jahreszahlen von Kapiteln in Studienbriefen können nicht maschinell verifiziert werden — manuelle Prüfung im Studienbrief selbst ist erforderlich.

**Sammelbände / Buchkapitel:** Datenbanken indexieren in der Regel nur den Herausgeberband, nicht die einzelnen Kapitel. Wenn der Herausgeberband verifiziert wird, klassifiziert der Skill das Kapitel als 🟡 II mit entsprechendem Hinweis.

**Modellwahl:** Bei langen Verzeichnissen (20+ Quellen) erhöht ein stärkeres Modell (z. B. Sonnet/Opus) die Trefferquote und die Regeltreue spürbar. Schwächere/schnellere Modelle neigen dazu, Abfragen abzukürzen (z. B. Crossref ohne `select=`) oder bei einem einzelnen API-Fehlschlag vorschnell „technisch nicht prüfbar" zu vergeben, statt die Kaskade und die Gegenprobe vollständig abzuarbeiten. Hinzu kommt: Schwächere Modelle fallen bei der Klassifizierung eher auf ihr Trainingswissen zurück und reproduzieren so die Halluzination, die der Skill gerade ausschließen soll.

**Tokenverbrauch:** Die Prüfung erzeugt pro Quelle mehrere Suchanfragen und wertet die Antworten aus; bei umfangreichen Verzeichnissen summiert sich das zu einem hohen Tokenverbrauch. Das ist der Preis dafür, dass jede Quelle live verifiziert statt aus dem Modellgedächtnis bewertet wird.

---

## Technischer Hintergrund

Der Skill ist ein reiner Prompt-Skill nach dem [agentskills.io](https://agentskills.io/specification)-Standard und benötigt zur Laufzeit nur Web-Zugang (`web_fetch`/WebSearch) — kein eigener Server, keine Installation von Abhängigkeiten. Die meisten Datenbanken sind ohne Schlüssel nutzbar; OpenAlex, Semantic Scholar und BASE lassen sich optional über API-Keys einbinden (siehe „API-Keys übergeben").

---

## Kontakt

Benedikt Engelmeier, FernUniversität in Hagen.
Rückmeldungen, Fehlerberichte und Verbesserungsvorschläge sind willkommen.
