# Konnektoren (MCP) — optional

Stellt die Umgebung Literatur-Konnektoren bereit, nutzt der Skill sie als Vorweg. Ohne Konnektor läuft
er unverändert.

## 1. Erkennung (Step 0c, ohne Rückfrage)

Verfügbare Werkzeuge prüfen, **auch verzögert geladene**: Gibt es eine Werkzeugsuche (z. B.
`ToolSearch`), dort nach `openalex` und `pubmed` suchen und Treffer laden.

| Konnektor | Erkennungsmuster (Werkzeugname) |
|---|---|
| OpenAlex | `…OpenAlex…resolve_references`, `…get_work`, `…search_works` |
| PubMed | `…PubMed…lookup_article_by_citation`, `…get_article_metadata` |

Kein Treffer → Konnektor-Schicht entfällt. Transparenz-Log: „Konnektoren: keine erkannt“.

## 2. OpenAlex-Konnektor

- `resolve_references`: bis zu **25 Quellen pro Aufruf** (Freitextzitat im Originalwortlaut, DOI oder
  PMID). Liefert je Quelle `match` (`exact`/`likely`/`uncertain`/`none`), `year_consistent`,
  `author_consistent`, `type` und den Datensatz mit W-ID und ggf. DOI.
- `get_work`: Vollrecord per W-ID, DOI, PMID oder arXiv-ID.
- Kanonische URL: `https://openalex.org/W…` bzw. `https://doi.org/…`.

### Annahme-Gate (HART)

Ein Treffer bestätigt eine Quelle nur, wenn **alle** Bedingungen gelten:

- `match` = `exact` und `author_consistent` = true,
- Titel des Treffers entspricht dem zitierten Haupttitel,
- `type` passt (`article` ↔ Artikel, `book` ↔ Buch, `book-chapter` ↔ Kapitel). **Nie** akzeptieren:
  `book-review`, `reference-entry`, `paratext`, oder `book-chapter` für ein zitiertes Buch,
- das Jahr wurde **selbst** verglichen. `year_consistent` toleriert ±1 und reicht nicht.

Danach:

- **Artikel:** Band, Heft und Seiten, falls zitiert, über die DOI per Crossref prüfen. Seitenangaben
  aus OpenAlex sind unzuverlässig.
- **Bücher:** Weicht das Jahr ab, zuerst die zitierte Ausgabe in K10plus suchen (`pica.jah=Zitatjahr`),
  bevor 🟡 II/🟠 III vergeben wird. OpenAlex ordnet Bücher oft einer anderen Ausgabe zu.

`likely`, `uncertain`, Typ-Konflikt oder `none` → normale Kaskade. Häufige Fehlzuordnungen:
Neuauflage, Nachdruck im Reader, Rezension, zitierendes Werk. `none` ist ein erster Hinweis, allein
aber nicht ausreichend, ältere deutsche Bücher und Herausgeberbände fehlen in OpenAlex teilweise.

## 3. PubMed-Konnektor (nur medizin / life sciences)

- `lookup_article_by_citation` (Zeitschrift, Jahr, Band, erste Seite, Erstautor) → PMID.
  `get_article_metadata` liefert DOI und Metadaten.
- Kanonische URL: `https://pubmed.ncbi.nlm.nih.gov/PMID/`.
- Ein Treffer bestätigt. Ein `null` kommt schon bei falschem Jahr und reicht allein nicht für 🔴 IV. Danach
  normale Artikel-Kaskade.
