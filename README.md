# competitor-recon

Skill do **Claude Code**, który z eksportu widoczności konkurenta (Senuto / Ahrefs / SEMrush)
typuje, **które strony konkurenta warto odtworzyć, by przejąć ich ruch** — z priorytetem
(P1/P2/P3), głównym słowem kluczowym, typem strony i odsianiem fraz brandowych oraz zbyt
generycznych/ambiwalentnych.

## Po co

Surowy wolumen myli. Najwyższe wolumeny to zwykle frazy, na które **nie ma sensu** rankować:
brandowe (`kaman`), generyczne/ambiwalentne (`landing`, `biogram`, `dropship`), obcojęzyczne.
A największą wartość mają często strony, które konkurent ledwo zajmuje, choć jest popyt
(**open goals**). Ten skill rozdziela jedno od drugiego.

**Domyślnie domenowo-agnostyczny:** nie zakłada żadnej „twojej" strony ani branży — ocenia każdą
stronę na gruncie samego konkurenta (czy to realna money/autorytetowa strona, czy brand/śmieć/
przynęta na ruch) i wypluwa listę stron do odtworzenia z najważniejszymi frazami. Filtr pod
własną niszę włączasz **opcjonalnie** flagą `--our-topics`.

## Architektura (świadomy podział pracy)

```
eksport.xlsx ──ingest──► reguły (brand/generyk/komercja/lokal) + agregacja keyword→URL
                         + latent_potential (open goals) + winsoryzacja CPC
                         ▼
                    worksheet.md (3 pule: ruch + CPC + open-goals)
                         ▼
                 [Claude: osąd semantyczny → judgments.json]
                         ▼
                    score ──► tiery WZGLĘDNE P1/P2/P3 + SKIP + TAIL
                              report.md + priorities.csv
```

- **Skrypt (`recon_tools.py`)** robi mechanikę: parsowanie, reguły, scoring, tiering — deterministycznie.
- **Claude** robi osąd semantyczny: bo „landing" vs „landing page", a „biogram" (CV) vs „bio na IG"
  rozróżni tylko rozumienie znaczenia, nie reguła.

Composite score = `relevance × (√(ruch+latent) + szerokość fraz + wartość komercyjna) − kara za KD`.
Tiery są **względne** (ranking osądzonych stron) — działają dla małej i dużej domeny. Strony
nieosądzone trafiają do **TAIL** (poza rekomendacjami), więc brand/generyk nie zaśmieca planu.

## Instalacja

Skopiuj katalog do skilli Claude Code:

```bash
cp -r competitor-recon ~/.claude/skills/        # globalnie
# albo do projektu:
cp -r competitor-recon <repo>/.claude/skills/
```

Wymaga Pythona 3 z `openpyxl` (`pip install openpyxl`) do czytania `.xlsx`. CSV działa bez zależności.

## Użycie

W Claude Code: podaj plik eksportu i poproś o analizę („które strony konkurenta odtworzyć, żeby
przejąć ruch"). Skill uruchomi workflow. Ręcznie:

```bash
# 1. Ingest — normalizacja, reguły, worksheet do osądu
#    (--our-topics OPCJONALNE: filtr pod własną niszę; bez niego ranking wg wartości strony)
python3 recon_tools.py ingest "eksport.xlsx" --domain konkurent.pl --top 40 --out recon_out

# 2. (Claude) czyta recon_out/worksheet.md i pisze recon_out/judgments.json
#    per URL: page_type, main_keyword, relevance (0-1), junk_keywords, content_type, note

# 3. Score — tiery + raport
python3 recon_tools.py score --in recon_out --judgments recon_out/judgments.json --out recon_out
```

Wynik: `report.md` (plan w tierach P1/P2/P3 + SKIP + 🎯 open goals + długi ogon) i `priorities.csv`.

## Obsługiwane formaty wejścia

Senuto (PL), Ahrefs / SEMrush (EN). Auto-mapowanie nagłówków kolumn (keyword, volume, position,
traffic, cpc, kd, url) i separatora CSV. Rozpoznaje też typy stron ecommerce z kodów-sufiksów
URL (`-cNNN` kategoria, `-pNNN` produkt, `-rNNN` region) oraz ścieżek (`/kategoria/`, `/produkt/`).

## Zwalidowane na

3 modelach biznesowych: agencja marketingowa, usługi/eventy (team building), ecommerce (armatura).
Szczegóły kalibracji i historia zmian w [`CHANGELOG.md`](CHANGELOG.md).

---

Zbudowany pierwotnie dla [Double Digital](https://double-digital.pl), ale **domyślnie działa na
dowolnym eksporcie bez żadnej konfiguracji** — pod własną niszę dorzuć `--our-topics`.
